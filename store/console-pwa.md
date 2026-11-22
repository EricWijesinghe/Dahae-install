# Owner console: v1 distribution as a PWA

**Status: proposed.** The owner has not yet confirmed this choice in writing.
Decided by: pending owner sign-off. Date: pending.
The install steps and offline behaviour below are not yet verified on any device (see the verification record).

## Decision (proposed)

For v1 the owner console (`apps/console`) would ship as an installable web app (PWA) at `https://DOMAIN/admin/`.
It would not be published to the Microsoft Store, the Play Store or the App Store, and no desktop shell would be built.

Reasons:

- The console is used by the owner and a few staff on the same hosted origin. Store review adds a second release channel to maintain for a small audience.
- The web build already serves the console from the same API, so every fix reaches staff on the next load.
- The Windows MSIX script (`scripts/store/build-msix.ps1`) packages only the customer and partner apps (`$Apps` default). Signing a console MSIX is needed only if the owner asks for a Store listing later.
- The Android console keeps its signing block in `apps/console/android/app/build.gradle.kts`. Adding a `console-android` job to `store-apps.yml` and a listing is the upgrade path if a store app is chosen. It is not part of v1.

To switch to signed store apps later, add a `console` app to `.github/workflows/store-apps.yml` (mirroring `partner`), add a console listing to `docs/store`, and add `console` to the `Apps` set in `scripts/store/build-msix.ps1` if Windows signing is required.

### Audit item APPS-2

APPS-2 (Android admin panel with a store listing) is met for v1 only by the PWA install path above, if the owner confirms this decision. It stays open until an Android install is tested on a real device (case 8 and the install steps in the verification record). Do not close APPS-2 before then.

## Install steps (owner and staff)

The owner console is opened in the browser first, then installed as an app. The owner needs no terminal.

The button labels and menu items below are the expected browser wording. They are not yet confirmed on each browser; correct them after the verification run.

| Device | Steps |
|---|---|
| Windows PC, Edge | Open `https://DOMAIN/admin/` in **Edge**. Click the **앱 설치** icon (⊕) at the right end of the address bar, then **설치**. The app appears in the Start menu and the taskbar as "다해 관리자". |
| Windows PC, Chrome | Open `https://DOMAIN/admin/` in **Chrome**. Click the **앱 설치** icon at the right end of the address bar, then **설치**. |
| Windows PC, if no icon | Edge or Chrome menu (⋯ / ⋮) → **앱** or **앱 설치** → **이 사이트를 앱으로 설치**. |
| Android phone | Open the address in **Chrome**. Menu (⋮) → **앱 설치** (or **홈 화면에 추가**). |
| iPhone | Open the address in **Safari**. Share → **홈 화면에 추가**. (iOS runs it as a home-screen web app, without the install prompt.) |

The installed app reads its name, icons and colours from `app/apps/console/web/manifest.json` (`display: standalone`, `lang: ko`, `id: malkkeum-console`). This is checked against the file; it is not a device test.

## Offline-state test list

The console needs the network to sign in and to load bookings, payouts and settings. It has no local data store in `apps/console/lib`, so it does not keep a copy of the data. The tests below check that each state is shown clearly in Korean and that no screen hangs or loses a staff action. Run each case on Edge and Chrome (Windows) and on Chrome for Android, in both the installed app and a normal tab. Mark pass or fail with the date and the tester's role.

Code reading behind case 2 (not a device test): the shared `AsyncView` (`app/packages/core/lib/src/async_view.dart`) keeps the loaded data when a reload fails and shows a toast with the error text. The toast text comes from the `errorNetwork` string in `app/tool/strings_customer.py`.

1. **Not installed, no network at first load.** Opening `/admin/` with the network off shows the browser's offline page. It must not show a blank screen after the app starts.
2. **Installed app, network drops while on Bookings.** The list keeps the last loaded rows. A message shows `인터넷 연결을 확인해 주세요` (errorNetwork). Reconnecting and pressing the refresh control loads the current rows.
3. **Network drops during a write** (approve a booking, mark a payout, edit a setting). The action shows a failure message and nothing is shown as saved. Retrying after reconnecting succeeds once, with no duplicate booking or payout row.
4. **Network returns while the app is open.** Lists refresh on the next visit or refresh action. No stale "saved" state remains.
5. **Session expiry while offline.** After reconnecting, the app sends the user to sign-in instead of showing empty lists. Signing in again restores the previous screen.
6. **Slow network (throttled to 3G).** Loading indicators appear for every list. No screen shows zero bookings or zero payouts while still loading.
7. **Staff MFA and payout account screens** load only when online. Offline, the screen shows the network message and does not show partial bank or account data.
8. **Update after deploy.** A cached copy from before a deploy must not break the console. `/admin/` is served with `Cache-Control: no-cache` (`deploy/Caddyfile`), so a reload picks up the new build. Close and reopen the installed app once after each deploy and confirm it loads the new version. Note: the Flutter web build may still register a service worker; whether it caches the console shell is not verified and this case must confirm it.
9. **Uninstall.** Removing the app from the Start menu, Android launcher or iPhone home screen leaves the browser tab working. No data is kept on the device after sign-out.

## Verification record

**Status: not run.** No case above has been executed on any browser or device. Do not describe the install steps or the offline behaviour as verified until this record is filled in.

- Date:
- Tester role:
- Browser and version:
- Cases 1-9 pass or fail:
- Corrections to wording or behaviour found:
- Notes:
