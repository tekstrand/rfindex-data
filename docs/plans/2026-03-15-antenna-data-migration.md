# Antenna Data Migration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Move 25 antennas from inline TypeScript in rfindex to JSON files in the rfindex-data submodule, with a prebuild pipeline that generates TypeScript and copies images at build time.

**Architecture:** Antenna JSON files live in `rfindex-data/data/meshtastic_antennas/`. A one-off extraction script converts the existing TypeScript array to individual JSON files. The existing `lib/load-devices.ts` is renamed to `lib/prebuild.ts` and extended to also generate `data/antennas-generated.ts` and copy antenna images. All consumers already import from `lib/data.ts`, which just needs its antenna import source changed.

**Tech Stack:** Node.js, TypeScript (tsx), Next.js 15, JSON Schema (ajv), Decap CMS

---

### Task 1: Create JSON Schema for Antennas

**Files:**
- Create: `rfindex-data/schemas/meshtastic_antennas.json`

**Context:** This schema is used by `scripts/validate.js` (via ajv) and runs in CI on every PR. See `schemas/meshtastic_devices.json` for the pattern. The antenna fields come from `types/antenna.ts` in rfindex.

**Step 1: Create the schema file**

```json
{
  "type": "object",
  "required": [
    "slug", "title", "manufacturer", "suppliers",
    "test_results", "connector_type"
  ],
  "additionalProperties": false,
  "properties": {
    "slug": { "type": "string", "minLength": 1 },
    "title": { "type": "string", "minLength": 1 },
    "manufacturer": {
      "type": "object",
      "required": ["brand_name", "part_number", "description", "freq_spec", "datasheet"],
      "additionalProperties": false,
      "properties": {
        "brand_name": { "type": "string", "minLength": 1 },
        "url": { "type": "string" },
        "part_number": { "type": "string", "minLength": 1 },
        "description": { "type": "string", "minLength": 1 },
        "freq_spec": {
          "type": "array",
          "items": { "type": "string" },
          "minItems": 1
        },
        "datasheet": { "type": "string", "minLength": 1 }
      }
    },
    "suppliers": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "purchase_cost", "url"],
        "additionalProperties": false,
        "properties": {
          "name": { "type": "string", "minLength": 1 },
          "part_number": { "type": "string" },
          "purchase_cost": { "type": "string", "minLength": 1 },
          "url": { "type": "string", "minLength": 1 }
        }
      },
      "minItems": 1
    },
    "test_results": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["markers", "notes", "metadata"],
        "additionalProperties": false,
        "properties": {
          "markers": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["frequency", "vswr"],
              "additionalProperties": false,
              "properties": {
                "frequency": { "type": "string", "minLength": 1 },
                "vswr": { "type": "string", "minLength": 1 }
              }
            },
            "minItems": 1
          },
          "notes": { "type": "string" },
          "metadata": {
            "type": "object",
            "required": ["tester", "date"],
            "additionalProperties": false,
            "properties": {
              "tester": { "type": "string", "minLength": 1 },
              "date": { "type": "string", "minLength": 1 }
            }
          }
        }
      },
      "minItems": 1
    },
    "suggested": { "type": "boolean" },
    "category": {
      "type": "string",
      "enum": [
        "Internal Device Antennas",
        "Fixed (Base) Antennas",
        "Portable Antennas",
        "Vehicle Antennas"
      ]
    },
    "pdf": { "type": "string" },
    "dimensions": {
      "type": "object",
      "required": ["length"],
      "additionalProperties": false,
      "properties": {
        "length": { "type": "number" },
        "width": { "type": ["number", "null"] }
      }
    },
    "connector_type": {
      "type": "string",
      "enum": [
        "u.FL/IPEX",
        "N-Type Male",
        "N-Type Female",
        "SMA Male",
        "RP-SMA Male",
        "NMO"
      ]
    },
    "gain": { "type": "string" },
    "image": { "type": "string" },
    "description": { "type": "string" }
  }
}
```

**Step 2: Verify schema is valid JSON**

Run: `node -e "JSON.parse(require('fs').readFileSync('schemas/meshtastic_antennas.json','utf8')); console.log('Valid JSON')"`
Expected: `Valid JSON`

**Step 3: Commit**

```bash
git add schemas/meshtastic_antennas.json
git commit -m "feat: add JSON schema for meshtastic antennas"
```

---

### Task 2: Add Antennas Collection to CMS Config

**Files:**
- Modify: `rfindex-data/admin/config.yml`

**Context:** Decap CMS reads this file to generate the admin UI. Each collection needs field definitions. See the `meshtastic_devices` collection (line ~157) for the pattern. Add the new collection after the existing `meshtastic_devices` block.

**Step 1: Add meshtastic_antennas collection to config.yml**

Add this block at the end of the `collections:` array in `admin/config.yml`:

```yaml
  - name: "meshtastic_antennas"
    label: "Meshtastic Antennas"
    label_singular: "Meshtastic Antenna"
    folder: "data/meshtastic_antennas"
    create: true
    extension: "json"
    slug: "{{fields.slug}}"
    summary: "{{manufacturer.brand_name}} - {{title}}"
    sortable_fields: ["title", "manufacturer"]
    fields:
      - { label: "Slug", name: "slug", widget: "string", hint: "URL slug used for routing. Must be unique." }
      - { name: "title", label: "Title", widget: "string" }
      - label: "Manufacturer"
        name: "manufacturer"
        widget: "object"
        fields:
          - { name: "brand_name", label: "Brand Name", widget: "string" }
          - { name: "url", label: "URL", widget: "string", required: false }
          - { name: "part_number", label: "Part Number", widget: "string" }
          - { name: "description", label: "Description", widget: "string" }
          - label: "Frequency Spec"
            name: "freq_spec"
            widget: "list"
            field: { name: "freq", label: "Frequency Range", widget: "string" }
          - { name: "datasheet", label: "Datasheet URL", widget: "string" }
      - label: "Suppliers"
        label_singular: "Supplier"
        name: "suppliers"
        widget: "list"
        fields:
          - { name: "name", label: "Name", widget: "string" }
          - { name: "part_number", label: "Part Number", widget: "string", required: false }
          - { name: "purchase_cost", label: "Purchase Cost", widget: "string" }
          - { name: "url", label: "URL", widget: "string" }
      - label: "Test Results"
        label_singular: "Test Result"
        name: "test_results"
        widget: "list"
        fields:
          - label: "Markers"
            label_singular: "Marker"
            name: "markers"
            widget: "list"
            fields:
              - { name: "frequency", label: "Frequency", widget: "string" }
              - { name: "vswr", label: "VSWR", widget: "string" }
          - { name: "notes", label: "Notes", widget: "text" }
          - label: "Metadata"
            name: "metadata"
            widget: "object"
            fields:
              - { name: "tester", label: "Tester", widget: "string" }
              - { name: "date", label: "Date", widget: "string" }
      - { name: "suggested", label: "Suggested", widget: "boolean", required: false }
      - label: "Category"
        name: "category"
        widget: "select"
        required: false
        options:
          - "Internal Device Antennas"
          - "Fixed (Base) Antennas"
          - "Portable Antennas"
          - "Vehicle Antennas"
      - { name: "pdf", label: "PDF", widget: "string", required: false }
      - label: "Dimensions"
        name: "dimensions"
        widget: "object"
        required: false
        fields:
          - { name: "length", label: "Length (mm)", widget: "number", value_type: "float" }
          - { name: "width", label: "Width (mm)", widget: "number", value_type: "float", required: false }
      - label: "Connector Type"
        name: "connector_type"
        widget: "select"
        options:
          - "u.FL/IPEX"
          - "N-Type Male"
          - "N-Type Female"
          - "SMA Male"
          - "RP-SMA Male"
          - "NMO"
      - { name: "gain", label: "Gain", widget: "string", required: false }
      - label: "Image"
        name: "image"
        widget: "image"
        required: false
        media_folder: "images"
        media_library:
          config:
            multiple: false
            max_file_size: 200000
      - { name: "description", label: "Description", widget: "text", required: false }
```

**Step 2: Commit**

```bash
git add admin/config.yml
git commit -m "feat: add meshtastic_antennas collection to CMS config"
```

---

### Task 3: Extract Antenna Data to JSON Files

**Files:**
- Create: `rfindex/scripts/extract-antennas.ts` (one-off, delete after use)
- Create: `rfindex-data/data/meshtastic_antennas/*.json` (25 files)

