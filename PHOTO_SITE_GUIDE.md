# Building a Nature Photo Site with Deep Zoom
### Backblaze B2 · Cloudflare · libvips · OpenSeadragon · Beantools

A complete end-to-end guide for building a static nature photography site with deep zoom tile viewing, cloud image storage, and a zero-cost-at-scale serving architecture.

This guide covers the full workflow: from exporting photos in Lightroom, AI-assisted renaming, generating deep zoom tiles, uploading to Backblaze B2, and serving everything efficiently through Cloudflare.

---

## Table of Contents

1. [Overview & Stack](#1-overview--stack)
2. [Why This Stack?](#2-why-this-stack)
3. [Prerequisites](#3-prerequisites)
4. [Backblaze B2 Setup](#4-backblaze-b2-setup)
5. [Cloudflare Setup](#5-cloudflare-setup)
6. [Photo Export from Lightroom](#6-photo-export-from-lightroom)
7. [AI Photo Renaming with Beanlemon Renamer](#7-ai-photo-renaming-with-beanlemon-renamer)
8. [Generating Deep Zoom Tiles with libvips](#8-generating-deep-zoom-tiles-with-libvips)
9. [Uploading to B2 with Beanlemon Uploader](#9-uploading-to-b2-with-beanlemon-uploader)
10. [The Cloudflare Worker](#10-the-cloudflare-worker)
11. [OpenSeadragon Integration](#11-openseadragon-integration)
12. [Caching Architecture & B2 Transaction Costs](#12-caching-architecture--b2-transaction-costs)
13. [Folder Structure Reference](#13-folder-structure-reference)
14. [Cost Breakdown](#14-cost-breakdown)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Overview & Stack

This guide walks through building a static photo site where:

- Photos are stored in **Backblaze B2** (cheap, S3-compatible object storage)
- The site is hosted on **Cloudflare Pages** (free static hosting)
- A **Cloudflare Worker** acts as a secure middleware layer between your site and B2
- Deep zoom tile sets are generated locally with **libvips** and served via **OpenSeadragon**
- **Beantools** (free, open source) handles photo renaming and B2 uploading on Windows

**The full workflow at a glance:**

```
Lightroom export (JPEG)
        ↓
Beanlemon Renamer  ←  AI identifies species + suggests structured filename
        ↓
libvips (vips dzsave)  ←  generates deep zoom tile sets locally
        ↓
Beanlemon B2 Uploader  ←  uploads .dzi + tile folders to Backblaze B2
        ↓
Cloudflare Worker  ←  proxies B2 files with long-term cache headers
        ↓
OpenSeadragon  ←  renders deep zoom viewer in the browser
        ↓
Your site, live
```

---

## 2. Why This Stack?

### Why Backblaze B2?
- **10 GB free storage**, 1 GB/day free egress to Cloudflare (they have a bandwidth alliance — no egress fees between them)
- S3-compatible API
- Much cheaper than AWS S3 or Google Cloud Storage at scale
- Simple bucket structure — works like a file system

### Why Cloudflare?
- Free CDN, free Pages hosting, free Workers (up to 100,000 requests/day)
- Global edge caching means files served near your visitors
- Zero egress fees to/from B2 (bandwidth alliance)
- Worker gives you a secure middleware layer so B2 credentials never touch the browser

### Why libvips + OpenSeadragon?
- Deep zoom lets visitors explore full-resolution images by zooming in — without downloading the entire file upfront
- libvips is fast and free; generates the tile sets locally before upload
- OpenSeadragon is a mature, well-documented open source deep zoom viewer

### Why Beantools?
- Handles the tedious parts: AI-assisted naming with a structured filename format, and batch uploading tile sets to B2 with the correct folder structure
- Free and open source: [github.com/duobound/beantools](https://github.com/duobound/beantools)
- Windows desktop apps, no command line required for the main workflow

---

## 3. Prerequisites

**Accounts needed:**
- [Backblaze](https://www.backblaze.com) — free tier is sufficient to start
- [Cloudflare](https://www.cloudflare.com) — free tier is sufficient
- [Anthropic](https://console.anthropic.com) — for AI photo renaming (pay-per-use, very cheap)
- GitHub — for hosting your site source (optional but recommended)

**Software needed (Windows):**
- [Beantools](https://github.com/duobound/beantools/releases) — Beanlemon Renamer + B2 Uploader
- [libvips](https://www.libvips.org/install.html) — deep zoom tile generation (Windows 64-bit w64 build)
- [Adobe Lightroom](https://lightroom.adobe.com) or any JPEG exporter — photo editing and export
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) — for deploying the Cloudflare Worker (`npm install -g wrangler`)

**Optional but useful:**
- [Cyberduck](https://cyberduck.io) — GUI for browsing/managing your B2 bucket
- [Node.js](https://nodejs.org) — required for Wrangler

---

## 4. Backblaze B2 Setup

### Create a Bucket

1. Log into Backblaze → **B2 Cloud Storage → Buckets → Create a Bucket**
2. Give it a name (e.g. `my-photos`)
3. Set **Files in Bucket** to **Public** — this is required for Cloudflare to serve your files
4. Leave all other settings at default
5. Click **Create a Bucket**

### Get Your Bucket Details

From the bucket page, note down:
- **Bucket Name** (e.g. `my-photos`)
- **Bucket ID** (shown on the bucket page)
- **Endpoint** (e.g. `s3.us-west-004.backblazeb2.com`)

Your public file URL format will be:
```
https://<bucket-name>.s3.<region>.backblazeb2.com/<file-path>
```

### Create an Application Key

1. Go to **B2 Cloud Storage → Application Keys → Add a New Application Key**
2. Give it a name (e.g. `my-site-uploader`)
3. Set access to your specific bucket only
4. Allow **Read and Write**
5. Click **Create New Key**
6. **Copy both the Key ID and the Application Key immediately** — the Application Key is only shown once

---

## 5. Cloudflare Setup

### Connect Your Domain

1. Sign up for Cloudflare and add your domain
2. Update your domain's nameservers at your registrar to the two Cloudflare nameservers provided
3. Wait for DNS propagation (usually a few minutes to a few hours)

### Deploy Your Site with Cloudflare Pages

1. Go to **Cloudflare → Workers & Pages → Create → Pages**
2. Connect your GitHub repo
3. Set build output directory to `/` (or wherever your HTML files are) for a static site with no build step
4. Deploy

Your site will be live at `your-project.pages.dev` and at your custom domain once DNS is connected.

### Set Up the Cloudflare Worker

The Worker is the critical piece — it acts as a secure proxy between your site and B2. See [Section 10](#10-the-cloudflare-worker) for full setup details.

---

## 6. Photo Export from Lightroom

Export your photos as JPEGs before running them through the renaming and tiling workflow.

**Recommended export settings:**
- **Format:** JPEG
- **Quality:** 90–95%
- **Long edge:** 5000px (good balance of detail vs file size for tiling)
- **Color space:** sRGB
- **Metadata:** Copyright only (strips GPS and personal metadata)
- **Watermark:** Bake your watermark in at export time — once tiled, it's embedded in every tile

> **Why 5000px?** libvips will tile this into many zoom levels. A 5000px source gives you enough resolution for viewers to zoom in meaningfully without creating enormous tile sets.

Save these as a Lightroom export preset so you can repeat it consistently.

---

## 7. AI Photo Renaming with Beanlemon Renamer

[Beanlemon Renamer](https://github.com/duobound/beantools) is a Windows desktop app that reads your exported JPEGs, sends each one to Claude (Anthropic's AI) for species identification, and suggests a structured filename. Every field is editable before you confirm.

### Filename Format

```
DATE_LOCATION_CATEGORY_SPECIES_BEHAVIOR_FOLDER_FILENUMBER.jpg

Example:
20250420_bolsa-chica-ecological-reserve_birds_forsters-tern_landing_100234_7117.jpg
```

| Field | Example | Notes |
|-------|---------|-------|
| Date | `20250420` | YYYYMMDD from EXIF |
| Location | `bolsa-chica-ecological-reserve` | Lowercase, hyphenated |
| Category | `birds` | birds, mammals, insects, plants, aquatic, landscapes, other |
| Species | `forsters-tern` | AI-suggested, editable |
| Behavior | `landing` | AI-suggested, editable |
| Folder | `100234` | Camera folder number from EXIF |
| File number | `7117` | Camera file number from EXIF |

### Why This Naming Format?

The category and species fields are used downstream by Beanlemon Uploader to automatically create the correct folder structure in B2. A consistent naming format also makes your archive searchable and sortable without a database.

### First Launch Setup

On first launch, the app will ask for:
- Your **Anthropic API key** (from [console.anthropic.com](https://console.anthropic.com))
- Your **export folder path**

These are saved locally on your machine and never sent anywhere except to Anthropic's API for species identification.

### Usage

1. Launch **Beanlemon Renamer**
2. Each photo is shown full size on the left
3. AI suggests category, species, and behavior — edit any field if needed
4. Preview the final filename before confirming
5. Press **Enter** or click **✓ Rename & Next**
6. Press **Escape** to skip a photo

---

## 8. Generating Deep Zoom Tiles with libvips

Deep zoom works by pre-slicing your image into a pyramid of tiles at multiple zoom levels. The browser only loads the tiles for the current zoom level and viewport — so a 50MB image can be explored smoothly without ever downloading more than a few hundred KB at a time.

### Install libvips (Windows)

1. Go to [libvips.org/install](https://www.libvips.org/install.html)
2. Download the Windows 64-bit **w64** build (e.g. `vips-dev-w64-all-8.x.x.zip`)
3. Extract to a permanent location (e.g. `C:\vips`)
4. Add `C:\vips\bin` to your system PATH

Verify the install:
```
vips --version
```

### Generate Tiles for a Single Photo

```bash
vips dzsave input.jpg output_name --layout dz --tile-size 256 --overlap 1
```

This creates:
- `output_name.dzi` — the manifest file OpenSeadragon reads
- `output_name_files/` — the folder of tile images at all zoom levels

### Batch Process a Folder

Create a simple batch script (`tile_all.bat`):
```bat
@echo off
for %%f in (*.jpg) do (
    vips dzsave "%%f" "%%~nf" --layout dz --tile-size 256 --overlap 1
)
echo Done.
pause
```

Run this in the folder containing your renamed JPEGs. It will generate a `.dzi` and `_files/` folder for each one.

### Tile Output Size

A typical 5000px JPEG will produce a tile set of roughly 15–30 MB depending on image complexity. Tile sets are stored in B2 but are served efficiently because the browser only requests the tiles it actually needs.

---

## 9. Uploading to B2 with Beanlemon Uploader

[Beanlemon Uploader](https://github.com/duobound/beantools) is a Windows desktop app that reads your renamed `.dzi` files, parses the category and species from the filename, and uploads the `.dzi` + `_files/` pair to the correct folder in your B2 bucket.

### B2 Folder Structure Created

```
your-bucket/
├── birds/
│   ├── forsters-tern/
│   │   ├── 20250420_..._forsters-tern_landing_...dzi
│   │   └── 20250420_..._forsters-tern_landing_..._files/
│   │       ├── 0/
│   │       ├── 1/
│   │       └── ...
├── mammals/
│   └── fox-squirrel/
└── landscapes/
    └── tide-pools/
```

### First Launch Setup

Click the **⚙️ B2 Settings** button and enter:
- **Key ID** — your B2 Application Key ID
- **Application Key** — your B2 Application Key
- **Bucket ID** — your bucket's ID (from the Backblaze bucket page)

These are saved locally.

### Usage

1. Launch **Beanlemon Uploader**
2. Click **Browse Folder** and select the folder containing your `.dzi` files and `_files/` folders
3. The app detects all `.dzi` + `_files/` pairs automatically
4. Uncheck any you want to skip
5. Click **☁️ Upload All**
6. Each pair uploads in sequence with a live progress bar

> **Note:** The Uploader handles tile sets only. JPEG thumbnails for your gallery are uploaded separately via your site's admin panel or directly through Cyberduck/the Backblaze web UI.

---

## 10. The Cloudflare Worker

The Cloudflare Worker is the backbone of the serving architecture. It sits between your site and B2, handling:

- **Secure API proxying** — B2 credentials never touch the browser
- **File proxying with cache headers** — B2 files get long-term cache headers so Cloudflare caches them at the edge
- **CORS** — cross-origin headers so your site can fetch from the Worker

### Why a Worker Instead of Direct B2 URLs?

If you point your site directly at B2 URLs, every request hits B2 and counts as a transaction. With the Worker in front:

```
Without Worker:
Browser → B2 (every request = 1 transaction)

With Worker + Cloudflare Cache:
Browser → Cloudflare Edge (cache hit = 0 B2 transactions)
                ↓ (cache miss, first request only)
           Cloudflare Worker → B2 (1 transaction, then cached)
```

### Worker Setup

**1. Install Wrangler**
```bash
npm install -g wrangler
wrangler login
```

**2. Create the Worker project**
```bash
wrangler init my-photo-worker
cd my-photo-worker
```

**3. Set your B2 secrets**
```bash
wrangler secret put B2_KEY_ID
wrangler secret put B2_APPLICATION_KEY
wrangler secret put B2_BUCKET_NAME
wrangler secret put B2_DOWNLOAD_URL
```

Your `B2_DOWNLOAD_URL` is your public bucket URL:
```
https://<bucket-name>.s3.<region>.backblazeb2.com
```

**4. Worker logic**

The Worker handles two types of requests:

- **API routes** (`/auth`, `/upload`, `/ai-name`, etc.) — proxied to your backend logic with credentials
- **File routes** (everything else) — fetched from B2 and returned with long-term cache headers

```javascript
// Simplified Worker structure
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const path = url.pathname.slice(1); // remove leading /
    const firstSegment = path.split('/')[0];

    // Reserved API routes
    const apiRoutes = ['auth', 'upload', 'photos', 'publish', 'ai-name', 'ai-species', 'ai-describe'];
    
    if (request.method === 'GET' && !apiRoutes.includes(firstSegment)) {
      // File proxy with caching
      const b2Url = `${env.B2_DOWNLOAD_URL}/${path}`;
      const b2Response = await fetch(b2Url);
      
      if (b2Response.ok) {
        const response = new Response(b2Response.body, b2Response);
        response.headers.set('Cache-Control', 'public, max-age=31536000, immutable');
        response.headers.set('Access-Control-Allow-Origin', '*');
        return response;
      }
      
      return b2Response; // pass through errors unchanged
    }

    // Handle API routes here...
  }
};
```

**5. Deploy**
```bash
wrangler deploy
```

Your Worker will be live at `https://your-worker.your-subdomain.workers.dev`.

**6. Update your site**

Update all B2 file URLs in your site to point to your Worker instead of B2 directly:
```
Before: https://my-photos.s3.us-west-004.backblazeb2.com/birds/tern/photo.dzi
After:  https://your-worker.workers.dev/birds/tern/photo.dzi
```

---

## 11. OpenSeadragon Integration

[OpenSeadragon](https://openseadragon.github.io) is an open source deep zoom viewer. It reads your `.dzi` manifest and loads tiles on demand as the user zooms and pans.

### Add OpenSeadragon to Your Page

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/openseadragon/4.1.0/openseadragon.min.js"></script>
```

### Basic Viewer Setup

```html
<div id="viewer" style="width: 100%; height: 600px;"></div>

<script>
  const viewer = OpenSeadragon({
    id: 'viewer',
    prefixUrl: 'https://cdnjs.cloudflare.com/ajax/libs/openseadragon/4.1.0/images/',
    tileSources: 'https://your-worker.workers.dev/birds/forsters-tern/photo.dzi',
    showNavigator: true,
    navigatorPosition: 'BOTTOM_RIGHT',
    minZoomImageRatio: 0.5,
    maxZoomPixelRatio: 2
  });
</script>
```

### Pointing to Your B2 Tiles via the Worker

The `tileSources` URL should point to your Worker, not B2 directly. OpenSeadragon will fetch the `.dzi` manifest, parse the tile URL pattern, and request tiles through the same Worker URL pattern — all of which will be cached by Cloudflare after the first load.

### Multiple Images

```javascript
tileSources: [
  'https://your-worker.workers.dev/birds/tern/photo1.dzi',
  'https://your-worker.workers.dev/birds/tern/photo2.dzi'
]
```

---

## 12. Caching Architecture & B2 Transaction Costs

Understanding the cost model is important for keeping your site free (or very cheap) to run.

### Backblaze B2 Transaction Classes

| Class | Operations | Free Tier |
|-------|-----------|-----------|
| Class A | Uploads, bucket ops | Unlimited free |
| Class B | Downloads, file reads | 2,500/day free |
| Class C | Listing files | 2,500/day free |

Class B is the one to watch — every file read that reaches B2 counts. On a photo site with deep zoom tiles, a single page load without caching could trigger 50–300+ Class B transactions.

### How the Cache Eliminates This

The Worker returns this header on every successful B2 file response:

```
Cache-Control: public, max-age=31536000, immutable
```

| Value | Meaning |
|-------|---------|
| `public` | Cloudflare (and any CDN) may cache this response |
| `max-age=31536000` | Cache is valid for 1 year (in seconds) |
| `immutable` | The file will never change — never revalidate |

Once a file is cached at Cloudflare's edge, all subsequent requests for that file are served from cache. B2 is never contacted. For static photos that never change after upload, this is ideal — effectively infinite free serving after the first cache warm.

### B2 + Cloudflare Bandwidth Alliance

Backblaze and Cloudflare have a bandwidth partnership — there are **no egress fees** for data transferred from B2 to Cloudflare. This means your only real cost is storage ($0.006/GB/month after the free 10GB) and the rare cache-miss transaction.

### Monitoring Transactions

To check your transaction usage: **Backblaze → B2 Cloud Storage → Reports → Transaction Summary**

Watch for high `b2_download_file_by_name` counts — that's a signal that files are being requested without hitting the cache (typically during development or if cache headers aren't being set).

---

## 13. Folder Structure Reference

### Local (before upload)

```
export-folder/
├── 20250420_bolsa-chica_birds_forsters-tern_landing_100234_7117.jpg   ← renamed JPEG
├── 20250420_bolsa-chica_birds_forsters-tern_landing_100234_7117.dzi   ← tile manifest
├── 20250420_bolsa-chica_birds_forsters-tern_landing_100234_7117_files/← tile folder
│   ├── 0/  ← zoom level 0 (thumbnail)
│   ├── 1/
│   ├── ...
│   └── 13/ ← zoom level 13 (full resolution tiles)
```

### Backblaze B2 (after upload)

```
your-bucket/
├── birds/
│   └── forsters-tern/
│       ├── photo.dzi
│       └── photo_files/
│           └── [zoom level folders]/
├── mammals/
├── insects/
├── plants/
├── aquatic/
├── landscapes/
└── other/
```

### Your Site

```
your-site/
├── index.html
├── gallery.html
├── assets/
│   ├── css/
│   └── js/
└── [no large images — all served from B2 via Worker]
```

---

## 14. Cost Breakdown

For a typical hobby photo site:

| Service | Cost |
|---------|------|
| Cloudflare Pages | Free |
| Cloudflare Workers (up to 100k req/day) | Free |
| Cloudflare CDN & caching | Free |
| Backblaze B2 storage (first 10GB) | Free |
| Backblaze B2 storage (beyond 10GB) | $0.006/GB/month |
| Backblaze B2 Class B transactions (first 2,500/day) | Free |
| Backblaze B2 egress to Cloudflare | Free (bandwidth alliance) |
| Anthropic API for photo renaming | ~$0.01–0.05 per photo |
| Domain name | ~$10–15/year |

**For a site with ~500 photos:** roughly $0–$3/month depending on storage size, essentially free to serve due to caching.

---

## 15. Troubleshooting

### High B2 transaction counts
- Check that your Worker is setting `Cache-Control: public, max-age=31536000, immutable` on responses
- Verify your site is pointing to the Worker URL, not B2 directly
- During development, avoid repeatedly reloading pages with deep zoom active — each cache miss during dev burns transactions

### Deep zoom tiles not loading
- Check browser DevTools → Network tab for 404s or CORS errors
- Verify the `.dzi` path in `tileSources` matches the actual path in B2
- Make sure the Worker's file proxy route isn't accidentally catching a reserved API route name

### Worker returning 403 from B2
- B2 bucket must be set to **Public** for the Worker to fetch files without auth headers
- Verify your B2 Application Key has read access to the bucket

### libvips not found in PATH
- Make sure you added `C:\vips\bin` to your system PATH and restarted your terminal
- Run `vips --version` to confirm it's accessible

### Beanlemon Uploader not detecting tile pairs
- The `.dzi` file and `_files/` folder must share the same base name and be in the same folder
- Filenames must follow the Beantools naming format for category/species parsing

---

## Further Resources

- [Beantools on GitHub](https://github.com/duobound/beantools) — Renamer + Uploader source and releases
- [OpenSeadragon Documentation](https://openseadragon.github.io/docs/) — full viewer API reference
- [libvips Documentation](https://www.libvips.org/API/current/) — tile generation options
- [Backblaze B2 Documentation](https://www.backblaze.com/docs/cloud-storage) — bucket and key management
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/) — Worker development guide
- [Cloudflare Pages Documentation](https://developers.cloudflare.com/pages/) — static site deployment

---

*Guide based on the architecture developed for [Beanlemon Field Notes](https://beanlemon.com) — a nature photography journal by Bean & Lemon.*
