# PDF metadata sanitization — workflow

Use absolute paths for `INPUT` and scratch files. Replace spaces in paths with careful quoting.

## Variables

```bash
INPUT="/absolute/path/to/document.pdf"
TMP1="/tmp/pdf_sanitize_1.pdf"
TMP2="/tmp/pdf_sanitize_2.pdf"
```

---

## Phase 1 — Baseline (read-only)

**1.1 Document info (Info dictionary / poppler view)**

```bash
pdfinfo "$INPUT"
```

**1.2 ExifTool (includes XMP where present)**

```bash
exiftool "$INPUT"
```

**1.3 Forensic strings (catches leftovers ExifTool “removed” from the catalog only)**

```bash
strings "$INPUT" | grep -iE 'xmp|rdf:|dc:|xmlns|/Metadata|producer|creator|author|uuid:|microsoft|word 20' | sort -u | head -50
```

If this prints **sensitive or old tool lines**, plan for **Phase 3** (pypdf) after qpdf, or **before** final qpdf if the file was never rewritten.

---

## Phase 2 — ExifTool (optional first pass)

Clears many writable tags; on PDFs the edit can be **reversible** and leave orphan bytes — **always run Phase 4 and Phase 1 string checks after**.

```bash
exiftool -all= -overwrite_original "$INPUT"
```

---

## Phase 3 — pypdf page-only rewrite (conditional)

**When:** `strings "$INPUT" | grep -iE 'xmp|rdf:|dc:|uuid:|author|producer'` still shows removed metadata **after** ExifTool, or user requires **no** old XML blobs in the binary.

**How:** use a venv (PEP 668 safe):

```bash
VENV="/tmp/pdfstrip_venv"
python3 -m venv "$VENV"
"$VENV/bin/pip" install -q pypdf
"$VENV/bin/python" - <<'PY'
from pathlib import Path
from pypdf import PdfReader, PdfWriter

src = Path("/absolute/path/to/document.pdf")
out = Path("/tmp/document_pypdf.pdf")
reader = PdfReader(str(src))
w = PdfWriter()
for p in reader.pages:
    w.add_page(p)
with open(out, "wb") as f:
    w.write(f)
print("pages", len(reader.pages))
PY
mv /tmp/document_pypdf.pdf "$INPUT"
```

Then **always** run **Phase 4** on `"$INPUT"` so `/Producer (pypdf)` and structure are normalized (qpdf pass removes that).

---

## Phase 4 — qpdf (required for “clean binary”)

**4.1 Fresh page graph** (drops old xref-linked garbage in practice when combined with later steps):

```bash
qpdf --empty --pages "$INPUT" 1-z -- "$TMP1"
```

**4.2 Drop unreferenced page resources + normalize content streams**

```bash
qpdf --remove-unreferenced-resources=yes --normalize-content=y "$TMP1" "$TMP2"
```

**4.3 Recompress** (often shrinks file after normalize; reduces “huge rewrite” artifact):

```bash
qpdf --recompress-flate "$TMP2" "$INPUT"
rm -f "$TMP1" "$TMP2"
```

**4.4 Structural check**

```bash
qpdf --check "$INPUT"
```

---

## Phase 5 — ExifTool (second pass, optional)

```bash
exiftool -all= -overwrite_original "$INPUT"
```

---

## Phase 6 — Verification checklist

| Step | Command / check | Pass criteria |
|------|-----------------|---------------|
| A | `pdfinfo "$INPUT"` | No Author/Creator/Producer/Title/Subject/CreationDate/ModDate; **Metadata Stream: no** (unless user intentionally keeps a stream — default is no) |
| B | `exiftool -a -G0:1 "$INPUT"` | No `[XMP]` / XML document tags; `[PDF]` only structural (version, page count, linearized) |
| C | `grep -abo $'%%EOF' "$INPUT" \| wc -l` | **1** (no incremental update chain) |
| D | `strings "$INPUT" \| grep 'trailer <<'` | Trailer has **`/Root`**, **`/Size`**, **`/ID`** — ideally **no `/Info`**, **no `/Metadata`** |
| E | `strings` + grep for old names, `uuid:`, `xmp`, `rdf:`, `Microsoft`, `Word 20`, `exiftool`, `pypdf` | **No matches** (allow false positives only if regex is sloppy — tighten patterns if needed) |
| F | `grep -aob '/Metadata' "$INPUT" \| wc -l` | **0** |

**Note:** Trailer **`/ID`** is normal for PDFs and is **not** legacy author metadata.

---

## Phase 7 — macOS extended attributes (optional)

```bash
xattr -c "$INPUT"
xattr -l "$INPUT"
```

`com.apple.macl` may **reappear or resist deletion**; it is **OS provenance**, not PDF author metadata. For maximum separation from local history, copy the finished file to a **new name** and share from that copy.

---

## Sharing and messaging apps

### WhatsApp (document)

- WhatsApp sends PDFs as **files**; it does **not** typically re-export the PDF like compressed photos/videos.
- **Embedded PDF metadata** (Info/XMP/producer) **normally unchanged** in the received file.
- **WhatsApp** still records **message metadata** (sender, time, chat) on their infrastructure — that is **not inside the PDF**.

### When metadata on the recipient’s copy *can* change

- The recipient opens the file in **Word / Pages / Acrobat** and **Save** or **Export** again.
- Some **enterprise DLP / mail gateways** rewrite attachments (uncommon for personal WhatsApp).

---

## Troubleshooting

| Symptom | Action |
|--------|--------|
| ExifTool warns edits are **reversible** | Run **Phase 4** (qpdf) at minimum; re-check `strings`. |
| `pip install pypdf` fails (externally managed env) | Use **venv** (Phase 3). |
| File balloons in size after `--normalize-content` | Run **`qpdf --recompress-flate`** (Phase 4.3). |
| `pdfinfo` still shows Producer after pypdf | Run qpdf **empty+pages** pipeline (Phase 4). |

---

## One-shot shell sequence (after setting `INPUT`)

Adjust if you skip pypdf (omit Phase 3 block manually).

```bash
set -euo pipefail
INPUT="/absolute/path/to/document.pdf"
TMP1="/tmp/pdf_sanitize_1.pdf"
TMP2="/tmp/pdf_sanitize_2.pdf"

command -v qpdf >/dev/null
command -v exiftool >/dev/null || true
command -v pdfinfo >/dev/null || true

exiftool -all= -overwrite_original "$INPUT" 2>/dev/null || true

qpdf --empty --pages "$INPUT" 1-z -- "$TMP1"
qpdf --remove-unreferenced-resources=yes --normalize-content=y "$TMP1" "$TMP2"
qpdf --recompress-flate "$TMP2" "$INPUT"
rm -f "$TMP1" "$TMP2"

exiftool -all= -overwrite_original "$INPUT" 2>/dev/null || true
qpdf --check "$INPUT"
pdfinfo "$INPUT"
```

Then run **Phase 6** string/trailer checks manually.
