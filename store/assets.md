# Store asset list · 스토어 이미지 목록

What each store asks for, and the file that answers it. Checked against the store help pages on 2026-11-22
(Play: support.google.com/googleplay/android-developer/answer/9866151, Apple: developer.apple.com/help/app-store-connect/reference/app-information/screenshot-specifications).

The images are generated, not committed (`.gitignore`): run `scripts/store/generate.sh` (default languages ko,en).
They show the brand saved in the console. **Regenerate after the final name, logo or colour is saved**, before upload.
All PNGs are 24-bit RGB with no alpha, which both stores require for screenshots and the feature graphic.

## Google Play (each app)

| Asset | Store rule | File | Count |
|---|---|---|---|
| App icon | 512 x 512, 32-bit PNG, max 1 MB | Console [Setup] brand, `icon_512` slot (also `{API_BASE}/v1/brand`) | 1 |
| Feature graphic | 1024 x 500, JPEG or 24-bit PNG, required | `graphics/customer-feature-<lang>.png`, `graphics/partner-feature-<lang>.png` | 1 per language |
| Phone screenshots | 2-8, sides 320-3840 px, long side at most 2.3 x short side | `screenshots/<app>/android/<lang>/*.png` (1080 x 2400, ratio 2.22) | customer 6, partner 4 |
| 7" / 10" tablet, Chromebook | Optional. The apps do not declare tablet layouts | none | 0 |
| Short / full description | 80 / 4000 characters | `listing-ko.md`, `listing-en.md` (checked by `check_listings.py`) | |

## App Store (each app, iPhone only: `TARGETED_DEVICE_FAMILY = 1`)

| Asset | Store rule | File | Count |
|---|---|---|---|
| App icon | 1024 x 1024, no alpha | Inside the build (`scripts/store_brand.py` writes it from the console brand) | 1 |
| iPhone 6.9" | 1320 x 2868 portrait, up to 10 | `screenshots/<app>/ios-6.9/<lang>/*.png` | customer 6, partner 4 |
| iPhone 6.5" | 1284 x 2778; only needed when 6.9" is missing | `screenshots/<app>/ios-6.5/<lang>/*.png` | spare |
| iPad 13" | Only when the app runs on iPad | `screenshots/<app>/ipad-13/` exists but is **not uploaded** (iPhone only) | 0 |
| Name / subtitle / keywords / promo text | 30 / 30 / 100 / 170 | `listing-ko.md`, `listing-en.md` | |

## URLs both stores ask for

| Field | Value |
|---|---|
| Privacy policy | `https://<domain>/privacy.html` |
| Account deletion (Play) | `https://<domain>/delete-account.html` |
| Support URL (Apple, required) | `https://<domain>/` (the home page shows the customer-centre phone and email from the console business profile) |
| Marketing URL (Apple, optional) | `https://<domain>/` |
