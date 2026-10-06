# Bitwarden browser extension: study guide

This folder is a personal study guide for the Bitwarden browser extension, written to prepare for a
software engineering interview. It is not official documentation.

It was checked against the code at commit `5a00c60` of this repository (the commits after it on this
branch only add these study files). Treat the code as the source of truth: if the repo has moved on,
re-check a path before relying on it.

Ground rules used while writing:

- Every feature below cites real paths, relative to the repo root. Each path was checked to exist.
- "Flag" means a server-controlled feature flag from `libs/common/src/enums/feature-flag.enum.ts`.
  Every flag mentioned here defaults to `false` in `DefaultFeatureFlagValue` in that file. The value
  a real user gets comes from the server config, which this repo cannot tell you.
- "Dev flag" means a build-time flag (`apps/browser/src/platform/flags.ts`,
  `libs/common/src/platform/misc/flags.ts`, `apps/browser/config/*.json`). These are separate from
  server feature flags. `apps/browser/webpack.base.js` injects `DEV_FLAGS` only when
  `ENV === "development"`, so dev-flag features are absent from production builds.
- Anything still unverified is listed at the end.

## Companion docs in this folder

| File | Topic |
| --- | --- |
| [`concepts.md`](./concepts.md) | Core concepts (background, content scripts, MV2 vs MV3, messaging, state) |
| [`vuln-cve-2018-25081-iframe-autofill.md`](./vuln-cve-2018-25081-iframe-autofill.md) | CVE-2018-25081, autofill into iframes. Related features: F2, F3, F4 |
| [`vuln-2025-dom-clickjacking.md`](./vuln-2025-dom-clickjacking.md) | 2025 DOM-based clickjacking of autofill UI. Related features: F11, F12, F13 |

---

## 1. How the extension is laid out

### Source tree

| Path | What lives there |
| --- | --- |
| `apps/browser/src/background/` | `main.background.ts` (builds every background service), `runtime.background.ts`, `commands.background.ts`, `idle.background.ts`, `nativeMessaging.background.ts` |
| `apps/browser/src/popup/` | Angular popup shell (files inside this folder): `main.ts`, `app.module.ts`, `app-routing.module.ts`, `services/services.module.ts` |
| `apps/browser/src/autofill/` | Autofill, inline menu, notifications, passkeys (`apps/browser/src/autofill/fido2/`), context menus, content scripts |
| `apps/browser/src/vault/`, `apps/browser/src/tools/`, `apps/browser/src/auth/`, `apps/browser/src/key-management/`, `apps/browser/src/billing/`, `apps/browser/src/dirt/`, `apps/browser/src/admin-console/` | Feature code, mostly popup screens plus some background services |
| `apps/browser/src/platform/` | Browser glue: storage, messaging, popout windows, badge, offscreen document, IPC, task scheduler |
| `apps/browser/src/safari/` | Xcode projects and Swift code for the Safari app extension |
| `bitwarden_license/bit-browser/` | The commercial build of the extension (adds the Health tab). Uses the same webpack base. |
| `libs/common`, `libs/angular`, `libs/vault`, `libs/auth`, `libs/key-management`, `libs/tools/*`, `libs/importer`, `libs/components` | Shared code consumed by the extension |

Two manifests exist: `apps/browser/src/manifest.json` (Manifest V2) and
`apps/browser/src/manifest.v3.json` (Manifest V3). `apps/browser/webpack/manifest.js` rewrites them per
browser: a key prefixed `__chrome__`, `__firefox__`, `__safari__`, `__opera__` or `__edge__` overrides
the plain key for that browser, keys for other browsers are dropped, and a `null` value deletes the
key.

MV2 vs MV3 is chosen by the `MANIFEST_VERSION` env var. In `apps/browser/webpack.base.js`
(`getEnv`), anything other than `3` means MV2. In `apps/browser/package.json`, the Chrome, Edge and
Opera build scripts set `MANIFEST_VERSION=3`; the Firefox and Safari build scripts do not, so they
build MV2 unless you add `MANIFEST_VERSION=3` (`build:watch:firefox:mv3`, `build:watch:safari:mv3`,
`dist:firefox:mv3`, `dist:safari:mv3`). CI follows the npm scripts:
`.github/workflows/build-browser.yml` ships Firefox from `dist:firefox` (MV2), labels the MV3 Firefox
artifact `DO-NOT-USE-FOR-PROD-dist-firefox-MV3`, and builds Safari with `dist:safari` (MV2). The Nx
targets in `apps/browser/project.json` differ: their `firefox` and `safari` configurations set
`MANIFEST_VERSION=3`, with separate `firefox-mv2` and `safari-mv2` configurations.

### Runtime contexts

| Context | What it is | Entry points |
| --- | --- | --- |
| Background | MV3: a service worker (`background.js`) that the browser can stop and restart; Firefox MV3 loads the same file as a background script (`__firefox__background.scripts`). MV2: a persistent background page (`background.html` + `background.js`). Owns all services and all decrypted-vault logic that content scripts need. | `apps/browser/src/platform/background.ts` creates `MainBackground` and calls `bootstrap()`; wiring is in `apps/browser/src/background/main.background.ts` |
| Popup (and popout, sidebar, tab, side panel) | Angular app. The same `popup/index.html` is shown in the toolbar popup, in a separate popout window (`?uilocation=popout`), in the Firefox/Opera sidebar (`?uilocation=sidebar`), in a full tab (`?uilocation=tab`, used by the new import picker), and in a Chrome side panel (`?uilocation=sidepanel`). | `apps/browser/src/popup/main.ts`, `apps/browser/src/popup/app.module.ts`, `apps/browser/src/popup/app-routing.module.ts` |
| Content scripts | Run inside web pages. Two are declared in the manifest; the rest are injected by the background on demand. | Manifest: `content/content-message-handler.js` (top frame only) and `content/trigger-autofill-script-injection.js` (all frames, plus `content/autofill.css`), both at `document_start` on `*://*/*` and `file:///*`, excluding `*.xml*` URLs. Injected: the bootstrap and autofiller scripts listed below. |
| Page-world script | `content/fido2-page-script.js` runs in the page's own JavaScript world so it can replace `navigator.credentials`. In MV3 it is registered with `world: "MAIN"`; in MV2 a small script appends it to the DOM. | `apps/browser/src/autofill/fido2/content/fido2-page-script.ts`, `fido2-page-script-delay-append.mv2.ts`, registration in `apps/browser/src/autofill/fido2/background/fido2.background.ts` |
| Extension iframes injected into pages | `notification/bar.html` (save/update password bar) and `overlay/menu.html` (inline autofill menu container, which embeds `overlay/menu-button.html` or `overlay/menu-list.html` in a `sandbox="allow-scripts"` iframe). All are listed in `web_accessible_resources`; the button and list pages are also manifest `sandbox` pages (except on Firefox, `__firefox__sandbox: null`). | `apps/browser/src/autofill/notification/bar.ts`, `apps/browser/src/autofill/overlay/inline-menu/pages/` |
| Offscreen document | MV3, not Firefox. A hidden page the service worker creates on demand for DOM-only work: clipboard read/write and `window.localStorage` access (the secondary copy of disk state). | `apps/browser/src/platform/offscreen-document/offscreen-document.ts`, `offscreen-document.service.ts` |
| Safari | Safari wraps the web extension in a macOS app. A Swift handler answers native messages (clipboard, popover, file download, biometrics, a timer workaround). | `apps/browser/src/safari/safari/SafariWebExtensionHandler.swift`, `apps/browser/src/browser/safariApp.ts` |

### Webpack entry names

Defined in `apps/browser/webpack.base.js` (`buildConfig`). `apps/browser/webpack.config.js` passes in the
OSS popup entry (`apps/browser/src/popup/main.ts`) and background entry (`apps/browser/src/platform/background.ts`).
`bitwarden_license/bit-browser/webpack.config.js` does the same with commercial entries
(`bitwarden_license/bit-browser/src/popup/main.ts`, `bitwarden_license/bit-browser/src/platform/background.ts`)
and aliases `@bitwarden/sdk-internal` to `@bitwarden/commercial-sdk-internal`.

Main config (all builds):

| Entry name | Source file |
| --- | --- |
| `popup/polyfills` | `apps/browser/src/popup/polyfills.ts` |
| `popup/main` | `apps/browser/src/popup/main.ts` |
| `content/trigger-autofill-script-injection` | `apps/browser/src/autofill/content/trigger-autofill-script-injection.ts` |
| `content/bootstrap-autofill` | `apps/browser/src/autofill/content/bootstrap-autofill.ts` |
| `content/bootstrap-autofill-overlay` | `apps/browser/src/autofill/content/bootstrap-autofill-overlay.ts` |
| `content/bootstrap-autofill-overlay-menu` | `apps/browser/src/autofill/content/bootstrap-autofill-overlay-menu.ts` |
| `content/bootstrap-autofill-overlay-notifications` | `apps/browser/src/autofill/content/bootstrap-autofill-overlay-notifications.ts` |
| `content/autofiller` | `apps/browser/src/autofill/content/autofiller.ts` |
| `content/auto-submit-login` | `apps/browser/src/autofill/content/auto-submit-login.ts` |
| `content/contextMenuHandler` | `apps/browser/src/autofill/content/context-menu-handler.ts` |
| `content/content-message-handler` | `apps/browser/src/autofill/content/content-message-handler.ts` |
| `content/fido2-content-script` | `apps/browser/src/autofill/fido2/content/fido2-content-script.ts` |
| `content/fido2-page-script` | `apps/browser/src/autofill/fido2/content/fido2-page-script.ts` |
| `content/ipc-content-script` | `apps/browser/src/platform/ipc/content/ipc-content-script.ts` |
| `notification/bar` | `apps/browser/src/autofill/notification/bootstrap-bar.ts` |
| `overlay/menu-button` | `.../overlay/inline-menu/pages/button/bootstrap-autofill-inline-menu-button.ts` |
| `overlay/menu-list` | `.../overlay/inline-menu/pages/list/bootstrap-autofill-inline-menu-list.ts` |
| `overlay/menu` | `.../overlay/inline-menu/pages/menu-container/bootstrap-autofill-inline-menu-container.ts` |
| `content/send-on-installed-message` | `apps/browser/src/vault/content/send-on-installed-message.ts` |
| `content/send-popup-open-message` | `apps/browser/src/vault/content/send-popup-open-message.ts` |

(`...` abbreviates `apps/browser/src/autofill`.) `content/autofill.css` is not an entry; it is copied
as-is by `CopyWebpackPlugin`, along with the manifest, `managed_schema.json`, `_locales` and images.

Added by build type:

| Build | Extra entry or config |
| --- | --- |
| MV2 | `background` (in the main config, as `params.background.entry`) and `content/fido2-page-script-delay-append-mv2`. Also emits `background.html` from `apps/browser/src/platform/background.html`, with a `vendor` chunk split out of the background's `node_modules`. |
| MV3 | A second webpack config named `background` builds `background.js` (target `webworker`, or `web` for Firefox; `ts-loader`; depends on the `main` config). |
| MV3, not Firefox | `offscreen-document/offscreen-document`, with `offscreen-document/index.html`. |
| MV3, Chrome only | `sidepanel-disabled.html`, generated from `apps/browser/src/sidepanel-disabled.html` with no scripts injected. The MV3 manifest's `__chrome__side_panel.default_path` points at it. |
| MV3, Safari | `apps/browser/src/safari/mv3/fake-background.html` and `fake-vendor.js` are copied to the output root as `background.html` and `vendor.js`, because the desktop Xcode project expects those files. |
| Chrome (any MV) | `core-js` polyfill imports are replaced with an empty module (`webpack/empty-module.js`); the comment relies on `minimum_chrome_version: "134.0"` in `manifest.v3.json`. |

