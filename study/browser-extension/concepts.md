# Bitwarden Browser Extension: Concepts You Need to Read the Code

A study guide for someone preparing for a Bitwarden software engineering interview who is also studying cybersecurity. Everything below is grounded in the code in this repository (`/home/user/clients`). The extension lives in `apps/browser/`; shared code lives in `libs/`.

## How to read this document

- **All paths are relative to the repo root.** Every path cited here was checked to exist.
- **"In the repo:"** marks a statement that comes from reading code. **"Platform fact:"** marks a general web or extension platform fact that is not specific to Bitwarden. I kept platform facts standard and conservative. If I could not confirm something from the code, the text says so, and the open questions are collected at the end in [Open questions and caveats](#open-questions-and-caveats).
- Code excerpts are verbatim, at most 10 lines, and carry their file path. A line containing only `...` means lines were elided.
- The repo is the only source of truth. Where this repo differs from what you may remember about the public Bitwarden clients (for example, the inline menu's nested sandboxed iframe, or the `AutofillOrchestrator`), trust the code and the design docs.
- Two companion write-ups in this folder cover vulnerabilities in depth: [`vuln-cve-2018-25081-iframe-autofill.md`](./vuln-cve-2018-25081-iframe-autofill.md) and [`vuln-2025-dom-clickjacking.md`](./vuln-2025-dom-clickjacking.md).

### Table of contents

- [Part A: Browser extension platform concepts](#part-a-browser-extension-platform-concepts)
- [Part B: Bitwarden architecture concepts](#part-b-bitwarden-architecture-concepts)
- [Part C: Autofill concepts (the security-critical core)](#part-c-autofill-concepts-the-security-critical-core)
- [Part D: Security concepts glossary](#part-d-security-concepts-glossary)
- [Suggested reading order through the code](#suggested-reading-order-through-the-code)
- [Open questions and caveats](#open-questions-and-caveats)

---

# Part A: Browser extension platform concepts

## A1. Manifest V2 vs Manifest V3

**Platform fact:** An extension's `manifest.json` declares its name, permissions, scripts, UI entry points, and security policy. Manifest V2 (MV2) and Manifest V3 (MV3) are two versions of that format and of the extension runtime model. The biggest practical difference for Bitwarden: MV2 allows a persistent background page; MV3 does not. Chromium-based browsers (and Safari) run the MV3 background as a service worker that the browser can stop and restart at will. Firefox's MV3 instead runs `background.scripts` as a non-persistent "event page": it has a DOM, but Firefox can still unload it when idle. Either way, MV3 background code must expect to be torn down and restarted.

**In the repo:** There are two manifest sources, and the build picks one.

| File | `manifest_version` | Background declaration |
| --- | --- | --- |
| `apps/browser/src/manifest.json` | 2 | `"background": { "page": "background.html", "persistent": true }` |
| `apps/browser/src/manifest.v3.json` | 3 | `"background": { "service_worker": "background.js" }` plus `"__firefox__background": { "scripts": ["background.js"] }` |

### How the build picks one

`apps/browser/webpack.base.js` reads two environment variables:

```js
  const manifestVersion = process.env.MANIFEST_VERSION == 3 ? 3 : 2;
  const browser = process.env.BROWSER ?? "chrome";
```

(`apps/browser/webpack.base.js`, `getEnv`.) Anything other than `3` falls back to MV2. The same file then copies one manifest source to `manifest.json` and runs it through a transform:

```js
          from:
            manifestVersion == 3
              ? path.resolve(__dirname, "src/manifest.v3.json")
              : path.resolve(__dirname, "src/manifest.json"),
          to: "manifest.json",
          transform: manifest.transform(browser),
```

(`apps/browser/webpack.base.js`.) The npm and Nx targets in `apps/browser/project.json` set `MANIFEST_VERSION` and `BROWSER` per build target (for example `BROWSER=firefox MANIFEST_VERSION=2` for the `firefox-mv2-dev` target).

**Which manifest actually ships.** In `apps/browser/package.json`, `build:chrome` and `build:edge` set `MANIFEST_VERSION=3`, but `build:firefox` and `build:safari` set no version, so they fall back to MV2. The release workflow `.github/workflows/build-browser.yml` builds `dist:chrome`, `dist:edge` and `dist:opera:mv3` (MV3), `dist:firefox` (MV2), and `dist:safari` (MV2). It also builds a Firefox MV3 package, but names that artifact `DO-NOT-USE-FOR-PROD-dist-firefox-MV3`. So in this snapshot, Chrome, Edge and Opera ship MV3, while Firefox and Safari ship MV2. That matches the lifecycle design doc quoted in A6.

### Per-browser key prefixes

`apps/browser/webpack/manifest.js` implements a small convention: a key prefixed with `__<browser>__` overrides or removes the unprefixed key for that browser, and keys for other browsers are dropped. A `null` value deletes the key. Examples from the manifests:

- `"__firefox__background": { "scripts": ["background.js"] }` in `manifest.v3.json`. Firefox MV3 builds get a background script (an event page), and other browsers get the service worker. `webpack.base.js` matches this by building the MV3 background with webpack `target: "web"` for Firefox and `"webworker"` for the others.
- `"__firefox__sandbox": null` removes the `sandbox` key for Firefox.
- `"__chrome__side_panel": { "default_path": "sidepanel-disabled.html" }` only exists in Chrome builds.
- `"__safari__permissions"` and `"__firefox__permissions"` give those browsers their own permission lists.

For the beta channel, `transformChannel` in the same file merges `apps/browser/webpack/manifest-beta-overrides.json`.

### Runtime check

Code can ask which manifest it runs under:

```ts
  static get manifestVersion() {
    return chrome.runtime.getManifest().manifest_version;
  }
```

(`apps/browser/src/platform/browser/browser-api.ts`.) You will see `BrowserApi.isManifestVersion(3)` everywhere that behavior forks, for example storage selection in `apps/browser/src/background/main.background.ts` and script injection in `apps/browser/src/platform/services/browser-script-injector.service.ts`.

### Other MV2/MV3 differences visible in the manifests

- **Host permissions.** MV2 lists host patterns inside `permissions` (`"<all_urls>"`, `"*://*/*"`) and also requests `webRequestBlocking`. MV3 has a separate `"host_permissions": ["https://*/*", "http://*/*"]`. Its default permission list adds `scripting`, `offscreen`, `sidePanel`, `activeTab`, and `webRequestAuthProvider`. The Firefox MV3 list (`__firefox__permissions`) leaves out `offscreen` and `sidePanel`.
- **Minimum Chrome.** `manifest.v3.json` sets `"minimum_chrome_version": "134.0"`. The comment in `apps/browser/webpack.base.js` (in `requiredPlugins`) gives the reason. The Chrome build drops core-js polyfills, because the Chrome Web Store flags core-js's IE-era `Object.create` shim as "obfuscated code". The minimum is pinned so the store never serves the build to a Chrome without native `Symbol.dispose` and explicit resource management, which "the SDK's `using` disposal needs".
- **Web-accessible resources format** and **CSP format** (next two sections).
- **Offscreen document.** Only built when `manifestVersion == 3` and the browser is not Firefox (`apps/browser/webpack.base.js`: `if (browser !== "firefox")`, with the comment "Firefox does not use the offscreen API"). See [A6](#a6-execution-contexts).

## A2. Permissions and host permissions

**Platform fact:** `permissions` lists API capabilities the extension may use. Host permissions (in MV3, `host_permissions`) say which web origins the extension may script and read. The `tabs` permission exposes sensitive fields like `tab.url` and `tab.title` for every tab. A host permission also exposes them, but only for tabs whose URL it matches.

**In the repo** (`apps/browser/src/manifest.v3.json`, the default `permissions` list; Firefox and Safari get their own lists, see A1): `activeTab`, `alarms`, `clipboardRead`, `clipboardWrite`, `contextMenus`, `idle`, `offscreen`, `scripting`, `sidePanel`, `storage`, `tabs`, `unlimitedStorage`, `webNavigation`, `webRequest`, `webRequestAuthProvider`, `notifications`. Optional: `nativeMessaging`, `privacy` (the `privacy` one relates to controlling the browser's built-in password and address autofill; see `BrowserApi.browserAutofillSettingsOverridden` and `updateDefaultBrowserAutofillSettings` in `apps/browser/src/platform/browser/browser-api.ts`).

Why they matter for reading the code:

- `tabs` + `webNavigation`: the background knows each tab's URL and each frame's URL. Autofill matching uses `tab.url` (see [C2](#c2-uri-matching-and-why-it-is-security-critical)), and frame URLs come from `BrowserApi.getFrameDetails`.
- `scripting`: lets the background inject content scripts on demand and choose the execution world (`ISOLATED` or `MAIN`).
- `clipboardRead/Write`: copying passwords, usernames, and TOTP codes; see the offscreen document in A6.
- `contextMenus`, `idle`, `alarms`: right-click autofill and copy, idle detection for locking, and the MV3-safe task scheduler.
- Broad host permission: autofill must be able to run on any website, so the extension is very powerful. That is why most of the security design in Part C is about not misusing that power.

## A3. `content_scripts` and dynamic injection

**Platform fact:** A content script is JavaScript the extension runs inside web pages. The manifest can declare static `content_scripts` with URL `matches`, `run_at`, and `all_frames`. The extension can also inject scripts later through the scripting API.

**In the repo:** The manifests declare exactly two static content scripts. The second one, which runs in every frame at `document_start`:

```json
    {
      "all_frames": true,
      "css": ["content/autofill.css"],
      "js": ["content/trigger-autofill-script-injection.js"],
      "matches": ["*://*/*", "file:///*"],
      "exclude_matches": ["*://*/*.xml*", "file:///*.xml*"],
      "run_at": "document_start"
    }
```

(`apps/browser/src/manifest.v3.json`.) The first entry is `content/content-message-handler.js` with `"all_frames": false` (top frame only). Both entry points are tiny. The "trigger" script's whole job is:

```ts
(function () {
  void chrome.runtime.sendMessage({ command: "triggerAutofillScriptInjection" });
})();
```

(`apps/browser/src/autofill/content/trigger-autofill-script-injection.ts`.) The background handles that message:

```ts
      case "triggerAutofillScriptInjection":
        await this.autofillService.injectAutofillScripts(sender.tab, sender.frameId);
        break;
```

(`apps/browser/src/background/runtime.background.ts`, `processMessageWithSender`.) `AutofillService.injectAutofillScripts` (`apps/browser/src/autofill/services/autofill.service.ts`) then decides which real scripts to inject into that frame, based on user settings and auth state:

- one of four bootstrap bundles (`bootstrap-autofill.js`, `bootstrap-autofill-overlay-notifications.js`, `bootstrap-autofill-overlay-menu.js`, or `bootstrap-autofill-overlay.js`), chosen by the inline-menu and notification-bar settings (`getBootstrapAutofillContentScript`),
- `autofiller.js` only when `triggeringOnPageLoad && autoFillOnPageLoadIsEnabled`, and that flag can be true only when there is an active account and its vault is unlocked,
- `contextMenuHandler.js` always,
- and, only when the call is not triggered by a page load (for example the re-injection into open tabs at install time), `content-message-handler.js` as well.

It then hands the frame to the lifecycle service (`autofillLifecycleService.startMonitoringFrame(tab, frameId)`).

This two-step design (a trivial static script that asks the background to inject the heavy scripts) lets the background decide per frame, and respects the user's blocked-domains list: `BrowserScriptInjectorService.inject` (`apps/browser/src/platform/services/browser-script-injector.service.ts`) refuses to inject when the tab URL is blocked or, for a sub-frame injection, when that frame's URL is blocked (`findBlockedInjectionUrl`). The lifecycle design doc states that this static script also "wakes the service worker on every navigation regardless of auth state" (`apps/browser/src/autofill/lifecycle.design.md`).

The injector picks the MV3 or MV2 API under the hood. In MV3 it calls `chrome.scripting.executeScript` with `world` defaulting to `ISOLATED` (`BrowserApi.executeScriptInTab` in `browser-api.ts`).

## A4. `web_accessible_resources`

**Platform fact:** By default, web pages cannot load files from an extension's package. `web_accessible_resources` lists the files that web pages are allowed to load, for example as an iframe `src` or a script `src`. In MV3, each entry also says which sites (`matches`) may load the resources, and it can set `use_dynamic_url`.

**In the repo** (MV3, abridged; `apps/browser/src/manifest.v3.json`):

```json
      "resources": [
        "content/fido2-page-script.js",
        "notification/bar.html",
        ...
        "overlay/menu-button.html",
        "overlay/menu-list.html",
        "overlay/menu.html",
        ...
      ],
      "matches": ["<all_urls>"],
      "use_dynamic_url": true
```

Why this matters for injected iframes: the inline menu (`overlay/menu.html`, `overlay/menu-button.html`, `overlay/menu-list.html`) and the notification bar (`notification/bar.html`) are extension pages that the content script embeds in a web page as iframes. That only works if they are web-accessible. The flip side is a security consequence: **a web-accessible page can be loaded by any site listed in `matches`** (here, all sites). So those pages cannot assume only Bitwarden's own code talks to them. This is exactly why the menu container validates message sources, tokens, and origins (see [A8](#a8-message-passing) and [C4](#c4-the-inline-menu-overlay)).

`content/fido2-page-script.js` is listed because MV2 builds inject it into the page as a `<script src=...>`, see the MAIN-world discussion in A7.

**Platform fact:** `use_dynamic_url` is a Chrome MV3 option. Chrome documents it as allowing the resources to be loaded only through a dynamic ID, which is regenerated each browser session or extension reload, instead of the extension's fixed ID. Its purpose is to make it harder for a page to detect or fingerprint the extension by probing its fixed URLs. It does not stop a page that already knows the current URL from loading the resource. The repo only sets the flag, and no code depends on how it works. Firefox already gives each installation a random `moz-extension://` UUID. The repo does show that predictable extension URLs are a known concern: a custom ESLint rule, `libs/eslint/platform/no-page-script-url-leakage.mjs`, flags script elements that receive `chrome.runtime.getURL()` results, because "This pattern exposes predictable extension URLs to web pages, enabling fingerprinting attacks" (its file header) and tells developers to "Use secure page script registration instead". That is consistent with the MV3 FIDO2 path registering the page script with `world: "MAIN"` instead of inserting a `<script src>`; the MV2 helper that still inserts one carries an `eslint-disable-next-line` for this rule (A7).

## A5. Content Security Policy (CSP) and sandboxed pages

**Platform fact:** An extension's CSP restricts what scripts its own pages can run. `script-src 'self'` means only scripts bundled in the extension may run (no inline scripts, and, by default, no `eval`). `'wasm-unsafe-eval'` allows WebAssembly compilation. A CSP on a web page is a different thing; that is defined by the website, not by Bitwarden.

**In the repo:**

```json
    "extension_pages": "script-src 'self' 'wasm-unsafe-eval'; object-src 'self'",
    "__chrome__sandbox": "sandbox allow-scripts; script-src 'self'"
```

(`apps/browser/src/manifest.v3.json`, `content_security_policy`. Because of the `__chrome__` prefix, only Chrome builds get the `sandbox` CSP key here.) The MV2 manifest uses the string form for `content_security_policy` and a separate `sandbox` object with its own CSP. The `wasm-unsafe-eval` allowance is consistent with the extension loading the Rust SDK as WebAssembly (`apps/browser/src/platform/services/sdk/browser-sdk-load.service.ts` imports `./wasm`), though I am inferring the reason rather than reading it from a comment.

**Sandboxed pages.** Both manifests list:

```json
  "sandbox": {
    "pages": ["overlay/menu-button.html", "overlay/menu-list.html"]
  },
```

**Platform fact:** a manifest "sandbox page" (a Chrome feature) is served in a unique opaque origin and has no access to extension APIs. In Firefox builds, `"__firefox__sandbox": null` removes this key. The code also loads the same pages in an iframe with the HTML attribute `sandbox="allow-scripts"` (see C4). That attribute gives the iframe an opaque origin on its own, in every browser. So the actual button and list UI run in a context that cannot call `chrome.*` and must communicate through their parent extension page.

## A6. Execution contexts

Different parts of the extension run in different JavaScript environments. Knowing which one a file runs in is the first thing to work out when reading the code.

| Context | What it is | In the repo |
| --- | --- | --- |
| Background page (MV2) | A hidden persistent page with a DOM | `apps/browser/src/platform/background.html` (MV2 build only), built by `HtmlWebpackPlugin` in `apps/browser/webpack.base.js` |
| Service worker (MV3, Chromium and Safari) | Background code with no DOM, which the browser may stop when idle and restart on events. Firefox MV3 builds use a non-persistent background script instead (A1) | `background.js`, same entry source as MV2 (`apps/browser/src/platform/background.ts`) |
| Offscreen document (MV3, not Firefox) | A hidden page created on demand so the service worker can use DOM-only APIs | `apps/browser/src/platform/offscreen-document/` |
| Popup | The toolbar UI (Angular app) | `apps/browser/src/popup/` |
| Popout window, sidebar, side panel | The same Angular app opened in other containers | `uilocation` query param, `BrowserPopupUtils` |
| Content scripts | Extension code running inside web pages (isolated world) | `apps/browser/src/autofill/content/` |
| Page-world scripts (MAIN world) | Extension-supplied script running in the page's own JS world | `apps/browser/src/autofill/fido2/content/fido2-page-script.ts` |
| Extension iframes inside pages | Extension pages embedded in the page by the content script | `overlay/menu.html`, `notification/bar.html` |

### Background: persistent page vs service worker

Both build modes use the same entry file:

```ts
const logService = new ConsoleLogService(false);
const bitwardenMain = ((self as any).bitwardenMain = new MainBackground());
bitwardenMain.bootstrap().catch((error) => logService.error(error));
```

(`apps/browser/src/platform/background.ts`.) `MainBackground` (`apps/browser/src/background/main.background.ts`) constructs every background service by hand; see [B2](#b2-angular-in-the-popup-and-manual-construction-in-the-background).

The lifecycle design doc states the consequence directly: "Firefox runs Manifest V2 with a persistent background page; Chrome runs Manifest V3, whose background is a service worker the browser terminates and restarts at will. In-memory background state is therefore durable on Firefox but ephemeral on Chrome, where it must be reconstructed on each restart." (`apps/browser/src/autofill/lifecycle.design.md`.) This describes the production builds (A1). The not-for-production Firefox MV3 build would not get that durability, because a Firefox MV3 background script can be unloaded when idle. And `apps/browser/CLAUDE.md` says: "DON'T assume background page persists indefinitely" and "`chrome.extension.getBackgroundPage()` returns `null` in MV3".

What follows for you as a reader:

- **No DOM** in the service worker. Anything needing `localStorage`, `document`, or clipboard DOM tricks needs a different home (the offscreen document).
- **No durable memory.** Memory state must be persisted somewhere that survives a worker restart (see [B3](#b3-rxjs-and-the-state-provider-framework)).
- **Timers are unreliable.** `setInterval` in a worker that gets terminated is lost, so the repo has a task scheduler built on `chrome.alarms` in `apps/browser/src/platform/services/task-scheduler/browser-task-scheduler.service.ts`. The alarms API cannot schedule anything shorter than a minute. Per the class comments, delays under a minute use plain `setTimeout`/`setInterval`, with alarms created as a backup in case the timer is lost. `VaultTimeoutService` (`libs/common/src/key-management/vault-timeout/services/vault-timeout.service.ts`) uses this scheduler.
- **Startup order matters.** In `main.background.ts`, `bootstrap()` has a comment that the autofill lifecycle is wired "before runtime init triggers script injection, so the onConnect listener is registered before any frame connects".

### The offscreen document

**Platform fact:** An MV3 service worker has no DOM. Chrome's `chrome.offscreen` API lets an extension create a hidden document for specific declared reasons.

**In the repo:** `apps/browser/src/platform/offscreen-document/offscreen-document.ts` handles a small, explicit set of commands sent to it:

```ts
  private readonly extensionMessageHandlers: OffscreenDocumentExtensionMessageHandlers = {
    offscreenCopyToClipboard: ({ message }) => this.handleOffscreenCopyToClipboard(message),
    offscreenReadFromClipboard: () => this.handleOffscreenReadFromClipboard(),
    localStorageGet: ({ message }) => this.handleLocalStorageGet(message.key),
    localStorageSave: ({ message }) => this.handleLocalStorageSave(message.key, message.value),
    localStorageRemove: ({ message }) => this.handleLocalStorageRemove(message.key),
  };
```

So it exists for two reasons. The first is **clipboard access**. Inside the offscreen page, the copy and read handlers call `BrowserClipboardService` (`apps/browser/src/platform/services/browser-clipboard.service.ts`). The background decides when to use the offscreen page in `BrowserPlatformUtilsService.copyToClipboard` / `readFromClipboard` (`apps/browser/src/platform/services/platform-utils/browser-platform-utils.service.ts`). Safari always goes through the native app (`SafariApp.sendMessageToApp("copyToClipboard", ...)`). Otherwise the offscreen page is used only when the build is MV3, `chrome.offscreen` exists (`offscreenApiSupported()`), and the caller has no `document`, as in the service worker. A caller with a document, such as the popup, uses the clipboard directly. The second reason is **`window.localStorage`**: the MV3 background wraps it in `OffscreenStorageService` (`apps/browser/src/platform/storage/offscreen-storage.service.ts`). `DefaultOffscreenDocumentService.withDocument` (`apps/browser/src/platform/offscreen-document/offscreen-document.service.ts`) creates the document if missing, runs a callback, and closes the document when the last caller finishes (a reference count in `workerCount`). `MainBackground` picks the storage backend by manifest version:

```ts
    const localStorageStorageService = BrowserApi.isManifestVersion(3)
      ? new OffscreenStorageService(this.offscreenDocumentService)
      : new WindowStorageService(self.localStorage);
```

(`apps/browser/src/background/main.background.ts`.)

### Popup, popout, sidebar, side panel

The popup is an Angular single-page app (`apps/browser/src/popup/main.ts` bootstraps `AppModule`). The same app is opened in other containers, distinguished by a `uilocation` URL parameter: `popup`, `sidebar` (Firefox/Opera sidebar action), `popout` (a standalone window), or `sidepanel` (checked in `apps/browser/src/platform/browser/browser-popup-utils.ts`).

**Platform fact:** a popup is destroyed when it loses focus or closes, so state in it is short-lived. The repo caches popup view and router state in the background (`apps/browser/src/platform/services/popup-view-cache-background.service.ts` and `popup-router-cache-background.service.ts`) so reopening the popup feels continuous.

**Side panel.** The MV3 manifest requests `sidePanel` and declares `__chrome__side_panel` pointing to a blank page. At startup the background disables the panel globally:

```ts
    if (BrowserApi.isSidePanelApiSupported) {
      await BrowserApi.setSidePanelOptions({ enabled: false });
    }
```

(`apps/browser/src/background/main.background.ts`, `bootstrap`.) It is enabled per tab only for the "autofill triage" diagnostics feature, from the context-menu handler, with a comment that `sidePanel.open()` needs a user gesture (`apps/browser/src/autofill/browser/context-menu-clicked-handler.ts`, `autofillTriageAction`).

### Content scripts and extension iframes

These run on web pages and are the main subject of Part C. Two important facts:

- The **inline menu and notification bar are only created in the top frame**. The bootstraps check `globalThis.self === globalThis.top` before building the inline menu and notification services (`apps/browser/src/autofill/content/bootstrap-autofill-overlay.ts`, and the same check in `bootstrap-autofill-overlay-menu.ts` and `bootstrap-autofill-overlay-notifications.ts`). Sub-frames still collect fields; the top frame draws the UI using computed offsets (see A9).
- The extension also builds **extension-origin iframes** inside page DOM. They are the reason `web_accessible_resources` exists (A4).

## A7. Isolated worlds vs the page's main world

**Platform fact:** A content script shares the page's DOM but runs in its own *isolated world*, a separate JavaScript environment with its own globals and its own copies of built-in objects and prototypes. The page's scripts cannot read the content script's variables or call its functions, and the content script cannot see the page's JavaScript variables. Because the built-ins are separate, a page that monkey-patches something like `Element.prototype.attachShadow` or `JSON.parse` does not affect the content script. Both worlds see the same DOM tree and can change it. The page can also change the DOM after the content script reads it, and the content script can only read what the DOM exposes, for example attribute values that the page controls. Of the two, only the content script can use extension messaging such as `chrome.runtime.sendMessage`, and even it gets only a limited subset of extension APIs. A web page can message an extension only if the extension declares `externally_connectable`, and Bitwarden does not (Part D).

**What a malicious page can and cannot do:**

| The page can | The page cannot (by default isolation) |
| --- | --- |
| Read and rewrite any DOM node the extension adds to the page. A closed shadow root hides the node's *contents* from page script, and a cross-origin iframe hides its document, but the page can still move, restyle, cover, or remove the host element itself | Read the content script's JS variables (such as `bitwardenAutofillInit`) or call its functions |
| Change CSS, attributes, `opacity`, `z-index`, and `pointer-events` on its own elements (and on the extension's host elements; see [C4](#c4-the-inline-menu-overlay)) | Call `chrome.runtime.*` as the extension |
| Dispatch synthetic events, and call `window.postMessage` (the resulting `message` events are dispatched by the browser, so they are `isTrusted === true`) | Synthesize an input event (click, key) with `isTrusted === true`. It can, however, trick the user into producing a real one (clickjacking) |
| Observe values that autofill writes into form fields (after the fill, the data is in the page's DOM) | Read extension-origin iframe contents (cross-origin) |

In the repo, you can see this boundary being exploited as a design tool:

- `windowContext.bitwardenAutofillInit` is stored on the content script's own `window` (`apps/browser/src/autofill/content/bootstrap-autofill-overlay.ts`), which the page cannot see, so the page cannot re-enter the content script through that handle.
- `EventSecurity.isEventTrusted(event)` (`apps/browser/src/autofill/utils/event-security.ts`) is just `event.isTrusted`. **Platform fact:** events created and dispatched by script (`dispatchEvent`, `element.click()`) have `isTrusted === false`. Events the browser dispatches itself have `isTrusted === true`, including real user input and the `message` event produced by any `window.postMessage` call. The code rejects synthetic events in many places, for example the inline-menu cipher click handler (`apps/browser/src/autofill/overlay/inline-menu/pages/list/autofill-inline-menu-list.ts`) and the submit-button handler in `autofill-overlay-content.service.ts`. Two limits matter here:
  - On user input, `isTrusted` proves that a real click or keypress happened. It does not prove the user saw what they clicked, which is why clickjacking defenses (A10) are also needed.
  - On `message` events, such as in `content-message-handler.ts` and the FIDO2 messenger, the check only blocks a hand-built `new MessageEvent(...)`. A page calling `window.postMessage` still passes, so the source and origin checks are what actually filter those messages.
- The FIDO2 content script comments that its Permissions Policy check "runs in the content script's isolated world, so its view of `document.permissionsPolicy` / `document.featurePolicy` cannot be tampered with by page-world script" (`apps/browser/src/autofill/fido2/content/fido2-content-script.ts`).
- Values the page controls are treated as untrusted data. The FIDO2 types are named accordingly:

```ts
export type InsecureCreateCredentialParams = Omit<
  CreateCredentialParams,
  "origin" | "sameOriginWithAncestors"
>;
```

(`apps/browser/src/autofill/fido2/content/messaging/message.ts`.) The comment above it says params from the page script "are created in an insecure environment and should not be trusted", and the content script supplies `origin: globalContext.location.origin` itself (`fido2-content-script.ts`, `respondToCredentialRequest`).

### The one script that deliberately runs in the MAIN world

To offer its own passkeys, Bitwarden overrides `navigator.credentials.create/get`, and that override must live in the page's own world, so Bitwarden must run a script there. In MV3 it registers the page script with `world: "MAIN"` and the companion content script in the default isolated world (for already-open tabs it also injects with `chrome.scripting.ExecutionWorld.MAIN`):

```ts
        {
          id: Fido2ContentScriptId.PageScript,
          js: [Fido2ContentScript.PageScript],
          world: "MAIN",
          ...this.sharedRegistrationOptions,
        },
```

(`apps/browser/src/autofill/fido2/background/fido2.background.ts`, `updateMv3ContentScriptsRegistration`.) In MV2 there is no `world` option, so `apps/browser/src/autofill/fido2/content/fido2-page-script-delay-append.mv2.ts` creates a `<script>` element whose `src` is `chrome.runtime.getURL("content/fido2-page-script.js")` and prepends it to the page. This is also why that file is in `web_accessible_resources`.

The page script and the content script talk through `window.postMessage` plus a per-request `MessageChannel` (`apps/browser/src/autofill/fido2/content/messaging/messenger.ts`). The receiver rejects opaque-origin (sandboxed) contexts, untrusted events, a mismatched `event.origin`, and messages without a transferred port. Because the page script and the page share a world and an origin, those checks cannot tell the page script apart from other page code. That is why the content script treats everything from that side as untrusted data and fills in `origin` itself. Because the page script lives in the page's world, **anything it holds is visible to the page**. The repo treats it as a hostile environment. When the script was added as a `<script>` element (the MV2 path), it also removes that element after loading (`document.currentScript` handling in `fido2-page-script.ts`).

Separately from the MAIN-world script, remember that anything the extension writes into the DOM (for instance a filled password in an `<input>`) is readable by page scripts. That is inherent to autofill.

## A8. Message passing

There are several different channels. Learn to tell them apart.

### A8.1 One-shot runtime messages: `chrome.runtime.sendMessage`

A content script (or any extension page) sends a message; the background receives it with a `sender` object describing where it came from. Content scripts use the raw helper:

```ts
export async function sendExtensionMessage(
  command: string,
  options: Record<string, any> = {},
): Promise<any> {
```

(`apps/browser/src/autofill/utils/index.ts`.) The convention across the code is a message object with a string `command` field plus fields. Receivers dispatch on `message.command`.

`sender` is the key security object. Platform fact: `sender.tab` is present when the message came from a content script or extension frame inside a tab, and `sender.frameId` says which frame. In `apps/browser/src/autofill/background/overlay.background.ts`:

```ts
  private async withSenderTab(
    sender: chrome.runtime.MessageSender,
    action: (tab: chrome.tabs.Tab) => void | Promise<void>,
  ): Promise<void> {
    if (sender.tab === null || sender.tab === undefined) {
      this.logService.error("Extension message handler called without sender.tab");
      return;
    }
```

Key habit to notice: the background derives the tab and frame from the browser-supplied `sender`, not from fields inside the message body (the body is whatever the sending context chose to put there). The code is not uniform about this. For instance, `runtime.background.ts` takes `frameId` from `sender.frameId` but passes `tab: msg.tab` and `details: msg.details` from the body into `AutofillOrchestrator`, and `collectPageDetailsFromTab$` was written to read the frame id from the stamped sender (`getWebExtSender`) rather than from the body (see A8.2).

### A8.2 The hardened internal messaging layer

The repo has an internal abstraction: `MessageSender` and `MessageListener` in `libs/messaging/src/message.sender.ts` and `message.listener.ts`, re-exported through `libs/common/src/platform/messaging/index.ts` (`export * from "@bitwarden/messaging";`). Concretely:

- `MessageSender.send(command, payload)` is the typed way to send. The browser's cross-context version is `ChromeMessageSender` (`apps/browser/src/platform/messaging/chrome-message.sender.ts`), which wraps `chrome.runtime.sendMessage` and handles two expected errors ("Receiving end does not exist", "message port closed") by logging at debug level instead of warning.
- `MessageSender.combine(...)` fans out to several senders. `MainBackground` uses `MessageSender.combine(this.#intraprocessMessageSender, new ChromeMessageSender(this.logService))` so a message reaches both same-context and other-context listeners.
- `IntraprocessMessageSender` (`libs/messaging/src/intraprocess-message.sender.ts`) is a same-context channel. Its subject is a `#messages` private field because, per the source comment, "the subject has to be unreachable at runtime", and its docs warn that it "attests where a message came from, never what it contains".
- `fromChromeRuntimeMessaging()` (`apps/browser/src/platform/utils/from-chrome-runtime-messaging.ts`) turns `chrome.runtime.onMessage` into an observable at the ingest boundary, tags each message as external with `tagAsExternal`, and stamps the browser's sender onto it:

```ts
      // Stamp the browser-authoritative sender under a private symbol so consumers that need
      // trustworthy provenance can read it via `getWebExtSender`.
      stampWebExtSender(message, sender);
```

- `getWebExtSender(message)` (`apps/browser/src/platform/utils/web-ext-sender.ts`) reads that stamp. Its docs say to prefer it over any `webExtSender` property in the message body, "that property is part of the body and a page can populate it". The stamp is non-writable and non-configurable, and it returns `undefined` for same-context messages, so callers must "fail closed".
- `isExternalMessage(message)` (`libs/messaging/src/is-external-message.ts`) tells you whether a message crossed a context boundary. Its warning: absence of the tag "is not evidence of trustworthiness". Consumers use it like this:

```ts
    this.messageListener
      .messages$(RETRY_WHEN_UNLOCK_COMPLETED)
      .pipe(filter((message) => !isExternalMessage(message)))
```

(`apps/browser/src/autofill/background/overlay.background.ts`, `setupExtensionListeners`.) A message that should only come from inside the background is accepted only when it arrived over the in-process channel.

Two lint rules push developers toward these abstractions: `**/platform/messaging/**` and `**/platform/**/internal` are forbidden import patterns in `eslint.config.mjs` (`buildNoRestrictedImports`), which is why you see `// eslint-disable-next-line no-restricted-imports` before imports of `tagAsExternal` and `getCommand`.

### A8.3 Checking who sent it: `senderIsInternal`

```ts
    if (!urlOriginsMatch(extensionUrl, sender.origin)) {
      ...
      return false;
    }

    // frameId is absent for popups, so use an 'in' check rather than direct comparison.
    if ("frameId" in sender && sender.frameId !== 0) {
```

(`apps/browser/src/platform/browser/browser-api.ts`, `senderIsInternal`; condensed to the two checks.) It returns true only when `sender.origin` equals the extension's own origin and the sender is not a sub-frame (`frameId` is either absent, as for popups, or `0`). Its doc calls it "a best-effort check that relies on the browser correctly populating `sender.origin`". Both conditions matter:

- A content script can open a port with any name it likes, but its `sender.origin` is the web page's origin, so the origin check rejects it.
- An extension page embedded inside a web page, such as the inline menu's `menu.html`, does have the extension origin. But it sits in a sub-frame, so the `frameId` check rejects it.

Callers run this check before accepting a port: `apps/browser/src/platform/storage/background-memory-storage.service.ts`, `apps/browser/src/platform/services/local-backed-session-storage.service.ts`, `apps/browser/src/platform/services/task-scheduler/background-task-scheduler.service.ts`, and `apps/browser/src/platform/services/popup-view-cache-background.service.ts`.

### A8.4 Long-lived ports: `chrome.runtime.connect` / `onConnect`

A port is a persistent channel between two contexts. A port has a `name`, and the receiver sees `port.sender`. Examples:

- **Content script liveness.** Every injected autofill script opens a port named `autofill-injected-script-port` (`AutofillPort.InjectedScript`, `apps/browser/src/autofill/enums/autofill-port.enum.ts`) via `setupExtensionDisconnectAction` (`utils/index.ts`). The background learns which `(tab, frame)` pairs are alive from those ports, and the content script learns the extension was reloaded or unloaded when `onDisconnect` fires, so it can clean up (`DefaultAutofillLifecycleService.handleInjectedScriptPortConnection` in `apps/browser/src/autofill/services/autofill-lifecycle.service.ts`).
- **State sharing.** `ForegroundMemoryStorageService` (`apps/browser/src/platform/storage/`) in the popup connects a port to the background's memory store. In MV2, that store is `BackgroundMemoryStorageService`, which backs all popup memory state. In MV3, the popup reads ordinary memory state straight from `chrome.storage.session`, and uses the port only for the `memory-large-object` store (`LocalBackedSessionStorageService`). The wiring is in `apps/browser/src/popup/services/services.module.ts`, `OBSERVABLE_MEMORY_STORAGE` / `OBSERVABLE_LARGE_OBJECT_MEMORY_STORAGE`.
- **Inline menu.** Four named ports exist, defined in `apps/browser/src/autofill/enums/autofill-overlay.enum.ts`:

```ts
export const AutofillOverlayPort = {
  Button: "autofill-inline-menu-button-port",
  ButtonMessageConnector: "autofill-inline-menu-button-message-connector",
  List: "autofill-inline-menu-list-port",
  ListMessageConnector: "autofill-inline-menu-list-message-connector",
} as const;
```

The `Button`/`List` ports are opened by the content script (`AutofillInlineMenuIframeService`, once its iframe has loaded). The `...MessageConnector` ports are opened by the extension-origin container iframe (`menu.html`). The background ignores ports with unknown names. For any message arriving on these ports, it runs a handler only if the message carries the right **port key** and names a command in that connector's handler table:

```ts
    const tabPortKey = this.portKeyForTab[tabId];
    if (!tabPortKey || tabPortKey !== message?.portKey) {
      return;
    }
```

(`apps/browser/src/autofill/background/overlay.background.ts`, `handleOverlayElementPortMessage`.) The key is a per-tab random string, created in `handlePortOnConnect` the first time a `Button`/`List` port connects from that tab and reused until the tab's key is deleted: `this.portKeyForTab[port.sender.tab.id] = generateRandomChars(12)`, where `generateRandomChars` (`apps/browser/src/autofill/utils/index.ts`) uses `crypto.getRandomValues` and maps bytes onto 26 lowercase letters (about 56 bits for 12 characters, with a slight modulo bias; I note this as an observation, not a finding). The key is sent in the `initAutofillInlineMenu*` message over the content-script port; the content script relays it into the iframe with `postMessage` targeted at the extension origin (A8.6). A web page never sees that port, and the messages to the iframe name the extension origin as the target, so the page has no direct channel to learn it.

### A8.5 Messages to a specific tab and frame: `tabs.sendMessage` with `frameId`

The background addresses content scripts like this (`apps/browser/src/autofill/services/autofill.service.ts`, `doAutoFill`):

```ts
        void BrowserApi.tabSendMessage(
          tab,
          {
            command: options.autoSubmitLogin ? "triggerAutoSubmitLogin" : "fillForm",
            fillScript: fillScript,
            ...
          },
          { frameId: pd.frameId },
        );
```

`BrowserApi.tabSendMessage` (`browser-api.ts`) wraps `chrome.tabs.sendMessage(tab.id, obj, options, cb)`, where `options.frameId` selects one frame. Platform fact: per the Chrome documentation for this option, it sends to one specific frame "instead of all frames in the tab", so omitting it addresses every frame in the tab. The repo relies on both modes. `collectPageDetailsFromTab$` omits `frameId` to collect from every frame (the popup does this), but passes it when given one, as the page-load path in `AutofillOrchestrator` does. `doAutoFill` always passes `frameId`, so each fill script lands in exactly one frame. Because the fill script goes to a specific frame, **the frame, not just the tab, is the unit of trust** (see A9).

### A8.6 `window.postMessage` between the page and extension iframes

`postMessage` is the standard way for frames to talk across origins. The receiver gets `event.source`, `event.origin`, and `event.data`. Three sets of code use it:

1. **Web vault to extension.** `apps/browser/src/autofill/content/content-message-handler.ts` listens for `message` events on the page `window`. It is intentionally reachable by page script, because the Bitwarden web vault is a page. It drops hand-built (untrusted) events and messages whose `source` is not this window, and handles only a fixed set of commands. It also attaches a `referrer`, which is the hostname taken from the browser-supplied `event.origin`, for the background to verify:

```ts
  if (!EventSecurity.isEventTrusted(event) || source !== window || !data?.command) {
    return;
  }
```

The background then checks `referrer` with `isValidVaultReferrer` (`apps/browser/src/platform/utils/valid-vault-referrer.ts`) in `runtime.background.ts` for commands such as `authResult`. That file's own doc notes the allowlist is client-side only and can miss a self-hosted vault with a drifted hostname. The comment on `forwardCommands` is also a good read: "Do not add commands containing sensitive information to this set."

2. **Content script to the extension iframe.** `AutofillInlineMenuIframeService.postMessageToIFrame` always passes the extension origin as the target origin:

```ts
    this.iframe.contentWindow?.postMessage(
      { portKey: this.portKey, ...message },
      this.extensionOrigin,
    );
```

(`apps/browser/src/autofill/overlay/inline-menu/iframe-content/autofill-inline-menu-iframe.service.ts`.) **Platform fact:** the second argument is the `targetOrigin`. The browser delivers the message only if the receiving window's origin matches, so if the page navigated or swapped that iframe for something else, the new document would not receive the data. A related platform fact matters on the receiving side: a message posted by a content script arrives with `event.source` set to the page's window and `event.origin` set to the page's origin. Those are exactly the values page script would produce, so a receiver cannot use them to tell the content script apart from the page.

3. **Inside the iframe stack.** See [C4](#c4-the-inline-menu-overlay). The container checks `event.source` identity, requires a `portKey` to be present (the background checks its value), checks a session token on messages from the inner iframe, and forwards only an allowlist of commands.

### A8.7 Page-script bridge using `MessageChannel`

See the FIDO2 messenger in A7. Each request uses a private `MessageChannel` port transferred with `postMessage`, so the response goes back only to the requester.

## A9. Frames

**Platform fact:** A page can contain nested browsing contexts via `<iframe>`. The top-level one is the *top frame*; others are *sub-frames*. Each frame has its own document, origin, and JavaScript global. The same-origin policy says script in one origin cannot read the DOM of another origin; `window.postMessage` is the sanctioned cross-origin channel. An iframe with the `sandbox` attribute gets restricted privileges, and without `allow-same-origin` it gets an opaque origin, which serializes as the string `"null"`. An iframe sandboxed with *both* `allow-scripts` and `allow-same-origin` keeps its real origin. If that origin is also the parent's, its script can reach up and remove the sandbox entirely.

**In the repo:**

- `frameId`: `0` is the top frame, and each sub-frame gets another id from the browser. The background gets it from `sender.frameId` and uses it to target messages.
- `all_frames: true` in the static content script entry means the trigger script runs in every frame, and `injectDetails.frame: "all_frames"` or a numeric frame id is how dynamic injection targets frames (`buildInjectionDetails` in `browser-script-injector.service.ts`).
- Per-frame bookkeeping: the background keys state by `${tabId}:${frameId ?? -1}` (`monitorFrameKey` in `autofill-lifecycle.service.ts`). `AutofillOrchestrator` serializes fills per tab and per frame: it groups requests by `request.tabId`, then by `request.frameId ?? -1`, and runs each group with `concatMap` (`apps/browser/src/autofill/background/autofill-orchestrator.ts`).
- Sub-frame geometry: the inline menu is drawn only in the top frame, so for a field inside an iframe the background walks up the frame tree using `getFrameDetails` and `getSubFrameOffsets` messages to add up pixel offsets, with a depth cap `MAX_SUB_FRAME_DEPTH = 8` (`overlay.background.ts`, `buildSubFrameOffsets`; the constant is in `autofill-overlay.enum.ts`).
- Sandboxed iframes: autofill refuses to fill them.

```ts
export function currentlyInSandboxedIframe(): boolean {
  if (String(self.origin).toLowerCase() === "null" || globalThis.location.hostname === "") {
    return true;
  }
  ...
```

(`apps/browser/src/autofill/utils/index.ts`.) The first test catches opaque origins and documents with no hostname (for example `file:` pages). The elided part reads `globalThis.frameElement`, which is only available when the parent is same-origin, and looks at its `sandbox` attribute. It treats the frame as sandboxed unless the attribute includes both `allow-scripts` and `allow-same-origin`. The companion CVE note describes this as a later relaxation, so that sites that frame their own login form that way can be filled. The function is called from `InsertAutofillContentService.fillForm`, from `AutofillOverlayContentService.setupOverlayListeners` (so fields in sandboxed frames get no inline-menu listeners), and from both FIDO2 scripts.

### Why frames matter for autofill

The browser tells the background the tab's top URL (`tab.url`). Credentials are chosen by matching that URL to saved logins. But the form being filled may sit in a sub-frame served from a different site than the tab. The sub-frame's own URL reaches the background as `details.url` in its page details. The frame's content script reads that from `location.href`. Page script can change that value only within its own origin, for example with `history.pushState`. If autofill blindly filled it, a page that matches your saved login could embed an attacker's form in an iframe and receive your password. Part C explains the safeguard (`untrustedIframe`). The companion note [`vuln-cve-2018-25081-iframe-autofill.md`](./vuln-cve-2018-25081-iframe-autofill.md) covers the vulnerability class in depth.

## A10. DOM concepts autofill relies on

Only items I found in the code are listed.

### Shadow DOM (open and closed)

**Platform fact:** A *shadow root* attaches an encapsulated DOM subtree to a host element. In open mode (`mode: "open"`), page script can reach it via `element.shadowRoot`; in closed mode (`mode: "closed"`), `element.shadowRoot` returns `null`, so page scripts that do not hold the root reference cannot reach inside. `querySelector` from outside does not descend into it, and page style rules do not match elements inside it. Two things still cross the boundary:

- Inherited CSS properties (such as `color`, `font`, and `visibility`) flow from the host into the shadow tree.
- Page styles can always target the host element itself.

That is why Bitwarden resets its host elements with `all: initial`. Closed mode is an encapsulation feature, not a full security boundary. For example, same-world script could patch `attachShadow` before the root is created. It works well here because the root is created by the content script, whose isolated world the page cannot patch (A7).

Bitwarden uses closed roots on its own injected UI:

```ts
    const shadow: ShadowRoot = element.attachShadow({ mode: "closed" });
    shadow.prepend(style);
```

(`apps/browser/src/autofill/overlay/inline-menu/iframe-content/autofill-inline-menu-iframe-element.ts`; the `style` node is an internal stylesheet for pseudo-elements, see A10 `pointer-events`.) The notification bar does the same with `mode: "closed"` and `delegatesFocus: true` on a fixed-name host, `bit-notification-bar-root` (`apps/browser/src/autofill/overlay/notifications/content/overlay-notifications-content.service.ts`). Pages inside the extension iframes (`AutofillInlineMenuPageElement`) also attach closed roots.

Bitwarden must also **read other sites' shadow DOM** to find login fields. `DomQueryService.getShadowRoot` (`apps/browser/src/autofill/services/dom-query.service.ts`) tries `node.shadowRoot` first, then extension-only APIs that can see closed roots:

```ts
    if ((chrome as any).dom?.openOrClosedShadowRoot) {
      try {
        return (chrome as any).dom.openOrClosedShadowRoot(node);
      } catch {
        return null;
      }
    }

    // Firefox-specific equivalent of `openOrClosedShadowRoot`
    return (node as any).openOrClosedShadowRoot;
```

Platform fact: these extension-only APIs let a content script see inside closed shadow roots; that is a privilege page scripts do not have. There is a lot of machinery here (`apps/browser/src/autofill/services/shadow-host-hydration-tracker.ts`, `dom-query.service.ts`) because `attachShadow()` emits no mutation record, so late-attached roots must be discovered separately. `getShadowRoot` also skips the extension's own hosts (`isOwnedShadowHost`) so the autofill scanner does not scan its own UI.

### Custom elements

**Platform fact:** `customElements.define(name, class extends HTMLElement)` registers a new tag. Custom element names must start with a lowercase ASCII letter and contain a hyphen.

The inline menu host is a custom element with a random name per page load, defined on the fly:

```ts
    const self = this;
    globalThis.customElements?.define(
      customElementName,
      class extends HTMLElement {
        constructor() {
          super();
          self.buttonIframe = new AutofillInlineMenuButtonIframe(this);
        }
      },
    );
```

(`apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts`, `createButtonElement`; the elided line just above is `const customElementName = this.generateRandomCustomElementName();`, which is then passed to `define` and later to `document.createElement`.) The random name (8 to 12 letters with hyphens, from `generateRandomCustomElementName` in `apps/browser/src/autofill/utils/index.ts`, which uses `Math.random`) stops a page from targeting the host by a fixed, known tag name in CSS or `querySelector`. It is an obstacle, not a secret: the element is in the page DOM, so a page can still find it by other means, for example by watching for newly added elements. In Firefox the code uses a plain `<div>` instead (`isFirefoxBrowser` branch). The extension-origin pages define fixed element names (`customElements.define(AutofillOverlayElement.List, AutofillInlineMenuList)` in `bootstrap-autofill-inline-menu-list.ts`), which is fine because those pages are isolated from the website. The notification and some in-page UI are built with Lit (`apps/browser/src/autofill/content/components/`), per `.claude/rules/autofill-content-scripts.md`.

### `MutationObserver`

**Platform fact:** observes DOM changes (attribute changes, child list changes, subtree changes) asynchronously.

Used in two opposite directions:

- **To discover page fields** that appear after load (`CollectAutofillContentService`, with `AUTOFILL_ATTRIBUTES` from `libs/common/src/autofill/constants/index.ts`, and `DomQueryService` enrolling shadow roots).
- **To defend the extension's own UI** against page tampering: in the content script, observers watch the host element's attributes, the container's children, and `<html>`/`<body>` attributes. For example:

```ts
      if (record.attributeName === "style") {
        element.removeAttribute("style");
        this.updateCustomElementDefaultStyles(element);

        continue;
      }
```

(`autofill-inline-menu-content.service.ts`, `handleInlineMenuElementMutationObserverUpdate`.) A foreign `popover` value is reset to `"manual"`, a changed `style` is wiped and re-applied, and any other added attribute is removed. The host-element and container observers share an "excessive iterations" circuit breaker (`isTriggeringExcessiveMutationObserverIterations`). The counter resets only after 2 seconds with no callbacks, and if it passes 100, the menu closes. The `<html>`/`<body>` observers do not use the breaker; they re-run `checkPageRisks` instead. The iframe service has its own observer that resets the iframe's `style` and attributes to defaults. It force-closes the menu once it has had to undo 10 foreign changes (`foreignMutationsCount`), or after more than 20 observer callbacks without a 2-second pause (`autofill-inline-menu-iframe.service.ts`).

### `IntersectionObserver`

Used in two places:

- **As a measuring tool.** To place the menu next to a field, `getBoundingClientRectFromIntersectionObserver` creates an observer on the focused field to get its rectangle, and falls back to `getBoundingClientRect()` if the result is empty (`apps/browser/src/autofill/services/autofill-overlay-content.service.ts`). It uses `threshold: 0.9999` with a comment: "Safari doesn't seem to function properly with a threshold of 1".
- **To re-check visibility.** `CollectAutofillContentService` observes fields that were not viewable when collected and recomputes `viewable` when they scroll into view (`handleFormElementIntersection` in `collect-autofill-content.service.ts`).

### The top layer and the Popover API

**Platform fact:** The *top layer* is a browser-managed stacking layer that sits above all normal content regardless of `z-index`. Modal `<dialog>` elements (`:modal`), elements shown via the Popover API, and fullscreen elements are placed in it. Within the top layer, `z-index` has no effect: later additions render on top of earlier ones. So any page that can add its own top-layer element after Bitwarden's can cover it, and the only counter is to re-add (re-promote) one's own element. `popover="manual"` means the element is shown and hidden only through script (`showPopover()` / `hidePopover()`) and is not light-dismissed.

In the repo this is the main weapon against being covered by a page:

```ts
      this.appendInlineMenuElementToDom(this.buttonElement);
      this.updateInlineMenuElementIsVisibleStatus(AutofillOverlayElement.Button, true);
      this.buttonElement.showPopover();
```

(`autofill-inline-menu-content.service.ts`, `appendButtonElement`.) The elements are created with `setAttribute("popover", "manual")`. The service also:

- finds other top-layer content (`:modal`, `:popover-open`, and optionally `[popover], dialog`) via `getUnownedTopLayerItems`,
- re-promotes itself (`hidePopover()` then `showPopover()`) in `refreshTopLayerPosition` so it remains on top of a page's new dialog,
- appends the menu inside an open modal `<dialog>` or an ARIA modal (`[role="dialog"]` or `[role="alertdialog"]` with `aria-modal="true"`) that contains the focused field, otherwise to `document.body` (`getInlineMenuContainerElement`), so the page's focus trap does not steal focus from the menu,
- counts re-promotions and turns the inline menu off entirely when a page fights too aggressively:

```ts
        // Set inline menu to be off; page is aggressively trying to take top position of top layer
        this.inlineMenuEnabled = false;
        void this.checkPageRisks();

        const warningMessage = chrome.i18n.getMessage("topLayerHijackWarning");
        globalThis.window.alert(warningMessage);
```

(`checkAndUpdateRefreshCount`.) Limits are in `experienceValidationBackoffThresholds` at the top of the file: top layer count 5 per 5000 ms, popover attribute count 10 per 5000 ms.

### CSS stacking, z-index, opacity, `pointer-events`

**Platform fact:** `z-index` orders positioned elements within a stacking context; `opacity` below 1 creates a stacking context and makes the element and all descendants translucent; `pointer-events: none` makes an element ignore clicks so they hit whatever is beneath it.

- The extension's host elements use the maximum 32-bit `z-index`: `zIndex: "2147483647"` in `customElementDefaultStyles` (content service) and in the iframe's `iframeStyles` (iframe service), together with `all: "initial"` and `position: "fixed"` to reset inherited page styles.
- **Opacity risk.** Because opacity applies to descendants, a transparent `<html>` or `<body>` would make the menu invisible while still clickable, a UI-redressing building block. The content service checks it:

```ts
      // These are computed style values, so we don't need to worry about non-float values
      // for `opacity`, here
      const elementOpacity = globalThis.window.getComputedStyle(element)?.opacity || "0";

      // Any value above this is considered "opaque" for our purposes
      const opacityThreshold = 0.6;
```

(`getPageIsOpaque`; if any of `html`/`body` is at or below 0.6 the menu is closed in `checkPageRisks`.)
- **Visibility checks for target fields.** `DomElementVisibilityService.isElementHiddenByCss` (`apps/browser/src/autofill/services/dom-element-visibility.service.ts`) rejects a field when its own opacity or any ancestor's opacity is below 0.1, when it has `display: none`, `visibility: hidden`/`collapse`, or a clip-path that hides it, and `isElementOutsideViewportBounds` rejects fields smaller than 10 px or off-screen. `formFieldIsNotHiddenBehindAnotherElement` uses `elementFromPoint` at the field's center, so a field covered by another element is not "viewable".
- **Another element covering the menu.** `verifyInlineMenuIsNotObscured` uses `document.elementFromPoint` at the center of the menu and closes it if the topmost element is a foreign last child of the container.
- **`pointer-events`.** The iframes set `pointerEvents: "auto"` explicitly as part of their reset styles, and the pseudo-element reset in `autofill-inline-menu-iframe-element.ts` sets `pointer-events: all !important` on `::backdrop/::before/::after`, so a page cannot style those pseudo-elements to interfere. I did not find code that reads the page's `pointer-events` values.

The 2025 DOM clickjacking issue is covered separately in [`vuln-2025-dom-clickjacking.md`](./vuln-2025-dom-clickjacking.md). Many of the defenses above exist to counter it.

## A11. `BrowserApi` and cross-browser differences

`apps/browser/CLAUDE.md` makes this a rule: **never call `chrome.*` or `browser.*` directly in business logic; use `BrowserApi`** (`apps/browser/src/platform/browser/browser-api.ts`). The exception is injected content scripts (`.claude/rules/autofill-content-scripts.md`: "Content scripts cannot import the `BrowserApi` abstraction ... direct use of `chrome.*` / `browser.*` APIs ... is expected here").

What `BrowserApi` provides:

- **Detection flags:** `isWebExtensionsApi` (`typeof browser !== "undefined"`), `isSafariApi`, `isChromeApi`, `isFirefox`, `isFirefoxOnAndroid`, `isManifestVersion(2|3)`.
- **Promise wrappers** over callback APIs (`tabsQuery`, `getFrameDetails`, `getAllFrameDetails`, `tabSendMessage`, `createNewTab`).
- **Manifest-version forks:** `executeScriptInTab` uses `chrome.scripting.executeScript` in MV3 (with `injectImmediately` from `runAt === "document_start"`) and `chrome.tabs.executeScript` in MV2; `registerContentScriptsMv2/Mv3`; `executeFunctionInTab`.
- **Listener management:** `BrowserApi.addListener` / `removeListener`. In Safari popups it records listeners and removes them on `pagehide`, to avoid memory leaks:

```ts
    if (BrowserApi.isSafariApi && !BrowserApi.isBackgroundPage(self)) {
      BrowserApi.trackedChromeEventListeners.push([event, callback]);
      BrowserApi.setupUnloadListeners();
    }
```

- **Safari tab bug workaround:** `tabsQueryFirstCurrentWindowForSafari`. Its doc says Safari "sometimes returns >1 tabs unexpectedly" for a current-window query; the helper picks the tab matching the current window id. `apps/browser/CLAUDE.md` makes this a rule too.
- **Side panel helpers:** `isSidePanelApiSupported`, `openSidePanel`, `setSidePanelOptions`.
- **Sender checks:** `senderIsInternal` (A8.3).

Other cross-browser differences visible in code:

| Area | Firefox | Chrome-family | Safari |
| --- | --- | --- | --- |
| Manifest | MV3 `background.scripts` (`__firefox__background`); MV2 builds exist | MV3 service worker | MV2 and MV3 build targets; Safari-specific permission list |
| Offscreen document | Not built (`browser !== "firefox"`) | Built for MV3 | `offscreen` is in the Safari MV3 permissions list; I did not verify what Safari does with it |
| Sandbox pages in manifest | Removed (`__firefox__sandbox: null`) | Present | Present |
| Closed shadow root access | `node.openOrClosedShadowRoot` | `chrome.dom.openOrClosedShadowRoot(node)` | not checked; any browser without `chrome.dom` falls through to `node.openOrClosedShadowRoot` (`dom-query.service.ts`) |
| Inline menu host element | Plain `div` (`isFirefoxBrowser`) | Random custom element | Random custom element |
| Popup event listener cleanup | No tracking in `BrowserApi` | No tracking in `BrowserApi` | Tracked and removed on `pagehide` (`apps/browser/CLAUDE.md`: "Safari requires manual cleanup to prevent memory leaks") |
| Window messages in content scripts | | | A code comment in `content-message-handler.ts` says Safari seems to mishandle window message events from content scripts when the listener is registered inside a class, so the listeners are registered at the top level of that file |

---

# Part B: Bitwarden architecture concepts

## B1. Monorepo layout and dependency rules

From `.claude/CLAUDE.md`:

- `apps/<client>/`: single-client code (`browser`, `cli`, `desktop`, `web`). Each is self-contained.
- `libs/common/`: shared by **all** clients including the non-Angular CLI. "No Angular APIs here: no `@Injectable`, no `inject()`, no decorators."
- `libs/angular/`: shared by Angular clients (browser, desktop, web). Angular is allowed.
- Other libs (`ui`, `platform`, `key-management`, `vault`, `state`, `messaging`, `unlock`, and so on) are domain-scoped and follow the same Angular or non-Angular split.
- `bitwarden_license/` holds commercial variants (for example `bit-browser`, and the `commercial-*` build targets in `apps/browser/project.json`).
- Guidance: "place it as deep and as narrow as possible. Promote to a shared `libs/` only when a second client needs it."

The rules are **enforced by lint**, not just convention (`eslint.config.mjs`):

- `import/no-restricted-paths`: `libs/**` may not import from `apps/**` or `bitwarden_license/**`; `libs/common` may not import Angular or `libs/node`.
- `COMMON_FORBIDDEN_PACKAGES`: files under `libs/common/src/**` may not import `@bitwarden/angular`, `@bitwarden/auth`, `@bitwarden/key-management`, `@bitwarden/vault`, and so on. Common is at the base of the dependency graph.
- A shared group in `buildNoRestrictedImports`: no relative imports across libs (`**/src/**/*`) and no imports of `**/platform/**/internal` or `**/platform/messaging/**`.
- A `LEGACY_CRYPTO_RESTRICTED_PATTERN` forbids new imports of `@bitwarden/legacy-crypto`, with the message: "holds crypto primitives that are being retired in favour of the SDK. Do not add new imports if possible — implement the operation in the SDK and contact the Key Management team."

Libraries are consumed through TypeScript path aliases in `tsconfig.base.json`, for example `"@bitwarden/common/*": ["./libs/common/src/*"]` and `"@bitwarden/messaging": ["./libs/messaging/src/index.ts"]`. Note that `libs/common/src/platform/state/index.ts` and `libs/common/src/platform/messaging/index.ts` are thin re-exports of `@bitwarden/state` and `@bitwarden/messaging`, kept for import-path compatibility.

For autofill content scripts there is an extra rule: they are not part of the Angular app, so no Angular patterns, no DI, no RxJS Observable Data Services (`.claude/rules/autofill-content-scripts.md`).

## B2. Angular in the popup, and manual construction in the background

**The popup (Angular).** `apps/browser/src/popup/main.ts` calls `platformBrowser().bootstrapModule(AppModule, ...)`. The app is a hybrid:

- Older pieces are NgModule-based. `AppComponent` has `standalone: false` and `apps/browser/src/popup/app.module.ts` is an `@NgModule` with a long `imports` list.
- Newer components are standalone: they list their own `imports: [...]` and set `changeDetection: ChangeDetectionStrategy.OnPush`. A small example with signals is `apps/browser/src/vault/popup/components/vault/fill-assist-active-banner/fill-assist-active-banner.component.ts` (uses `signal`, `computed`, and `toSignal` to bridge an RxJS stream). I infer standalone-ness from the `imports` array on the decorator; no `standalone: true` flag is set there.
- Routing is in `apps/browser/src/popup/app-routing.module.ts`. The root route uses an auth-status redirect guard:

```ts
    canActivate: [
      popupRouterCacheGuard,
      redirectGuard({ loggedIn: "/tabs/current", loggedOut: "/login", locked: "/lock" }),
    ],
```

The guards (`authGuard`, `lockGuard`, `redirectGuard`, and so on) come from `libs/angular/src/auth/guards/`. The `locked` route is the visible effect of the auth status in [B4](#b4-accounts-auth-status-vault-timeout-and-lock).

**Dependency injection in the popup.** Angular's DI container is configured in `apps/browser/src/popup/services/services.module.ts` using `safeProvider({ provide, useClass/useFactory, deps })`, a wrapper that type-checks that the `deps` array matches the constructor. The repo rules (`.claude/rules/angular.md`) say to use `inject()` for new code and to wrap providers in `safeProvider`. (That rule file says to import `safeProvider` from `@bitwarden/ui-common`; `services.module.ts` imports it from `@bitwarden/angular/platform/utils/safe-provider`. I did not investigate which is current.)

The popup builds its **own copies** of services (for example its own `AutofillService` provider, and a `ForegroundMemoryStorageService` for memory state) and synchronizes through shared storage and messages, rather than calling the background's objects. That is why the memory-storage port pair exists.

**The background (no Angular DI).** `MainBackground` (`apps/browser/src/background/main.background.ts`) is one very large class with a very large constructor that builds every service by hand with `new`, in dependency order:

```ts
    this.autofillService = new AutofillService(
      this.cipherService,
      this.autofillSettingsService,
      this.totpService,
      this.eventCollectionService,
      this.logService,
      ...
      this.autofillLifecycleService,
    );
```

Reasons are practical: a service worker has no Angular application, `libs/common` services cannot use `@Injectable` anyway, and construction order must be explicit and synchronous. Then `bootstrap()` runs migrations, loads the WASM SDK (`sdkLoadService.loadAndInit()`), tries auto-unlock, and calls `init()` on each background piece (`runtimeBackground`, `overlayBackground` via its init, `commandsBackground`, `fido2Background`, `autofillOrchestrator`, `vaultTimeoutService`, and others).

**Reading tip:** when something in the popup "just works," look at `services.module.ts`. When something in the background works, look at the constructor sequence in `main.background.ts`. Same abstractions (`CipherService`, `AuthService`), two wirings.

## B3. RxJS and the State Provider framework

**RxJS as the main state pattern.** Services expose state as `Observable`s (a `$` suffix by convention): `activeAccount$`, `activeAccountStatus$`, `autofillOnPageLoad$`, `inlineMenuVisibility$`. Consumers use `firstValueFrom(x$)` for a one-time read and `pipe(...)` for reactive flows. In the autofill code you will see `switchMap`, `combineLatest`, `groupBy`, `concatMap`, `withLatestFrom`, `takeUntil`, and `shareReplay`. `AutofillOrchestrator.init()` (`apps/browser/src/autofill/background/autofill-orchestrator.ts`) is a good example: one subject of fill requests, grouped by tab and frame, run one at a time with `concatMap`.

**The State Provider framework** lives in `libs/state` (`@bitwarden/state`). It's documented in `libs/state/README.md`. Key types (all under `libs/state/src/core/`):

- `StateDefinition` (`state-definition.ts`): a named storage namespace with a default location of `"disk"` or `"memory"`, and optional per-client overrides.
- `KeyDefinition` (`key-definition.ts`): one key inside that namespace for global state.
- `UserKeyDefinition` (`user-key-definition.ts`): per-user state. It requires `clearOn: ["lock" | "logout"]`.
- `StateProvider` (`state.provider.ts`): the entry point (`getUserState$`, `getGlobal`, `getActive`, `getUser`, and so on). Implementations are in `libs/state-internal/src/` (`DefaultStateProvider`), and `MainBackground` constructs them.
- The master list of definitions is `libs/state/src/core/state-definitions.ts`, for example `AUTOFILL_SETTINGS_DISK = new StateDefinition("autofillSettings", "disk")`.

`libs/common/src/platform/state/index.ts` re-exports `@bitwarden/state` so most code imports from `@bitwarden/common/platform/state`.

A real definition ties it together. The user key is held in memory only, and cleared on lock and logout:

```ts
export const USER_KEY = new UserKeyDefinition<UserKey>(CRYPTO_MEMORY, "userKey", {
  deserializer: (obj) => SymmetricCryptoKey.fromJSON(obj) as UserKey,
  clearOn: ["logout", "lock"],
  // Prevents the state from caching and rxjs observable becoming hot observable.
  cleanupDelayMs: 0,
});
```

(`libs/common/src/key-management/state-definitions.ts`.) And an autofill setting that survives lock:

```ts
const AUTO_COPY_TOTP = new UserKeyDefinition(AUTOFILL_SETTINGS_DISK, "autoCopyTotp", {
  deserializer: (value: boolean) => value ?? true,
  clearOn: [],
});
```

(`libs/common/src/autofill/services/autofill-settings.service.ts`.)

### Storage locations

`StorageLocation` is `"disk" | "memory"` (`libs/storage-core/src/storage-location.ts`), and `ClientLocations` lets a state definition override per client; for the browser it also allows `"memory-large-object"` and `"disk-backup-local-storage"` (`libs/storage-core/src/client-locations.ts`). The browser maps these in `BrowserStorageServiceProvider` (`apps/browser/src/platform/storage/browser-storage-service.provider.ts`):

| Location | Browser backing store (MV3) | Notes |
| --- | --- | --- |
| `disk` | `BrowserLocalStorageService` over `chrome.storage.local` (`apps/browser/src/platform/services/browser-local-storage.service.ts`) | Persistent. The code comment in `MainBackground` says secure storage "is not supported in browsers, so we use local storage and warn users when it is used". |
| `memory` | `BrowserMemoryStorageService` over `chrome.storage.session` (`browser-memory-storage.service.ts`) | Survives a service worker restart but not a browser restart. |
| `memory-large-object` | `LocalBackedSessionStorageService` | Encrypted copy in local storage, keyed by an ephemeral session key held in session storage. |
| `disk-backup-local-storage` | `PrimarySecondaryStorageService` using the offscreen `localStorage` | A secondary copy for data that needs one. |

In MV2 the memory store is a real in-process object, `BackgroundMemoryStorageService`, shared with popups over ports:

```ts
    if (BrowserApi.isManifestVersion(3)) {
      // manifest v3 can reuse the same storage. They are split for v2 due to lacking a good sync mechanism, which isn't true for v3
      this.memoryStorageForStateProviders = new BrowserMemoryStorageService(); // mv3 stores to storage.session
```

(`apps/browser/src/background/main.background.ts`.)

### Why MV3 needs memory state persisted

Because the service worker can be terminated at any time, plain JavaScript variables would be lost, including the in-memory user key, so the vault would appear locked after every worker restart. So "memory" state in MV3 is written to `chrome.storage.session`, an area that the platform keeps only for the browser session and does not write to disk (Platform fact; I rely on this being the documented behavior of `storage.session`). For bigger memory objects, `LocalBackedSessionStorageService` (`apps/browser/src/platform/services/local-backed-session-storage.service.ts`) stores encrypted data in local storage under a key kept only in session storage. Its header comment explains the effect: "When the session key is unavailable, any encrypted items in local storage cannot be decrypted and must be cleared". The same lifecycle problem explains why the lifecycle design says gate timers and monitoring state "are in-memory too" and must be rebuilt on restart (`lifecycle.design.md`, "Routing").

## B4. Accounts, auth status, vault timeout, and lock

**Accounts.** `AccountService` (`libs/common/src/auth/abstractions/account.service.ts`) holds the account list (`accounts$`), the active account (`activeAccount$`), and last-activity times (`accountActivity$`). Multiple accounts can be logged in; most per-user state is keyed by `UserId`.

**Auth status** is a three-value enum, derived (not stored):

```ts
export enum AuthenticationStatus {
  LoggedOut = 0,
  Locked = 1,
  Unlocked = 2,
}
```

(`libs/common/src/auth/enums/authentication-status.ts`; the file's doc comments define each.) It is computed from two facts, whether there is an access token and whether there is a user key in state:

```ts
        map(([userKey, hasAccessToken]) => {
          if (!hasAccessToken) {
            return AuthenticationStatus.LoggedOut;
          }

          if (!userKey) {
            return AuthenticationStatus.Locked;
          }

          return AuthenticationStatus.Unlocked;
        }),
```

(`libs/common/src/auth/services/auth.service.ts`.) This is the whole security meaning of "locked": **the user key is gone from memory**, so nothing in the vault can be decrypted. Auth status drives behavior everywhere, for example `injectAutofillScripts` only enables autofill-on-page-load when `authStatus === AuthenticationStatus.Unlocked`, and the popup's `redirectGuard` sends users to `/lock` or `/login`.

**Vault timeout.** `VaultTimeoutService` (`libs/common/src/key-management/vault-timeout/services/vault-timeout.service.ts`) checks every 10 seconds (`startCheck`, scheduled through the task scheduler) whether any user should lock. `shouldLock` skips a user whose vault view is focused and active, skips already locked or logged-out users, and otherwise compares last activity with the timeout. Timeout values are the `VaultTimeout` type (`types/vault-timeout.type.ts`): numeric minutes, or the strings `never`, `onRestart`, `onLocked`, `onSleep`, `onIdle`, `custom`. The action on timeout is `VaultTimeoutAction.Lock` or `LogOut`.

**Lock.** `DefaultLockService.lockUser` (`libs/unlock/src/lock.service.ts`) logs out instead if the user cannot lock, otherwise it:

1. wipes decrypted state: `folderService.clearDecryptedFolderState`, `cipherService.clearCache`, `keyService.clearStoredUserKey`, then `stateEventRunnerService.handleEvent("lock", userId)`, which clears every state whose `clearOn` includes `"lock"` (including `USER_KEY`),
2. waits until the auth status reads `Locked`,
3. clears pending clipboard clearing and runs platform actions,
4. sends the `"locked"` message,
5. reloads the process ("Wipe the current process to clear active secrets in memory", per the source comment). `DefaultProcessReloadService.reloadProcess` (`libs/common/src/key-management/process-reload/default-process-reload.service.ts`) skips the reload while any account is still unlocked or an after-first-unlock PIN is active, so with several accounts the wipe happens only once none is unlocked.

In the browser, when the reload does proceed it is a real extension reload:

```ts
    // Wait for the popup to actually close, otherwise the reload leaves behind a
    // zombie popup with an invalidated extension context.
    await BrowserPopupUtils.waitForAllPopupsClose();

    BrowserApi.reloadExtension();
```

(`apps/browser/src/key-management/browser-process-reload.service.ts`.) This is also what produces the "extension context invalidated" situation that content scripts must handle, which is why every content script registers `setupExtensionDisconnectAction`. **Memory hygiene** here means the sensitive material lives in memory only while unlocked and, once no account is unlocked, the whole JS process is torn down. `main.background.ts` `logout` also ends with `await this.processReloadService.reloadProcess();`. Autofill's own lifecycle across these transitions is described in `apps/browser/src/autofill/lifecycle.design.md` ("Logging in", "Logging out", "Locking the vault", "Unlocking the vault" sequences).

## B5. Ciphers, zero-knowledge, and the key hierarchy

### Ciphers (vault items) and their types

A *cipher* is a vault item. Its type is a numeric constant:

```ts
const _CipherType = Object.freeze({
  Login: 1,
  SecureNote: 2,
  Card: 3,
  Identity: 4,
  SshKey: 5,
  BankAccount: 6,
  DriversLicense: 7,
  Passport: 8,
} as const);
```

(`libs/common/src/vault/enums/cipher-type.ts`.) The autofill code mostly deals with `Login`, `Card`, `Identity`, and `SshKey` (see the `switch (options.cipher.type)` in `AutofillService.generateFillScript`).

### `Cipher` vs `CipherView`

Two representations of the same item:

- **`Cipher`** (`libs/common/src/vault/models/domain/cipher.ts`) is the encrypted form, as stored and synced. Text fields are `EncString` values. Its class comment is worth reading: only metadata (id, type, flags, dates, and so on) "are stable to read", the content fields are "Encryption-format internal, not public API", and callers should "Decrypt to a `CipherView` / `CipherListView` to inspect item contents".
- **`CipherView`** (`libs/common/src/vault/models/view/cipher.view.ts`) is the decrypted form, with plain-text `name`, `notes`, `login: LoginView`, `card: CardView`, and so on. Everything in autofill (`AutofillService`, `OverlayBackground`) works on `CipherView`s.

The general naming: `domain` models are encrypted, `view` models are decrypted, `data` models are the JSON as stored/synced, and `api` models are server response shapes.

`Cipher.decrypt` (marked deprecated in favor of SDK-based decryption) shows the per-item key step: if the cipher has a `key`, it is unwrapped with the user or organization key and used for that item's fields:

```ts
    if (this.key != null) {
      const encryptService = Utils.getContainerService().getEncryptService();

      try {
        const cipherKey = await encryptService.unwrapSymmetricKey(this.key, userKeyOrOrgKey);
```

In current code, decryption and encryption of ciphers go through the Rust SDK: `DefaultCipherEncryptionService` (`libs/common/src/vault/services/default-cipher-encryption.service.ts`) calls `this.sdkService.userClient$(userId)` and then `ref.value.vault().ciphers().encrypt(...)` (and `decrypt` variants).

### Zero-knowledge model, at a high level

**What the repo shows:**

- Vault contents are **encrypted and decrypted on the client**. The server stores and returns `Cipher` objects whose sensitive fields are `EncString`s. The client decrypts after unlock, and the decrypted `CipherView` list is held in client memory (cleared on lock with `cipherService.clearCache`).
- The server authenticates the user with a derived value that is not the decryption key: `MasterPasswordAuthenticationHash` is documented as "The Base64-encoded master password authentication hash, that is sent to the server for authentication" (`libs/common/src/key-management/master-password/types/master-password.types.ts`).
- The repo rule in `.claude/CLAUDE.md`: "**NEVER** send unencrypted vault data to API services."

**Key hierarchy, only at the level the code supports.** The chain, from these files:

1. **Master password** (typed by the user) plus a **salt** and **KDF settings** derive a **master key**. `MasterPasswordUnlockData` holds `salt`, `kdf`, and `masterKeyWrappedUserKey`. The KDF is configurable: `master-password.types.ts` imports `PBKDF2KdfConfig` and `Argon2KdfConfig` (from `@bitwarden/legacy-crypto`), and the per-user setting is stored as `KDF_CONFIG` in `libs/common/src/key-management/state-definitions.ts`. The `MasterKey` type is marked `@deprecated Interacting with the master key directly is prohibited. Use a high level function from MasterPasswordService instead.` (`libs/common/src/types/key.ts`).
2. The master key **wraps (encrypts) the user key**: `MasterKeyWrappedUserKey` is a distinct opaque type. The wrapped user key is stored; the user key itself is produced by unwrapping at unlock.
3. The **user key** (`UserKey`) is the main symmetric key. It is what is held in memory while unlocked (`USER_KEY`), and its presence is what makes status `Unlocked`.
4. The user key (or an **organization key** `OrgKey` for org items) **decrypts ciphers**, either directly or by unwrapping a **per-cipher key** (`Cipher.key`) that then encrypts that item's fields.

Other keys exist in the same files (device key, PRF key, provider key, an asymmetric key pair for sharing and org key delivery: see `libs/common/src/types/key.ts` and `libs/key-management/src/abstractions/key.service.ts`), but I am not covering them. `KeyService.userKey$(userId)` (the abstraction in `libs/key-management/src/abstractions/key.service.ts`) is the stream of the current user key (or `null`), and `AuthService` uses it.

**Rule about crypto in this repo:** "**CRITICAL**: new encryption logic should not be added to this repo. If significant encryption related logic is added or changed, make sure @bitwarden/team-key-management-dev is aware of the PR." The mechanisms backing that rule are visible: much cryptography lives in the Rust SDK exposed as `@bitwarden/sdk-internal` (see `package.json`, and the WASM loader `apps/browser/src/platform/services/sdk/browser-sdk-load.service.ts`), and `@bitwarden/legacy-crypto` is a "holding pen" that lint prevents growing (see B1). I did not read the Rust SDK; it is not in this repo.

## B6. Feature flags and server config

Feature flags are defined in one place:

```ts
export enum FeatureFlag {
  ...
  /* Autofill */
  UseUndeterminedCipherScenarioTriggeringLogic = "undetermined-cipher-scenario-logic",
  FillAssistTargetingRules = "fill-assist-targeting-rules",
  DefaultPasswordManagerPrompt = "pm-39071-default-password-manager-prompt",
  LitInlineMenuComponents = "lit-inline-menu-components",
```

(`libs/common/src/enums/feature-flag.enum.ts`.) Its header comments set policy: "Flags MUST be short lived and SHALL be removed once enabled", and `DefaultFeatureFlagValue` says "DO NOT enable previously disabled flags, REMOVE them instead." Every flag has a default in `DefaultFeatureFlagValue` (mostly `FALSE`). A flag with an explicit warning: `EnableBasicAuthResponse` is annotated "This flag gates security risks and should not be turned on without changes to the underlying experience".

`ConfigService` (`libs/common/src/platform/abstractions/config/config.service.ts`) exposes `serverConfig$`, `getFeatureFlag$(flag)` (observable, updates as server config changes), `getFeatureFlag(flag)` (promise), and `userCachedFeatureFlag$(flag, userId)`. The values come from the server's config (`ServerConfig.featureStates`; `getFeatureFlagValue` falls back to the default when the server has no value). Server-driven config therefore changes client behavior without a new extension release. For example `OverlayBackground` reads:

```ts
  useLitInlineMenuComponents$ = this.configService.getFeatureFlag$(
    FeatureFlag.LitInlineMenuComponents,
  );
```

(`apps/browser/src/autofill/background/overlay.background.ts`.) Fill assist ("targeting rules") is another example. It needs both a flag and a user or policy setting, resolved in `DomainSettingsService.resolvedEnableFillAssist$` (`libs/common/src/autofill/services/domain-settings.service.ts`).

`ConfigService` also provides `checkServerMeetsVersionRequirement$` for gating on the server version, which matters for self-hosted servers older than the client.

## B7. Policies (enterprise policies)

Organizations can enforce policies on members. `PolicyService` (`libs/common/src/admin-console/abstractions/policy/policy.service.abstraction.ts`) offers `policies$(userId)`, `policiesByType$(type, userId)`, and `policyAppliesToUser$(type, userId)`. `PolicyType` values come from the SDK and are re-exported in `libs/common/src/admin-console/enums/policy-type.enum.ts`.

Policies that touch the browser extension, found by searching the repo:

| Policy type | Where used | Effect |
| --- | --- | --- |
| `ActivateAutofill` | `libs/common/src/autofill/services/autofill-settings.service.ts` (`activateAutofillOnPageLoadFromPolicy$`), applied by `AutofillService.setAutoFillOnPageLoadOrgPolicy` | Turns on autofill-on-page-load for members. |
| `AutomaticAppLogIn` | `apps/browser/src/autofill/background/auto-submit-login.background.ts` | Enables auto-submit login for configured IdP hosts (see C3). |
| `UriMatchDefaults` | `libs/common/src/autofill/services/domain-settings.service.ts` (`defaultUriMatchStrategyPolicy$`) | Forces a default URI match strategy; the code validates it against `Object.values(UriMatchStrategy)`. |
| `FillAssist` | `domain-settings.service.ts` | Org-driven fill assist, with an optional custom rules URL. |
| `MaximumVaultTimeout` | `libs/common/src/key-management/vault-timeout/services/vault-timeout-settings.service.ts` | Caps the vault timeout. |
| `OrganizationDataOwnership` | `apps/browser/src/autofill/background/notification.background.ts`, vault list filters | Used when deciding how new items are saved and filtered; I did not trace the details. |
| `RemoveUnlockWithPin` | `apps/browser/src/auth/popup/settings/account-security.component.ts` | Affects the PIN unlock option in account security settings; I did not trace the details. |

Policy handling pattern: observe `policiesByType$`, and re-evaluate when the policy changes. `AutoSubmitLoginBackground.init` is a clean example: it filters to the unlocked state, then `switchMap`s into `policiesByType$(PolicyType.AutomaticAppLogIn, userId)`, and registers or tears down listeners according to `policy.enabled`.

---

# Part C: Autofill concepts (the security-critical core)

Autofill is the one feature where the extension reads a hostile environment (arbitrary web pages), then writes secrets into it. The core question is always: **is this the right page, in the right frame, and is the user in control?**

Orientation docs in the repo: `apps/browser/src/autofill/README.md` (index and build flags), `autofill.design.md` (fill mechanics), `lifecycle.design.md` (when frames are monitored). The design docs say of themselves that `autofill.design.md` is "correct but incomplete".

## C1. The pipeline

```
 content script               background                          content script
 (collect)                    (decide and build script)           (insert)
 -------------                ---------------------------         ----------------
 collectPageDetails  --->     collectPageDetailsResponse  --->    fillForm
 (CollectAutofill-            AutofillOrchestrator /               (InsertAutofill-
  ContentService)             AutofillService.doAutoFill           ContentService)
                              -> generateFillScript
```

### Step 1: collect page details (content script)

`CollectAutofillContentService.getPageDetails()` (`apps/browser/src/autofill/services/collect-autofill-content.service.ts`) scans the DOM (forms, fields, including shadow DOM) and returns an `AutofillPageDetails` (`apps/browser/src/autofill/models/autofill-page-details.ts`): `title`, `url`, `documentUrl`, `forms`, `fields`, `collectedTimestamp`. Each field gets an **`opid`** (an index-based id like `__0`, `__form__0`, set in `buildAutofillFieldItem`) so the background can refer to fields without holding DOM references. Field entries include attributes used for classification (`htmlID`, `htmlName`, `type`, `autoCompleteType`, label text, and a `viewable` flag computed by `DomElementVisibilityService`).

How a collect happens:

- **Requested by the background**: `BrowserApi.tabSendMessage(tab, { command: "collectPageDetails", tab, sender }, { frameId })`. See `MainBackground.collectPageDetailsForContentScript` and `AutofillService.collectPageDetailsFromTab$`. The `sender` string says who asked (for example `ExtensionCommand.AutofillCommand` = `"autofill_cmd"`, `"autofillInit"`, `"contextMenu"`).
- **Self-initiated** on load: `AutofillInit.collectPageDetailsOnLoad` sends `bgCollectPageDetails` after a 750 ms delay.
- The content script handler gates on monitoring state: `collectPageDetails: ({ message }) => this.isMonitoring ? this.collectPageDetails(message) : undefined` (`apps/browser/src/autofill/content/autofill-init.ts`).

The content script replies with a **`collectPageDetailsResponse`** runtime message carrying `{ tab, details, sender }`.

### Step 2: the background decides and builds the fill script

`apps/browser/src/background/runtime.background.ts` receives `collectPageDetailsResponse` and dispatches on `msg.sender` to the right action. For the keyboard shortcut:

```ts
          case ExtensionCommand.AutofillCommand:
            this.autofillOrchestrator.autofillActiveTabFromCommand({
              frameId: sender.frameId,
              tab: msg.tab,
              details: msg.details,
            });
            break;
```

`AutofillOrchestrator` (`apps/browser/src/autofill/background/autofill-orchestrator.ts`) is the "single owner of runtime-message-driven autofill dispatch". It funnels three request kinds (`pageLoad`, `command`, `cipherType`) through one stream, **serialized per tab and per frame**, so two fills cannot race on one frame. The design doc explains why: it keeps "two fills from racing on a single frame — a race that could fill twice, or, if the page navigated between collecting its details and dispatching the fill, place a credential chosen for the old page onto the new one" (`autofill.design.md`).

The orchestrator calls into `AutofillService` (`apps/browser/src/autofill/services/autofill.service.ts`):

- `doAutoFillActiveTab` / `doAutoFillOnTab`: choose which cipher. For a user-initiated fill (`fromCommand = true`) it uses `cipherService.getNextCipherForUrl(tabUrl, userId)` (cycles through matches); for page load it uses the last launched or last used cipher for the URL. It then checks password reprompt (C5).
- `doAutoFill(options)`: for each frame's page details, calls `generateFillScript(...)`, applies the untrusted-iframe gate, and sends the script.
- `generateFillScript` (private): picks a path by cipher type (`generateLoginFillScript`, `generateCardFillScript`, `generateIdentityFillScript`, `generateSshKeyFillScript`), or `generateTargetedFillScript` when the page has fields flagged by targeting rules. The result is an `AutofillScript` (`apps/browser/src/autofill/models/autofill-script.ts`):

```ts
export type FillScript = [action: FillScriptActions, opid: string, value?: string];
```

with actions `fill_by_opid`, `click_on_opid`, `focus_by_opid`, plus `properties`, `savedUrls`, and `untrustedIframe`. So the fill script is **data**: a list of (action, field id, value). The content script interprets only these three verbs.

### Step 3: insert (content script)

The background sends `fillForm` to the one frame via `tabSendMessage(..., { frameId: pd.frameId })` (shown in A8.5). `AutofillInit.fillForm` first verifies the page has not navigated since collection:

```ts
    if ((document.defaultView || window).location.href !== pageDetailsUrl || !fillScript) {
      return;
    }
```

(`apps/browser/src/autofill/content/autofill-init.ts`.) It then calls `InsertAutofillContentService.fillForm` (`apps/browser/src/autofill/services/insert-autofill-content.service.ts`), which refuses in sandboxed iframes, shows insecure-page and untrusted-iframe confirmation prompts, and runs each action with a 20 ms delay: `fill_by_opid` looks up the element by opid via `collectAutofillContentService.getAutofillFieldElementByOpid`, sets `.value`, and fires simulated pre- and post-insert events (click, focus, keyboard, `input`/`change`) so framework-driven pages register the change.

### Message command names, collected

| Command | Direction | Purpose |
| --- | --- | --- |
| `triggerAutofillScriptInjection` | content (static) to background | ask for per-frame script injection |
| `startAutofillMonitors` / `stopAutofillMonitors` | background to content | lifecycle toggles (`AutofillLifecycleCommand`) |
| `disableAutofiller` | background to content | halt the page-transition poller (`AutofillerCommand.disable`) |
| `pageTransitionDetected` | autofiller to background | report a URL change (`AutofillMessageCommand`) |
| `bgCollectPageDetails` | content to background | content asks background to request a collect |
| `collectPageDetails` | background to content | request page details (a `sender` string rides along) |
| `collectPageDetailsResponse` | content to background | reply with `{tab, details, sender}` |
| `fillForm` / `triggerAutoSubmitLogin` | background to content | deliver the fill script |
| `updateIsFieldCurrentlyFilling` | content to background | tells the overlay a fill is in progress |

The page-load variant adds a front half: `autofiller.js` polls `window.location.href` every 500 ms and reports changes as `pageTransitionDetected` (`apps/browser/src/autofill/content/autofiller.ts`). The background's lifecycle service buffers the transition (pending, paused, resolved, or retired) until the tab is committed and an account is logged in, and emits an opportunity; `AutofillOrchestrator` turns that into a `pageLoad` fill request if autofill-on-page-load is enabled. The state-machine and the "committed tab" rule (fill only the tab the user is looking at) are documented in `lifecycle.design.md` and `autofill.design.md`.

### Fill targeting and retry classification (read these two sections of the design doc)

- *Fill targeting* (`autofill.design.md`): a page-load fill must target the frame "resolved live, by id, at the moment of the fill", not a snapshot, because "Filling from the transition's stale snapshot would put a cipher chosen for the _old_ page into whatever page now occupies that frame — a credential handed to the wrong origin." In code, `AutofillOrchestrator.resolveFreshTarget` re-reads the tab and the frame's live URL and abandons if the URL differs from the one reported with the transition, and `dispatch` compares `details?.url === request.frameUrl`.
- *Retry classification*: only "a cipher matched, but no field accepted a value yet" is retryable.

## C2. URI matching, and why it is security critical

A saved login has zero or more URIs, each with an optional **match strategy**. Strategies:

```ts
export const UriMatchStrategy = {
  Domain: 0,
  Host: 1,
  StartsWith: 2,
  Exact: 3,
  RegularExpression: 4,
  Never: 5,
} as const;
```

(`libs/common/src/models/domain/domain-service.ts`, whose header comment quotes the user-facing definitions: Domain means "the top-level domain and second-level domain of the URI match the detected resource", Host means hostname and port match, and so on.) A per-URI `match` overrides the user's default (`defaultUriMatchStrategy`, default `Domain`; possibly forced by the `UriMatchDefaults` policy).

The matching function is `LoginUriView.matchesUri` (`libs/common/src/vault/models/view/login-uri.view.ts`):

```ts
    switch (matchType) {
      case UriMatchStrategy.Domain:
        return this.matchesDomain(targetUri, matchDomains);
      ...
      case UriMatchStrategy.Exact:
        return targetUri === this.uri;
      case UriMatchStrategy.StartsWith:
        return targetUri.startsWith(this.uri);
```

`Host` (elided above) compares `Utils.getHost` of both URLs, which is hostname plus port when a port is present.

`RegularExpression` builds `new RegExp(this.uri, "i")` and returns `false` on an invalid pattern; `Never` returns `false`. `LoginView.matchesUri` ORs across all URIs, and `CipherService.filterCiphersForUrl` (`libs/common/src/vault/services/cipher.service.ts`) filters the vault: it skips deleted and archived ciphers, and applies `matchesUri` to logins using the page's equivalent domains and the default strategy. `getAllDecryptedForUrl` is the usual entry point.

### Domain strategy, `Utils.getDomain`, and the Public Suffix List

`Domain` compares the **registrable domain** (for `login.example.co.uk`, that is `example.co.uk`). That needs the Public Suffix List (PSL): a community list of suffixes under which anyone can register names (`.com`, `.co.uk`, and also private suffixes such as `github.io`). The repo uses the `tldts` library:

```ts
      const parseResult = parse(uriString, {
        validHosts: this.validHosts,
        allowPrivateDomains: true,
      });
```

(`libs/common/src/platform/misc/utils.ts`, `Utils.getDomain`; `import { getHostname, parse } from "tldts"` at the top.) Details visible there:

- `validHosts` is `["localhost"]`; `localhost` and IP addresses are returned as-is.
- `data:` and `about:` URIs return `null`.
- `allowPrivateDomains: true` makes tldts treat private-section PSL entries as suffixes, so two different users' subdomains under such a suffix have different registrable domains (this is how the library behaves; I did not test it in this environment).
- `Utils.DomainMatchBlacklist = new Map([["google.com", new Set(["script.google.com"])]])`, used in `matchesDomain`: even if the domain matches, `script.google.com` is excluded for a `google.com` login, because `script.google.com` hosts user-controlled content.
- `matchesDomain` normalizes punycode and Unicode forms before comparing (`punycodeToUnicode` in `libs/common/src/autofill/utils/punycode.ts`), so an internationalized domain saved as Unicode matches its punycode (`xn--...`) form. This is a correctness feature; it also shows why homograph/lookalike domains are a concern, as each distinct string is a different domain.

### Equivalent domains

Users and Bitwarden define groups of domains treated as the same site (for example, a company's login on two domains). `DomainSettingsService.getUrlEquivalentDomains(url)` (`libs/common/src/autofill/services/domain-settings.service.ts`):

```ts
        const equivalents = equivalentDomains.filter((ed) => ed.includes(domain)).flat();

        return new Set(equivalents);
```

The data comes from sync: `default-sync.service.ts` merges the user's own `equivalentDomains` with the server's `globalEquivalentDomains` and stores them with `setEquivalentDomains`. In `matchesUri`, the target domain is added to that set (`equivalentDomains.add(targetDomain)`), then `matchesDomain` checks whether the saved URI's domain is in the set.

### Why this is security critical (phishing)

Autofill is only safe if "this page" really is "the saved site". If the match is too loose, a lookalike or attacker-controlled page gets your password with no typing. The design pushes safety in several ways:

- Default `Domain` strategy compares registrable domains, so `evil-example.com` does not match `example.com`. A subdomain of the legit domain does match under `Domain`, which is why `Host`, `Exact` and `StartsWith` exist for stricter needs.
- **`StartsWith` is a prefix match**, so a saved `https://example.com` also matches `https://example.com.evil.test/` (string prefix). The strategy exists for flexibility, and the safety of the pattern is left to whoever chooses it. I flag it here as the type of thing an interviewer might probe.
- **`RegularExpression`** runs user-supplied patterns against URLs, so a bad pattern can match too much (or be slow; I did not evaluate that).
- **HTTP vs HTTPS and frames** are checked separately (C5).
- **The URL the browser reports for the tab** (`tab.url`, from the privileged `tabs` API) is what is matched, not anything the page claims about itself.

## C3. Autofill triggers

| Trigger | Entry point | Notes |
| --- | --- | --- |
| **Manual, from the popup** | `VaultPopupAutofillService.doAutofill` (`apps/browser/src/vault/popup/services/vault-popup-autofill.service.ts`) | Collects page details with `collectPageDetailsFromTab$`, then `autofillService.doAutoFill({ tab, cipher, pageDetails, doc: window.document, fillNewPassword: true, allowTotpAutofill: true })`. Password reprompt is asked in the popup unless skipped. Copies TOTP to clipboard after a successful fill. |
| **Keyboard shortcut** | `CommandsBackground.processCommand` (`apps/browser/src/background/commands.background.ts`) | Manifest `commands` `autofill_login` (suggested `Ctrl+Shift+L`), `autofill_card`, `autofill_identity`. If the vault is not unlocked it opens the unlock popout and queues a retry. The retry keeps its sender under a symbol (`RETRY_SENDER`) and is accepted only if it arrived via the intraprocess channel: "This ensures that the command to retry never round-tripped through a untrusted environment." |
| **Context menu** | `ContextMenuClickedHandler` (`apps/browser/src/autofill/browser/context-menu-clicked-handler.ts`) and `main-context-menu-handler.ts` | Per-cipher "autofill" and "copy username/password/TOTP" entries; checks password reprompt first and opens `openVaultItemPasswordRepromptPopout` when needed. |
| **Autofill on page load** | `autofiller.ts` to `pageTransitionDetected` to lifecycle service to `AutofillOrchestrator` `pageLoad` request | Off by default; the policy `ActivateAutofill` can switch it on. Tab must be "committed". Never fills untrusted iframes (it passes `allowUntrustedIframe: false` because `fromCommand` is false) and never fills reprompt-protected ciphers. |
| **Inline menu (overlay)** | `OverlayBackground.fillInlineMenuCipher` (`overlay.background.ts`) | User clicks an item in the in-page menu. See C4. |
| **TOTP copying** | `AutofillService.getShouldAutoCopyTotp`, `getTotpCopyCode`; `AutofillOrchestrator.copyTotp`; also the inline menu | After a successful fill of a login that has a TOTP secret, the code may be copied to the clipboard (the `autoCopyTotp` setting defaults to true). Premium or org-TOTP access is checked (`canUseTotp`). |
| **Auto-submit login** | `AutoSubmitLoginBackground` (`apps/browser/src/autofill/background/auto-submit-login.background.ts`) and `content/auto-submit-login.ts` | Policy-gated (`AutomaticAppLogIn`). See below. |

### Auto-submit login in more detail

This is an enterprise feature for SSO-style flows: after a redirect, fill and submit a login form automatically. Safeguards in code:

- Only active if the `AutomaticAppLogIn` policy is enabled and applies to the user, the vault is `Unlocked`, and the policy lists IdP hosts (`parseIpdHostsFromPolicy`, `validIdpHosts`).
- Triggered by a URL hash containing `autosubmit=1` (`urlContainsAutoSubmitHash`), but only after a request whose **initiator** is a valid IdP host or already-valid auto-submit host. The redirect handler documents why: "The initiator check prevents any origin from 302'ing to `target#autosubmit=1` to force autofill there."
- Uses `chrome.webRequest.onBeforeRequest` and `onBeforeRedirect` listeners limited to `main_frame` and `sub_frame` types, and clears state if a POST is seen or the main frame goes to an invalid host.
- Injects `content/auto-submit-login.js` only when the vault is unlocked, and fills with `triggerAutoSubmitLogin` using the same fill-script mechanism.

## C4. The inline menu (overlay)

The **inline menu** is the small Bitwarden icon inside a login field and the dropdown of matching items. It is the most exposed UI, because it is rendered inside the untrusted web page.

### Structure: four nested layers

1. **Host element in the page** (content script, top frame only). A custom element with a random name (Firefox: a `div`) with `popover="manual"`, appended to `document.body` (or into an open modal dialog), forced to the top layer with `showPopover()`. Pages see this element and can touch its attributes, styles, and position in the DOM.
2. **Closed shadow root** on the host (`attachShadow({ mode: "closed" })`, see A10). The only thing inside is the iframe, so the page cannot enumerate or query it.
3. **Extension-origin iframe** (`src` is `overlay/menu.html`, the "menu container"), created by `AutofillInlineMenuIframeService.initMenuIframe`. Its attributes include `credentialless` and `tabIndex: "-1"`. It is cross-origin to the page, so the page cannot reach its DOM, and the `<iframe>` element itself sits inside the closed shadow root. However, `menu.html` is web-accessible (A4), so a page could also load a copy of it in an iframe of its own; that is why the container authenticates its messages (below).
4. **Nested sandboxed iframe** created *by the container page* (`apps/browser/src/autofill/overlay/inline-menu/pages/menu-container/autofill-inline-menu-container.ts`). Its attributes:

```ts
  private readonly defaultIframeAttributes: Record<string, string> = {
    src: "",
    title: "",
    sandbox: "allow-scripts",
    credentialless: "",
    allowtransparency: "true",
    tabIndex: "-1",
  };
```

`sandbox="allow-scripts"` without `allow-same-origin` gives an opaque origin, so this innermost iframe (which loads `overlay/menu-button.html` or `overlay/menu-list.html`, drawn by `AutofillInlineMenuButton` / `AutofillInlineMenuList` custom elements with their own closed shadow roots) has no extension API access and, being cross-origin to both the page and the container, cannot be scripted by either. All real work goes through message passing.

I could not find a comment explaining the choice of two nested extension iframes; the structure suggests the middle layer is a trusted **broker** with extension API access, and the innermost layer is the **untrusted renderer** that only shows data and relays user clicks. That reading is my inference from the code, not a documented rationale.

### Why it is built this way

- **Isolation from page scripts**: A page cannot read inside a cross-origin iframe or a closed shadow root, so it cannot read cipher names, usernames, or TOTP codes shown in the menu (`buildCipherData` in `overlay.background.ts` sends the iframe `name`, `username`, TOTP code, favicon, and so on, but not the password, as I read the code).
- **Isolation from page CSS**: styles do not cross a shadow root or an iframe boundary; the host element resets with `all: "initial"` and uses `!important` pseudo-element rules (`autofill-inline-menu-iframe-element.ts`).
- **Resistance to being covered or hidden**: top layer, max z-index, `getPageIsOpaque`, `elementFromPoint` checks, and mutation observers (A10).
- **No sensitive data in the page's reach**: the iframe receives cipher *display* data with synthetic ids of the form `inline-menu-cipher-<index>` (built in `OverlayBackground.handleOverlayCiphersUpdate`), and when the user clicks an item, only that id travels back; the background looks up the real `CipherView` itself in `fillInlineMenuCipher`. As I read `buildCipherData`, neither the real cipher id nor the password is part of the data sent into the page-side layers.

### Message authentication across the layers

Messages inside the stack (container, `autofill-inline-menu-container.ts`):

- Rejects any window message with no `portKey` or not from the parent window or the one iframe it created (`isForeignWindowMessage`).
- Accepts the initial `initAutofillInlineMenu*` message only from the extension origin or its parent window (`isMessageFromExtensionOrigin`, `isMessageFromParentWindow`).
- Identifies the inner iframe by **object identity**, `this.inlineMenuPageIframe.contentWindow === event.source`, and requires a **session token** generated per container instance: `this.token = generateRandomChars(32)`.
- Forwards to the background only commands in an allowlist:

```ts
const ALLOWED_BG_COMMANDS = new Set<string>([
  "addNewVaultItem",
  "autofillInlineMenuBlurred",
  "autofillInlineMenuButtonClicked",
  "checkAutofillInlineMenuButtonFocused",
  "checkInlineMenuButtonFocused",
  "fillAutofillInlineMenuCipher",
  ...
]);
```

- Validates that URLs it is told to load are extension URLs under the expected origin (`isExtensionUrlWithOrigin` checks the protocol is `chrome-extension:`, `moz-extension:`, or `safari-web-extension:`).
- Posts to its own inner iframe with target origin `"*"` in a method literally named `postMessageToInlineMenuPageUnsafe`, presumably because the sandboxed child has an opaque origin that cannot be named as a target (my inference). The method's comment says it is for trusted messages, and the safe wrapper adds the session token. The container therefore relies on `contentWindow` identity and the token instead of a target origin on that hop.

The innermost page (`autofill-inline-menu-page-element.ts`) only accepts messages from `globalThis.parent`, pins `messageOrigin` to the first parent message's origin, ignores events from other origins, and refuses to post to its parent without a token and an established origin ("never send messages containing authentication tokens without a valid token and an established messageOrigin"). The background then re-checks `portKey` against the tab (A8.4).

## C5. Security safeguards to know

### Untrusted and cross-origin iframes

The background marks a fill script `untrustedIframe` when the frame's URL does not match the saved login:

```ts
    if (pageUrl === options.tabUrl) {
      return false;
    }
    ...
    const matchesUri = options.cipher.login.matchesUri(
      pageUrl,
      equivalentDomains,
      options.defaultUriMatch,
    );
    return !matchesUri;
```

(`AutofillService.inUntrustedIframe`, `autofill.service.ts`; condensed. Its doc comment: "If the pageUrl (from the content script) matches the tabUrl (from the sender tab), we are not in an iframe".) Then the policy depends on the trigger:

```ts
        if (
          fillScript.untrustedIframe &&
          options.allowUntrustedIframe != undefined &&
          !options.allowUntrustedIframe
        ) {
          this.logService.info("Autofill on page load was blocked due to an untrusted iframe.");
          return;
        }
```

(`doAutoFill`.) Page-load fills pass `allowUntrustedIframe: fromCommand`, i.e. `false`, so they are silently blocked. User-initiated paths (shortcut passes `true`; popup and inline menu leave it undefined) are not blocked in the background, but the content script **asks the user**:

```ts
    return !globalThis.confirm(confirmationWarning);
```

(`InsertAutofillContentService.userCancelledUntrustedIframeAutofill`.) The code comment notes `confirm()` "is blocked by sandboxed iframes, but we don't want to fill sandboxed iframes anyway", and `fillForm` already refuses sandboxed iframes with `currentlyInSandboxedIframe()`. The text shown to the user comes from the locale string `autofillIframeWarning`: "The form is hosted by a different domain than the URI of your saved login. Choose OK to autofill anyway, or Cancel to stop." (`apps/browser/src/_locales/en/messages.json`.)

### HTTP vs HTTPS warning

```ts
    if (
      !savedUrls?.some((url) => url.startsWith(`https://${globalThis.location.hostname}`)) ||
      globalThis.location.protocol !== "http:" ||
      !this.isPasswordFieldWithinDocument()
    ) {
      return false;
    }
```

(`InsertAutofillContentService.userCancelledInsecureUrlAutofill`.) It prompts only when: a saved URL for this hostname starts with `https://`, but the current page is `http:`, and there is a password field in the document. Then `confirm()` shows the `insecurePageWarning` strings ("This Login was originally saved on a secure (HTTPS) page."). Note the scope: a login saved only as `http://...` does not trigger the warning (my reading of the condition).

### Password reprompt

Per-item "master password re-prompt" is the `CipherRepromptType`:

```ts
export const CipherRepromptType = {
  None: 0,
  Password: 1,
} as const;
```

(`libs/common/src/vault/enums/cipher-reprompt-type.ts`.) Enforcement in `AutofillService`:

- `doAutoFillOnTab` returns `{ didAutofill: false }` when `cipher.reprompt === CipherRepromptType.Password && !fromCommand`. In other words, a page-load fill never fills reprompt-protected items.
- `isPasswordRepromptRequired(cipher, tab)`: if `cipher.reprompt === Password` and the user has a master password, it opens the reprompt popout (debounced) and returns `true`, so the fill is not performed.
- The inline menu (`fillInlineMenuCipher`), context menu (`ContextMenuClickedHandler`), and popup (`_internalDoAutofill` calls `passwordRepromptService.showPasswordPrompt()`) each check it too. The inline menu also receives `reprompt` in its cipher data so it can display the state.

### User-gesture and visibility checks

- **Trusted events only** (`event.isTrusted`) on every inline-menu action and form-submit capture (A7).
- **Visibility**: `viewable` fields only (A10), and `generateFillScript` skips fields where `!field.viewable && field.tagName !== "span"`.
- **Page risk checks** close the inline menu if `html` or `body` is translucent, or the page keeps fighting for the top layer (A10).
- **User gesture for side panel**: comment in the context menu handler (A6).
- **Committed tab only**: fills land only on the tab the user is working in (`autofill.design.md`, "Autofill and the monitoring lifecycle").
- I did not find use of `navigator.userActivation`; "user gesture" in this repo mostly means `isTrusted` input events.

### Other safeguards worth knowing

- `fillForm` re-checks `location.href === pageDetailsUrl` (C1).
- `senderHasValidTab` and `withSenderTab` in `overlay.background.ts` (A8.1).
- Inline-menu messages are tied to the focused field's tab and frame: `senderTabHasFocusedField` and `senderFrameHasFocusedField` compare `sender.tab?.id` and `sender.frameId` against `focusedFieldData` (`apps/browser/src/autofill/background/overlay.background.ts`).
- The inline-menu cipher cache is only (re)built while the vault is `Unlocked` (`updateOverlayCiphers` checks `AuthenticationStatus.Unlocked` before populating); on lock the extension process is reloaded (B4). I did not trace every place the cache is cleared.
- Blocked domains: `BrowserScriptInjectorService.findBlockedInjectionUrl` prevents injecting scripts into the user's blocked pages and sub-frames.

## C6. Vulnerability history (short)

- **Iframe autofill (CVE-2018-25081 per the companion doc filename).** A page that matches a saved login could embed a different-origin form that received the credential. The safeguard you can read today is the `untrustedIframe` flow above. See [`vuln-cve-2018-25081-iframe-autofill.md`](./vuln-cve-2018-25081-iframe-autofill.md).
- **2025 DOM-based extension clickjacking.** A malicious page manipulates the visibility, opacity, or position of the extension's injected UI so that a user's click triggers an autofill they did not intend. The defenses in `autofill-inline-menu-content.service.ts` and `autofill-inline-menu-iframe.service.ts` (A10, C4) are the relevant code. See [`vuln-2025-dom-clickjacking.md`](./vuln-2025-dom-clickjacking.md).
- `fido2-content-script.ts` mentions "VULN-582 / VULN-398" for the Permissions Policy attacker model; I did not find more detail in the repo.

I have not independently verified the details of either named vulnerability; read the companion files for those.

---

# Part D: Security concepts glossary

Short, and tied to the code where I can.

**Threat model of a password-manager extension.** Think in terms of who can run code or influence what, and what they want (secrets, or a trick that makes the user or extension hand secrets over).

- *Malicious web page.* Controls all page JavaScript in the main world, the DOM, CSS, and the page's iframes. Goals: read filled credentials, trigger a fill into its own form, cover or fake the UI to induce a click (clickjacking), forge messages to the extension. Defenses: isolated world, closed shadow DOM, extension-origin iframes, URL matching, untrusted-iframe prompts, `isTrusted` checks, sender and token checks (Parts A and C). Limit: once a legitimately matched page receives a fill, its scripts can read the DOM values; no extension can prevent that.
- *Compromised or malicious sub-frame.* An iframe from another origin inside a legitimate page, for example an ad or widget. Goal: get a credential chosen for the parent page. Defense: per-frame fill targeting and the `untrustedIframe` flag (A9, C5). Also, content scripts treat `sender.frameId` from the browser as authoritative.
- *Malicious other extension.* Can run its own content scripts in the same page and see the same DOM, but not Bitwarden's isolated world. It could observe filled values in the DOM and could try to interfere with the injected UI. In this repo there is no `externally_connectable` key and no `onMessageExternal` listener, and the messaging layer tags and stamps messages (A8.2). I'm describing what the code registers, not making a claim about every browser's cross-extension behavior.
- *Local attacker.* Someone with access to the machine or profile. Defenses: the user key is only in memory while unlocked (`USER_KEY` is `CRYPTO_MEMORY`, cleared on lock), lock reloads the extension process, vault timeout, and the clipboard clearing (`systemService.clearClipboard` / `clearPendingClipboard`). Limit: a `MainBackground` comment says secure storage "is not supported in browsers, so we use local storage and warn users when it is used", so anything the extension keeps for convenience unlock sits in ordinary extension storage. I did not audit exactly what is stored there.
- *Network attacker and malicious server.* Mitigated mostly by TLS and the zero-knowledge design: the server holds only encrypted vault data and an authentication hash, not the user key.

**XSS (cross-site scripting).** Injecting attacker script into a page that then runs with the page's privileges. In the extension it matters because extension pages have strong powers. Defenses visible here: extension-page CSP `script-src 'self'` (A5), Angular templates in the popup, and in-page UI code that assigns `innerHTML` only to clear (`""`) and uses `textContent` for text (`autofill-inline-menu-list.ts`; `autofill-inline-menu-button.ts` also clears with `innerHTML = ""`). A search of the autofill folder (non-test code) found no `innerHTML` assignment of data. Also, page-controlled strings (field names, labels) are only classified, not evaluated.

**Clickjacking.** Tricking a user into clicking something other than what they think they are clicking, usually by overlaying or hiding UI. Classic form: invisible iframe above a button. In extensions, the page can do the same to *extension-injected DOM* because it shares the page's DOM and CSS. Defenses: A10, C4.

**UI redressing.** The umbrella for clickjacking and similar visual deception: opacity, covering, overlapping, misleading positioning, fake lookalike dialogs. The repo guards against it in the real UI (menu opacity checks, `elementFromPoint` obstruction checks) and prevents the extension from being fooled by invisible fields on the way in (`DomElementVisibilityService` only fills fields that are viewable).

**Phishing and lookalike domains.** A fake site resembling the real one. A password manager helps because autofill is keyed to the real domain: if the lookalike is a different registrable domain, nothing is offered. Related attack surface: homographs (non-Latin characters that look like Latin ones), `StartsWith` matching mistakes, subdomains under a legit domain, equivalent-domain lists, and `Domain` matching on shared-hosting suffixes (the PSL). Code: `Utils.getDomain`, `LoginUriView.matchesUri` (C2).

**CSP (Content Security Policy).** A header or manifest field restricting what resources a document can load or run. Two different uses in this repo: (1) the extension's own CSP (A5); (2) websites' CSPs, which the extension does not control. The MV2 FIDO2 path inserts a `<script src>` element into the page, which is a place where a page's own CSP could matter; I did not investigate how that case is handled.

**Origin vs site.** An *origin* is scheme + host + port (`https://a.example.com:443`); the same-origin policy works on origins. A *site* is scheme + registrable domain (eTLD+1), roughly `https://example.com`, which groups subdomains. Bitwarden's `Domain` match strategy is a *site-like* comparison; `Host` and `Exact` are closer to origin-like. FIDO2 code uses `location.origin` (`respondToCredentialRequest`). The popup/extension origin comparison in `senderIsInternal` is an origin comparison (`urlOriginsMatch`).

**Zero-knowledge encryption.** The service provider cannot read your data because encryption keys derive from a secret it never receives (the master password), and data leaves the client only encrypted. In this repo see B5. The practical consequence for engineers: any code path that sends decrypted vault data to an API breaks the model, hence the rule "**NEVER** send unencrypted vault data to API services".

**Memory hygiene for secrets.** Minimize how long and where secrets live. Visible practices: the user key only in memory state cleared on lock and logout; extension process reload after lock (`BrowserProcessReloadService`); `cleanupDelayMs: 0` on key state so observables do not retain secrets; session-keyed encryption of persisted memory state (`LocalBackedSessionStorageService`); the clipboard auto-clear for copied passwords and TOTP; explicit `destroy()` of content-script services so decrypted display data does not linger (`AutofillInit.destroy`). JavaScript offers no reliable way to zero strings, which is one reason the heavy crypto lives in the Rust/WASM SDK. I did not inspect SDK memory handling.

**The CLAUDE.md rules.** In `.claude/CLAUDE.md`:

- "**NEVER** use code regions" (refactor instead).
- "**CRITICAL**: new encryption logic should not be added to this repo", and notify the key-management team of significant crypto changes.
- "**NEVER** send unencrypted vault data to API services."
- "**NEVER** commit secrets, credentials, or sensitive information."
- "**NEVER** log decrypted data, encryption keys, or PII. No vault data in error messages or console logs." Look at how logging is done around fills: `this.logService.info("Autofill on page load was blocked due to an untrusted iframe.")` states what happened, never the credential or the cipher content. A custom ESLint rule, `@bitwarden/platform/no-page-script-url-leakage` (`libs/eslint/platform/no-page-script-url-leakage.mjs`), guards against exposing extension URLs to pages (see A4); `fido2-page-script-delay-append.mv2.ts` disables it for one line with a comment.
- Tailwind classes need the `tw-` prefix. (Style rule, not security.)
- Respect config files (`eslint.config.mjs`, `tsconfig.json`) and the verification commands (`npm run lint:fix`, `npm run prettier`, `npm run test:types`, `npm test`).

---

# Suggested reading order through the code

A path that builds understanding from the outside in. Spend most of your time on 8 to 13.

1. `apps/browser/src/manifest.v3.json`, then `apps/browser/src/manifest.json`, and `apps/browser/webpack/manifest.js`. What gets shipped and where each context starts.
2. `apps/browser/src/platform/background.ts` and `apps/browser/src/background/main.background.ts` (`bootstrap()` and the `autofillService`, `overlayBackground`, `autofillOrchestrator` constructions). The service graph.
3. `apps/browser/src/platform/browser/browser-api.ts` (flags, `senderIsInternal`, `tabSendMessage`, `executeScriptInTab`). The platform seam.
4. `apps/browser/src/platform/utils/from-chrome-runtime-messaging.ts`, `apps/browser/src/platform/utils/web-ext-sender.ts`, `libs/messaging/src/intraprocess-message.sender.ts`, `libs/messaging/src/is-external-message.ts`. Message provenance.
5. `libs/state/src/core/user-key-definition.ts`, `libs/common/src/key-management/state-definitions.ts`, and `apps/browser/src/platform/storage/browser-storage-service.provider.ts`. How state and storage locations map to MV3.
6. `libs/common/src/auth/services/auth.service.ts`, `libs/unlock/src/lock.service.ts`, `libs/common/src/key-management/vault-timeout/services/vault-timeout.service.ts`. Locked vs unlocked in practice.
7. `libs/common/src/vault/models/domain/cipher.ts`, `libs/common/src/vault/models/view/cipher.view.ts`, `libs/common/src/vault/services/default-cipher-encryption.service.ts`. Encrypted vs decrypted models and the SDK boundary.
8. `apps/browser/src/autofill/content/trigger-autofill-script-injection.ts`, `apps/browser/src/background/runtime.background.ts` (`processMessageWithSender`), and `AutofillService.injectAutofillScripts`. How content scripts get into pages.
9. `apps/browser/src/autofill/content/autofill-init.ts`, `apps/browser/src/autofill/services/collect-autofill-content.service.ts`, `apps/browser/src/autofill/services/insert-autofill-content.service.ts`. The content-script side of the pipeline.
10. `apps/browser/src/autofill/services/autofill.service.ts` (`doAutoFill`, `doAutoFillOnTab`, `generateFillScript`, `generateLoginFillScript`, `inUntrustedIframe`) and `apps/browser/src/autofill/background/autofill-orchestrator.ts`. The decision logic.
11. `libs/common/src/vault/models/view/login-uri.view.ts`, `libs/common/src/platform/misc/utils.ts` (`getDomain`), `libs/common/src/autofill/services/domain-settings.service.ts` (`getUrlEquivalentDomains`). Matching.
12. `apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts` and `apps/browser/src/autofill/overlay/inline-menu/iframe-content/autofill-inline-menu-iframe.service.ts` (plus `autofill-inline-menu-iframe-element.ts`). Page-facing defenses.
13. `apps/browser/src/autofill/overlay/inline-menu/pages/menu-container/autofill-inline-menu-container.ts`, `apps/browser/src/autofill/overlay/inline-menu/pages/shared/autofill-inline-menu-page-element.ts`, and `apps/browser/src/autofill/background/overlay.background.ts` (`handlePortOnConnect`, `handleOverlayElementPortMessage`, `fillInlineMenuCipher`). Message authentication across the stack.
14. `apps/browser/src/autofill/lifecycle.design.md`, `autofill.design.md`, and `apps/browser/src/autofill/services/autofill-lifecycle.service.ts`. The monitoring lifecycle and fill targeting rules.
15. `apps/browser/src/autofill/fido2/background/fido2.background.ts`, `apps/browser/src/autofill/fido2/content/fido2-content-script.ts`, `fido2-page-script.ts`, and `messaging/messenger.ts`. The MAIN-world case and the page-to-content-script bridge.

---

# Open questions and caveats

Things I could not confirm from the code, or that you should double check before relying on them in an interview:

1. **Companion docs.** `vuln-cve-2018-25081-iframe-autofill.md` and `vuln-2025-dom-clickjacking.md` were not present in `study/browser-extension/` when I wrote this, so I could not check my short summaries in C6 against them.
2. **Firefox MV3 background lifetime.** The MV3 manifest gives Firefox `background.scripts`. The lifecycle design doc describes Firefox as "Manifest V2 with a persistent background page". I did not verify how Firefox treats the MV3 `scripts` background (whether persistent or event-driven).
3. **`use_dynamic_url` and `credentialless`.** The repo sets both but I found no comment explaining them. My descriptions are general knowledge or just "the code sets this".
4. **Why the inline menu has two nested extension iframes.** The broker/renderer explanation in C4 is my inference from reading the code. I found no documentation of the intent.
5. **`wasm-unsafe-eval` rationale.** Inferred from the SDK WASM loader; not stated in a comment.
6. **Which Chrome feature motivates `minimum_chrome_version: 134.0`.** `apps/browser/webpack.base.js` has a comment near line 152 I only skimmed.
7. **Safari.** The Safari MV3 permission list includes `offscreen`; I did not trace how Safari builds use it. The Safari build artifacts are under `apps/browser/src/safari/` (Xcode projects), which I did not read.
8. **SDK internals.** Encryption, decryption, and key derivation are in `@bitwarden/sdk-internal` (Rust). The key hierarchy in B5 is only what TypeScript types and comments show; the exact algorithms are not in this repo.
9. **`safeProvider` import path** differs between the rule file (`@bitwarden/ui-common`) and `services.module.ts` (`@bitwarden/angular/platform/utils/safe-provider`); I did not check which is current.
10. **Regex and `StartsWith` match risks** are my reading of how those strategies work, not documented vulnerabilities.
11. **Platform facts** (isolated worlds, `isTrusted`, shadow DOM modes, top layer, PSL behavior, `storage.session`) are standard web and extension behavior as I understand it; I did not test them in a browser here.
12. The repo is a snapshot with features (fill assist targeting rules, the autofill lifecycle service, the orchestrator, the Lit inline menu components flag) that you may not see in older public versions of Bitwarden. Rely on the code in this checkout.
