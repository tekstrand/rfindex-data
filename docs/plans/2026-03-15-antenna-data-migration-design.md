# Antenna Data Migration Design

## Goal

Move antenna data from inline TypeScript (`data/antennas.ts`) in rfindex to JSON files in the rfindex-data submodule, matching the pattern established by the device migration.

## Architecture

Antenna data follows the same prebuild pipeline as devices: JSON files in the submodule are read at build time by a prebuild script that generates a static TypeScript file and copies images. The existing `lib/load-devices.ts` is renamed to `lib/prebuild.ts` to serve as a single entry point for both devices and antennas.

## JSON Structure & Field Mapping

Each antenna becomes `rfindex-data/data/meshtastic_antennas/{slug}.json`. Fields are 1:1 with the current `Antenna` TypeScript type, with one exception: `image` stores a bare filename (`foo.webp`) instead of a full path (`/meshtastic/antennas/foo.webp`). The prebuild script maps it back.

```json
{
  "slug": "molex-206764-0050",
  "title": "Molex PCB ISM Dipole",
  "manufacturer": {
    "brand_name": "Molex",
    "part_number": "206764-0050",
    "description": "ISM Dipole Flex Ant 868/915 MHz 50MM",
    "freq_spec": ["862-876 MHz", "902-928 MHz"],
    "datasheet": "https://..."
  },
  "suppliers": [
    { "name": "Mouser", "part_number": "538-206764-0050", "purchase_cost": "$3.76", "url": "https://..." }
  ],
  "test_results": [
    {
      "markers": [{ "frequency": "902MHz", "vswr": "1.220:1" }],
      "notes": "...",
      "metadata": { "tester": "Ric Letson, NB2E", "date": "15 September 2023" }
    }
  ],
  "suggested": true,
  "category": "Internal Device Antennas",
  "pdf": "PDFs/206764-0050.pdf",
  "dimensions": { "length": 100.0, "width": 16.0 },
  "connector_type": "u.FL/IPEX",
  "gain": "1.3 dBi",
  "image": "molex-206764-0050.webp",
  "description": ""
}
```

## rfindex-data Repo Changes

- `data/meshtastic_antennas/` — 25 JSON files, one per antenna, named `{slug}.json`
- `data/meshtastic_antennas/images/` — 23 optimized webp images (copied from rfindex)
- `admin/config.yml` — new `meshtastic_antennas` collection with all fields
- `schemas/meshtastic_antennas.json` — JSON Schema for CI validation
- `CLAUDE.md` — update structure docs

## rfindex Changes

- Rename `lib/load-devices.ts` → `lib/prebuild.ts` — single prebuild entry point for devices + antennas
- Prebuild generates `data/antennas-generated.ts` and copies antenna images to `public/meshtastic/antennas/`
- `package.json` — update scripts to reference `lib/prebuild.ts`
- `lib/data.ts` — import from `@/data/antennas-generated` instead of `@/data/antennas`
- `.gitignore` — add `data/antennas-generated.ts` and `public/meshtastic/antennas/`
- Delete `data/antennas.ts`
- Update `CLAUDE.md`
