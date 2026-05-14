---
name: pdf-metadata-sanitization
description: >-
  Strips PDF document metadata (Info, XMP, producer/author/dates) and rewrites
  the file so no forensic strings or incremental-update tails remain. Use when
  the user asks to remove, clear, or sanitize PDF metadata, anonymize a PDF,
  prepare a contract or CV for sharing, or verify a PDF has no XMP/Info after
  editing. Also covers sharing via WhatsApp and what changes (or does not) in
  transit.
---

# PDF metadata sanitization

## Goal

Remove **embedded** document metadata and avoid **leftover bytes** that imply prior author/producer/XMP (ExifTool-only edits on PDFs can be **reversible** and leave strings behind).

## Prerequisites

- **qpdf** (required for rewrite + cleanup): `brew install qpdf`
- **ExifTool** (optional pass on tags): `brew install exiftool`
- **poppler** (verification): `brew install poppler` → `pdfinfo`
- **pypdf** (only if `strings` still shows old XMP/author after ExifTool): use a **venv** (`python3 -m venv .venv && .venv/bin/pip install pypdf`) — avoid `pip install` on PEP 668–managed system Python.

## Agent workflow (execute in order)

Follow the full checklist and commands in **[workflow.md](workflow.md)**.

Summary:

1. **Baseline** — `pdfinfo`, `exiftool`, `strings` + targeted `grep` for names/tool strings.
2. **Optional ExifTool** — `exiftool -all= -overwrite_original file.pdf` (may not remove binary leftovers).
3. **Rewrite pages** — If strings still show XMP/author: **pypdf** copy pages only → replace file, or skip if clean.
4. **qpdf sanitize** — `qpdf --empty --pages in.pdf 1-z -- tmp.pdf` then `qpdf --remove-unreferenced-resources=yes --normalize-content=y tmp.pdf step2.pdf` then replace input (adjust paths).
5. **Recompress** — `qpdf --recompress-flate step2.pdf out.pdf` to avoid an oversized “obvious rewrite” fingerprint when normalize expanded streams.
6. **ExifTool again** — `exiftool -all= -overwrite_original out.pdf` if any tag reappears.
7. **Verify** — `pdfinfo`, `exiftool -a -G0:1`, single `%%EOF`, trailer has **no `/Info`** and **no `/Metadata`**, `strings` greps clean.
8. **macOS (optional)** — `xattr -c file.pdf`; note **`com.apple.macl`** may persist.

## Sharing (e.g. WhatsApp)

Sending the PDF as a **document** does **not** normally re-embed producer/author/XMP; recipients get the same bytes. **WhatsApp still has message metadata** (who/when) on their side. If the recipient **re-saves** the PDF in Word/Acrobat, **that app** may add metadata again. Details in [workflow.md](workflow.md#sharing-and-messaging-apps).

## Do not

- Rely on **ExifTool alone** for PDFs when the user needs **no recoverable traces** — always confirm with `strings` and rewrite with **qpdf** (and pypdf if needed).
- Run destructive `git` commands or store secrets in the skill.

## Progressive disclosure

- Step-by-step commands, verification table, and troubleshooting: [workflow.md](workflow.md)
