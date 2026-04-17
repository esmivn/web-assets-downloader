---
name: web-assets-downloader
description: Find, verify, and download official web-hosted assets such as product images from a brand or publisher website into the current project. Use when Codex must source files from a specific official site, extract direct asset URLs from page source or CDN references, avoid false matches, download the highest valid resolution available, and leave behind a traceable manifest of what was kept and why.
---

# Web Assets Downloader

Use this skill to collect official assets from a specific website with a bias toward accuracy over volume.

## Execution Contract

When this skill triggers, execute the workflow in order unless a step is truly not applicable.

- Do not silently skip workflow steps because the conversation is long or the task looks familiar.
- Treat steps 1 through 7 as required gates for a normal successful run.
- If a step is not applicable, say why in the manifest and final note.
- If a step is blocked, stop calling the task complete and report the blocker explicitly.
- Before the final answer, verify that every required gate below is either completed, marked not applicable with a reason, or blocked with a reason.

Required gates for a complete run:
- official source page identified
- candidate asset IDs or URLs extracted from the official source
- candidates narrowed against the user's criteria
- highest valid resolution probed for kept assets
- visual verification performed for final candidates when accuracy matters
- kept files saved under `asset/<criteria-slug>/`
- manifest written inside `asset/<criteria-slug>/`

Default outcome:
- Create a target directory at `asset/<criteria-slug>/` and place all outputs inside it.
- Download only assets that match the user's criteria.
- Prefer direct CDN/image-service URLs over screenshots or page thumbnails.
- Verify each candidate before saving it as final output.
- Record source URLs and exclusion reasons in a manifest.

## Workflow

### 1. Start from the official source

Begin with the exact site the user named. If they give a page, use it. If they only give a domain and a product name, find the official product page on that domain first.

Completion requirement:
- Record the exact source page URL in the manifest.

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

For client-rendered pages, do not rely only on the rendered DOM or a browser snapshot. Also inspect:
- raw HTML source
- inline JS assignments
- escaped JSON strings
- custom-element attributes
- bundled config blobs that contain asset IDs without obvious full URLs

Completion requirement:
- Capture the kept candidates and rejected candidates in working notes or directly in the manifest before finishing.

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
- If the page looks like a JS shell, treat escaped asset strings and seed-like identifiers as first-class evidence. Do not conclude that no gallery or spin exists just because the HTML lacks obvious `360` or `rotator` keywords.
- For likely frame sets, validate a few spread-out frames first, for example `000`, `010`, `020`, `035`, before downloading the whole sequence.
- After sampled frames succeed, probe the boundary frame just past the likely end, for example `036`, to infer the valid range before bulk download.
- Prefer patterns already hinted at by the site itself: nearby asset IDs, JS constants, JSON keys, CDN path segments, or viewer component names.
- Do not require the user to provide a guessed pattern manually when the official page exposes enough evidence to infer it.

Subaru-specific specialization:
- Treat `<trim>_<color>` as a reusable seed when visible anywhere in the page, inline JSON, or bundled JS.
- On Subaru build-and-price color pages, expect the visible page source to expose static derivatives such as `Subaru.trimScaledLowImages` or escaped `*_pass_scaled_low` values even when the page does not visibly expose a 360 viewer.
- Treat `*_pass_scaled_low` as evidence of a reusable exterior seed, not as the final answer.
- If you find `*_pass_scaled_low`, also probe likely exterior spin patterns such as `*_360e_000` through `*_360e_035`.
- Do not require the source to contain literal `360`, `spin`, or `rotator` strings before probing `*_360e_*` for Subaru when a verified `<trim>_<color>` seed exists.
- If one Subaru color in the requested trim has a valid `*_360e_*` sequence, probe the other requested colors from the same trim family before concluding coverage is partial.
- If the user asked for a trim or model generally rather than a single named paint, treat all exposed same-trim color-specific `*_360e_*` sequences as in scope and download each full verified sequence, not just the first color that succeeds.
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

Completion requirement:
- Every rejected candidate must have a short rejection reason in the manifest.

### 4. Probe the highest valid resolution

Many image services expose a default preview and allow a larger render with width parameters. Test a small set of widths until you find the highest accepted size. Stop when the service rejects the request or returns an error payload such as `illegal image size`.

Do not rely exclusively on `HEAD` requests for image services. If `HEAD` is blocked, inconsistent, or ambiguous, fall back to a real `GET`, save the payload temporarily, and verify the result with `file`, image dimensions, or visual inspection.

Useful pattern:

```bash
curl -L -A 'Mozilla/5.0' -o /tmp/candidate.jpg 'https://cdn.example.com/asset-id?wid=2000'
sips -g pixelWidth -g pixelHeight /tmp/candidate.jpg
file /tmp/candidate.jpg
```

If a larger width fails, keep the highest clean result that still returns a real image.

Completion requirement:
- For each kept file, record the final download URL or the resolution limit discovered.

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
- Record the inferred frame range and how it was established, for example sampled frames plus the first failing boundary frame.

Completion requirement:
- Do not present ambiguous files as final output without noting the ambiguity and why they were excluded or kept.

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

Completion requirement:
- Keep all final files and the manifest together in the same `asset/<criteria-slug>/` directory.

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

Completion requirement:
- Do not end the task without writing the manifest unless filesystem access is blocked. If blocked, say so explicitly.

## Quality Rules

- Favor correctness over quantity. If a candidate is ambiguous, do not keep it.
- Prefer the direct official asset over any transformed preview unless the preview is the only valid full-resolution source exposed by the site.
- Verify uniqueness when the user asks for multiple angles.
- Infer official sequence patterns from verified asset seeds when the naming convention is strong and supported by page or JS evidence.
- Keep interiors, crops, banners, and comparison graphics out of the final set unless the user asked for them.
- If the site's image service supports higher sizes only up to a limit, state that limit explicitly in the final answer.
- If network or sandbox restrictions block page fetches, retry with the appropriate approval flow instead of switching to an unofficial source.
- Do not compress the workflow into a best-effort answer. Either complete the gates or report the missing gate and blocker.

## Output Pattern

For a normal request, produce:
- one `asset/<criteria-slug>/` directory in the current repo/workdir
- downloaded files inside that directory
- one manifest inside that directory documenting source and filtering decisions
- a concise final note listing how many files were kept and any important exclusions or limits
- a gate status summary confirming completion, non-applicability, or blockers for steps 1 through 7

## Example Triggers

- `Find and download high quality 2025 Subaru WRX images from subaru.com.`
- `Pull official product photos for the Sony A7C II from sony.com and keep only full-camera angles.`
- `Download the highest-resolution hero and gallery images for the 2026 Kia EV9 from kia.com, but exclude interior shots.`

## Notes

- Use `rg` for source extraction and de-duplication work.
- Use parallel downloads when candidates are independent.
- Use `view_image` for final verification on high-risk candidates.
- Create extra helper files only when they materially improve repeatability; otherwise keep the skill lightweight.
