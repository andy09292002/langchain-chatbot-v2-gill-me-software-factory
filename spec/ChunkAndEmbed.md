# Chunking and Embedding: PDF, DOCX and TXT

How an uploaded file becomes searchable vectors. This covers parsing, cleaning, chunking, embedding and indexing for each file type.

> **Status (2026-10-15):** PDF, DOCX and TXT are implemented end to end, both from the CLI and through the upload API with background ingestion. Legacy `.doc` (via LibreOffice) is planned for M5.

---

## 1. Pipeline overview

```
inspect_upload()          register_upload()                    process_document()   (background job)
────────────────          ─────────────────                    ──────────────────────────────────────────────
clean filename            SHA-256 dedupe                       status=processing, chunks_done=0
ext allow-list       ──►  documents row       ──►  job    ──►  parse   (per-type parser → pages or sections)
content sniff             status=queued            queue       clean   (clean_pages for PDF, normalise for DOCX/TXT)
size limit + hash         store bytes                          chunk   (chunk_pages / chunk_sections → ChunkResult)
                                                               needs_ocr? → status=needs_ocr, stop
                                                               embed   (batches of 64)
                                                               upsert into Chroma + insert into SQLite
                                                               status=indexed   (any exception → status=failed)
```

There are two ways in:

- **API** (`POST /api/documents`): inspects every file first, then registers each one and submits the new ones to the job queue. It returns `202` straight away, and the client polls for progress (§8).
- **CLI** (`cli.py ingest`): `register_file(path)` runs the same `inspect_upload` + `register_upload`, then calls `process_document` directly in the foreground.

Code locations:

| Step | File |
|---|---|
| Upload checks (type, content, size, hash) | `backend/app/ingestion/validation.py` |
| Upload and status endpoints | `backend/app/api/documents.py` |
| Register, process, delete | `backend/app/ingestion/pipeline.py` |
| Restart recovery | `backend/app/db/recovery.py` |
| PDF parser | `backend/app/ingestion/parsers/pdf.py` |
| DOCX pars/cer | `backend/app/ingestion/parsers/docx.py` |
| TXT parser | `backend/app/ingestion/parsers/txt.py` |
| Clean + chunk | `backend/app/ingestion/chunker.py` |
| Embedding model | `backend/app/retrieval/embedder.py` |
| Chroma collection + model guard | `backend/app/retrieval/vectorstore.py` |
| CLI for trying it out | `backend/app/ingestion/cli.py` |

### Shared contract

`PARSERS` in `pipeline.py` maps each extension to a `(parser, chunker)` pair:

| Ext | Parser returns | Chunker |
|---|---|---|
| `.pdf` | `list[str]`, one string per page | `chunk_pages` |
| `.docx` | `list[Section]`, each a `(heading, body text)` pair | `chunk_sections` |
| `.txt` | `[Section(heading=None, text=<whole file>)]` | `chunk_sections` |

`chunk_file(path)` looks up the pair and runs both. The pipeline and the CLI's `chunks` command both use it. Both chunkers return the same `ChunkResult`, so embedding and indexing don't care about the file type.

The key rule is that **a chunk never crosses a unit boundary.** Each page or section is split on its own, so every chunk belongs to exactly one place, and a citation can point to it.

---

## 2. Register (all types)

### 2.1 Upload checks (`inspect_upload` in `validation.py`)

These run on every file before anything is stored. A failure raises `UploadRejected` (a `ValueError`) carrying an HTTP status. The API returns that status, and **one bad file rejects the whole request**, so nothing in it is stored.

1. **Filename.** Client-side folders are dropped (either `/` or `\`), and the name is capped at 255 characters. The result is only used for display.
2. **Extension allow-list.** The extension must be a key in `ALLOWED` (`.pdf`, `.docx`, `.txt`), otherwise **415** `unsupported file type`. `ALLOWED` has to stay in step with `PARSERS` in `pipeline.py`.
3. **Empty file.** **400** `file is empty`.
4. **Content sniff.** `filetype` checks the first 8 KB. The `Content-Type` the browser sends is ignored. A mismatch is **415** `content does not match a .xxx file`, so a renamed `.exe` doesn't get through.
   - `.pdf` must sniff as `application/pdf`.
   - `.docx` may sniff as DOCX or as a plain zip, because `filetype` only recognises DOCX when `word/` appears in the first 8 KB. A zip that isn't really a DOCX is rejected later by `python-docx`, and the document ends up `failed`.
   - `.txt` has no signature, so it must not sniff as any known binary format.
5. **Size limit and hash.** One streaming pass computes the SHA-256 and stops as soon as the file is larger than `MAX_UPLOAD_MB` (default 50), which is **413**.

### 2.2 Register (`register_upload` in `pipeline.py`)

1. **Dedupe.** If a document with the same SHA-256 already exists and is **not** `failed`, it returns `(doc, is_new=False)` and does nothing more. The API returns the existing document. If the earlier attempt failed, the row is reset to `queued` with its error cleared, so it can be retried.
2. **Row.** A new `documents` row is created with `status=queued`. If the same file arrives in two uploads at once, the unique hash makes the second commit fail. That upload then gets the first one's document back as `is_new=False`.
3. **Store.** The bytes are written to storage under the key `{doc_id}{ext}`, for example `data/documents/<uuid>.pdf`.

Only documents with `is_new=True` are handed to `process_document`.

---

## 3. Parsing per file type

### 3.1 PDF (implemented)

`parse_pdf(path) -> list[str]`, using `pypdf.PdfReader`:

- **Encryption.** Many PDFs are encrypted only to restrict printing or editing, and they open with an empty password, so the parser tries `decrypt("")` first. If that fails, it raises `PasswordProtectedPDF("PDF is password-protected")`, and the document ends up `failed` with that message.
- **Text extraction.** `page.extract_text() or ""` is called for each page, in order. A page with no text layer becomes `""`, so page numbering stays correct.
- **Location.** Page number = list index + 1. It is stored as `page` on each chunk.

### 3.2 DOCX (implemented)

`parse_docx(path) -> list[Section]`, using `python-docx`:

- **Walk the body in document order.** `iter_inner_content()` yields paragraphs and tables interleaved. Page headers and footers aren't part of the body, so they never get in.
- **Sections.** A paragraph whose style is `Title` or `Heading …` starts a new section. The body paragraphs and tables after it, up to the next heading, belong to that section. Text before the first heading forms a section with `heading=None`.
- **Body text.** One line per non-empty paragraph. The heading itself is **not** part of the text (see §6). Blank paragraphs are skipped.
- **Tables.** Each row becomes one line with its cells joined by ` | `, for example `Setup | 100`. A merged cell is reported once per grid column it spans, so consecutive duplicate cells are collapsed. Empty cells are dropped.
- **Empty headings dropped.** A heading with no body before the next heading (for example `Chapter 1` directly above `1.1 Scope`) produces no section.
- **No text at all.** For example a document that is only images. The parser raises `ValueError("DOCX contains no text")`, and the document ends up `failed`.
- **Location.** The heading text, with whitespace collapsed, is stored as `section`, for example `"3.2 Termination"`. `page` is `NULL`, because DOCX has no fixed pagination. `documents.page_count` is also `NULL`.
- **Not handled yet:** Word's automatic heading numbering isn't part of the paragraph text, so a heading numbered "3.2" by Word comes through as just `Termination`. Text boxes, footnotes and comments are not extracted.
- **Legacy `.doc`** (M5): convert it with `soffice --headless --convert-to docx`, then handle it as DOCX. This needs LibreOffice in the image.

### 3.3 TXT (implemented)

`parse_txt(path) -> list[Section]`:

- **Decoding** (`decode_text`):
  1. Try `utf-8-sig` first. This covers plain UTF-8 and UTF-8 with a BOM.
  2. If that fails, use `charset-normalizer`. It handles legacy encodings such as cp1252, Big5 and UTF-16.
  3. **cp1252 tie-break.** Western text often decodes equally well as cp1250, cp1252, cp1257 and others, with the same chaos and coherence scores. The first tied candidate would win, so French text came out as cp1250 (`crème` → `crčme`). When cp1252 scores as well as the winner, it is chosen instead. Text that only fits cp1250 (for example Czech) is unaffected.
  4. If nothing decodes cleanly (binary data), the parser raises `ValueError("TXT file is not readable text ...")`.
  - Very short non-UTF-8 snippets can be misdetected, because detection is statistical. Real files are long enough.
- **Line endings.** `\r\n` and `\r` become `\n`.
- **One unit.** The whole file becomes a single `Section` with no heading. System Design §5.1 suggested splitting on blank-line blocks, but because chunks never cross a unit boundary, that would mean one tiny chunk per paragraph. Instead, the splitter's preference for `\n\n` keeps paragraphs together where possible, and short paragraphs are merged into full-size chunks.
- **Empty file.** A file with only whitespace raises `ValueError("TXT file is empty")`.
- **Location.** No `page` or `section`. A chunk is located by its `idx` plus `char_start`/`char_end` within the normalised text.

---

## 4. Cleaning

`normalise(text)` does steps 2 and 3 below, and runs on every page or section. `clean_pages` (PDF only) runs step 1 first and then calls `normalise`. DOCX and TXT skip step 1: the heuristic is meant for paginated layouts and could remove legitimate repeated lines such as a recurring "Notes:" label, and DOCX headers and footers are already excluded by the parser.

1. **Repeated header/footer removal (PDF only).** This only applies when there are 3 or more pages.
   - Candidates are the first 2 and last 2 non-empty lines of each unit (`EDGE_LINES = 2`).
   - Each candidate is normalized to a key: lowercased, whitespace collapsed. Page-number lines like `Page 3 of 10`, `3 / 10` or `- 3 -` have their digits replaced with `#`, so they all become one key. Ordinary body lines that just contain numbers keep their digits.
   - A key that appears on **more than half** of the units counts as a header or footer, and it is removed from the edge positions of every unit.
2. **Hyphenation repair.** `termi-\nnation` becomes `termination`. This only happens between two lowercase letters.
3. **Whitespace normalization.** Runs of spaces, tabs and NBSPs collapse to one space, spaces around newlines are trimmed, and 3 or more newlines collapse to a single blank line.

---

## 5. Chunking (`chunk_pages` / `chunk_sections`)

1. **Scanned-document check (PDF only).** If more than half of the cleaned units have fewer than `MIN_CHARS_PER_PAGE = 30` characters, it returns `needs_ocr=True` with no chunks. The pipeline then sets `status=needs_ocr` and stops before embedding.
2. **Split.** Each non-empty unit goes through its own `RecursiveCharacterTextSplitter`, with these settings:
   - `chunk_size = CHUNK_SIZE` (default **1000** characters)
   - `chunk_overlap = CHUNK_OVERLAP` (default **150**)
   - separators in order of preference: paragraph `\n\n` → line `\n` → sentence `". "` → word `" "` → character
   - `add_start_index=True`, so each chunk knows its offset
3. **Chunk record (`ChunkData`).**
   - `idx`: running index across the whole document (0, 1, 2, …)
   - `text`
   - `page`: the 1-based page number (PDF), or `None` (DOCX, TXT)
   - `section`: the heading (DOCX), or `None` (PDF, TXT, and DOCX text before the first heading)
   - `char_start` / `char_end`: offsets **within the cleaned text of that page or section**. These are used to highlight the quote in the source panel.

Empty pages produce no chunks, but the page numbering stays the same.

To preview chunking without writing anything to the database:

```
python -m app.ingestion.cli chunks FILE.pdf --show 10
python -m app.ingestion.cli chunks FILE.docx --show 10
python -m app.ingestion.cli chunks FILE.txt --show 10
```

On a legacy console code page such as cp950, characters the console can't show print as `?`. The stored text is unaffected.

---

## 6. Embedding

`get_embedder()` returns a `HuggingFaceEmbeddings` (sentence-transformers) model. It runs locally on the CPU, it's free, and it's loaded lazily the first time it's used.

| Setting | Value |
|---|---|
| Model | `EMBEDDING_MODEL`, default `BAAI/bge-small-en-v1.5` |
| Device | `cpu` |
| Normalization | `normalize_embeddings=True` (unit vectors, so cosine = dot product) |
| Doc encode batch | 32 |
| Query prefix | For bge v1.5 English models (not bge-m3), queries get the prefix `"Represent this sentence for searching relevant passages: "`. Documents are embedded **without** it, which is what bge expects. |

The pipeline sends chunks to the embedder in groups of `EMBED_BATCH = 64`.

**Section heading prefix.** For a chunk with a `section`, the embedder gets `"{section}\n{text}"` (`_embed_text` in `pipeline.py`). The stored text stays body-only. This has two effects:

- **Every** chunk of a long section carries its topic in the vector, not just the first chunk.
- The splitter never sees the heading line, so a long first paragraph can't leave a chunk that contains only the heading.

Quotes and `char_start`/`char_end` still refer to verbatim body text.

---

## 7. Indexing

For each batch of 64 chunks:

1. **Chroma upsert** into the collection `CHROMA_COLLECTION` (default `documents`):
   - `id = "{doc_id}:{idx}"`
   - `embedding`, plus `document` = the chunk text
   - `metadata = {doc_id, filename, idx, page?, section?}`. Chroma rejects `None` values, so `page` or `section` is left out when it's empty.
2. **SQLite insert** into the `chunks` table, using the same id, with `document_id, idx, text, page, section, char_start, char_end`. This mirror is used for the source viewer and for BM25 later.
3. **Progress.** `documents.chunks_done += len(batch)`, so the UI can show progress against `chunk_count`.

When every batch is written, the document becomes `status=indexed`.

### Safety rules

- **Re-runnable.** Before inserting, `_remove_chunks(doc_id)` deletes the document's existing vectors (by `where={"doc_id": ...}`) and its SQLite chunks. Re-processing a document replaces its chunks instead of duplicating them.
- **Errors don't propagate.** Any exception is logged, and the document becomes `failed` with the first 500 characters of the error.
- **Embedding-model guard.** The collection is created with `hnsw:space=cosine` and `embedding_model=<name>` in its metadata. `check_embedding_model()` refuses to run if `EMBEDDING_MODEL` doesn't match, because vectors from different models can't be compared. Changing the model means re-indexing.
- **Delete.** `delete_document()` removes the vectors, the SQLite chunks, the stored file and the `documents` row.

---

## 8. Background jobs, progress and restart recovery

- **Job queue.** The API submits `process_document(doc_id)` to `app.state.jobs`, the background job interface (`JOB_BACKEND`). Today that's an in-process queue that runs one job at a time on a background thread. A Redis worker can replace it later without changing the pipeline.
- **Progress.** `GET /api/documents` lists documents, newest first. `GET /api/documents/{id}` returns one document's `status` and progress, and the UI polls it while the document is processing:
  - `status` moves through `queued` → `processing` → `indexed`, or ends at `needs_ocr` or `failed` (with `error` set).
  - `chunk_count` is set once chunking finishes, and `chunks_done` then rises by up to 64 per batch until the two are equal.
- **Restart recovery** (`db/recovery.py`, run at startup in `main.py`). The in-process queue doesn't survive a restart, so:
  1. Documents left in `processing` are reset to `queued`, with `chunks_done=0`.
  2. Every `queued` document, oldest first, is resubmitted to the job queue. That covers both the interrupted ones and any that never started.

  Because processing is re-runnable (§7), a document interrupted halfway finishes without duplicate vectors.
- **Speed.** Embedding on the CPU dominates. A 100-page, text-heavy PDF (986 chunks) took about 3.5 minutes from upload to `indexed` (2026-10-15).

---

## 9. Open points

1. **Many small DOCX sections.** A document with a heading every few lines gives one small chunk per section. This is deliberate, for exact section citations, but if retrieval quality suffers, consider merging consecutive short sections under the same parent heading.
2. **Non-English documents.** The default embedding model, `BAAI/bge-small-en-v1.5`, is English-only. Chinese (or other non-English) text is parsed and stored correctly, but retrieval over it will be poor. For multilingual documents, switch `EMBEDDING_MODEL` to a multilingual model such as `BAAI/bge-m3`. That needs a re-index (§7), and the query prefix is already skipped for m3.
