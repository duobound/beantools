# Backblaze B2 + Cloudflare Caching Architecture

## The Problem

Beanlemon Field Notes stores all photos and deep zoom tiles in a **Backblaze B2** bucket (`Beanlemon-photos`). Backblaze gives you 2,500 free **Class B transactions per day** — a Class B transaction is any file read/download request that hits B2 directly.

On a photo-heavy site with a tile viewer (OpenSeadragon), it's easy to burn through that cap fast:

- Each gallery image load = 1 Class B transaction
- Each deep zoom tile = 1 Class B transaction (a single deep zoom session can trigger 100–300+)
- Admin panel browsing = more transactions
- Every page refresh = repeats all of the above if nothing is cached

The cap is hit not from public traffic, but from your own dev/testing sessions.

---

## The Solution: Cloudflare Worker as a Caching Proxy

Instead of the browser fetching files from B2 directly, all requests are routed through a **Cloudflare Worker**. The Worker fetches from B2 once, then Cloudflare caches the response at the edge.

```
Browser → Cloudflare Worker → (first request only) → Backblaze B2
                ↓
         Cloudflare Edge Cache
                ↓
         All future requests served from cache (B2 never touched)
```

### How it works

1. The browser makes a GET request to the Worker URL with the B2 file path
2. The Worker checks if it's a known API route (auth, upload, AI endpoints, etc.)
3. If not — it's a file request — the Worker fetches it from B2 using the download URL
4. On a successful B2 response, the Worker returns the file with:

```
Cache-Control: public, max-age=31536000, immutable
```

5. Cloudflare caches this response at the edge for 1 year
6. Every subsequent request for the same file is served from Cloudflare's cache — **B2 is never contacted again**

---

## Why These Cache Headers?

| Header | Value | Meaning |
|--------|-------|---------|
| `Cache-Control` | `public` | Cloudflare (and any CDN) is allowed to cache this |
| `max-age` | `31536000` | Cache is valid for 1 year (31,536,000 seconds) |
| `immutable` | — | The file will never change, so never revalidate |

For a photo site with static images, these are ideal settings. Photos don't change after upload — so once cached, they can stay cached indefinitely.

---

## Worker Route Logic

The Worker distinguishes between API routes and file routes:

**Reserved API paths (bypass the cache proxy):**
- `auth` — authentication
- `photos` — photo listing
- `upload` — file uploads
- `publish`, `add-photos` — publishing
- `ai-name`, `ai-species`, `ai-describe` — AI endpoints
- `debug` — debugging

**Everything else** is treated as a B2 file path and proxied with cache headers.

---

## What This Fixes

| Before | After |
|--------|-------|
| Every image load = 1 B2 transaction | First load = 1 B2 transaction, all repeat loads = 0 |
| Gallery refresh = full transaction count again | Gallery refresh = served from Cloudflare cache |
| Deep zoom tiles = 100–300+ transactions per session | Deep zoom tiles cached after first view |
| 2,500/day cap easily hit during dev | Cap pressure dramatically reduced |

---

## Diagnosing Transaction Spikes

If you hit the cap again, check **Backblaze → B2 Cloud Storage → Reports → Transaction Summary**. Look for:

- `b2_download_file_by_name` — the main Class B transaction type; high counts mean cache is missing
- `b2_get_file_info` — metadata lookups; also Class B
- `b2_download_file_by_id` — rare, but also Class B

High `b2_download_file_by_name` counts after this fix would indicate requests are bypassing the Worker (going directly to B2 URLs) or cache headers aren't being set correctly.

To enable detailed logging going forward: **Backblaze → Bucket → Access Logs** (note: logs are stored as files in the bucket and count toward storage).

---

## Key Takeaway

Cloudflare's free tier is extremely generous with caching and edge serving. Once a file is cached, it costs nothing to serve — no B2 transactions, no bandwidth charges, near-zero latency. The Worker acts as a secure middleware layer that handles both API requests and file serving, with all static assets naturally benefiting from Cloudflare's global cache.

This pattern works for any static asset stored in B2: photos, thumbnails, deep zoom tile sets, or any other file that doesn't change after upload.