HTML outputs from `HtmlWebpackPlugin` in every build: `popup/index.html`, `notification/bar.html`,
`overlay/menu-button.html`, `overlay/menu-list.html`, `overlay/menu.html` (plus the per-build pages above).

### Background boot order (worth knowing)

`MainBackground` in `apps/browser/src/background/main.background.ts` builds services in its constructor
(which ends by registering the system-notification click handlers, `initNotificationSubscriptions`)
and does startup work in `bootstrap()`. Order in `bootstrap()`: managed config read, SDK load,
migrations, token storage cleanup, auto-unlock for accounts that have an auto-unlock key, i18n and
event upload init, popup view-cache listeners, vault timeout init, FIDO2 init, autofill lifecycle
init, then `RuntimeBackground.init()` (which triggers script injection), then the autofill
orchestrator, notification, overlay-notification, commands and context-menu services, side panel
disabled globally, idle, web-request, sync listener, auto-submit and fill-assist rules services, a
switch to the next account if the active one is logged out, then the overlay and tabs backgrounds,
then IPC and shared unlock, the badge service, and finally (after a 500 ms timeout) a full sync,
background sync registration, server notifications, and the task and end-user notification
listeners. The code comment in `bootstrap()` explains that the lifecycle service must be wired before
runtime init so its `onConnect` listener exists before any frame connects.

---

## 2. Feature catalog

Use the numbers (F1, F2, ...) to ask for a deep dive. Each entry has: what the user sees, key code,
gating, and one interview angle.

### Index

| ID | Feature | Area |
| --- | --- | --- |
| F1 | Content-script injection and bootstrap variants | Autofill |
| F2 | Page scanning and fill-script generation | Autofill |
| F3 | Autofill on page load and the monitoring lifecycle | Autofill |
| F4 | Autofill from the popup (confirmation, reprompt) | Autofill |
| F5 | Card, identity and SSH key fill | Autofill |
| F6 | TOTP copy on autofill | Autofill |
| F7 | Auto-submit login (enterprise policy) | Autofill |
| F8 | HTTP basic-auth autofill | Autofill |
| F9 | Fill assist targeting rules and dev tools | Autofill |
| F10 | URI matching, blocked and excluded domains | Autofill |
| F11 | Inline autofill menu architecture | Inline menu |
| F12 | Inline menu tamper defenses | Inline menu |
| F13 | Inline menu contents, account creation, generator | Inline menu |
| F14 | Save and update password prompts | Notifications |
| F15 | Unlock prompt and locked-vault command retry | Notifications |
| F16 | Passkey interception and ceremony UI | Passkeys |
| F17 | WebAuthn Permissions Policy gate | Passkeys |
| F18 | Vault tab: list, search, filters, nudges | Vault |
| F19 | New vault list table and vault switcher | Vault |
| F20 | View, add, edit, clone items; TOTP QR capture | Vault |
| F21 | Folders, trash, archive, attachments, history, collections | Vault |
| F22 | Temporary item sharing | Vault |
| F23 | Credential generator and history | Tools |
| F24 | Send | Tools |
| F25 | Import and export | Tools |
| F26 | Login methods and registration | Auth |
| F27 | Two-factor, SSO results, vault-origin validation | Auth |
| F28 | Login with device and auth-request notifications | Auth |
| F29 | Account switcher, logout, process reload | Auth |
| F30 | Account security settings | Auth |
| F31 | Lock screen and unlock options | Unlock |
| F32 | Biometrics and shared unlock via desktop | Unlock |
| F33 | Vault timeout, idle and system lock | Unlock |
| F34 | At-risk passwords (security tasks) | Security health |
| F35 | Health report (commercial build) | Security health |
| F36 | Phishing detection | Security health |
| F37 | Keyboard shortcuts | Shortcuts and menus |
| F38 | Context menus | Shortcuts and menus |
| F39 | Toolbar badge and icon | Shortcuts and menus |
| F40 | Settings screens | Settings |
| F41 | Default password manager prompt | Settings |
| F42 | Sync | Sync and push |
| F43 | Server push and system notifications | Sync and push |
| F44 | Policies and managed configuration | Enterprise |
| F45 | Admin auto-confirm, premium, families screens | Enterprise |
| F46 | Event logging | Enterprise |
| F47 | State and storage | Platform |
| F48 | Offscreen document and clipboard | Platform |
| F49 | Popup shell: popouts, sidebar, side panel, caches | Platform |
| F50 | Page and web vault messaging channels | Platform |
| F51 | Safari app extension | Safari |
| F52 | Install, startup re-injection and welcome page | Platform |

---

### Autofill

**F1. Content-script injection and bootstrap variants**

- What the user sees: nothing directly. When a page loads, the extension decides which autofill code
  to load into it. A new page gets a smaller or larger script depending on whether the inline menu and
  the save/update prompts are turned on.
- Code:
  - `apps/browser/src/autofill/content/trigger-autofill-script-injection.ts` (manifest content script; sends `triggerAutofillScriptInjection`)
  - `apps/browser/src/background/runtime.background.ts` (handles that message)
  - `apps/browser/src/autofill/services/autofill.service.ts` (`injectAutofillScripts`, `getBootstrapAutofillContentScript`)
  - `apps/browser/src/platform/services/browser-script-injector.service.ts`
- How it works (from the code): `injectAutofillScripts` always injects one bootstrap bundle chosen from
  four (`bootstrap-autofill.js`, `bootstrap-autofill-overlay-notifications.js`,
  `bootstrap-autofill-overlay-menu.js`, `bootstrap-autofill-overlay.js`) by inline menu visibility and
  the two prompt settings; adds `autofiller.js` only on a page-load injection when the account is
  unlocked and autofill on page load is enabled; and always adds `contextMenuHandler.js`. It then calls
  `autofillLifecycleService.startMonitoringFrame` (F3). The injector skips the injection when the tab
  URL, or the target frame's URL, is on the blocked-domains list (F10). On every background start the
  same code re-injects into all open `http*` tabs and frames (F52); that path also injects
  `content-message-handler.js`.
- Gating: none; runs for logged-out users too (the injected scripts stay inert, see F3).
- Interview angle: why a tiny always-on trigger script plus on-demand injection, and what changes
  between MV2 (`tabs.executeScript`) and MV3 (`scripting.executeScript` in the isolated world, via
  `BrowserApi.executeScriptInTab`).

**F2. Page scanning and fill-script generation**

- What the user sees: a login form is detected and filled with the right username, password (and
  TOTP) fields, including on pages with shadow DOM or iframes.
- Code:
  - `apps/browser/src/autofill/services/collect-autofill-content.service.ts` (scan forms and fields)
  - `apps/browser/src/autofill/services/dom-query.service.ts`, `apps/browser/src/autofill/services/dom-element-visibility.service.ts`
  - `apps/browser/src/autofill/services/autofill.service.ts` (`generateFillScript`, `doAutoFill`)
  - `apps/browser/src/autofill/services/insert-autofill-content.service.ts` (runs the script, 20 ms between actions)
- How it works: the content script sends page details to the background; the background builds a
  "fill script" (a list of `fill_by_opid` style actions) using the decrypted cipher and sends it only
  to the frame that produced the page details (`tabSendMessage` with that `frameId`).
  `inUntrustedIframe` returns false when the frame URL equals the tab URL; otherwise it checks the
  frame URL against the cipher's URIs (with equivalent domains and the default match type). If it does
  not match, the script is marked `untrustedIframe`. A page-load fill is then refused; a user-triggered
  fill makes the content script show a `confirm()` warning first. The content script also refuses to
  fill inside a sandboxed iframe, and asks for confirmation before filling a password on an `http:`
  page when the saved URL is `https:` for the same host.
- Interview angle: the iframe trust decision is the security boundary of autofill (see
  `vuln-cve-2018-25081-iframe-autofill.md`). Also: what is sent to a content script, and what
  could a hostile page read.

**F3. Autofill on page load and the monitoring lifecycle**

- What the user sees: with the "autofill on page load" setting on, a matching login fills itself
  when the page opens. The setting is off by default (`autofillOnPageLoad$` defaults to `false` in
  `libs/common/src/autofill/services/autofill-settings.service.ts`).
- Code:
  - `apps/browser/src/autofill/background/autofill-orchestrator.ts` (serialized per-frame fill dispatch)
  - `apps/browser/src/autofill/services/autofill-lifecycle.service.ts` (`DefaultAutofillLifecycleService`: starts and stops content-script monitors, buffers page transitions)
  - `apps/browser/src/autofill/content/autofiller.ts` (reports page transitions, including SPA navigations)
  - `apps/browser/src/autofill/lifecycle.design.md`, `apps/browser/src/autofill/autofill.design.md`
- How it works (code at this commit): the lifecycle service tells a freshly injected frame to start
  monitoring if an account is logged in (locked or unlocked), broadcasts start on login, and broadcasts
  stop plus "disable autofiller" on logout; lock and unlock send nothing. The autofiller polls
  `location.href` every 500 ms, reports each change as `pageTransitionDetected`, and reports once more
  after 1.5 s if no fill arrived for that URL. A transition resolves once its frame is monitoring and is
  dropped if the frame disconnects first. The orchestrator queues fills per `(tab, frame)` so collect
  and fill never interleave, re-reads the frame's live URL and abandons the fill if it differs from the
  reported URL, checks the URL again after collecting page details, and (temporary guard, `FIXME
  (PM-39579)`) fills only if the tab is the active tab of the current window.
- Design versus code: the design docs describe "commit" and "cool-down" gates (monitor only a tab that
  has been active for a while, keep monitoring briefly after the user leaves) and an outcome-based
  retry rule. Neither is implemented at this commit: no code reads tab activation, and the orchestrator
  FIXME says "the tab gate replaces this". The design doc gives no interval values ("tuned constants"),
  and the code has none yet.
- Gating: user setting, re-checked at fill time; policy `PolicyType.ActivateAutofill` turns it on
  (`activateAutofillOnPageLoadFromPolicy$`, applied by `setAutoFillOnPageLoadOrgPolicy` in
  `apps/browser/src/autofill/services/autofill.service.ts` after unlock and after each sync).
- Interview angle: MV3 service workers lose in-memory state, so frame liveness is rebuilt by
  re-injecting after a restart; and "fill only the tab the user is looking at" as a security rule.

**F4. Autofill from the popup (confirmation and reprompt)**

- What the user sees: choosing autofill on an item in the popup fills the current tab. If a login's
  saved URLs do not match the page, a dialog lists the saved sites and offers "autofill only" or
  "autofill and add this URL" (the second is hidden when the user cannot edit the item). Items with
  master-password reprompt ask for the password first.
- Code:
  - `apps/browser/src/vault/popup/services/vault-popup-autofill.service.ts` (`doAutofill`, `doAutofillAndSave`)
  - `apps/browser/src/vault/popup/components/vault/item-more-options/item-more-options.component.ts` (`passwordRepromptCheck`, URL match, dialog)
  - `apps/browser/src/vault/popup/components/vault/autofill-confirmation-dialog/autofill-confirmation-dialog.component.ts`
- Reprompt: in the popup it is `PasswordRepromptService` from `libs/vault`. Fills started outside the
  popup (shortcut, context menu) use `isPasswordRepromptRequired` in
  `apps/browser/src/autofill/services/autofill.service.ts`, which opens a reprompt popout instead.
- Interview angle: user intent versus URL match; the dialog is a guard against filling a credential
  on the wrong site.

