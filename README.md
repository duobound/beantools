# 🌿 Beantools

A small collection of desktop tools built for [Beanlemon Field Notes](https://beanlemon.com) — a nature photography journal by Bean & Lemon.

These tools are built around a specific workflow for processing, naming, and uploading deep zoom nature photography, but are designed to be usable by anyone with their own Backblaze B2 or Cloudflare R2 bucket and Anthropic API key.

---

## Tools

### 🔄 Beanlemon Pipeline *(new)*
All-in-one photo processing pipeline — rename, tile, hero+thumb, and upload in a single app.

Combines all four steps of the Beanlemon workflow into one GUI with individually toggleable steps. Run the full pipeline end-to-end or pick exactly which steps to run. Each step shows live progress in a shared log. Before uploading, a confirmation dialog shows exactly what files will be sent and where.

**Steps:**
1. **AI Rename** — opens an interactive per-photo review window. Claude identifies species, category, and behavior. Every field is editable before confirming. Runs photo by photo, pipeline continues when all photos are done.
2. **Tile** — runs `vips dzsave` on each renamed JPEG to generate deep zoom `.dzi` + `_files/` tile sets. Skips photos already tiled.
3. **Hero + Thumb** — renames hero JPEGs to match the export filename, generates a resized thumbnail in a separate folder.
4. **Upload** — uploads the selected files (`.dzi`, tiles, hero, and/or thumb) to Cloudflare R2 or Backblaze B2. Configurable per-step checkboxes control exactly what gets uploaded.

**What you need:**
- An Anthropic API key (get one at [console.anthropic.com](https://console.anthropic.com))
- Cloudflare R2 or Backblaze B2 credentials
- [libvips](https://www.libvips.org/) installed and on your PATH (`brew install vips` / `choco install vips`)

**Settings panel (⚙):**
All folder paths, credentials, thumbnail size, upload batch size, and upload toggles are configurable and saved locally. Settings are never hardcoded — the app works out of the box for anyone with their own accounts.

**Folder paths (all configurable):**
- Exports folder — where renamed JPEGs and tiles live
- Hero folder — where hero JPEGs are stored
- Thumbs folder — where thumbnails are generated

**Upload options:**
- Storage provider toggle: R2 (recommended) or B2
- Checkboxes for what gets uploaded: `.dzi` + tiles, hero JPG, thumb JPG
- Configurable batch size (default: 12)

**Upload confirmation:**
Before any upload begins, a summary dialog shows exactly what will be sent — folder paths, `.dzi` files, tile counts, hero and thumb files — so you can confirm or cancel before anything leaves your machine.

**Filename format expected:**
```
DATE_LOCATION_CATEGORY_SPECIES_BEHAVIOR_HHMMSS_NNNN.jpg
20250420_bolsa-chica_birds_forsters-tern_landing_100234_7117.jpg
```

**Storage folder structure produced:**
```
your-bucket/
├── birds/forsters-tern/photo-name.dzi
├── birds/forsters-tern/photo-name_files/
├── birds/forsters-tern/photo-name.jpg        (hero, if enabled)
├── birds/forsters-tern/photo-name_thumb.jpg  (thumb, if enabled)
└── ...
```

**Dependencies:**
```
pip install anthropic pillow requests boto3
```
vips must be on PATH.

---

### 🐦 Beanlemon Renamer
AI-powered photo renaming tool for nature photographers.

Reads EXIF data from your exported JPEGs, sends each photo to Claude (Anthropic's AI) for species/subject identification, and suggests a structured filename based on date, location, category, species, and behavior. Every field is editable before confirming.

**Filename format produced:**
```
DATE_LOCATION_CATEGORY_SPECIES_BEHAVIOR_FOLDER_FILENUMBER.jpg
20250420_bolsa-chica-ecological-reserve_birds_forsters-tern_landing_100234_7117.jpg
```

**Categories supported:**
`birds` · `mammals` · `insects` · `plants` · `aquatic` · `landscapes` · `other`

**What you need:**
- An Anthropic API key (get one at [console.anthropic.com](https://console.anthropic.com))
- A folder of exported JPEGs to rename

**Usage:**
1. Launch **Beanlemon Renamer**
2. Each photo is shown full size on the left
3. AI suggests category, species, and behavior — edit any field if needed
4. Preview the filename before confirming
5. Press **Enter** or click **✓ Rename & Next**
6. Press **Escape** to skip a photo

---

### ☁️ Beanlemon Uploader
Batch deep zoom tile uploader for Backblaze B2 and Cloudflare R2.

After processing photos with [libvips](https://www.libvips.org/) (`vips dzsave`) to generate deep zoom tile sets, this tool uploads them to your selected storage bucket automatically. It reads the category and species from your filename, creates the correct folder structure, and uploads the `.dzi` manifest and `_files/` tile folder together.

**Storage folder structure produced:**
```
your-bucket/
├── birds/forsters-tern/photo-name.dzi
├── birds/forsters-tern/photo-name_files/
├── mammals/fox-squirrel/photo-name.dzi
├── landscapes/tide-pools/photo-name.dzi
└── ...
```

**What you need:**
- For R2: Cloudflare account, R2 bucket, Access Key ID, Secret Access Key, Account ID
- For B2: Backblaze account, Key ID, Application Key, Bucket ID
- Photos renamed using the Beanlemon Renamer (or matching the filename format above)
- Deep zoom tiles already generated locally using `vips dzsave`

**Usage:**
1. Launch **Beanlemon Uploader**
2. Click **Browse Folder** and select the folder containing your `.dzi` files and `_files/` folders
3. The app detects all matching pairs automatically and lists them
4. Uncheck any you want to skip
5. Click **☁️ Upload All**
6. Each pair uploads in sequence with a live progress bar

**Note:** This tool uploads tiles only — not JPEG thumbnails. Thumbnails are handled separately by your CMS or admin panel.

---

## Workflow Overview

### Original workflow (separate tools):
```
Lightroom export (JPEG)
        ↓
Beanlemon Renamer      ←  AI identifies species + category
        ↓
vips dzsave            ←  generate deep zoom tiles locally
        ↓
Beanlemon Uploader     ←  upload .dzi + _files/ to R2 or B2
        ↓
Admin panel            ←  upload JPEG thumbnail, update gallery JSON
        ↓
Gallery live at beanlemon.com/gallery.html
```

### Pipeline workflow (all-in-one):
```
Lightroom export (JPEG)
        ↓
Beanlemon Pipeline
  ├── Step 1: AI Rename     ←  interactive per-photo review window
  ├── Step 2: Tile          ←  vips dzsave on each JPEG
  ├── Step 3: Hero + Thumb  ←  rename hero, generate thumbnail
  └── Step 4: Upload        ←  .dzi + tiles to R2 or B2
        ↓
Admin panel            ←  upload JPEG thumbnail, update gallery JSON
        ↓
Gallery live at beanlemon.com/gallery.html
```

---

## Installation

All tools are available for **Mac** and **Windows** in the [Releases](../../releases) section.

### Beanlemon Pipeline
- `Beanlemon-Pipeline-mac.zip` — extract and run `Beanlemon Pipeline.app`
- `Beanlemon-Pipeline-win.exe` — run directly, no install needed

### Beanlemon Renamer & Uploader (Windows)
- `Beanlemon-Renamer-Setup-1.0.0.exe`
- `Beanlemon-Uploader-Setup-1.0.0.exe`

Download, run the installer, and launch from your Start Menu or Desktop shortcut.

> **Mac note:** On first launch macOS may block the app. Right-click → Open → Open Anyway. Or run once in Terminal: `xattr -cr "/Applications/Beanlemon Pipeline.app"`

---

## Building from Source

### Beanlemon Pipeline (Python)
```bash
pip install anthropic pillow requests boto3 pyinstaller

# Mac
pyinstaller --onedir --windowed --name "Beanlemon Pipeline" \
  --icon beanlemon_pipeline.icns beanlemon_pipeline.py

# Windows
pyinstaller --onefile --windowed --name "Beanlemon Pipeline" \
  --icon beanlemon_pipeline.ico \
  --hidden-import boto3 --hidden-import botocore \
  --hidden-import requests --hidden-import anthropic \
  --collect-all boto3 --collect-all botocore --collect-all anthropic \
  beanlemon_pipeline.py
```

### Beanlemon Renamer (Python)
```bash
pip install pyinstaller anthropic pillow
pyinstaller --onefile --windowed --name "BeanlemanRenamer" beanlemon_rename_gui.py
```

### Beanlemon Uploader (Electron)
```bash
npm install
npm run dist
```

---

## Notes

- All tools are free to use with your own API keys and storage credentials
- AI naming uses your own Anthropic API key — you pay for your own usage
- B2/R2 uploads use your own credentials — your bucket, your files
- No data is sent to Beanlemon servers
- Credentials are saved locally on your machine and never included in the app

---

## 📖 Docs

- [Full Photo Site Guide](docs/PHOTO_SITE_GUIDE.md) — end-to-end walkthrough for building a photo site with this stack
- [Caching Architecture](docs/BACKBLAZE_CLOUDFLARE_CACHING.md) — deep dive on Backblaze B2 + Cloudflare caching

---

*Built with 🌿 by [Beanlemon Field Notes](https://beanlemon.com)*