**Context:** The 25 antennas are currently defined as a TypeScript array in `rfindex/data/antennas.ts`. We need to convert each antenna object into an individual JSON file in the rfindex-data submodule. The `image` field needs its path prefix stripped (e.g., `/meshtastic/antennas/foo.webp` → `foo.webp`).

**Step 1: Write the extraction script**

Create `rfindex/scripts/extract-antennas.ts`:

```typescript
import { antennas } from "../data/antennas"
import fs from "fs"
import path from "path"

const outDir = path.join(process.cwd(), "rfindex-data", "data", "meshtastic_antennas")
fs.mkdirSync(outDir, { recursive: true })

for (const antenna of antennas) {
  const json: Record<string, unknown> = { ...antenna }
  // Strip path prefix from image — prebuild script will add it back
  if (typeof json.image === "string" && (json.image as string).startsWith("/meshtastic/antennas/")) {
    json.image = (json.image as string).replace("/meshtastic/antennas/", "")
  }
  // Remove empty image strings
  if (json.image === "") {
    delete json.image
  }
  const filename = `${antenna.slug}.json`
  fs.writeFileSync(path.join(outDir, filename), JSON.stringify(json, null, 2) + "\n")
}

console.log(`Extracted ${antennas.length} antennas to ${outDir}`)
```

**Step 2: Run the extraction script from rfindex root**

Run: `cd /Users/tekstrand/git-repos/rfindex && npx tsx scripts/extract-antennas.ts`
Expected: `Extracted 25 antennas to /Users/tekstrand/git-repos/rfindex/rfindex-data/data/meshtastic_antennas`

**Step 3: Verify a sample JSON file**

Run: `cat rfindex-data/data/meshtastic_antennas/molex-206764-0050.json | head -20`
Expected: JSON object with `slug`, `title`, `manufacturer`, `image: "molex-206764-0050.webp"` (bare filename, no path prefix)

**Step 4: Verify file count**

Run: `ls rfindex-data/data/meshtastic_antennas/*.json | wc -l`
Expected: `25`

**Step 5: Delete the one-off script**

Run: `rm scripts/extract-antennas.ts`

**Step 6: Register antennas in validate.js and run validation**

Edit `rfindex-data/scripts/validate.js` — add to the `collections` object:

```javascript
  meshtastic_antennas:  { schema: "meshtastic_antennas.json",  dir: "meshtastic_antennas" },
```

**Step 7: Run validation**

Run: `cd /Users/tekstrand/git-repos/rfindex-data && npm test`
Expected: `Validated N files across 10 collections. All files passed.`

If validation fails, fix the JSON files or schema as needed before proceeding.

**Step 8: Commit in rfindex-data**

```bash
cd /Users/tekstrand/git-repos/rfindex-data
git add data/meshtastic_antennas/*.json scripts/validate.js
git commit -m "feat: add 25 antenna JSON files with validation"
```

---

### Task 4: Copy Antenna Images to rfindex-data

**Files:**
- Create: `rfindex-data/data/meshtastic_antennas/images/*.webp` (23 files)

**Context:** Antenna images are currently in `rfindex/public/meshtastic/antennas/` as optimized .webp files. Copy them to the submodule. The `large/` subfolder and non-webp originals stay in rfindex (not migrated).

**Step 1: Copy webp images**

Run:
```bash
mkdir -p /Users/tekstrand/git-repos/rfindex-data/data/meshtastic_antennas/images
cp /Users/tekstrand/git-repos/rfindex/public/meshtastic/antennas/*.webp /Users/tekstrand/git-repos/rfindex-data/data/meshtastic_antennas/images/
```

**Step 2: Verify image count**

Run: `ls /Users/tekstrand/git-repos/rfindex-data/data/meshtastic_antennas/images/*.webp | wc -l`
Expected: `23`

**Step 3: Commit in rfindex-data**

```bash
cd /Users/tekstrand/git-repos/rfindex-data
git add data/meshtastic_antennas/images/
git commit -m "feat: add antenna images"
```

---

### Task 5: Update rfindex-data Documentation

**Files:**
- Modify: `rfindex-data/CLAUDE.md`

**Context:** The CLAUDE.md documents the repo structure and conventions. Add `meshtastic_antennas` to the repo structure section and any relevant conventions.

**Step 1: Update CLAUDE.md**

Add `meshtastic_antennas` to the repository structure section alongside `meshtastic_devices`, and add antenna-specific conventions (image naming, category enums, connector type enums).

**Step 2: Commit**

```bash
cd /Users/tekstrand/git-repos/rfindex-data
git add CLAUDE.md
git commit -m "docs: add meshtastic_antennas to repo structure"
```

---

### Task 6: Push rfindex-data

**Step 1: Push all commits**

Run: `cd /Users/tekstrand/git-repos/rfindex-data && git push`

---

### Task 7: Rename Prebuild Script and Add Antenna Loading

**Files:**
- Rename: `rfindex/lib/load-devices.ts` → `rfindex/lib/prebuild.ts`
- Modify: `rfindex/lib/prebuild.ts` (add antenna generation + image copy)

**Context:** The existing prebuild script reads device JSON and generates `data/devices-generated.ts`. We need to extend it to also generate `data/antennas-generated.ts` from the antenna JSON files, and copy antenna images to `public/meshtastic/antennas/`. The `image` field in JSON stores a bare filename (`foo.webp`) — the script must prepend `/meshtastic/antennas/` when generating the TypeScript.

**Step 1: Rename the file**

Run: `cd /Users/tekstrand/git-repos/rfindex && git mv lib/load-devices.ts lib/prebuild.ts`

**Step 2: Add antenna loading to lib/prebuild.ts**

Append this code after the existing device image copying section (after line 89):

```typescript
// --- Antenna loading ---

const antennasDir = path.join(process.cwd(), "rfindex-data", "data", "meshtastic_antennas")

if (!fs.existsSync(antennasDir)) {
  console.error("rfindex-data submodule antenna data not found at", antennasDir)
  process.exit(1)
}

const antennaFiles = fs.readdirSync(antennasDir).filter((f) => f.endsWith(".json"))
const antennaData = antennaFiles.map((file) => {
  const raw = JSON.parse(fs.readFileSync(path.join(antennasDir, file), "utf-8"))
  // Map bare image filename to full path
  if (raw.image && !raw.image.startsWith("/")) {
    raw.image = `/meshtastic/antennas/${raw.image}`
  }
  return raw
})

const antennaOutput = `import type { Antenna } from "@/types/antenna"

// Auto-generated from rfindex-data submodule. Do not edit manually.
// Regenerate with: npx tsx lib/prebuild.ts

export const antennas: Antenna[] = ${JSON.stringify(antennaData, null, 2)}
`

const antennaOutPath = path.join(process.cwd(), "data", "antennas-generated.ts")
fs.writeFileSync(antennaOutPath, antennaOutput)
console.log(`Generated ${antennaOutPath} with ${antennaData.length} antennas`)

// Copy antenna images from submodule to public/meshtastic/antennas/
const antennaImagesSource = path.join(antennasDir, "images")
const antennaImagesDest = path.join(process.cwd(), "public", "meshtastic", "antennas")

if (fs.existsSync(antennaImagesSource)) {
  fs.mkdirSync(antennaImagesDest, { recursive: true })
  const antennaImages = fs.readdirSync(antennaImagesSource).filter((f) => f.endsWith(".webp"))
  for (const img of antennaImages) {
    fs.copyFileSync(path.join(antennaImagesSource, img), path.join(antennaImagesDest, img))
  }
  console.log(`Copied ${antennaImages.length} antenna images to public/meshtastic/antennas/`)
}
```

Also update the file's doc comment at the top to mention antennas:

```typescript
/**
 * Prebuild script: reads device and antenna JSON files from rfindex-data submodule
 * and generates data/devices-generated.ts and data/antennas-generated.ts
 * for the Next.js app to import.
 *
 * Run via: npx tsx lib/prebuild.ts
 */
```

**Step 3: Commit**

```bash
cd /Users/tekstrand/git-repos/rfindex
git add lib/prebuild.ts
git rm lib/load-devices.ts
git commit -m "refactor: rename load-devices to prebuild, add antenna generation"
```

---

### Task 8: Update package.json, .gitignore, and lib/data.ts

