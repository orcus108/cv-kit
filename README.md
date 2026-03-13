# cv-kit

> https://github.com/orcus108/cv-kit

A LaTeX-based resume generator. Maintain one master resume as structured blocks, then assemble tailored variants for different job applications — each compiling to a professional PDF via `pdflatex`.

## The problem it solves

Most people keep multiple copies of a LaTeX resume file, one per job application, and manually keep them in sync. cv-kit replaces that with:

- A **master profile** — one source of truth for all your content (projects, experience, education, skills, achievements)
- **Resume variants** — select which blocks to include and in what order, per application
- **Application tracker** — log where you sent each variant and track status (Applied → Interview → Offer / Rejected)

---

## Running locally

**Prerequisites:** Node.js 18+, TeX Live with `pdflatex` on PATH (or set `PDFLATEX_PATH`).

```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). The app redirects to `/editor`.

If `pdflatex` is not on your PATH (e.g. macOS with TeX Live installed to `/Library/TeX/texbin`), create `.env.local`:

```
PDFLATEX_PATH=/Library/TeX/texbin/pdflatex
```

Data is persisted as JSON files in `/data/` (gitignored, auto-created on first run).

---

## Running with Docker

```bash
docker compose up
```

The Docker image bundles Node.js + TeX Live (`texlive-latex-base`, `texlive-fonts-recommended`, `texlive-latex-extra`). The `./data` directory is volume-mounted so data persists between container restarts.

To build and run manually:

```bash
docker build -t cv-kit .
docker run -p 3000:3000 -v $(pwd)/data:/app/data cv-kit
```

Note: the image is ~800MB due to TeX Live.

---

## Project structure

```
/app
  /editor/page.tsx          Master resume editor (tabbed)
  /variants/page.tsx        Variant list — create, duplicate, download
  /variants/[id]/page.tsx   Variant builder — toggle blocks, reorder sections, live preview
  /tracker/page.tsx         Application tracker
  /api/profile/             GET, PUT — master profile
  /api/variants/            GET, POST
  /api/variants/[id]/       GET, PUT, DELETE
  /api/applications/        GET, POST
  /api/applications/[id]/   PUT, DELETE
  /api/generate/            POST — returns PDF binary

/components
  /editor/                  PersonalInfoForm, EducationSection, ProjectsSection,
                            ExperienceSection, SkillsSection, AchievementsSection
  /variants/                VariantCard, BlockToggle, SectionOrderer (dnd-kit)
  NavLinks.tsx              Active-link-aware nav (client component)

/lib
  types.ts                  All TypeScript interfaces
  storage.ts                JSON file read/write (profile, variants, applications)
  latex.ts                  JSON → .tex string generation with LaTeX escaping
  compiler.ts               pdflatex runner, temp dir lifecycle, error handling

/data                       Runtime data (gitignored)
  profile.json
  variants/[id].json
  applications.json

Dockerfile
docker-compose.yml
```

---

## PDF generation pipeline

```
POST /api/generate { variantId | variantData }
  → load profile from /data/profile.json
  → filter blocks by variant inclusion lists
  → generateLatex() → .tex string
  → compileTex() → writes /tmp/cvkit-{uuid}/resume.tex
                 → runs pdflatex twice (second pass for hyperref links)
                 → reads resume.pdf
                 → cleans up temp dir
  → Response: application/pdf binary
```

The endpoint accepts either `variantId` (looks up saved variant) or `variantData` (inline — used by the live preview so the iframe reflects current UI state before autosave).

---

## Data model

```typescript
ResumeProfile     — personal info + education[], projects[], experience[], skills[], achievements[]
ResumeVariant     — name + included_* (arrays of block IDs) + section_order[]
Application       — company, role, variant_id, date_applied, status, notes
```

Variants reference blocks by ID — no data duplication. Changing a block in the master profile is immediately reflected across all variants.

---

## Key implementation notes

- **LaTeX escaping** — all 10 special characters (`\ & % $ # _ { } ~ ^`) plus smart quotes and em/en dashes are escaped before insertion into the template (`lib/latex.ts`)
- **pdflatex runs twice** — required for `hyperref` cross-reference resolution
- **Non-zero exit ≠ failure** — pdflatex exits 1 on warnings; the real check is whether a `.pdf` file was produced
- **`process.cwd()` not `__dirname`** — Next.js resolves `__dirname` to `.next/server/` at runtime; `process.cwd()` reliably points to the project root
- **`export const runtime = "nodejs"`** — required on all API routes that use `child_process` or `fs` to prevent accidental Edge runtime execution

---

## Scripts

```bash
npm run dev          # development server with hot reload
npm run build        # production build
npm run start        # production server
npm run lint         # ESLint
npx tsc --noEmit     # type-check without building
```
