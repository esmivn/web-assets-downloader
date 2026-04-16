---
name: web-assets-downloader
description: Find, verify, and download official web-hosted assets such as product images from a brand or publisher website into the current project. Use when Codex must source files from a specific official site, extract direct asset URLs from page source or CDN references, avoid false matches, download the highest valid resolution available, and leave behind a traceable manifest of what was kept and why.
---

# Web Assets Downloader

Use this skill to collect official assets from a specific website with a bias toward accuracy over volume.

Default outcome:
- Create a target directory at `asset/<criteria-slug>/` and place all outputs inside it.
- Download only assets that match the user's criteria.
- Prefer direct CDN/image-service URLs over screenshots or page thumbnails.
- Verify each candidate before saving it as final output.
- Record source URLs and exclusion reasons in a manifest.

## Workflow

### 1. Start from the official source

Begin with the exact site the user named. If they give a page, use it. If they only give a domain and a product name, find the official product page on that domain first.

Prefer:
- The exact product page on the official domain
- Official CDN/image-service assets referenced by that page
- First-party media pages over third-party mirrors

Avoid:
- Search-engine thumbnail URLs
- Dealer pages, forums, Pinterest, or press aggregators unless the user asked for those
- Downloading from a result before confirming it belongs to the target page/product

### 2. Extract candidate asset IDs and URLs

Pull the page HTML and search for:
- `og:image`
- direct image URLs
- CDN asset IDs
- JSON blobs that enumerate galleries, hero images, or feature images

Useful command patterns:

```bash
curl -L -A 'Mozilla/5.0' 'https://example.com/product-page' | head -n 200
curl -L -A 'Mozilla/5.0' 'https://example.com/product-page' | rg 'product|gallery|hero|feature' -o
curl -L -A 'Mozilla/5.0' 'https://example.com/product-page' | rg -o 'https://cdn.example.com/[^" )]+'
```

When the site uses an image service such as Scene7, Cloudinary, Imgix, or similar, identify the asset IDs first, then request the original service URL directly instead of downloading the page-rendered preview.

Do not stop at asset IDs that are explicitly listed in the page HTML. If the page exposes a reusable asset seed or a static derivative, automatically probe adjacent official naming patterns before concluding that only one image exists.

Generic pattern-inference rules:
- Treat the stable portion of a verified asset ID as a seed. Example: from `foo_bar_pass_scaled_low`, infer that `foo_bar` may be the reusable asset base.
- Look for derivative suffixes that imply variants or sequences, such as `hero`, `gallery`, `angle`, `spin`, `rotator`, `360`, `frame`, `thumb`, `scaled`, `low`, `med`, `high`, `crop`, `ext`, `int`.
- If the page or JS mentions rotators, 360 viewers, frame counts, indexes, or current-frame state, probe the corresponding asset family even if the full frame URLs are not listed inline.
- For likely frame sets, validate a few spread-out frames first, for example `000`, `010`, `020`, `035`, before downloading the whole sequence.
- Prefer patterns already hinted at by the site itself: nearby asset IDs, JS constants, JSON keys, CDN path segments, or viewer component names.
- Do not require the user to provide a guessed pattern manually when the official page exposes enough evidence to infer it.

Subaru-specific specialization:
- Treat `<trim>_<color>` as a reusable seed when visible anywhere in the page, inline JSON, or bundled JS.
- If you find `*_pass_scaled_low`, also probe likely exterior spin patterns such as `*_360e_000` through `*_360e_035`.
- If the page or JS references interior 360 features, also probe likely interior spin patterns such as `*_360i_000` through `*_360i_035` when the user asked for interiors.

### 3. Narrow candidates before downloading

Do not assume every page asset is relevant. Separate:
- full-product exterior shots
- interiors
- detail crops
- comparison tiles
- social banners
- logos and color chips

Use the user's criteria literally. Example:
- `"2025 Subaru WRX"` means the image must match the `2025 WRX`, not another WRX year or another Subaru model.
- `"other angles"` means prefer unique views and reject duplicates.
- `"high quality"` means probe for the largest valid official render, not just the default page size.