**F5. Card, identity and SSH key fill**

- What the user sees: cards and identities can be filled from the popup, the inline menu, the context
  menu, and the "autofill card" and "autofill identity" commands (no default key). SSH keys can be
  filled from the popup and the inline menu only.
- Code:
  - `apps/browser/src/autofill/services/autofill.service.ts` (`generateCardFillScript`, `generateIdentityFillScript`, `generateSshKeyFillScript`)
  - `apps/browser/src/autofill/background/autofill-orchestrator.ts` (`cipherType` requests)
  - `apps/browser/src/autofill/services/inline-menu-field-qualification.service.ts`
- Gating: policy `RestrictedItemTypes` (via `RestrictedItemTypesService`); the autofill settings screen
  and the context menu both check it (the context menu drops its card items when cards are restricted).
- Interview angle: field heuristics versus explicit rules; expiry-date format guessing is a good
  example of messy real-world forms.

**F6. TOTP copy on autofill**

- What the user sees: after a login fills, the one-time code is copied to the clipboard if the
  "copy TOTP automatically" setting is on (it defaults to on, `autoCopyTotp$`).
- Code:
  - `apps/browser/src/autofill/services/autofill.service.ts` (`doAutoFill`, `getShouldAutoCopyTotp`, `getTotpCopyCode`)
  - `apps/browser/src/autofill/background/autofill-orchestrator.ts` (`copyTotp` after page-load and shortcut fills)
  - `apps/browser/src/background/runtime.background.ts` (`autofillPage` copies the returned code)
- Gating: the cipher must have a TOTP secret and the user needs premium or an organization that has
  TOTP (`canUseTotp = canAccessPremium || cipher.organizationUseTotp`).
- Interview angle: clipboard exposure and the clear-clipboard timer (see F40, F48).

**F7. Auto-submit login (enterprise policy)**

- What the user sees: for organization-configured identity providers, the login form is filled and
  submitted without a click.
- Code:
  - `apps/browser/src/autofill/background/auto-submit-login.background.ts`
  - `apps/browser/src/autofill/content/auto-submit-login.ts`
- Gating: policy `PolicyType.AutomaticAppLogIn` whose data lists IdP hosts (`idpHost`,
  comma-separated). Listeners are only registered when the policy applies to the active user and the
  host list is non-empty.
- Interview angle: it uses `webRequest.onBeforeRequest` and `onBeforeRedirect` on main and sub
  frames to follow multi-step logins; why an explicit host allow-list matters.

**F8. HTTP basic-auth autofill**

- What the user sees: when a site shows the browser's own username/password dialog, Bitwarden answers
  it with a matching login.
- Code:
  - `apps/browser/src/autofill/background/web-request.background.ts`
  - `apps/browser/src/autofill/popup/settings/autofill.component.ts` (the toggle)
- Gating: flag `EnableBasicAuthResponse` AND a per-user setting; listeners are registered only while
  both are true and fail closed on error. The flag's comment in
  `libs/common/src/enums/feature-flag.enum.ts` says it "gates security risks and should not be turned
  on without changes to the underlying experience". Uses `webRequest.onAuthRequired` (`asyncBlocking`
  on Chrome, `blocking` on Firefox), which needs `webRequestAuthProvider` in MV3 and
  `webRequestBlocking` in MV2.
- How it matches: only while unlocked, and only when exactly one login matches the request URL with
  `UriMatchStrategy.Host`; otherwise it returns an empty response and the browser shows its dialog.
  Each request ID is answered once.
- Interview angle: a listener on every 401 is powerful; host-only matching and the "exactly one"
  rule are the safeguards.

**F9. Fill assist targeting rules and dev tools**

- What the user sees: an optional "fill assist" setting; on supported domains, fields are found with
  downloaded rules instead of heuristics. A banner says when rules are active on the tab.
- Code:
  - `apps/browser/src/autofill/services/targeting-rules-data.service.ts` (downloads and caches rules; manifest check, 5-minute retry, cached rules cleared on first failure)
  - `libs/common/src/autofill/services/domain-settings.service.ts` (`fillAssistPolicy$`, `effectiveFillAssistRulesUrl$`, `resolvedEnableFillAssist$`)
  - `apps/browser/src/autofill/services/autofill.service.ts` (`generateTargetedFillScript`)
  - `apps/browser/src/vault/popup/components/vault/fill-assist-active-banner/fill-assist-active-banner.component.ts`
  - Dev tools: `apps/browser/src/autofill/popup/autofill-tools/autofill-tools.guard.ts`, `apps/browser/src/autofill/popup/autofill-triage/autofill-triage.component.ts`, `apps/browser/src/autofill/webmapper/menu.ts`
- Gating: flag `FillAssistTargetingRules` plus the user's fill-assist setting (no fetch if either is
  off). Policy `PolicyType.FillAssist` turns fill assist on for users who never touched the setting and
  can supply an `https` rules URL; otherwise the URL comes from server config or defaults to
  `https://fillassist.bitwarden.com`. The triage and "webmapper" authoring tools exist only with the
  dev flag `fillAssistDevTools` (route `/autofill-triage`, which renders `AutofillToolsComponent`;
  context menu "Triage Autofill Issues"; opens in the Chrome side panel or a popout).
- Interview angle: remote rules change what code paths run, so they are treated as untrusted input;
  the targeted path still runs the iframe origin check (comment in `generateTargetedFillScript`).

**F10. URI matching, blocked and excluded domains**

- What the user sees: per-login URI match type (domain, host, starts with, exact, regex, never), a
  default match setting, "equivalent domains", a Blocked domains screen and an Excluded domains
  screen.
- Code:
  - `libs/common/src/models/domain/domain-service.ts` (`UriMatchStrategy`)
  - `apps/browser/src/autofill/popup/settings/blocked-domains.component.ts`, `apps/browser/src/autofill/popup/settings/excluded-domains.component.ts`
  - `apps/browser/src/platform/services/browser-script-injector.service.ts` (`findBlockedInjectionUrl` uses `blockedInteractionsUris$`)
  - `apps/browser/src/autofill/background/notification.background.ts` (`getExcludedDomains` uses `neverDomains$`)
- How it works: a blocked domain gets no autofill scripts at all (no autofill, no inline menu), and
  context-menu items marked `requiresUnblockedUri` are hidden there. An excluded domain is read by the
  notification code (never-save list).
- Gating: policy `UriMatchDefaults` locks the default match setting (`defaultUriMatchStrategyPolicy$`;
  `isDefaultUriMatchDisabledByPolicy` in `apps/browser/src/autofill/popup/settings/autofill.component.ts`).
- Interview angle: URL matching is the root of most autofill security bugs (subdomains, ports,
  `Never`).

### Inline autofill menu

**F11. Inline autofill menu architecture**

- What the user sees: a small Bitwarden icon inside username/password fields; clicking (or focusing,
  depending on the setting) opens a list of matching items.
- Code:
  - `apps/browser/src/autofill/services/autofill-overlay-content.service.ts` (watches fields, qualifies them)
  - `apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts` (creates the page elements)
  - `apps/browser/src/autofill/overlay/inline-menu/iframe-content/autofill-inline-menu-iframe.service.ts`
  - `apps/browser/src/autofill/background/overlay.background.ts` (state, ports, cipher list, positioning)
  - `apps/browser/src/autofill/overlay/inline-menu/pages/` (button, list and container pages)
- How it works: in the top frame only, the content service adds two custom elements (random tag
  names) holding iframes that load `overlay/menu.html`. That container page embeds the button or list
  page in an iframe with `sandbox="allow-scripts"`. The sandboxed page has an opaque origin and no
  extension APIs, so the container opens the named port to the background
  (`AutofillOverlayPort` in `apps/browser/src/autofill/enums/autofill-overlay.enum.ts`) and relays
  messages; the background adds a per-tab port key. The list is drawn in an extension-origin frame, so
  the host page's scripts cannot read it directly. Setting values are `Off`, `OnButtonClick`,
  `OnFieldFocus` (`AutofillOverlayVisibility` in `libs/common/src/autofill/constants/index.ts`); the
  stored default is `Off`, but a fresh install sets `OnFieldFocus` (F52).
- Gating: flag `LitInlineMenuComponents` switches the list page to Lit components
  (`useLitInlineMenuComponents$` / `useLitComponents` in `apps/browser/src/autofill/background/overlay.background.ts` and
  `apps/browser/src/autofill/overlay/inline-menu/pages/list/autofill-inline-menu-list.ts`; components under
  `apps/browser/src/autofill/content/components/`).
- Interview angle: why iframes plus sandbox plus ports, and how the page is kept from reading or
  driving the UI.

**F12. Inline menu tamper defenses**

- What the user sees: nothing, until a hostile page tries to hide, cover or restyle the menu; then the
  menu closes.
- Code:
  - `apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts` (mutation observers, opacity check via `getPageIsOpaque`, `verifyInlineMenuIsNotObscured` using `elementFromPoint`, top-layer and `popover` handling, refresh back-off counters)
  - `apps/browser/src/autofill/overlay/inline-menu/iframe-content/autofill-inline-menu-iframe.service.ts` (restores modified iframe attributes; iframe uses the `credentialless` attribute)
  - `apps/browser/src/autofill/utils/event-security.ts` (`isEventTrusted`, used by the menu pages, the Lit components and the message handler)
  - `apps/browser/src/autofill/overlay/inline-menu/pages/menu-container/autofill-inline-menu-container.ts` (session token and `contentWindow` identity checks on messages, extension-origin check on the iframe URL, and an allow-list `ALLOWED_BG_COMMANDS` for what reaches the background)
- Interview angle: the menu lives in the attacker's DOM, so every click path must assume the page is
  hostile (see `vuln-2025-dom-clickjacking.md`). Study the trusted-event check and the obscuring checks.

**F13. Inline menu contents, account creation and generator**

- What the user sees: logins for the site, plus cards, identities and SSH keys when those inline
  settings are on; TOTP codes with a countdown; passkeys; an unlock button when locked; "add new item";
  on sign-up forms, a generated password and a "save login" option.
- Code:
  - `apps/browser/src/autofill/background/overlay.background.ts` (`getInlineMenuCipherData`, passkey list, `fillGeneratedPassword`, `addNewVaultItem`)
  - `apps/browser/src/autofill/content/components/inline-menu/` (Lit components: `apps/browser/src/autofill/content/components/inline-menu/cipher-list.ts`, `apps/browser/src/autofill/content/components/inline-menu/password-generator.ts`, `apps/browser/src/autofill/content/components/inline-menu/totp-cipher-info.ts`, `apps/browser/src/autofill/content/components/inline-menu/prompt.ts`)
  - `apps/browser/src/autofill/enums/autofill-overlay.enum.ts` (`InlineMenuFillTypes`: `AccountCreationUsername`, `PasswordGeneration`, `CurrentPasswordUpdate`)
  - `apps/browser/src/autofill/services/inline-menu-field-qualification.service.ts`
- How it works: the background uses the credential generator service to produce a password and
  records it in generator history (`trackGeneratedCredential` in
  `apps/browser/src/autofill/utils/credential-history-utils.ts`). Cards, identities and SSH keys are
  fetched regardless of URL match (`getAllCipherTypeViews`).
- Interview angle: field qualification (is this a login, sign-up, card or identity form?) decides
  which UI appears, and mistakes are visible to users.

### Notifications

**F14. Save and update password prompts**

- What the user sees: after submitting a login form, a bar appears in the page offering
  to save the login or update a changed password, with folder, collection and vault choices and a
  "never" option.
