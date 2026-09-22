# GenReport — AI-Assisted Report Generation System

A full-stack web application (React + Node/Express + MongoDB) that turns keywords and
instructions into a complete, structured, editable report — with AI content generation,
an image manager with AI captioning, a template system (predefined or your own uploaded
DOCX/PDF), a real document-style preview, and PDF/DOCX export.

This is **not** a chatbot. AI is invoked through buttons, forms and contextual actions
inside a document editor (generate, regenerate, improve, expand, shorten, simplify,
convert paragraph ⇄ bullets ⇄ numbered list, generate image captions, etc.).

---

## 1. Project structure

```
project-root/
  frontend/     React + Vite + Tailwind CSS + React Router
  backend/      Node.js + Express (routes / controllers / services / models / middleware)
  README.md
```

Backend architecture:

```
backend/
  config/       env.js, db.js (MongoDB connection + local fallback store)
  models/       Report.js, Template.js (Mongoose schemas + Repository)
  controllers/  reportController, aiController, imageController, templateController,
                exportController, systemController
  routes/       reportRoutes, aiRoutes, templateRoutes, systemRoutes
  services/     aiService.js (real AI), demoAIService.js (Demo Mode), aiProvider.js
                (picks one), reportService.js, templateService.js (DOCX/PDF analysis),
                exportService.js (PDF via Puppeteer, DOCX via `docx`), prompts.js
  middleware/   upload.js (Multer), errorHandler.js
  utils/        Repository.js, memoryStore.js, figureNumbering.js, jsonExtract.js,
                logger.js, seedData.js
```

## 2. Prerequisites

- Node.js 18+ and npm
- **`poppler-utils`** installed as a system package (provides the `pdftoppm`/`pdftotext`/`pdfinfo` command-line tools) — required for analyzing PDF templates at all, and specifically for extracting a PDF template's header/logo banner so exported reports can reuse it. On Debian/Ubuntu: `sudo apt-get install poppler-utils`; on macOS: `brew install poppler`. If this isn't installed, PDF template uploads still work but silently produce no logo (a warning is logged server-side as `[TEMPLATE] pdftoppm rasterization failed...` — if your exported/previewed reports are missing a logo from a PDF template, check for that log line first).
- (Optional) A MongoDB Atlas free-tier cluster — or run without one; see below
- (Optional) An AI API key (Anthropic Claude) — or run without one in **AI Demo Mode**

## 3. Installation

```bash
# Backend
cd backend
npm install
cp .env.example .env      # then edit .env as needed (see below)
npm run seed              # optional: creates the sample "AI-Assisted Event Management" report
npm run dev                # starts the API on http://localhost:5000

# Frontend (in a second terminal)
cd frontend
npm install
cp .env.example .env      # optional, defaults to the Vite dev proxy
npm run dev                # starts the app on http://localhost:5173
```

Open http://localhost:5173 — the landing page loads with the dark theme by default.

> Puppeteer (used for PDF export) downloads its own bundled Chromium during
> `npm install`. This requires normal internet access; if your network blocks
> that download, set `PUPPETEER_EXECUTABLE_PATH` in `backend/.env` to point at
> an existing Chrome/Chromium binary on your machine instead.

## 4. MongoDB setup (optional but recommended)

1. Create a free cluster at https://www.mongodb.com/atlas
2. Get its connection string and put it in `backend/.env` as `MONGODB_URI`
3. Restart the backend

**If you skip this step**, the backend automatically falls back to a local,
JSON-file-backed in-memory data store (`backend/data/*.json`) so the entire
app — creating, editing, saving and reopening reports — still works exactly
the same way for local development and demos. This fallback is logged on
startup and shown in Settings → AI Settings → Database.

## 5. AI setup (optional — Demo Mode works out of the box)

Set in `backend/.env`:

```
AI_PROVIDER=anthropic
AI_API_KEY=sk-ant-...
```