When a site exposes only one static angle in HTML but the asset naming or viewer code strongly suggests a rotator or frame sequence exists, test that sequence before declaring that the page has only one angle.

### 4. Probe the highest valid resolution

Many image services expose a default preview and allow a larger render with width parameters. Test a small set of widths until you find the highest accepted size. Stop when the service rejects the request or returns an error payload such as `illegal image size`.

Useful pattern:

```bash
curl -L -A 'Mozilla/5.0' -o /tmp/candidate.jpg 'https://cdn.example.com/asset-id?wid=2000'
sips -g pixelWidth -g pixelHeight /tmp/candidate.jpg
file /tmp/candidate.jpg
```

If a larger width fails, keep the highest clean result that still returns a real image.

### 5. Verify visually before promoting a file

Always inspect candidates with `view_image` before treating them as final output when accuracy matters.

Check:
- correct product/model/year cues when visible
- correct angle or shot type
- not an interior when the request is for exteriors
- not a tight crop when the request is for full-car/full-product images
- not a banner with large padding or a low-value social crop

If two asset IDs produce the same image, keep one and mark the other as a duplicate.

When probing a frame sequence:
- Confirm sampled frames are real images, not error payloads or placeholders.
- Compare checksums or visual samples to verify the frames are actually different angles.
- If the sequence is real, download the full valid range that matches the user's request.

### 6. Save with descriptive stable filenames

Before saving final files, create a directory under `asset/` derived from the user's criteria. Normalize it to a filesystem-safe slug and keep it human-readable.

Examples:
- `2025 Subaru WRX` -> `asset/2025-subaru-wrx/`
- `2025 Subaru WRX official exterior angles` -> `asset/2025-subaru-wrx-official-exterior-angles/`

Put all kept assets and the manifest inside that directory.

Use names that encode the subject and shot type, for example:
- `2025-subaru-wrx-official-angle-01-front-3q-drift-2000w.jpg`
- `2025-subaru-wrx-official-angle-06-side-profile-drift-2000w.jpg`

Include:
- subject
- whether it is official
- rough angle/shot label
- resolution class when useful

### 7. Write a manifest

Create a markdown manifest inside the `asset/<criteria-slug>/` output directory.

Include:
- source page
- each kept file
- source asset ID or direct URL
- download URL used
- excluded asset IDs and why they were rejected
- any resolution limits discovered during probing

This makes later reuse auditable and prevents re-checking the same dead ends.

## Quality Rules

- Favor correctness over quantity. If a candidate is ambiguous, do not keep it.
- Prefer the direct official asset over any transformed preview unless the preview is the only valid full-resolution source exposed by the site.
- Verify uniqueness when the user asks for multiple angles.
- Infer official sequence patterns from verified asset seeds when the naming convention is strong and supported by page or JS evidence.
- Keep interiors, crops, banners, and comparison graphics out of the final set unless the user asked for them.
- If the site's image service supports higher sizes only up to a limit, state that limit explicitly in the final answer.
- If network or sandbox restrictions block page fetches, retry with the appropriate approval flow instead of switching to an unofficial source.

## Output Pattern

For a normal request, produce:
- one `asset/<criteria-slug>/` directory in the current repo/workdir
- downloaded files inside that directory
- one manifest inside that directory documenting source and filtering decisions
- a concise final note listing how many files were kept and any important exclusions or limits

## Example Triggers

- `Find and download high quality 2025 Subaru WRX images from subaru.com.`
- `Pull official product photos for the Sony A7C II from sony.com and keep only full-camera angles.`
- `Download the highest-resolution hero and gallery images for the 2026 Kia EV9 from kia.com, but exclude interior shots.`

## Notes

- Use `rg` for source extraction and de-duplication work.
- Use parallel downloads when candidates are independent.
- Use `view_image` for final verification on high-risk candidates.
- Create extra helper files only when they materially improve repeatability; otherwise keep the skill lightweight.