- Code:
  - `apps/browser/src/autofill/background/overlay-notifications.background.ts` (detects submits via `webRequest` POST/PUT/PATCH plus content-script `formFieldSubmitted`; picks the scenario)
  - `apps/browser/src/autofill/background/notification.background.ts` (queue, save, update, folder and collection data)
  - `apps/browser/src/autofill/overlay/notifications/content/overlay-notifications-content.service.ts` (creates the `credentialless` iframe to `notification/bar.html` with a `parentOrigin` parameter)
  - `apps/browser/src/autofill/notification/bar.ts`
- Gating: user settings for "ask to add login" and "ask to update password"; excluded domains
  (F10); flag `UseUndeterminedCipherScenarioTriggeringLogic` switches the add-versus-change decision
  to a single "cipher" scenario; the `OrganizationDataOwnership` policy removes the individual vault
  choice (`removeIndividualVault` in `apps/browser/src/autofill/background/notification.background.ts`).
- Interview angle: capturing the password the user typed is sensitive: it is kept in memory and
  cleared after `CLEAR_NOTIFICATION_LOGIN_DATA_DURATION` (60 s, `libs/common/src/autofill/constants/index.ts`);
  the bar is an extension page inside a page the site controls.

**F15. Unlock prompt and locked-vault command retry**

- What the user sees: pressing the autofill shortcut while locked opens an unlock popout; after
  unlocking, the original autofill command runs on the original tab.
- Code:
  - `apps/browser/src/background/commands.background.ts` (`triggerAutofillCommand`, `handleUnlockCompleted`)
  - `apps/browser/src/background/runtime.background.ts` (`ADD_TO_LOCKED_VAULT_PENDING_NOTIFICATIONS`, `loggedIn`/`unlocked` handling)
  - `apps/browser/src/autofill/background/abstractions/notification.background.ts` (retry constants)
  - `apps/browser/src/auth/popup/utils/auth-popout-window.ts` (`openUnlockPopout`)
- How it works: both the enqueue (`ADD_TO_LOCKED_VAULT_PENDING_NOTIFICATIONS`) and the replay
  (`RETRY_WHEN_UNLOCK_COMPLETED`) go through the in-process `IntraprocessMessageSender`; a copy that
  arrived as an external runtime message is dropped (`isExternalMessage`). Replayed commands must
  carry a sender tab (`RETRY_SENDER`) and cannot fall back to the active tab.
- Interview angle: a queued action that survives an unlock is a confused-deputy risk; study how the
  code ensures only the background can author the queue entry.

### Passkeys

**F16. Passkey interception and ceremony UI**

- What the user sees: on a site that asks to create or use a passkey, Bitwarden offers to save a new
  passkey or pick an existing one, in a popout (user verification is requested when needed).
- Code:
  - `apps/browser/src/autofill/fido2/content/fido2-page-script.ts` (replaces `navigator.credentials.create/get` in the page world)
  - `apps/browser/src/autofill/fido2/content/fido2-content-script.ts` (bridge via a DOM messenger; runs only on `text/html` documents served over HTTPS or from `http://localhost`, and not in sandboxed iframes)
  - `apps/browser/src/autofill/fido2/background/fido2.background.ts` (registers scripts, handles requests, per-tab active-request tracking)
  - `libs/common/src/platform/services/fido2/fido2-client.service.ts`, `libs/common/src/platform/services/fido2/fido2-authenticator.service.ts` (the WebAuthn client and authenticator logic, shared with other clients)
  - `apps/browser/src/autofill/fido2/services/browser-fido2-user-interface.service.ts`, `apps/browser/src/autofill/popup/fido2/fido2.component.ts` (popout UI; route `/fido2`)
  - `apps/browser/src/vault/services/fido2-user-verification.service.ts`
- Gating: user setting `enablePasskeys` (notifications settings screen); scripts register only for
  logged-in users. Registration uses `https://*/*` and `http://localhost/*`, all frames, excluding
  `https://*/*.xml*`; in MV3 the page script is registered with `world: "MAIN"`. The client validates
  the RP ID against the origin (`isValidRpId` in `libs/common/src/platform/services/fido2/domain-utils.ts`).
- Interview angle: hooking a browser API from the page world (can the page tamper with it?), and
  keeping the extension context out of reach of the page.

**F17. WebAuthn Permissions Policy gate**

- What the user sees: a passkey request from a cross-origin iframe is allowed or blocked the way the
  browser's own WebAuthn would be (based on the `Permissions-Policy` header and the iframe `allow`
  attribute).
- Code:
  - `apps/browser/src/autofill/fido2/background/permissions-policy/permissions-policy.background.ts`
  - `.../permissions-policy-header-cache.background.ts`, `apps/browser/src/autofill/fido2/background/permissions-policy/iframe-allow-cache.background.ts`
  - `.../webauthn-permissions-policy.background.ts`, `apps/browser/src/autofill/fido2/background/permissions-policy/permissions-policy-parser.ts`
  - `apps/browser/src/autofill/fido2/content/iframe-allow-reporter.ts`, `apps/browser/src/autofill/fido2/content/iframe-allow-scraper.ts`
  - `apps/browser/src/autofill/fido2/background/fido2.background.ts` (`enforcePermissionsPolicyGate`, called for both create and get)
- (`...` abbreviates `apps/browser/src/autofill/fido2/background/permissions-policy`.)
- Interview angle: the extension re-implements a browser rule because it replaced the API; a gap
  here would let embedded third-party frames request passkeys.

### Vault

**F18. Vault tab: list, search, filters, nudges**

- What the user sees: the main screen with items that match the current tab at the top, favorites, all
  items, a search box, type/folder/collection filters, and one-time tips ("spotlights") for new users.
- Code:
  - `apps/browser/src/vault/popup/components/vault/vault.component.ts`
  - `apps/browser/src/vault/popup/services/vault-popup-items.service.ts`, `apps/browser/src/vault/popup/services/vault-popup-list-filters.service.ts`
  - `apps/browser/src/vault/popup/components/vault/autofill-vault-list-items/`, `apps/browser/src/vault/popup/components/vault/vault-list-items-container/`, `apps/browser/src/vault/popup/components/vault/vault-list-filters/`
  - `apps/browser/src/vault/popup/components/vault/intro-carousel/intro-carousel.component.ts`, `apps/browser/src/vault/popup/guards/intro-carousel.guard.ts`
  - `apps/browser/src/vault/popup/services/browser-autofill-nudge.service.ts`
- Notes: the blocked-injection banner (`apps/browser/src/vault/popup/components/vault/blocked-injection-banner/`) and the fill-assist banner show on
  this screen. The autofill nudge is dismissed if the account is older than 30 days (inherited
  `NewAccountNudgeService`) or the browser's own password manager is already turned off.
- Interview angle: the list is built from reactive streams over decrypted ciphers; popup state is
  cached across popup closes (F49).

**F19. New vault list table and vault switcher**

- What the user sees: with the flag on, a different vault screen: a table-style list with its own
  search toolbar and section grouping, and a switcher to scope the list (all items, My Vault, an
  organization). A route `/tabs/vault/:vaultId` exists for the scoped view.
- Code:
  - `apps/browser/src/vault/popup/components/vault/vault-popup-list-table/vault-popup-list-table.component.ts`
  - `apps/browser/src/vault/popup/components/vault/vault-switcher/vault-switcher.component.ts`
  - `apps/browser/src/popup/app-routing.module.ts` (`vault/:vaultId` with `vaultScopeGuard` from `libs/vault/src/routing/vault-scope.guard.ts`; the route itself is not flag-guarded in the extension)
  - `apps/browser/src/vault/popup/guards/clear-vault-state.guard.ts` (with the flag on, search and filter state persists across navigation)
- Gating: flag `VFO1Foundation` (the template renders the old header and list in the `@else`
  branches). In the extension it also changes the popup header, pop-out button, account switcher,
  at-risk callout, Send list, archive, folders and trash screens.
- What VFO1 is: a cross-client vault redesign flag (also used heavily in `apps/web`, `apps/desktop`
  and `libs/vault`). Besides layout, it renames "collections" to "shared folders"
  (`libs/vault/src/services/vfo1-terminology.service.ts` maps `bwi-collection` icons to
  `bwi-shared-folder`; the web locale file has "VFO1 terminology variant" strings) and moves vault
  filters into namespaced URL query params (`libs/vault/src/routing/README.md`). The code never
  expands the acronym.
- Interview angle: how a risky UI rewrite ships behind a flag with the old path kept alive.

**F20. View, add, edit, clone items; item types; TOTP QR capture**

- What the user sees: full item screens for logins, secure notes, cards, identities and SSH keys,
  with copy buttons, custom fields, URIs and password history. When adding a login, a "scan QR code"
  action can read a TOTP secret from the visible tab.
- Code:
  - `apps/browser/src/vault/popup/components/vault/add-edit/add-edit.component.ts` (uses `CipherFormComponent` from `@bitwarden/vault`, `libs/vault`)
  - `apps/browser/src/vault/popup/components/vault/view/view.component.ts`
  - `apps/browser/src/vault/popup/services/browser-totp-capture.service.ts` (`captureVisibleTab` then `qrcode-parser`, accepts only `otpauth:` URLs with a `secret`)
  - `apps/browser/src/vault/popup/components/vault/new-item-page/new-item-page.component.ts`, `apps/browser/src/vault/popup/components/vault/new-item-dropdown/`
  - `libs/common/src/vault/enums/cipher-type.ts`, `libs/common/src/vault/types/cipher-menu-items.ts` (`DIALOG_CIPHER_MENU_ITEMS`)
- Gating: flag `PM32009NewItemTypes` turns on a new-item page and the extra types BankAccount,
  DriversLicense and Passport (the types are in `cipher-type.ts`; the flag adds them to the menus via
  `DIALOG_CIPHER_MENU_ITEMS` in `cipher-menu-items.ts`); route `/new-item` is guarded by it. The
  TOTP capture is not offered inside a popout (`canCaptureTotp`).
- Interview angle: shared form logic in `libs/vault` versus per-client glue; `captureVisibleTab` needs
  `<all_urls>` or `activeTab`, and the MV3 manifest lists host patterns, not `<all_urls>`.

**F21. Folders, trash, archive, attachments, history, collections**

- What the user sees: manage folders, restore or purge deleted items, archive items (premium),
  attach files, read previous passwords, and assign an item to collections.
- Code:
  - `apps/browser/src/vault/popup/settings/folders.component.ts`, `apps/browser/src/vault/popup/settings/trash.component.ts`, `apps/browser/src/vault/popup/settings/archive.component.ts`
  - `apps/browser/src/vault/popup/components/vault/attachments/attachments.component.ts`
  - `apps/browser/src/vault/popup/components/vault/vault-password-history/vault-password-history.component.ts`
  - `apps/browser/src/vault/popup/components/vault/assign-collections/assign-collections.component.ts`
- Notes: archive uses `CipherArchiveService.userCanArchive$`, which is "premium from any source"
  (`libs/common/src/vault/services/default-cipher-archive.service.ts`). Without it, the settings
  screen shows the Archive entry with a premium badge, and clicking it opens an upgrade prompt unless
  the user already has archived items (`apps/browser/src/vault/popup/settings/vault-settings.component.ts`). The
  attachments route uses `filePickerPopoutGuard` (F49).
- Interview angle: each screen is thin; the real logic is in `libs/common/src/vault`.

**F22. Temporary item sharing**

- What the user sees: a "share item" page opened from an item: choose recipient emails, an expiry in
  hours and an optional one-time view; get a link; list and delete active links.