If `AI_API_KEY` is left empty, the backend automatically runs in **AI Demo
Mode** (`services/demoAIService.js`): all AI actions still work end-to-end
(report generation, rewriting, expanding, image captioning, etc.), but using
templated/rule-based content generation instead of a live model call, so the
whole application can be demonstrated and graded without an API key. The UI
clearly labels this ("AI Demo Mode" banner + badge in Settings). Demo Mode and
live AI mode share the exact same function signatures
(`services/aiService.js` vs `services/demoAIService.js`), selected by
`services/aiProvider.js`.

## 6. Using the app

1. **Dashboard** → "Create New Report"
2. **Details**: title, report type, keywords (as tags), additional info, length, writing style
3. AI generates a structured, section-by-section report (Content step)
4. Edit paragraphs/bullets/numbered lists; use the per-block AI toolbar
   (Regenerate / Improve / Expand / Shorten / Simplify / Make Formal / Convert
   to Bullets / Convert to Paragraph / Custom Instruction)
5. **Images** step: upload JPG/PNG/WEBP images into any section, generate an
   AI caption, edit it freely, control alignment/size — figure numbers renumber automatically
6. **Format** step: choose a predefined template (Academic / Technical /
   Simple / Professional) or upload your own DOCX/PDF to use as the
   formatting source; choose PDF / DOCX / Both
7. **Preview** step: a real paginated, document-style preview (title page,
   headers/footers, page numbers, zoom, page navigation, thumbnails)
8. **Export** step: Download PDF / DOCX / Both — generated on the backend
   from the exact same structured report data used by the preview

A ready-made sample report ("AI-Assisted Event Management") is created by
`npm run seed` so the app can be explored immediately.

## 7. Template system — how it actually works (and its honest limits)

**Uploading a DOCX** is genuinely structure-preserving, not "extract text then
regenerate from scratch." `services/templateService.js` reads the OOXML
directly to extract real page size, margins, default font/size, headings
(Word "Heading" styles), tables, images, and header/footer/page-number
presence. Then, at export time, `services/templateDocxBuilder.js` opens the
**original uploaded .docx file itself** as a zip and edits `word/document.xml`
in place:

- Header/footer files, page numbers, margins and page size are never touched
  at all (they're separate parts of the zip the builder never opens).
- The title page is preserved byte-for-byte except the one "Title"-styled
  paragraph, whose text is swapped for the report's title — logos, subtitles,
  and other front-matter text/positioning are untouched.
- Each of the template's top-level heading paragraphs is mapped, in order, to
  one AI-generated section. The heading keeps the template's own paragraph/run
  styling (font, size, color, spacing); only the text changes.
- Everything between one template heading and the next (the template
  author's placeholder body text) is replaced with the new AI content, but
  the new paragraphs clone that section's own body/bullet paragraph style
  from the template, so the result still looks native to the document.
- New content-block images are embedded as real DOCX media parts with proper
  relationships (not just referenced by path).
- If the report has more sections than the template has headings, the extra
  sections are appended using the last template section's style. If the
  template has more headings than the report has sections, the unmatched
  trailing template sections are dropped rather than left as stale
  placeholder content.

If a template has no detectable Word "Heading" styles to map against, this
falls back to the from-scratch renderer below (still using the template's
extracted formatting profile — font/margins/page size — just not its literal
layout), and logs why.

**Uploading a PDF** is intentionally NOT edited in place — arbitrary PDFs are
not reliably re-editable, so the backend extracts text and heading-like lines
as structural cues and recreates the closest reasonable layout at the
template's detected page size, via the same from-scratch renderer. This is a
deliberate, documented limitation, not a fake "full reproduction," and is
recorded in `extractedStructure.analysisNotes`.

The from-scratch renderer (used for predefined templates, PDF-sourced
templates, or a DOCX template with no headings) lives in
`services/exportService.js`'s `generateDocxFromScratch`.

**Header/college-logo extraction** happens once, at upload/analysis time
(not at export time), and is stored on the template record
(`formatting.logoPath`/`formatting.logoUrl`) — so **a template uploaded
before this logo-extraction feature existed will never show a logo until
it is re-uploaded**, since generating a report from it just reuses the old,
already-analyzed record. For DOCX, two paths are tried, in order: first a
real Word "Header" part (`word/header*.xml`) with an embedded image;
if that's not there, the very common real-world case where the logo is
just a picture pasted at the top of the document body (never using Word's
actual Header feature) is also detected, by looking for an image in the
document's first few paragraphs. For PDF, page 1 is rasterized and its top
strip is cropped as the logo (see the `poppler-utils` prerequisite above —
without it, PDF logo extraction silently no-ops).

## 8. Preview/export parity

The web preview (`frontend/src/components/preview/ReportPreview.jsx` +
`frontend/src/utils/paginate.js`) and the exported PDF
(`backend/services/exportService.js`, real CSS `@page` rules rendered by
Puppeteer/Chromium) both follow the **same page-geometry rules** — page size,
margins, header/footer, page numbers — driven by the same `template.formatting`
object, so what you see in the preview should closely match the exported file.
The web preview estimates block heights (paragraph/bullet/image) to lay out
pages without a full browser layout engine; the PDF export renders real HTML/CSS,
which is the authoritative, pixel-accurate version.

## 9. REST API summary

```
GET    /api/reports                     list (search, reportType, status, template filters)
POST   /api/reports                     create report (Step 1 details)
GET    /api/reports/:id
PUT    /api/reports/:id
DELETE /api/reports/:id
POST   /api/reports/:id/duplicate

POST   /api/reports/:id/sections
PUT    /api/reports/:id/sections/:sectionId
DELETE /api/reports/:id/sections/:sectionId
POST   /api/reports/:id/sections/:sectionId/duplicate
PUT    /api/reports/:id/sections-order

POST   /api/reports/:id/sections/:sectionId/blocks
PUT    /api/reports/:id/sections/:sectionId/blocks/:blockId
DELETE /api/reports/:id/sections/:sectionId/blocks/:blockId
PUT    /api/reports/:id/sections/:sectionId/blocks-order

POST   /api/reports/:id/images                              (multipart: image)
POST   /api/reports/:id/images/:sectionId/:blockId/caption
PUT    /api/reports/:id/images/:sectionId/:blockId
DELETE /api/reports/:id/images/:sectionId/:blockId

POST   /api/reports/:id/apply-template
GET    /api/reports/:id/preview
POST   /api/reports/:id/export/pdf
POST   /api/reports/:id/export/docx

GET    /api/ai/status
POST   /api/ai/generate
POST   /api/ai/generate-section
POST   /api/ai/add-content
POST   /api/ai/rewrite | improve | expand | shorten | simplify | formalize
POST   /api/ai/convert-to-bullets | convert-to-numbered | convert-to-paragraph
POST   /api/ai/custom-instruction

GET    /api/templates
GET    /api/templates/:id
POST   /api/templates/upload                                 (multipart: template)

GET    /api/system/status
```

## 10. Notes on this build environment

This project was authored and fixed in a network-restricted sandbox that
cannot reach npm/package registries, so a real `npm install` / `npm run dev`
full-stack server could not be started there. That does **not** mean the
fixes described in this README were only syntax-checked, though — see
`CHANGES.md` for exactly what was independently verified: the demo AI content
engine was actually invoked and its output inspected, the PDF export pipeline
was actually run end-to-end with a stand-in Chromium and the output opened
and rendered to images, and the new template-preserving DOCX export was
actually run against a realistic test template (title page, header, footer
with page numbers, headings, bullets, a table, an image) and the resulting
.docx opened in LibreOffice and rendered to images to confirm it wasn't
corrupted and looked correct. Dependencies that couldn't be installed
(`jszip`, `puppeteer`→Chromium, `mongoose`, etc.) were stood in for with
minimal local shims for that testing only — the actual shipped code still
imports the real npm packages listed in `package.json`, which `npm install`
provides normally. Still, run the install/dev steps above on a normal machine
with internet access, and re-run the 14-step manual test in `CHANGES.md`,
before considering this fully verified end-to-end in your own environment.