**Files:**
- Modify: `rfindex/package.json`
- Modify: `rfindex/.gitignore`
- Modify: `rfindex/lib/data.ts`

**Context:** Three small changes:
1. `package.json` scripts reference `lib/load-devices.ts` — change to `lib/prebuild.ts`
2. `.gitignore` needs `data/antennas-generated.ts` and `public/meshtastic/antennas/` (antenna images will now come from submodule)
3. `lib/data.ts` imports from `@/data/antennas` — change to `@/data/antennas-generated`

**Step 1: Update package.json scripts**

Change all three occurrences of `lib/load-devices.ts` to `lib/prebuild.ts`:

```json
"prebuild": "git submodule update --init --recursive && npx tsx lib/prebuild.ts",
"dev": "git submodule update --init --recursive && npx tsx lib/prebuild.ts && next dev",
"build": "git submodule update --init --recursive && npx tsx lib/prebuild.ts && next build",
```

**Step 2: Update .gitignore**

Add these lines to the end of `.gitignore`:

```
data/antennas-generated.ts
public/meshtastic/antennas/
```

Note: `public/meshtastic/antennas/` was previously tracked. We need to remove it from git tracking:

Run: `cd /Users/tekstrand/git-repos/rfindex && git rm -r --cached public/meshtastic/antennas/`

**Step 3: Update lib/data.ts**

Change the antenna import and re-export:

Old:
```typescript
import { antennas } from "@/data/antennas"
// ...
// Re-export antenna data (still inline until rfindex-data adds antennas)
export { antennas } from "@/data/antennas"
```

New:
```typescript
import { antennas } from "@/data/antennas-generated"
// ...
// Re-export antenna data (generated from rfindex-data submodule at prebuild)
export { antennas } from "@/data/antennas-generated"
```

**Step 4: Delete inline antenna data**

Run: `rm /Users/tekstrand/git-repos/rfindex/data/antennas.ts`

**Step 5: Update submodule pointer**

Run:
```bash
cd /Users/tekstrand/git-repos/rfindex
git submodule update --remote rfindex-data
```

**Step 6: Commit**

```bash
cd /Users/tekstrand/git-repos/rfindex
git add package.json .gitignore lib/data.ts rfindex-data
git rm data/antennas.ts
git commit -m "feat: integrate antenna data from rfindex-data submodule"
```

---

### Task 9: Build Verification and Documentation

**Files:**
- Modify: `rfindex/CLAUDE.md`

**Context:** `npm run build` is the primary verification for this repo. It statically generates every device and antenna page. A successful build confirms the migration worked end-to-end.

**Step 1: Run the prebuild script manually first**

Run: `cd /Users/tekstrand/git-repos/rfindex && npx tsx lib/prebuild.ts`
Expected output:
```
Generated data/devices-generated.ts with 57 devices
Copied N device images to public/devices/
Generated data/antennas-generated.ts with 25 antennas
Copied 23 antenna images to public/meshtastic/antennas/
```

**Step 2: Run full build**

Run: `cd /Users/tekstrand/git-repos/rfindex && npm run build`
Expected: Build completes successfully with no errors. All antenna detail pages are statically generated.

If build fails, debug and fix before proceeding.

**Step 3: Update CLAUDE.md**

Update the Data Layer section to reflect that antenna data now comes from the submodule:

- Change `data/antennas.ts` references to `data/antennas-generated.ts`
- Update the `lib/load-devices.ts` reference to `lib/prebuild.ts`
- Update the "Adding Data > New Antenna" section to describe adding JSON to rfindex-data instead of editing `data/antennas.ts`
- Add `public/meshtastic/antennas/` to the list of gitignored generated files

**Step 4: Commit**

```bash
cd /Users/tekstrand/git-repos/rfindex
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for antenna data migration"
```

---

### Task 10: Push rfindex

**Step 1: Push**

Run: `cd /Users/tekstrand/git-repos/rfindex && git push`

**Step 2: Verify Vercel deploy succeeds**

Check that the Vercel deployment completes without errors. The deploy will:
1. Initialize the submodule (`git submodule update --init --recursive`)
2. Run the prebuild script (generates both device and antenna TypeScript + copies images)
3. Build Next.js (statically generates all pages)