- Code:
  - `apps/browser/src/tools/popup/share/share-item.component.ts`
  - `apps/browser/src/tools/popup/share/browser-share-item.presenter.ts` (navigates to `/share-item?cipherId=...` instead of a drawer)
  - `libs/tools/share/src/services/share-link.service.ts`
- How it works: a share link is a Send of the new type `SendType.Item`. `createShareLink` copies the
  decrypted cipher, strips attachments, passkeys, password history and the cipher key, turns reprompt
  off, and creates the Send through `SendSdkApiService.mutateSend` with `AuthType.Email` (only the
  listed emails can open it), deletion and expiration dates, and `maxAccessCount = 1` for one-time
  shares. The link is the Send URL plus `accessId` and the URL-safe base64 Send key; for the US cloud
  the Send URL ends in `/#`, so the key is in the fragment.
- Gating: route guard `canAccessFeature(PM34203TemporaryItemSharing)`. `cipherCanBeShared$` also
  requires premium, rejects archived, deleted and SSH key items, rejects users under a `SendControls`
  policy that disallows item Sends, requires email auth to be allowed, or disables Send, and requires
  edit access to one of the item's collections (org admins who can edit all items are exempt).
- Interview angle: the item is re-encrypted under a Send key that travels in the link, the same model
  as Send; the server sees only ciphertext plus access rules.

### Tools

**F23. Credential generator and history**

- What the user sees: a Generator tab with options and a history screen. The generator library's
  credential types are password, username and email (`libs/tools/generator/core/src/metadata/type.ts`);
  passphrase is one of the password algorithms and email includes forwarding services. The generator
  is also reachable from inside the add/edit form and the inline menu.
- Code:
  - `apps/browser/src/tools/popup/generator/credential-generator.component.ts`, `apps/browser/src/tools/popup/generator/credential-generator-history.component.ts`
  - `libs/tools/generator/` (components and core; imported as `@bitwarden/generator-components`)
  - `libs/tools/generator/extensions/history/src/local-generator-history.service.ts`, `libs/tools/generator/extensions/history/src/key-definitions.ts`
  - `apps/browser/src/vault/popup/services/browser-cipher-form-generation.service.ts`
  - `apps/browser/src/background/main.background.ts` (`generatePasswordToClipboard`)
- History storage: `LocalGeneratorHistoryService` keeps up to 200 entries per user, newest first,
  skipping duplicates. It is a `SecretState` under state key `localGeneratorHistory` on the
  `GENERATOR_DISK` definition: on disk, encrypted with the user key (`UserKeyEncryptor`, padded to
  2048-byte frames), cleared on logout. Old password history is migrated through a buffered key.
- Gating: org `PasswordGenerator` policies are applied by the shared library
  (`libs/tools/generator/core/src/policies/`). `generatePasswordToClipboard` returns nothing until
  `initOverlayAndTabsBackground()` has run, which happens only for authenticated users.
- Interview angle: the popup tab, the inline menu and the keyboard shortcut all go through
  `CredentialGeneratorService`, and each result is recorded in generator history
  (`trackGeneratedCredential`, called from `main.background.ts` and `overlay.background.ts`).

**F24. Send**

- What the user sees: a Send tab to create text or file Sends, copy the link, and manage them.
- Code:
  - `apps/browser/src/tools/popup/send-v2/send-v2.component.ts`, `apps/browser/src/tools/popup/send-v2/add-edit/send-add-edit.component.ts`, `apps/browser/src/tools/popup/send-v2/send-created/send-created.component.ts`
  - `libs/tools/send/send-ui/src/services/send-policy.service.ts`
  - `apps/browser/src/tools/popup/guards/file-picker-popout.guard.ts`
- Gating: policies `DisableSend` and `SendOptions`; with flag `SendControls` the `SendControls`
  policy is checked as well (in `libs/tools/send/send-ui/src/services/send-policy.service.ts`). The
  Send list screen also reads `VFO1Foundation`.
- Interview angle: file Sends in a popup can crash or close the popup on some browsers, so they are
  forced into a popout (F49); text Sends never are.

**F25. Import and export**

- What the user sees: Settings > Vault options > Import / Export. Import takes a file or a
  supported cloud source (Keeper and LastPass flows exist in the code); Export writes a vault file.
- Code:
  - `apps/browser/src/tools/popup/settings/import/import-browser-v2.component.ts` (wraps `ImportComponent` from `libs/importer`)
  - `apps/browser/src/tools/popup/settings/import/import-source-select-browser.component.ts`, `apps/browser/src/tools/popup/settings/import/import-upgrade-navigation.service.ts`
  - `apps/browser/src/tools/popup/settings/import/browser-keeper-sso-tab-monitor.ts` (waits for an SSO callback tab and reads its body text and HTML with `executeFunctionInTab` to extract a token)
  - `apps/browser/src/tools/popup/settings/export/export-browser-v2.component.ts` (wraps `ExportComponent`, `libs/tools/export-vault-ui`)
  - `apps/browser/src/background/runtime.background.ts` (`authResult` with `lastpass` leads to `importCallbackLastPass`)
- Gating: flag `ImportUpgrade` swaps the in-popup import page for a new source picker opened in its
  own extension tab (`popup/index.html?uilocation=tab#/import-source-select`;
  `apps/browser/src/tools/popup/guards/import-upgrade-redirect.guard.ts`,
  `apps/browser/src/tools/popup/guards/import-upgrade-required.guard.ts`).
- Interview angle: import runs parsing in the popup; the importer library has many format parsers.
  Cloud-import SSO callbacks are another place where a tab's content is trusted.

### Authentication and accounts

**F26. Login methods and registration**

- What the user sees: email and master password login, SSO, log in with passkey, log in with device,
  admin approval, password hint, new-device verification, sign-up, set initial password, TDE
  decryption options, key connector confirmation and "remove password", an environment selector for
  self-hosted servers.
- Code:
  - `apps/browser/src/popup/app-routing.module.ts` (all the auth routes and their guards)
  - `libs/auth/src/angular/` (login, SSO, two-factor, registration components; imported as `@bitwarden/auth/angular`)
  - `apps/browser/src/auth/popup/login/extension-login-component.service.ts`, `apps/browser/src/auth/popup/login/extension-sso-component.service.ts`, `apps/browser/src/auth/popup/login/extension-login-via-webauthn-component.service.ts`
  - `apps/browser/src/auth/services/new-device-verification/extension-new-device-verification-component.service.ts`
- Gating: login with passkey opens in a popout on Linux (`platformPopoutGuard(["linux"])`); the login
  route also runs `DefaultPasswordManagerPromptGuard` (F41) and `IntroCarouselGuard` (F18); the
  settings password page is flagged `PM32413_MultiClientPasswordManagement` (F30).
- Interview angle: the extension supplies thin per-client "component services" to shared Angular auth
  components in `libs/auth`.

**F27. Two-factor, SSO results, vault-origin validation**

- What the user sees: two-factor steps (authenticator code, email, Duo, WebAuthn/security key, etc.).
  Duo, WebAuthn and SSO open web pages that hand the result back to the extension.
- Code:
  - `apps/browser/src/auth/services/extension-two-factor-auth-duo-component.service.ts`, `apps/browser/src/auth/services/extension-two-factor-auth-webauthn-component.service.ts`, `apps/browser/src/auth/services/extension-two-factor-auth-component.service.ts`
  - `apps/browser/src/autofill/content/content-message-handler.ts` (receives `window.postMessage` from the web vault page: `authResult`, `webAuthnResult`, `duoResult`)
  - `apps/browser/src/platform/utils/valid-vault-referrer.ts` (`isValidVaultReferrer`)
  - `apps/browser/src/background/runtime.background.ts` (`authResult`, `webAuthnResult`)
- How it works: the content script rejects untrusted (synthetic) events and events whose source is not
  its own window, and sends the hostname of `event.origin` as `referrer` (`"null"` for opaque
  origins). The background (for SSO and WebAuthn results) and the Duo service (for Duo results) only
  accept a result when that hostname matches a known region's vault URL or the configured web vault
  URL; the background then closes the SSO tab.
- Interview angle: a web page sending messages to a privileged extension; every hop validates origin.
  The validator's own comment notes the allow-list is client-side and can miss a self-hosted vault
  whose configured hostname differs.

**F28. Login with device and auth-request notifications**

- What the user sees: approve a login request from another device. When a request arrives and the
  popup is open, the approval dialog opens; otherwise a system notification is shown, and clicking it
  opens the popup (`ExtensionAuthRequestAnsweringService`).
- Code:
  - `apps/browser/src/auth/services/auth-request-answering/extension-auth-request-answering.service.ts`
  - `apps/browser/src/platform/system-notifications/browser-system-notification.service.ts`
  - `apps/browser/src/background/main.background.ts` (`initNotificationSubscriptions`, called synchronously at the end of the `MainBackground` constructor, routes clicks by notification ID prefix)
- Gating: `notifications` permission (absent from the Safari permission lists); no feature flag found
  in the extension code.
- Interview angle: MV3 service workers must register event listeners synchronously at start-up, or
  events are lost.

**F29. Account switcher, logout, process reload**

- What the user sees: a switcher listing up to five accounts, add account, lock and log out; after
  logout, the extension reloads itself.
- Code:
  - `apps/browser/src/auth/popup/account-switching/account-switcher.component.ts`, `apps/browser/src/auth/popup/account-switching/services/account-switcher.service.ts` (`ACCOUNT_LIMIT = 5`)
  - `apps/browser/src/background/main.background.ts` (`switchAccount`, `logout`)
  - `apps/browser/src/key-management/browser-process-reload.service.ts`
  - `apps/browser/src/auth/popup/logout/extension-logout.service.ts`
- Gating: on Safari, account switching needs flag `SafariAccountSwitching`
  (`accountSwitchingEnabled$` in `apps/browser/src/auth/popup/account-switching/services/account-switcher.service.ts`); other browsers always have it.
- How it works: `logout` uploads pending events, switches to the next account, clears ciphers, folders,
  biometric and PIN state and the popup view cache, clears keys, state, tokens and the account, runs
  the state "logout" event, reseeds storage (F47), clears any pending clipboard, then calls the
  process reload service. The browser reload service clears scheduled tasks, waits for popups to
  close, and calls `runtime.reload()`. Autofill, badge and notification settings are deliberately kept.
- Interview angle: what must be wiped on logout, in which order, and why the process is reloaded.

**F30. Account security settings**

- What the user sees: Settings > Account security: unlock with PIN, biometrics, vault timeout and
  action, change master password, two-step login link, fingerprint phrase, device management, lock and
  log out, phishing detection toggle, shared unlock toggles.
- Code:
  - `apps/browser/src/auth/popup/settings/account-security.component.ts`
  - `apps/browser/src/auth/popup/settings/change-password-page.component.ts`, `apps/browser/src/auth/popup/settings/extension-device-management.component.ts`
  - `apps/browser/src/auth/popup/components/set-pin.component.ts`
  - `apps/browser/src/key-management/session-timeout/services/browser-session-timeout-settings-component.service.ts`
- Gating: PIN option hidden when policy `RemoveUnlockWithPin` applies; change password opens the
  in-extension page only with flag `PM32413_MultiClientPasswordManagement` (otherwise it sends the
  user to the web app); shared unlock toggles need `SharedUnlockPart2` (F32); phishing toggle appears
  only when detection is available (F36).
- Interview angle: security settings are a bundle of independent toggles with policy overrides; study
  how each toggle's state is persisted per user.

### Unlock, biometrics, timeout

**F31. Lock screen and unlock options**

- What the user sees: the lock screen offers master password, PIN, biometrics and (when available)
  passkey PRF unlock, depending on what the account has set up.
