# Source visuals in knowledge bases

DeepTutor can retain extracted source images when a PDF or EPUB parser emits
image files. The KB stores verified parser-extracted PNG, JPEG, GIF, or WebP
bytes with a deterministic asset ID, document hash, parser identity, source
locator, caption and nearby text. Structured parser blocks also preserve a page
index and bounding box
when available. The visual record lives under the KB's `visual_assets/`
directory, separate from the vector index. The access checked API route
`/api/knowledge-bases/{kb_name}/visual-assets/{asset_id}` serves those exact
bytes, and deleting a raw source file removes its visual assets.

The LlamaIndex KB pipeline indexes a text record for each source visual, so a
text embedding model can retrieve it by its caption and context. When `rag`
retrieves that record in the chat loop, a vision capable answer model receives
the verified image pixels in its next request. A text only model receives the
caption and context with an explicit warning that it has not seen the pixels.
At most 64 images per document are retained, each image is limited to 5 MiB,
and at most two retrieved images are sent in one model continuation.

Current extraction coverage depends on the selected parser. MinerU can emit
structured PDF figures. PyMuPDF4LLM can emit PDF and EPUB images when image
extraction is enabled, but its Markdown output does not supply page boxes.
The default text only parser and the current Docling adapter do not emit
source image assets. Images that are only vector drawing commands, or pages
that require OCR when no usable OCR engine is configured, may be absent.
Other RAG providers do not yet index this visual manifest. Interactive
exercises, visual annotations, and mastery updates are separate future work.

## Resuming MinerU cloud PDF slices

Large cloud PDFs are sliced using `engines.mineru.max_pages_per_part` in
Document Parsing settings (default 180; clamped to 1–200). This advanced setting
is preserved when saving the legacy MinerU settings form. Local parsing and
small/non-PDF inputs keep their existing behavior.

Merged content-list `page_idx` values refer to the original PDF, including
nested blocks. For example, page index 0 of the second 180-page slice becomes
180. Invalid local indices fail the parse instead of silently mislabeling
source pages. Other per-part diagnostic artifacts retain their original local
numbering and `partNN_` filename prefix.

Completed slice archives are saved under the active workspace's parse cache in
`.mineru-segments/`, outside disposable failed-parse directories. Retrying the
same source and parser settings reuses completed slices; a changed document,
endpoint, model, language, OCR/formula/table setting, or slice size starts a new
checkpoint set. API tokens and signed URLs are never written to checkpoints.
Each archive is SHA-256 checked before reuse; an incomplete or corrupt checkpoint
is downloaded again. Concurrent jobs may still duplicate a cloud request, but
checkpoint writes are atomic. Cache write failures do not fail a successful parse.

These archives contain parsed document content, images, and any source copies
returned by MinerU, just like the normal parse cache. They are retained until the workspace cache is cleared; deleting
`.mineru-segments/` while no parse is running only discards resumable progress.
This change versions the cloud parser signature so older merged page indices
are not reused from the normal parse cache. Existing knowledge-base indexes
need an explicit rebuild to consume corrected pages.

## Tiny scanned PDF pages with MinerU

Some scanned PDFs encode a full-resolution page in an unusually small physical
page box. MinerU's official cloud backend can optionally enlarge these pages in
a temporary upload copy. The original file, page order, compressed image bytes,
and soft masks are preserved; language, OCR, model, formula, and table settings
remain as configured. This is a geometry workaround, not a guarantee of better
OCR or preservation of arbitrary interactive PDF semantics.

The option defaults to off. An administrator can enable it through the existing
`PUT /api/settings/document-parsing` endpoint with this partial payload:

```json
{"engines": {"mineru": {"normalize_tiny_scans": true}}}
```

Use `false` to disable it. The option also lives at
`engines.mineru.normalize_tiny_scans` in `document_parsing.json`. Enabling it
changes the cloud parse-cache signature; old parses remain intact. Existing
indexes are not rebuilt automatically. The optional
`deeptutor[parse-pymupdf4llm]` extra provides the required PyMuPDF dependency.

Only text-free pages below 144 points on their longest side, with a
high-resolution image covering at least 80% of the page at an apparent density
of at least 1200 DPI, qualify. The longest side is scaled to 768 points.
Rotated, cropped, annotated, vector-bearing, or non-default UserUnit pages are
left unchanged. Local MinerU, custom cloud endpoints, normal PDFs, and non-PDF
inputs keep their existing behavior. Temporary copies are removed after success
or failure, and upload is refused if compressed image streams change.

## Optional image-description batches

The existing LlamaIndex image-description pass can send multiple images per
vision request. Set `image_description_batch_size` through
`PUT /api/knowledge-bases/rag-pipelines/llamaindex/config`, for example:

```json
{"image_description_batch_size": 4}
```

The default is `1`, preserving individual requests; accepted values are clamped
to 1–8. Concurrency limits count batches when enabled, and the existing timeout
covers the whole batch including any split attempts. Returned captions are
matched by explicit IDs, never by response order. Malformed JSON/ID maps and
explicit context overflow split into smaller groups, eventually using the
existing single-image prompt. A group of N images makes at most 2N−1 completion
calls, with provider retries disabled for this mode. Authentication and rate
limits stop queued groups in the job; other API/transport errors do not split.

This only affects subsequently processed images in the existing LlamaIndex
description pass. It does not enable descriptions for structured source visuals
or alter reading-material captions. Batches use a different structured prompt
and cache complete, successful groups separately from independent single-image
captions. The digest includes ordered image contents and metadata, prompts,
model identity, and retry policy. Failed, incomplete, or canceled groups are
not cached; successful split groups can be reused. Setting the size
back to `1` restores the normal single-image path. The model must support
multiple image blocks; unsupported API responses are reported without a burst
of fallback requests.
