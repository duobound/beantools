# 🌿 Beantools

A small collection of Windows desktop tools built for [Beanlemon Field Notes](https://beanlemon.com) — a nature photography journal by Bean & Lemon.

These tools are built around a specific workflow for processing, naming, and uploading deep zoom nature photography, but are designed to be usable by anyone with their own Backblaze B2 bucket or Anthropic API key.

---

## Tools

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

**First launch:**
The app will ask for your Anthropic API key and your export folder path. These are saved locally and never leave your machine.

**Usage:**
1. Launch **Beanlemon Renamer**
2. Each photo is shown full size on the left
3. AI suggests category, species, and behavior — edit any field if needed
4. Preview the filename before confirming
5. Press **Enter** or click **✓ Rename & Next**
6. Press **Escape** to skip a photo

---

### ☁️ Beanlemon B2 Uploader
Batch deep zoom tile uploader for Backblaze B2.

After processing photos with [libvips](https://www.libvips.org/) (`vips dzsave`) to generate deep zoom tile sets, this tool uploads them to your Backblaze B2 bucket automatically. It reads the category and species from your filename, creates the correct folder structure in B2 if it doesn't exist, and uploads the `.dzi` manifest and `_files/` tile folder together.

**B2 folder structure produced:**
```
your-bucket/
├── birds/forsters-tern/photo-name.dzi
├── birds/forsters-tern/photo-name_files/
├── mammals/fox-squirrel/photo-name.dzi
├── landscapes/tide-pools/photo-name.dzi
└── ...
```

**What you need:**
- A Backblaze B2 account and bucket
- A B2 Application Key ID, Application Key, and Bucket ID
- Photos already renamed using the Beanlemon Renamer (or matching the filename format above)
- Deep zoom tiles already generated locally using `vips dzsave`

**First launch:**
Click ⚙️ B2 Settings and enter your Key ID, Application Key, and Bucket ID. These are saved locally on your machine.

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

The full Beanlemon Field Notes photo workflow these tools are part of:

```
Lightroom export (JPEG)
        ↓
Beanlemon Renamer  ←  AI identifies species + category
        ↓
vips dzsave  (generate deep zoom tiles locally)
        ↓
Beanlemon B2 Uploader  (upload .dzi + _files/ to B2)
        ↓
Admin panel  (upload JPEG thumbnail, update gallery JSON)
        ↓
Gallery live at beanlemon.com/gallery.html
```

---

## Installation

Both tools are available as Windows installers in the [Releases](../../releases) section.

- `Beanlemon-Renamer-Setup-1.0.0.exe`
- `Beanlemon-Uploader-Setup-1.0.0.exe`

Download, run the installer, and launch from your Start Menu or Desktop shortcut.

---

## Building from Source

### Beanlemon Renamer (Python)
```
pip install pyinstaller anthropic pillow appdirs
pyinstaller --onefile --windowed --name "BeanlemanRenamer" beanlemon_rename_gui.py
```

### Beanlemon B2 Uploader (Electron)
```
npm install
npm run dist
```

---

## Notes

- These tools are free to use with your own API keys and B2 bucket
- AI naming uses your own Anthropic API key — you pay for your own usage

- ## 📖 Docs

- [Full Photo Site Guide](docs/PHOTO_SITE_GUIDE.md) — end-to-end walkthrough for building a photo site with this stack
- [Caching Architecture](docs/BACKBLAZE_CLOUDFLARE_CACHING.md) — deep dive on Backblaze B2 + Cloudflare caching
- B2 uploads use your own Backblaze credentials — your bucket, your files
- No data is sent to Beanlemon servers

---

*Built with 🌿 by [Beanlemon Field Notes](https://beanlemon.com)*