- Code:
  - `apps/browser/src/key-management/lock/services/extension-lock-component.service.ts` (`getAvailableUnlockOptions$`)
  - `apps/browser/src/popup/app-routing.module.ts` (`lock` route, `lockGuard`; `LockComponent` from `libs/key-management-ui`)
  - `libs/key-management-ui/src/lock/services/default-webauthn-prf-unlock.service.ts`
  - `apps/browser/src/key-management/unlock/background-unlock.service.ts`, `apps/browser/src/key-management/unlock/foreground-unlock.service.ts`, `apps/browser/src/key-management/unlock/unlock-messages.ts` (announce an unlock across popup and background)
- Notes: the biometrics option is queried when biometric unlock is enabled, or when `SharedUnlockPart2`
  is on and sharing with desktop is allowed.
- Interview angle: the popup and the background are separate JS contexts, so "unlocked" has to be
  announced between them; the background can unlock on its own (biometrics) and tells the popup.

**F32. Biometrics and shared unlock via desktop**

- What the user sees: "Unlock with biometrics" uses the Bitwarden desktop app. The user must grant the
  optional `nativeMessaging` permission, which reloads the extension. A newer "share unlock state with
  desktop/web" option lets another Bitwarden client unlock or lock the extension.
- Code:
  - `apps/browser/src/background/nativeMessaging.background.ts` (connects to host `com.8bit.bitwarden`)
  - `apps/browser/src/key-management/biometrics/background-browser-biometrics.service.ts`, `apps/browser/src/key-management/biometrics/foreground-browser-biometrics.ts`
  - `apps/browser/src/key-management/shared-unlock/popup/native-messaging-permission-dialog.component.ts`
  - `libs/common/src/key-management/shared-unlock/default-shared-unlock-peer.service.ts`, `libs/common/src/key-management/shared-unlock/shared-unlock-driver.ts`
  - `apps/desktop/src/main/native-messaging.main.ts`, `apps/desktop/src/services/biometric-message-handler.service.ts` (the desktop side)
  - `apps/browser/src/safari/safari/SafariWebExtensionHandler.swift` (Safari biometrics)
- How the channel works: the extension generates an RSA-2048 key pair and sends `setupEncryption`
  with the public key and the active user ID, unencrypted. The desktop rejects it with `wrongUserId`
  if that user is not logged in to the desktop app; otherwise it generates an AES-256-CBC-HMAC session
  key, encrypts it to the public key (RSA-OAEP with SHA-1) and returns it. Later messages are encrypted
  with that key, both sides drop messages whose timestamp is more than 10 s off
  (`MessageValidTimeout`), and the desktop can send `invalidateEncryption`, which makes the extension
  drop the channel and its keys. On a successful
  biometric prompt the desktop returns the user key (`userKeyB64`). Safari always uses native
  messaging to its bundled Swift handler instead.
- How the desktop trusts the extension: the native-messaging host manifests that the desktop writes
  restrict the caller to Bitwarden's extension IDs (`allowed_origins` for Chrome, Chrome beta, Edge and
  Opera; `allowed_extensions` for Firefox); the browser enforces this. Beyond that, the desktop checks
  only the user ID and the encryption; there is no fingerprint or approval step in this code, and the
  user still has to pass the OS biometric prompt.
- Gating: flag `BiometricsSDKIPC` switches non-Safari browsers from native messaging to an SDK IPC
  channel (`useSdkIpc`). Shared unlock: `SharedUnlockPart2` (user-facing) is checked in
  `default-shared-unlock-peer.service.ts`, `account-security.component.ts` and
  `extension-lock-component.service.ts`; with it off the peer has no destinations. The browser shares
  with desktop and/or web per the user's toggles; desktop and web share only with the browser. A
  shared unlock can also suppress the vault timeout (F33). `SharedUnlockPart1` is described in the
  enum comment as an emergency roll-back flag for the "leader" with no user-facing change; no
  TypeScript code reads it (see section 4). Shared unlock with desktop is hidden on Safari; with web
  is hidden on Firefox (`apps/browser/src/auth/popup/settings/account-security.component.ts`).
- Interview angle: how the extension trusts the desktop process, and what is sent (the user key
  travels over this channel).

**F33. Vault timeout, idle and system lock**

- What the user sees: choose when the vault locks or logs out: immediately, a number of minutes, a
  custom value, on browser restart, never, or when the system locks (not on every browser).
- Code:
  - `apps/browser/src/key-management/session-timeout/services/browser-session-timeout-type.service.ts` (which options exist per browser)
  - `apps/browser/src/background/idle.background.ts` (`chrome.idle` state handling, 5-minute detection interval)
  - `apps/browser/src/key-management/vault-timeout/vault-timeout.service.ts` (Safari timer workaround: loops on a native `sleep` that the Swift handler completes after 10 s)
  - `apps/browser/src/key-management/vault-timeout/foreground-vault-timeout.service.ts`
- Notes: "on system lock" is unavailable on Firefox, Safari, and Opera on Mac; an unavailable choice is
  promoted to "on restart". Idle state also disconnects the server notification channel and
  reconnects on activity. Shared unlock can suppress the timeout until a given time
  (`suppressVaultTimeout` from `shared-unlock-driver.ts`, checked with `isVaultTimeoutSuppressed`).
- Interview angle: timers in a service worker that can be killed; the extension uses a task
  scheduler backed by `chrome.alarms` (`apps/browser/src/platform/services/task-scheduler/`).

### Security health

**F34. At-risk passwords (security tasks)**

- What the user sees: an "at-risk passwords" page listing logins an admin asked the user to change, a
  callout on the vault tab, a different toolbar badge icon on affected sites, and a notification bar
  after the user logs in to an at-risk site.
- Code:
  - `apps/browser/src/vault/popup/components/at-risk-passwords/at-risk-passwords.component.ts` (also marks related end-user notifications as read)
  - `apps/browser/src/vault/popup/guards/at-risk-passwords.guard.ts`
  - `apps/browser/src/vault/services/at-risk-cipher-badge-updater.service.ts`
  - `apps/browser/src/autofill/background/notification.background.ts` (`triggerAtRiskPasswordNotification`)
  - `apps/browser/src/autofill/content/components/notification/at-risk-password/`
- Gating: security tasks must be enabled for the user (`TaskService.tasksEnabled$` in the guard),
  which is true when any of the user's organizations `canUseAccessIntelligence`. Task data comes from
  the server and is kept up to date by `DefaultTaskService` in `libs/common/src/vault/tasks`.
- Interview angle: how a server-assigned task becomes UI in four surfaces (page, badge, callout, bar).

**F35. Health report (commercial build)**

- What the user sees: a Health tab with a gauge and lists of exposed, weak and reused passwords.
- Code:
  - `apps/browser/src/popup/health-tab-nav-button.ts` (injection token the commercial build provides)
  - `bitwarden_license/bit-browser/src/popup/dirt/health/health.component.ts`, `bitwarden_license/bit-browser/src/popup/dirt/health/health-overview.component.ts`
  - `bitwarden_license/bit-browser/src/popup/dirt/health/services/health-access.service.ts`, `bitwarden_license/bit-browser/src/popup/dirt/health/services/health-scan.service.ts`
  - `bitwarden_license/bit-common/src/dirt/vault-health/models/risk-category.ts`
- Gating: not in the open-source build. Needs flag `BrowserExtensionHealthReport` and the user must be
  a personal user or only in Free or Families organizations (`healthEnabled$`).
- Interview angle: how a commercial module plugs into the OSS shell without the shell importing it.

**F36. Phishing detection**

- What the user sees: when the user navigates to a known phishing address, the tab is replaced with a
  warning page offering "close" or "continue to this site".
- Code:
  - `apps/browser/src/dirt/phishing-detection/services/phishing-detection.service.ts`
  - `apps/browser/src/dirt/phishing-detection/services/phishing-data.service.ts`, `apps/browser/src/dirt/phishing-detection/services/phishing-indexeddb.service.ts`, `apps/browser/src/dirt/phishing-detection/phishing-resources.ts`
  - `apps/browser/src/dirt/phishing-detection/popup/phishing-warning.component.ts` (route `/security/phishing-warning`)
  - `libs/common/src/dirt/services/phishing-detection/phishing-detection-settings.service.ts`
- How it works: on main-frame `webNavigation.onCommitted` and `onErrorOccurred`, the URL is looked up
  in a locally stored blocklist. The list, a manifest and incremental patches come from
  `https://assets.bitwarden.com/security/v1/`; a legacy MD5 checksum comes from the Phishing-Database
  project on GitHub. A hit navigates the tab to the extension warning page. "Continue" is remembered
  per host and tab.
- Gating: flag `PhishingDetection`, and the user needs personal premium or membership of a Families
  organization, or of an Enterprise organization with `usePhishingBlocker` (the org must also grant
  premium to users); not available on Safari. A user toggle (`enablePhishingDetection`) turns it on or
  off.
- Interview angle: privacy (the URL is matched locally, not sent to a server) and the cost of a
  `webNavigation` listener on every navigation. Org events are recorded (F46).

### Shortcuts, menus, badge

**F37. Keyboard shortcuts**

- What the user sees: `Ctrl+Shift+Y` opens the popup (`Ctrl+Shift+U` on Linux), `Ctrl+Shift+L`
  autofills a login, `Ctrl+Shift+9` generates a password to the clipboard. Autofill card, autofill
  identity and lock vault commands exist with no default key. Firefox and Opera also get a sidebar
  command (`Alt+Shift+Y`, `Alt+Shift+U` on Linux).
- Code:
  - `apps/browser/src/manifest.v3.json` (`commands`; the popup command is `_execute_action`, while the MV2 file uses `_execute_browser_action`)
  - `apps/browser/src/background/commands.background.ts`
  - `libs/common/src/autofill/constants/index.ts` (`ExtensionCommand`)
- Notes: `apps/browser/src/background/commands.background.ts` also handles `open_popup`, but that
  command is not declared in the manifests, and the handler does nothing except on Safari.
- Interview angle: shortcuts act on the active tab and need the vault unlocked; see F15 for the retry.

**F38. Context menus**

- What the user sees: right-click menu with autofill, copy username, copy password, copy verification
  code (premium only), autofill identity and card, generate password, "copy custom field name", and
  create-item shortcuts when no item matches.
- Code:
  - `apps/browser/src/autofill/browser/main-context-menu-handler.ts` (builds the menu)
  - `apps/browser/src/autofill/browser/cipher-context-menu-handler.ts` (per-site items)
  - `apps/browser/src/autofill/browser/context-menu-clicked-handler.ts` (actions, reprompt check, event log)
  - `apps/browser/src/autofill/background/context-menus.background.ts`
  - `apps/browser/src/autofill/content/context-menu-handler.ts` (finds the clicked element's identifier)
- Gating: user setting to show or hide the context menu (state `enableContextMenu`, form control
  `enableContextMenuItem`). Items marked `requiresUnblockedUri` (autofill, identity, card, "Copy
  custom field name") are hidden when the tab URL is on the blocked list. Card items are dropped when
  the `RestrictedItemTypes` policy restricts cards. Triage and webmapper items need the dev flag (F9).
- Interview angle: the menu is rebuilt on lock, unlock, sync and cipher changes (`refreshMenu` in
  `apps/browser/src/background/main.background.ts`), which makes it a cache-invalidation problem.

**F39. Toolbar badge and icon**

- What the user sees: the icon shows locked, logged out or unlocked; the badge shows the number of
  logins that match the current tab (if enabled); a berry icon marks sites with at-risk logins; a red
  "1" appears when a clipboard-setting notice is pending.
- Code:
  - `apps/browser/src/platform/badge/badge.service.ts`, `apps/browser/src/platform/badge/priority.ts`, `apps/browser/src/platform/badge/icon.ts`
  - `apps/browser/src/auth/services/auth-status-badge-updater.service.ts`
  - `apps/browser/src/autofill/services/autofill-badge-updater.service.ts`
  - `apps/browser/src/autofill/services/clipboard-notification-badge-updater.service.ts`
- Interview angle: several features want to own one badge; the service resolves them with a priority
  scheme (`Low` 0, `Default` 100, `High` 200).

### Settings

**F40. Settings screens**

- What the user sees: Settings lists Account security, Autofill, Notifications, Vault options,
  Appearance, Admin (conditional), About, Download Bitwarden, More from Bitwarden.
- Code:
  - `apps/browser/src/tools/popup/settings/settings-v2.component.ts`
  - `apps/browser/src/autofill/popup/settings/autofill.component.ts` (autofill on page load and its default, inline menu visibility and per-type toggles, context menu, TOTP auto-copy, clear clipboard, default match, fill assist, basic auth)
  - `apps/browser/src/autofill/popup/settings/notifications.component.ts` (add-login and change-password prompts, passkeys)
  - `apps/browser/src/vault/popup/settings/appearance.component.ts` (theme, favicons, badge counter, animations, compact mode, quick-copy actions, popup width, at-risk notifications)
  - `apps/browser/src/vault/popup/settings/vault-settings.component.ts` (folders, import, export, archive, trash, sync now)
- Gating: several controls are flag- or policy-dependent (basic auth: F8; fill assist: F9; default
  match: F10). A dev flag `useBitwardenAutofillAttributes` gates the `data-bwignore`/`data-bwautofill`
  settings.
- Interview angle: the settings are persisted per user through state providers (F47); some options
  exist for compatibility with the browser's own password manager.

**F41. Default password manager prompt**

- What the user sees: a screen asking the user to make Bitwarden the browser's default password
  manager, which turns off the browser's built-in saving and autofill.
- Code:
  - `apps/browser/src/autofill/popup/default-password-manager/default-password-manager-prompt.component.ts`, `apps/browser/src/autofill/popup/default-password-manager/default-password-manager-prompt.guard.ts`
  - `apps/browser/src/autofill/default-password-manager-session.util.ts`, `apps/browser/src/autofill/default-password-manager-prompt-feature.util.ts`
  - `apps/browser/src/platform/browser/browser-api.ts` (`updateDefaultBrowserAutofillSettings`, uses `chrome.privacy.services.*`)
  - `apps/browser/src/background/runtime.background.ts` (marks a fresh install as eligible)
- Gating: flag `DefaultPasswordManagerPrompt`; needs the optional `privacy` permission (declared in
  `optional_permissions` in both manifests and requested at the time of use; Safari has no optional
  permissions, `__safari__optional_permissions: null`).
- Interview angle: optional permissions are requested at runtime. The component has separate Firefox
  popout and popup paths, and a session-storage state (`pending`, `show-toast`) that the background
  finishes after the permission is granted (`handleSetBitwardenAsDefaultPasswordManager` in
  `runtime.background.ts`). I read this as a workaround for the popup's short life; the code does not
  say so in as many words.

### Sync and push

**F42. Sync**

- What the user sees: vault data refreshes in the background; "Sync now" in Vault options.
- Code:
  - `apps/browser/src/background/main.background.ts` (`fullSync` skips if the last sync was under 6 hours ago; `backgroundSyncService`)
  - `apps/browser/src/platform/sync/foreground-sync.service.ts`, `apps/browser/src/platform/sync/sync-service.listener.ts` (the popup asks the background to sync; the listener only honours `doFullSync` messages that came from another extension context, `isExternalMessage`)
  - `libs/common/src/platform/sync/` (shared sync service; not read in detail)
- Interview angle: only the background syncs; the popup requests it by message and waits for a
  `fullSyncFinished` reply.

**F43. Server push and system notifications**

- What the user sees: changes made in another client show up without a manual sync; auth requests
  raise a system notification.
- Code:
  - `apps/browser/src/background/main.background.ts` (builds `DefaultServerNotificationsService` with SignalR and, on MV3, `WorkerWebPushConnectionService`)
  - `libs/common/src/platform/server-notifications/README.md` and `libs/common/src/platform/server-notifications/internal/` (`worker-webpush-connection.service.ts`)
  - `apps/browser/src/background/idle.background.ts` (disconnect when the user is idle)
  - `apps/browser/src/platform/notifications/foreground-server-notifications.service.ts`
- Gating: web push needs MV3 and a service worker registration (`self.registration`); otherwise
  `UnsupportedWebPushConnectionService` is used. Then it is driven by server config, not a feature
  flag: `supportStatus$` uses web push only when `config.push.pushTechnology === PushTechnology.WebPush`
  and a `vapidPublicKey` is present. It subscribes with `pushManager.subscribe` and registers the
  subscription with the server (`putSubscription`); otherwise SignalR is used.
- Interview angle: a long-lived connection (SignalR) in an MV3 service worker that sleeps; why Web Push
  exists.

### Enterprise

**F44. Policies and managed configuration**

- What the user sees: org policies change or disable features; IT can pre-set the server URLs and push
  managed settings.
- Code:
  - Policy uses relevant to the extension: `AutomaticAppLogIn` (F7), `ActivateAutofill` (F3), `FillAssist` (F9), `UriMatchDefaults` (F10), `RestrictedItemTypes` (F5, F38), `OrganizationDataOwnership` (F14, vault filters in `apps/browser/src/vault/popup/services/vault-popup-list-filters.service.ts`), `RemoveUnlockWithPin` (F30), `PasswordGenerator` (F23), `DisableSend`/`SendOptions`/`SendControls` (F24, F22), `MaximumVaultTimeout` (`libs/common/src/key-management/vault-timeout/services/vault-timeout-settings.service.ts`), `FreeFamiliesSponsorship` (`apps/browser/src/billing/services/families-policy.service.ts`)
  - `apps/browser/src/managed_schema.json` (browser managed storage schema; keys under `environment`)
  - `apps/browser/src/platform/services/browser-environment.service.ts` (`getManagedEnvironment` reads `storage.managed` key `environment`)
  - `apps/browser/src/platform/services/browser-managed-config-reader.ts` (reads the whole managed area into `ManagedSettingsService` from `@bitwarden/managed-settings`, and re-reads on `storage.onChanged` for the `managed` area)
  - `apps/browser/src/admin-console/types/group-policy-environment.ts`
- How it works: on first install, if a managed environment exists, the extension applies its URLs
  (`hasManagedEnvironment`, `setUrlsToManagedEnvironment` in `apps/browser/src/background/runtime.background.ts`, F52).
  Separately, `bootstrap()` starts with `BrowserManagedConfigReader.init()`, which pushes a UEM/MDM
  profile into `ManagedSettingsService` and keeps it current. The schema is removed for Firefox in the
  manifest (`__firefox__storage: null`).
- Gating: the `ManagedDeviceFramework` flag is defined but not read by any TypeScript in the repo.
- Interview angle: policies are enforced client-side, so they are a UX control; the server must also
  enforce what matters.

**F45. Admin auto-confirm, premium and families screens**

- What the user sees: an "Admin" settings entry for people who can manage automatic user
  confirmation; a Premium page and upgrade dialogs; families sponsorship handling.
- Code:
  - `apps/browser/src/vault/popup/settings/admin-settings.component.ts`, guard `canAccessAutoConfirmSettings` from `libs/auto-confirm/src/angular/guards/automatic-user-confirmation-settings.guard.ts`
  - `apps/browser/src/billing/popup/settings/premium-v2.component.ts`, `apps/browser/src/billing/popup/services/browser-premium-upgrade-prompt.service.ts`
  - `libs/angular/src/billing/components/premium-upgrade-dialog/premium-upgrade-dialog.component.ts` (the dialog the browser prompt service opens)
  - `apps/browser/src/billing/services/families-policy.service.ts`
- Gating: auto-confirm needs `canManageAutoConfirm$` to be true, otherwise a toast and a redirect to
  the vault tab. Flag `PM34515_BrowserDesktopCheckout` reaches the extension through the shared
  upgrade dialog: when it is on and the environment is cloud, "upgrade" creates a premium checkout
  session (`createPremiumCheckoutSession({ platform: "browser" })`) and opens its URL; otherwise it
  opens the web vault's premium page.
- Interview angle: route guards that redirect with a toast are the pattern for permission-gated
  screens.

**F46. Event logging**

- What the user sees: nothing; organization admins see events such as "client autofilled" and
  phishing-blocker events.
- Code:
  - `libs/common/src/dirt/event-logs/` (`EventCollectionService`, `EventType`)
  - `apps/browser/src/dirt/event-logs/foreground-event-upload.service.ts` (a no-op so only the background uploads)
  - `apps/browser/src/autofill/services/autofill.service.ts` (`Cipher_ClientAutofilled`)
  - `apps/browser/src/dirt/phishing-detection/services/phishing-detection.service.ts` (`PhishingBlocker_SiteAccessed`, `PhishingBlocker_Bypassed`, `PhishingBlocker_SiteExited`)
- Interview angle: duplicate-upload avoidance between contexts, and flushing events on logout
  (`apps/browser/src/background/main.background.ts` calls `uploadEvents` first).

### Platform

**F47. State and storage**

- What the user sees: settings and the vault survive restarts; the unlocked state does not survive a
  browser restart unless the user chose "never" lock (then `bootstrap()` auto-unlocks with the stored
  auto-unlock key).
- Code:
  - `apps/browser/src/platform/services/browser-local-storage.service.ts` (`chrome.storage.local`)
  - `apps/browser/src/platform/services/browser-memory-storage.service.ts` (MV3: `storage.session`)
  - `apps/browser/src/platform/services/local-backed-session-storage.service.ts` (MV3, large session objects: stored in local storage, encrypted with an ephemeral key kept in session storage; if the key is gone after a browser restart, the stored items are cleared)
  - `apps/browser/src/platform/storage/background-memory-storage.service.ts`, `apps/browser/src/platform/storage/foreground-memory-storage.service.ts` (MV2: memory in the background, popup reaches it over ports)
  - `apps/browser/src/background/main.background.ts` (picks MV2 or MV3 storage; comment: secure storage is not supported in browsers so local storage is used; disk state is mirrored to `window.localStorage`, via the offscreen document on MV3)
  - `libs/state/` (state providers)
- Interview angle: where each secret lives in MV2 versus MV3, and what an attacker with disk access
  versus a running browser gets. After logout, Chrome, Vivaldi and Opera (not Edge or Firefox) get
  `reseedStorage` (in `apps/browser/src/background/main.background.ts`), which calls `fillBuffer` in
  `apps/browser/src/platform/services/browser-local-storage.service.ts`. Its code comment says it writes
  4 MB of filler data to force LevelDB log compaction so deleted values are removed from disk. It is
  skipped when the vault timeout is "never".

**F48. Offscreen document and clipboard**

- What the user sees: copying a password from the context menu or shortcut works even though a service
  worker has no DOM clipboard access. Clearing the clipboard after a timer also works.
- Code:
  - `apps/browser/src/platform/offscreen-document/offscreen-document.service.ts` (`withDocument` creates the page, counts users, closes it when idle)
  - `apps/browser/src/platform/offscreen-document/offscreen-document.ts` (handlers: copy, read, `localStorage` get/save/remove)
  - `apps/browser/src/platform/services/platform-utils/browser-platform-utils.service.ts` (`copyToClipboard` and `readFromClipboard` pick native (Safari), offscreen (MV3 when supported), or direct)
  - `apps/browser/src/platform/storage/offscreen-storage.service.ts`
- Gating: MV3 and not Firefox. The `offscreen` permission is in the default and Safari permission lists of `apps/browser/src/manifest.v3.json`, and not in the Firefox list.
- Interview angle: a service worker cannot use DOM APIs, so a hidden page is spawned with a declared
  reason (`chrome.offscreen.Reason.CLIPBOARD` for the clipboard).

**F49. Popup shell: popouts, sidebar, side panel, caches**

- What the user sees: the popup can be popped out into its own window, shown in the Firefox/Opera
  sidebar, or in a Chrome side panel (autofill dev tools only; `bootstrap()` disables the side panel
  globally and it is enabled per tab). The popup remembers where you were and half-typed forms across
  closes.
- Code:
  - `apps/browser/src/platform/browser/browser-popup-utils.ts` (`openPopout`, `inPopout`, `inSidebar`, `inSidePanel`, `inPopup`, single-action popouts)
  - `apps/browser/src/platform/popup/view-cache/popup-router-cache.service.ts`, `apps/browser/src/platform/popup/view-cache/popup-view-cache.service.ts`
  - `apps/browser/src/platform/services/popup-view-cache-background.service.ts`, `apps/browser/src/platform/services/popup-router-cache-background.service.ts`
  - `apps/browser/src/tools/popup/guards/file-picker-popout.guard.ts`
  - `apps/browser/src/popup/services/services.module.ts` (foreground service wiring)
  - `apps/browser/src/platform/browser/browser-api.ts` (wrapper over `chrome.*` and `browser.*`)
- How it works: the background owns the view cache. It is cleared two minutes after the popup closes
  (a task-scheduler delay that reopening cancels), on account switch and on logout; entries marked
  `clearOnTabChange` are dropped when the user switches tabs. File pickers close the toolbar popup in
  some browsers, so the guard opens a popout on Firefox (unless in the sidebar), Safari, and Chromium
  browsers on Linux/Mac; it also forces a popout for the import page unless `ImportUpgrade` is on.
- Interview angle: the toolbar popup closes when it loses focus, so any state worth keeping must live
  in the background or a cache.

**F50. Page and web vault messaging channels**

- What the user sees: the web vault knows the extension is installed (it hides "install" prompts);
  the web vault can open the extension on a given screen (for example at-risk passwords); the
  extension can tell the web vault when the popup opened.
- Code:
  - `apps/browser/src/autofill/content/content-message-handler.ts` (window `postMessage` handlers: `checkBwInstalled`, `OpenAtRiskPasswords`, `OpenBrowserExtensionToUrl`, `authResult`, `webAuthnResult`, `duoResult`)
  - `apps/browser/src/vault/content/send-on-installed-message.ts`, `apps/browser/src/vault/content/send-popup-open-message.ts`
  - `apps/browser/src/background/runtime.background.ts` (`executeMessageActionOrOpenPopup`, `announcePopupOpen`, `sendBwInstalledMessageToVault`)
  - `apps/browser/src/platform/ipc/ipc-content-script-manager.service.ts` and `apps/browser/src/platform/ipc/content/ipc-content-script.ts` (a newer IPC channel; the script accepts only messages whose origin equals the page's own)
  - `libs/common/src/vault/enums/vault-messages.enum.ts`
- Gating: the IPC content script is registered only on MV3, for `https://*/*`, and only with flag
  `ContentScriptIpcChannelFramework`; on start it is also re-injected into open tabs. The legacy
  message handler is unflagged.
- Origin checks: `executeMessageActionOrOpenPopup` drops a message that arrived over
  `runtime.onMessage` (`isExternalMessage`) unless the sender's origin is a known vault host; with no
  accounts it just opens the popup.
- Interview angle: the content script's `forwardCommands` allow-list (`bgUnlockPopoutOpened`,
  `addedCipher`) controls which background broadcasts it re-sends to other extension contexts; its
  comment warns that anything forwarded is reachable by whatever can reach the content script.

### Safari

**F51. Safari app extension**

- What the user sees: the extension is part of a macOS app; a few features use the native app.
- Code:
  - `apps/browser/src/safari/safari/SafariWebExtensionHandler.swift` (commands: `readFromClipboard`, `copyToClipboard`, `showPopover`, `downloadFile`, `sleep`, `authenticateWithBiometrics`, `getBiometricsStatus`, `unlockWithBiometricsForUser`, `getBiometricsStatusForUser`, `biometricUnlock`, `biometricUnlockAvailable`)
  - `apps/browser/src/browser/safariApp.ts` (`sendMessageToApp` to `com.bitwarden.desktop`)
  - `apps/browser/src/safari/desktop/` and `apps/browser/src/safari/desktop.xcodeproj`
  - `apps/browser/src/key-management/vault-timeout/vault-timeout.service.ts` (timer workaround)
  - `apps/browser/src/manifest.v3.json` (`__safari__permissions` includes `nativeMessaging` as a required permission; `apps/browser/src/manifest.json` does the same for MV2)
- Differences visible in code: no phishing detection (F36), no desktop shared unlock (F32), account
  switching behind a flag (F29), file picker forces a popout (F49), clipboard read and write go through
  the native app (`browser-platform-utils.service.ts`), no optional permissions and no `notifications`
  permission in the Safari lists, the Swift handler also implements `downloadFile`, and biometrics
  always use native messaging.
- Interview angle: how much of the extension is browser-agnostic, and where `isSafari()` branches
  appear.

### Install and startup

**F52. Install, startup re-injection and welcome page**

- What the user sees: on a fresh install a welcome page opens and the inline menu is already on; pages
  that were open before the install or update get autofill without a reload.
- Code:
  - `apps/browser/src/background/runtime.background.ts` (`onInstalled` listener in the constructor, `checkOnInstalled` from `init()`)
  - `apps/browser/src/platform/services/browser-initial-install.service.ts` (`displayWelcomePage`, `extensionInstalled$`)
  - `apps/browser/src/autofill/services/autofill.service.ts` (`loadAutofillScriptsOnInstall`, `injectAutofillScriptsInAllTabs`, `reloadAutofillScripts`)
- How it works: on every background start, `checkOnInstalled` (after 100 ms) re-injects autofill
  scripts into every open `http*` tab and frame and subscribes to inline-menu setting changes, which
  re-inject everywhere when the menu is turned on or off or the card/identity sub-settings change. On reason `install`, and only once
  (`extensionInstalled` state), it marks the user eligible for the default password manager prompt
  (F41, if flagged), opens `https://bitwarden.com/browser-start/` for user-initiated installs (not
  admin or sideloaded installs, not with dev flag `skipWelcomeOnInstall` or the `prereleaseBuild`
  flag), sets inline menu visibility to `OnFieldFocus`, and applies a managed environment (F44).
- Interview angle: in MV3 this start-up re-injection is also what rebuilds per-frame state after the
  service worker is killed (F3); and why "on field focus" by default matters for clickjacking (F12).

---

## 3. Feature flags touching the extension (quick table)

All flags are in `libs/common/src/enums/feature-flag.enum.ts`. The "Used in" paths are where the
extension or its shared libs check them. Defaults in this file are off.

| Flag | Feature | Used in |
| --- | --- | --- |
| `LitInlineMenuComponents` | F11 | `apps/browser/src/autofill/background/overlay.background.ts` |
| `FillAssistTargetingRules` | F9 | `apps/browser/src/autofill/services/targeting-rules-data.service.ts`, `apps/browser/src/autofill/popup/settings/autofill.component.ts`, `libs/common/src/autofill/services/domain-settings.service.ts` |
| `EnableBasicAuthResponse` | F8 | `apps/browser/src/autofill/background/web-request.background.ts`, `apps/browser/src/autofill/popup/settings/autofill.component.ts` |
| `UseUndeterminedCipherScenarioTriggeringLogic` | F14 | `apps/browser/src/autofill/background/notification.background.ts` |
| `DefaultPasswordManagerPrompt` | F41 | `apps/browser/src/autofill/default-password-manager-prompt-feature.util.ts` |
| `PhishingDetection` | F36 | `libs/common/src/dirt/services/phishing-detection/phishing-detection-settings.service.ts` |
| `BrowserExtensionHealthReport` | F35 | `bitwarden_license/bit-browser/src/popup/dirt/health/services/health-access.service.ts` |
| `VFO1Foundation` | F19 and others | `apps/browser/src/vault/popup/components/vault/vault.component.ts`, `apps/browser/src/vault/popup/services/vault-popup-items.service.ts`, `apps/browser/src/platform/popup/layout/popup-header.component.ts`, `apps/browser/src/vault/popup/guards/clear-vault-state.guard.ts` |
| `PM32009NewItemTypes` | F20 | `apps/browser/src/popup/app-routing.module.ts`, `apps/browser/src/vault/popup/components/vault/new-item-dropdown/new-item-dropdown.component.ts` |
| `PM34203TemporaryItemSharing` | F22 | `apps/browser/src/popup/app-routing.module.ts`, `libs/tools/share/src/services/share-link.service.ts` |
| `ImportUpgrade` | F25 | `apps/browser/src/tools/popup/guards/import-upgrade-redirect.guard.ts`, `apps/browser/src/tools/popup/guards/import-upgrade-required.guard.ts`, `apps/browser/src/tools/popup/guards/file-picker-popout.guard.ts`, `apps/browser/src/vault/popup/settings/vault-settings.component.ts` |
| `SendControls` | F24, F22 | `libs/tools/send/send-ui/src/services/send-policy.service.ts` |
| `SafariAccountSwitching` | F29 | `apps/browser/src/auth/popup/account-switching/services/account-switcher.service.ts` |
| `PM32413_MultiClientPasswordManagement` | F30 | `apps/browser/src/popup/app-routing.module.ts`, `apps/browser/src/auth/popup/settings/account-security.component.ts` |
| `SharedUnlockPart2` | F32 | `libs/common/src/key-management/shared-unlock/default-shared-unlock-peer.service.ts`, `apps/browser/src/auth/popup/settings/account-security.component.ts`, `apps/browser/src/key-management/lock/services/extension-lock-component.service.ts` |
| `SharedUnlockPart1` | F32 | Defined only; no TypeScript reads it (see section 4) |
| `BiometricsSDKIPC` | F32 | `apps/browser/src/key-management/biometrics/background-browser-biometrics.service.ts` |
| `ContentScriptIpcChannelFramework` | F50 | `apps/browser/src/platform/ipc/ipc-content-script-manager.service.ts` |
| `PM34515_BrowserDesktopCheckout` | F45 | `libs/angular/src/billing/components/premium-upgrade-dialog/premium-upgrade-dialog.component.ts` |
| `ManagedDeviceFramework` | F44 | Defined only; no TypeScript reads it |

---

## 4. Still unverified

- Whether any flag is on for real users. This repo has defaults only.
- `SharedUnlockPart1` and `ManagedDeviceFramework`: no TypeScript reads them. `DefaultSdkService`
  (`libs/common/src/platform/services/sdk/default-sdk.service.ts`, `loadFeatureFlags`) passes every
  boolean server flag to the SDK with `load_flags`, so the SDK may read them, but the SDK source is not
  in this repo.
- What "VFO" stands for. The code shows what `VFO1Foundation` gates (F19), not the name.
- F3: the commit and cool-down gates exist only in the design doc at this commit, so there are no
  interval values to report; check a later commit (the orchestrator's `FIXME (PM-39579)`).
- F44: the full list of policies enforced by shared libs; the list above covers the ones traced.
