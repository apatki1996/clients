# DOM-based extension clickjacking (Marek Toth, DEF CON 33, 2025) and the Bitwarden browser extension

Study notes for interview prep. Source of truth is git history plus the current code. Anything that comes from outside the repo is labelled **[external]** and never overrides what the code shows.

How this was produced:

- Upstream history: a full-history, blobless clone of `bitwarden/clients` (`git -C <bw-history> ...`). Current code: `/home/user/clients` at `5a00c60` (main, shallow).
- External sources were only partly reachable. `marektoth.com`, `marektoth.cz`, `thehackernews.com`, `bleepingcomputer.com` and `socket.dev` were **blocked by the network proxy**, and GitHub PR pages could not be read (`gh pr view` is blocked, GraphQL is not available). The external material below therefore comes only from **web-search result summaries** of those articles, not from the primary write-up. Treat it as secondhand.
- No browser was available in the sandbox (no Chrome binary), so **nothing here was tested dynamically**. Where a conclusion depends on browser rendering behaviour, it is marked "untested".

---

## 1. Summary

### External facts [external, secondhand via search summaries]

- Researcher Marek Toth presented "DOM-based extension clickjacking" at DEF CON 33 (August 2025). A single click on an attacker-controlled page could leak credit card data, personal data, logins and TOTP codes from password manager autofill UIs. Affected products reportedly included 1Password, iCloud Passwords, Bitwarden, LastPass and others.
- The subtypes reported in the coverage: direct DOM element opacity manipulation, root element opacity manipulation, parent element opacity manipulation, and partial or full overlaying. One summary describes an overlay variant using `pointer-events: none` so clicks pass through to the autofill UI, and a "partial overlay" variant that covers everything except a few pixels of the dropdown and uses "last in DOM with max z-index, or Top Layer" to stay on top.
- The researcher reportedly notified vendors privately in April 2025.
- Bitwarden reportedly said a fix was rolling out in browser extension **2025.8.0**. One search-summary snippet claims 2025.8.1 added "do not render the inline autofill menu if the page has an open popover". A forum-style snippet says "not all versions of the vulnerability require manipulation of opacity (see the Overlay section)", and mentions a fix "in version 2025.7.2".

### What the code shows

- Before the fix, 2025.7.0 had strong defenses on the inline menu's own elements (forced inline `!important` styles, mutation observers that wipe tampering, DOM-order enforcement, a basic "is something covering me" check). It had **no check on the `<html>`/`<body>` ancestors**, and the menu's host element is a direct child of `<body>`, so page-set opacity on `html` or `body` visually fades the menu. That is the gap PM-24936 closed.
- The fix was not one commit. Within about 10 days there were three follow-ups in the same file, and more hardening followed through 2026:
  - PM-24936: opacity check on `html`/`body` (first shipped in browser 2025.8.0).
  - PM-25025: close the menu if any `:popover-open` element exists (first shipped in 2025.8.1).
  - PM-25122: put the menu itself in the top layer (`popover="manual"`) and keep re-asserting its position above other top-layer content (first shipped in 2025.8.2).
  - PM-27797 (2025.12.0): detect pages fighting for top-layer position and turn the menu off with an `alert()`.
- So "opacity only" is accurate for the first commit only. The repo later added structural defenses against the overlay and top-layer variants.

---

## 2. Background concepts

### 2.1 Classic clickjacking vs DOM-based extension clickjacking

| | Classic clickjacking | DOM-based extension clickjacking |
|---|---|---|
| Victim UI | A third-party page framed in an `<iframe>` on the attacker's page | UI that the **extension injects into the attacker's own page** |
| Attacker trick | Make the framed page invisible or covered, line it up under a decoy button | Make the injected UI invisible or covered, or ensure the decoy sits on top of it |
| Classic defense | `X-Frame-Options`, CSP `frame-ancestors` | Not applicable. The page is the "host" and legitimately controls its own DOM and CSS |
| Why it works | Victim page cannot control how it is displayed | The extension UI lives **inside** the attacker's document tree, so it is subject to the attacker's CSS (inheritance of compositing effects, stacking, hit-testing) |

The attacker never needs to read the extension UI. They only need the user's click to land on a control in it. The result of that click (the fill) goes into **form fields the attacker controls**, which the attacker's page can read.

### 2.2 How Bitwarden injects the inline menu (from the code)

Structure on current main and at 2025.7.0:

```
<body>
  <random-custom-element popover="manual">   <- "host", created by the content script
    #shadow-root (closed)
      <iframe src="chrome-extension://.../overlay/menu.html">   <- extension page
        <iframe sandbox="allow-scripts" credentialless src=".../overlay/list.html">   <- real list/button UI
```

- Host element: random custom element name (or a plain `div` on Firefox). `apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts`, `createButtonElement` / `createListElement` (7.0: lines 217-261; main: lines 327-381). Appended as the last child of `document.body` (or of a modal dialog), see `appendInlineMenuElementToDom`.
- Closed shadow root: `element.attachShadow({ mode: "closed" })` in `.../iframe-content/autofill-inline-menu-iframe-element.ts` (7.0 line 11; main line 14).
- Iframe whose `src` is an extension page: `BrowserApi.getRuntimeURL("overlay/menu.html")` in `.../iframe-content/autofill-inline-menu-iframe.service.ts` (`initMenuIframe`; 7.0 line 84, main line 90). The page it loads, `pages/menu-container/autofill-inline-menu-container.ts`, creates a nested iframe with `sandbox: "allow-scripts"` and `credentialless` for the actual button or list page.
- Content scripts run in an isolated world, so the page's JavaScript cannot call into the extension's objects.

### 2.3 What the page can and cannot influence

**Cannot:**
- Read or script the contents of the iframe (cross-origin extension page, closed shadow root hides the iframe node from `host.shadowRoot`, and the content script is in an isolated JS world).
- Make the iframe's own inline styles stick: they are set with `!important` on the iframe element and guarded by a MutationObserver (see section 4).

**Can (general web-platform analysis, not specific to Toth's write-up):**

| Lever | Why it matters |
|---|---|
| Styles on the host element (by tag name, class, `*`, `::before`/`::after`, `::backdrop`) | The host lives in the page's style scope |
| Compositing properties on **ancestors** (`opacity`, `filter`, `transform`, `clip-path`, `mask`, `mix-blend-mode`, `visibility`, `content-visibility`) on `<html>`/`<body>` | These apply to the whole subtree, including a closed shadow root and iframe. Plain `opacity` is the clearest case, because the host's `all: initial` resets the host's own `opacity` but cannot cancel an ancestor's group opacity |
| Elements painted on top of the menu (higher z-index, later in DOM order, or in the top layer) | Hides or replaces what the user sees |
| `pointer-events: none` on the overlay | The decoy is visible but clicks fall through to the real menu under it |
| Partial overlay | Cover all of the menu except a small hole; the user clicks the hole, which is part of a list item |
| Top layer (Popover API, `<dialog>.showModal()`, fullscreen) | Top-layer elements paint above all normal z-index content, so a page can stack a decoy above the menu if the menu is not in the top layer, or above it if the menu is lower in the top-layer order |
| Page scripts focusing fields, scrolling, moving fields | Trigger when and where the menu opens |
| Mouse-following decoys | The attacker moves the decoy so the next click lands on a menu item |

### 2.4 Variants and attribution

- Variants I could attribute to Toth's research (only via search summaries of his work, see section 1): element opacity, root/parent opacity, full or partial overlay, `pointer-events: none` overlay, and the top-layer technique for staying on top.
- Everything else in the table above (filter, clip-path, mask, transform, visibility, content-visibility, mix-blend-mode, viewport tricks) is **my general analysis**, not confirmed to be in his write-up.

---

## 3. How the attack maps onto Bitwarden's code

1. **A field gets focus.** Page script calls `field.focus()` or the user clicks. `AutofillOverlayContentService.handleFormFieldFocusEvent` -> `triggerFormFieldFocusedAction` (`apps/browser/src/autofill/services/autofill-overlay-content.service.ts`, main line ~1019) -> background.
2. **Menu is shown.** Background `openInlineMenuOnEmptyField` (`apps/browser/src/autofill/background/overlay.background.ts`, main ~2487). If the user setting is `AutofillOverlayVisibility.OnFieldFocus` (value 2, `libs/common/src/autofill/constants/index.ts`), both button and list are positioned immediately; otherwise only the button, and the list opens when the button is clicked. The content service then handles `appendAutofillInlineMenuToDom` -> `appendButtonElement` / `appendListElement`.
3. **User clicks a list item.** In `pages/list/autofill-inline-menu-list.ts` the handler ends in:

```ts
// main, line ~1321 (7.0: line 956, identical body)
private triggerFillCipherClickEvent = (cipher: InlineMenuCipherData, usePasskey: boolean) => {
  ...
  this.postMessageToParent({
    command: "fillAutofillInlineMenuCipher",
    inlineMenuCipherId: cipher.id,
    usePasskey,
  });
};
```

4. **Message path.** The nested page posts to its parent, the container page forwards to the background port. `fillAutofillInlineMenuCipher` is in the container's allowlist `ALLOWED_BG_COMMANDS` (`pages/menu-container/autofill-inline-menu-container.ts`, main line 14-31).
5. **Fill.** Background `fillInlineMenuCipher` (`overlay.background.ts`, main line 1453; 7.0 line 1125):

```ts
if (await this.autofillService.isPasswordRepromptRequired(cipher, tab)) { return; }
...
const result = await this.autofillService.doAutoFill({
  tab, cipher, pageDetails,
  fillNewPassword: true,
  allowTotpAutofill: true,
  ...
});
...
if (totpCode) { this.platformUtilsService.copyToClipboard(totpCode); }
```

What one click can do (from code):

- **Login**: fills username and password into whatever fields the page declared (`doAutoFill`). With `allowTotpAutofill: true` it also fills a TOTP field and copies the TOTP code to the clipboard.
- **Card and identity**: the menu supports `CipherType.Card` and `CipherType.Identity` fill types (`inlineMenuFillType === CipherType.Card` at `autofill-inline-menu-list.ts` 7.0 line 1624; `InlineMenuFillType` in `enums/autofill-overlay.enum.ts`). The same `fillAutofillInlineMenuCipher` command fills them.
- **Passkey item**: `authenticatePasskeyCredential` (`overlay.background.ts` main 1566) only calls `request.subject.next({ type: Continue, credentialId })` on an **already active** WebAuthn request for that tab. That means the page must have started `navigator.credentials.get`, and the assertion is bound to the page's own origin. My analysis: low value for an attacker, but I did not test it.
- **Mitigation already present**: items with master password reprompt do not fill on click, they open a reprompt popout (`isPasswordRepromptRequired`, `autofill.service.ts` main ~685).
- **Where the data goes**: into the page's own inputs, which the attacker can read (or which can be non-visible). Filling is gated by `viewable` checks in `collect-autofill-content.service.ts` (`isElementViewable`), but I did not audit how well that stops off-screen or zero-size field tricks. That is a separate topic.

---

## 4. Vulnerable state: browser 2025.7.0

Tag `browser-v2025.7.0` (tag commit `62fe7ee44a`, 2025-07-18). 2025.7.1 (`90cea802b9`, 2025-08-07) also does not contain the opacity fix (checked: `git tag --contains e7059b790d` starts at 2025.8.0).

Defenses that **already existed** in 7.0, all in `apps/browser/src/autofill/overlay/inline-menu/`:

**a) Forced inline styles on the host.** `content/autofill-inline-menu-content.service.ts` (7.0):

```ts
40  private readonly customElementDefaultStyles: Partial<CSSStyleDeclaration> = {
41    all: "initial",
42    position: "fixed",
43    display: "block",
44    zIndex: "2147483647",
45  };
...
269 private updateCustomElementDefaultStyles(element: HTMLElement) {
270   this.unobserveCustomElements();
272   this.setElementStyles(element, this.customElementDefaultStyles, true);   // true => "important"
274   this.observeCustomElements();
```

`setElementStyles` (`apps/browser/src/autofill/utils/index.ts:139`) does `element.style.setProperty(prop, value, "important")`. Inline `!important` beats any author stylesheet rule.

**b) MutationObserver that wipes tampering on the host** (7.0 lines 338-359):

```ts
if (record.attributeName !== "style") { this.removeModifiedElementAttributes(element); continue; }
element.removeAttribute("style");
this.updateCustomElementDefaultStyles(element);
```

So setting `opacity:0` directly on the host element gets reverted. This covers the "direct element opacity" variant.

**c) Same pattern on the inner iframe** (`iframe-content/autofill-inline-menu-iframe.service.ts`, 7.0 lines 26-41, 399-420, 436-455): the iframe gets `visibility: visible`, `clipPath: "none"`, `pointerEvents: "auto"`, `zIndex: 2147483647`, plus `opacity` managed for the fade-in; style and attribute mutations are reverted; more than 10 foreign attribute changes or more than 20 mutation iterations in 2 s force-close the menu (`forceCloseInlineMenu`).

**d) Closed shadow root** (`iframe-content/autofill-inline-menu-iframe-element.ts` 7.0 line 11), so the page cannot get at the iframe node.

**e) DOM order enforcement and basic obscuring check** (7.0 lines 384-497). A `childList` observer on the container keeps the button/list as the last children. If some other element keeps forcing itself last (3 times), the service lowers its z-index and, 500 ms later, checks the element at the center of the menu:

```ts
471 private verifyInlineMenuIsNotObscured = async (lastChild: Element) => {
...
477   if (!!button && this.elementAtCenterOfInlineMenuPosition(button) === lastChild) { this.closeInlineMenu(); ...
492 private elementAtCenterOfInlineMenuPosition(position) {
493   return globalThis.document.elementFromPoint(
494     position.left + position.width / 2, position.top + position.height / 2);
```

**What was missing in 7.0** (verified by reading the whole file and by `git log -S`): nothing looked at `<html>` or `<body>` styles at all. `grep -n "getComputedStyle" ...content.service.ts` at tag 7.0 has no match; the file's only style logic is on the inline-menu elements themselves. Because the host sits directly under `<body>`, `body { opacity: 0 }` or `html { opacity: 0 }` fades the whole menu and no observer notices.

**Earlier related fix (context):** `4d05b008f0` "[PM-5035] Fix autofill overlay clickjacking vulnerability that can be triggered by a malicious extension (#7001)", 2023-12-11, first in browser 2024.1.0. It added the iframe attribute and mutation hardening and the forced-close counters. It targeted a different threat (another extension tampering) but is the origin of the style-reverting pattern.

---

## 5. The fixes, in chronological order

Each fix exists as **two or three commits**: one on `main`, and cherry-picks on release branches (different hashes, same subject, same diff). The table lists all hashes and the first `browser-v*` tag containing each, from `git tag --contains <hash> | grep browser | sort -V | head -1`.

| Change | main hash | Release-branch hash(es) | Authored | First browser tag containing it |
|---|---|---|---|---|
| PM-24936 opacity | `e942645d44` | `e7059b790d` | 2025-08-18 (rc commit 2025-08-19) | main: 2025.9.0; rc: **2025.8.0** |
| PM-25025 popover-open | `b87cb2ba24` | `4af4a863be` (in 2025.8.1), `b72dd28aa5` | 2025-08-22 | main: 2025.9.0; rc: **2025.8.1** |
| PM-25122 top-layer menu | `8aba7757ab` | `f921012080` (in 2025.8.2), `909d5bdf6b` | 2025-08-28 | main: 2025.9.0; rc: **2025.8.2** |
| Pseudo-element guard | `b0179bd105` | none | 2025-10-07 | 2025.11.0 |
| PM-27915 pseudo-element 2 | `df03664827` | none | 2025-11-18 | 2025.12.0 |
| PM-27797 popover attr + backoff | `7c4db701b9` | none | 2025-11-19 | 2025.12.0 |
| PM-27798 viewport | `f17890a26b` | none | 2025-12-02 | 2025.12.1 |
| top-layer refresh | `3f466c4b4c` | none | 2026-01-21 | 2026.1.0 |
| PM-28831 isTrusted | `11e2b25ede` | none | 2026-02-11 | 2026.2.0 |

Browser tag dates: 2025.8.0 `afbe27591b` 2025-08-19; 2025.8.1 `553e3ca18f` 2025-08-22; 2025.8.2 `b629d07014` 2025-08-28; 2025.9.0 `9612a4ac45` 2025-09-09.

Note: the lead commit `e7059b790d` is the **release-branch copy** that shipped in 2025.8.0. The same change on main is `e942645d44` (PR #16063, same title, author Jonathan Prusik, same 2 files, +98/-3). Commit-message PR numbers come from the commit titles; I could not open the PR pages.

### 5.1 PM-24936, `e942645d44` / `e7059b790d`: "Prevent inline menu inheritance of potentially dangerous opacity from host body and above" (#16063)

Files: `.../inline-menu/content/autofill-inline-menu-content.service.ts` (+59/-3) and its `.spec.ts` (+42 lines), 98 insertions and 3 deletions in total.

Key diff:

```ts
+  private htmlMutationObserver: MutationObserver;
+  private bodyMutationObserver: MutationObserver;
+  private pageIsOpaque = true;
...
   constructor() {
+    this.checkPageOpacity();
     this.setupMutationObserver();
   }
...
   private observeCustomElements() {
+    this.htmlMutationObserver?.observe(document.querySelector("html"), { attributes: true });
+    this.bodyMutationObserver?.observe(document.body, { attributes: true });
...
+  private checkPageOpacity = () => {
+    this.pageIsOpaque = this.getPageIsOpaque();
+    if (!this.pageIsOpaque) { this.closeInlineMenu(); }
+  };
+
+  private handlePageMutations = (mutations: MutationRecord[]) => {
+    for (const mutation of mutations) {
+      if (mutation.type === "attributes") { this.checkPageOpacity(); }
+    }
+  };
+
+  /** Checks the opacity of the page body and body parent, since the inline menu experience
+   *  will inherit the opacity, despite being otherwise encapsulated from styling changes
+   *  of parents below the body. Assumes the target element will be a direct child of the page
+   *  `body` (enforced elsewhere). */
+  private getPageIsOpaque() {
+    const htmlOpacity = globalThis.window.getComputedStyle(globalThis.document.querySelector("html")).opacity;
+    const bodyOpacity = globalThis.window.getComputedStyle(globalThis.document.querySelector("body")).opacity;
+    // Any value above this is considered "opaque" for our purposes
+    const opacityThreshold = 0.6;
+    return parseFloat(htmlOpacity) > opacityThreshold && parseFloat(bodyOpacity) > opacityThreshold;
+  }
...
   private processContainerElementMutation = async (containerElement: HTMLElement) => {
+    // If the computed opacity of the body and parent is not sufficiently opaque, tear
+    // down and prevent building the inline menu experience.
+    this.checkPageOpacity();
+    if (!this.pageIsOpaque) { return; }
```

What it checks: computed `opacity` of `<html>` and `<body>` must both be greater than 0.6. When it checks: after any attribute mutation on `html` or `body` (observers attached the first time an inline-menu element's styles are set), and whenever the menu's container gets a child added or removed (this includes the menu being appended). What it does on failure: `closeInlineMenu()` (removes button and list from the DOM and tells the background), and `processContainerElementMutation` returns before re-ordering.

Observations from the diff:
- The constructor call to `checkPageOpacity()` runs before any menu element exists, so I believe it has no effect in practice (nothing to close). Untested. PM-25025 removed it.
- Because the check is driven by observers, it is reactive. There is a window between the menu being appended and the idle-callback check (`requestIdleCallbackPolyfill`, timeout 500 ms). That is my reading of the code, not a demonstrated exploit.
- Only attribute mutations are watched, so a change made through a stylesheet (not through an attribute on `html`/`body`) is only caught at the next container mutation.

Tests added (spec): "closes the inline menu if the page body is not sufficiently opaque" (body 0), "... html is not sufficiently opaque" (html 0.3), "does not close ... if html and body is sufficiently opaque".

### 5.2 PM-25025, `b87cb2ba24` / `4af4a863be`: "Additional defense against top-layer content" (#16101)

Files: same service + spec (+53/-22 total). Renames `checkPageOpacity` to `checkPageRisks`, removes the constructor call and the `pageIsOpaque` field, and adds:

```ts
+  private checkPageRisks = async () => {
+    const pageIsOpaque = await this.getPageIsOpaque();
+    const pageTopLayerInUse = await this.getPageTopLayerInUse();
+    const risksFound = !pageIsOpaque || pageTopLayerInUse;
+    if (risksFound) { this.closeInlineMenu(); }
+    return risksFound;
+  };
...
+  private getPageTopLayerInUse = () => {
+    const pageHasOpenPopover = !!globalThis.document.querySelector(":popover-open");
+    return pageHasOpenPopover;
+  };
```

What it checks: any open popover on the page. On failure: close the menu. This is the "do not render the inline menu if the page has an open popover" behaviour that secondhand coverage attributes to 2025.8.1, and git agrees (4af4a863be is first in 2025.8.1). It is superseded 6 days later by PM-25122 (the `getPageTopLayerInUse` call is removed).

### 5.3 PM-25122, `8aba7757ab` / `f921012080`: "Top-layer inline menu population" (#16175)

Files (6): `content/autofill-inline-menu-content.service.ts`, abstractions, `services/autofill-overlay-content.service.ts`, `services/collect-autofill-content.service.ts`, specs. The strategy changes from "bail out if the page uses the top layer" to "join the top layer and stay on top".

The menu elements become manual popovers:

```ts
// createButtonElement / createListElement
+      this.buttonElement.setAttribute("popover", "manual");
...
   if (!(await this.isInlineMenuButtonVisible())) {
     this.appendInlineMenuElementToDom(this.buttonElement);
     this.updateInlineMenuElementIsVisibleStatus(AutofillOverlayElement.Button, true);
+    this.buttonElement.showPopover();
```

Top-layer ordering is the order of `showPopover()` calls, so the menu must re-open after any other top-layer element does:

```ts
+  getUnownedTopLayerItems = (includeCandidates = false) => {
+    ...
+    const selector = [":modal", inlineMenuTagExclusions, ...(includeCandidates ? ["[popover], dialog"] : [])].join(",");
+    return globalThis.document.querySelectorAll(selector);
+  };
+
+  refreshTopLayerPosition = () => {
+    const otherTopLayerItems = this.getUnownedTopLayerItems();
+    if (!otherTopLayerItems.length) { return; }
+    ...
+    if (buttonInDocument) { buttonInDocument.hidePopover(); buttonInDocument.showPopover(); }
+    if (listInDocument)   { listInDocument.hidePopover();   listInDocument.showPopover(); }
+  };
```

And `collect-autofill-content.service.ts` watches for top-layer candidates (`<dialog>`, elements with `popover`/`popovertarget`/`popovertargetaction` attributes), including ones added later via mutation records, and refreshes the menu when one opens:

```ts
+  element.addEventListener("toggle", (event: ToggleEvent) => {
+    if (event.newState === "open") {
+      // Add a slight delay (but faster than a user's reaction), to ensure the layer
+      // positioning happens after any triggered toggle has completed.
+      setTimeout(this.autofillOverlayContentService.refreshMenuLayerPosition, 100);
+    }
+  });
```

Also in this commit: html/body observers are moved out of `observeCustomElements()` into a new `observePageAttributes()` called from `setupMutationObserver()` (so they are active from construction), `getPageIsOpaque` now fails closed when `html`/`body` is missing, and `destroy()` unobserves them. A `@TODO` is added: "for definitive checks, traverse up the node tree from the inline menu container; nodes can exist between `html` and `body`".

### 5.4 Pseudo-element guards, `b0179bd105` then `df03664827` (PM-27915)

First attempt (2025-10-07): a `<style>` node inside the host (light DOM) with `::backdrop { background:none; pointer-events:none }` and `::before, ::after { content:"" }`. The second commit (2025-11-18) moved this into the **closed shadow root**, with a much larger rule set:

```css
:host::backdrop, :host::before, :host::after {
  all: initial !important;
  backdrop-filter: none !important;  filter: none !important;
  display: none !important;          position: relative !important;
  transform: none !important;        opacity: 1 !important;
  mix-blend-mode: normal !important; z-index: 0 !important;
  background: none !important;       width: 0 !important; height: 0 !important;
  content: "" !important;
  ...
}
```

File: `iframe-content/autofill-inline-menu-iframe-element.ts` (main lines 34-71). Reason: once the host is a popover, its `::backdrop` pseudo-element is styleable by page rules (for example a full-screen decoy backdrop), and `::before`/`::after` on the host are drawn by the page's CSS. The commit messages only say "prevent pseudo-elements from being targeted and styled by host page's global rules"; the attack scenario is my inference.

### 5.5 PM-27797, `7c4db701b9`: "Prevent host page manipulation of inline menu popover attribute" (#17400), 2025-11-19

Files: content service (+138 lines changed), spec (+490 lines), `_locales/en/messages.json`. Adds a fail-safe against pages that fight for the top of the top layer:

```ts
const experienceValidationBackoffThresholds = {
  topLayer:        { countLimit: 5,  timeSpanLimit: 5000 },
  popoverAttribute:{ countLimit: 10, timeSpanLimit: 5000 },
};
...
private inlineMenuEnabled = true;
...
// in the host-element mutation handler
if (record.attributeName === "popover" && this.inlineMenuEnabled) {
  if (element.getAttribute("popover") !== "manual") { this.refreshPopoverAttribute(element); }
  continue;
}
...
// in checkAndUpdateRefreshCount
} else {
  // Set inline menu to be off; page is aggressively trying to take top position of top layer
  this.inlineMenuEnabled = false;
  void this.checkPageRisks();
  const warningMessage = chrome.i18n.getMessage("topLayerHijackWarning");
  globalThis.window.alert(warningMessage);
}
```

Message text (`_locales/en/messages.json`): "This page is interfering with the Bitwarden experience. The Bitwarden inline menu has been temporarily disabled as a safety measure."

What it checks: how often the menu has had to be re-promoted (top-layer refreshes, or the page rewriting the `popover` attribute) within 5 s. On failure: permanently (until reload) disables creation and appending of menu elements (`if (!this.inlineMenuEnabled) return;` guards were added to `appendInlineMenuElements`, `appendButtonElement`, `appendListElement`, `createButtonElement`, `createListElement`), closes the menu, and shows a page `alert()`.

It also rewrote `getPageIsOpaque` to use `document.querySelectorAll("html, body")` and require **every** match to have opacity above 0.6 (covers duplicate `html`/`body` nodes in non-standard documents).

### 5.6 Supporting hardening in the same period (related, but I cannot tie them to this specific disclosure from git)

- `f17890a26b` PM-27798 (2025.12.1): iframe service `updateIframePosition` now calls `isElementCompletelyWithinViewport(this.iframe.getBoundingClientRect())` and `forceCloseInlineMenu()` if any edge is outside the (visual) viewport (`autofill-inline-menu-iframe.service.ts`, main 283-343).
- `11e2b25ede` PM-28831 (2026.2.0): `utils/event-security.ts` adds `EventSecurity.isEventTrusted(event)` (`return event.isTrusted`), and every click and key handler in the list page, page-element base class and Lit components rejects untrusted (script-dispatched) events. This blocks synthetic `click()`/`dispatchEvent` against the menu; it does **not** stop a real user click that was tricked (a real click is `isTrusted === true`).
- `b1acff7f5c` (PM-27900, 2025.12.0) and `3de3bee08f` (PM-27821, 2025.12.0): extension-frame validation, extension-origin checks on every `postMessage`, a per-session 32-character token between the container page and its iframe. These protect the messaging channel, not the visual layer.
- `364e9b3054` (2026.3.0): adds the `credentialless` attribute to injected iframes (main: `defaultIframeAttributes` in the iframe service and container). Isolation hardening, not a clickjacking defense.
- `3f466c4b4c` (2026.1.0) and `4ad91a3b46` (2026.9.0): refresh the menu position as soon as a top-layer candidate listener is set up; de-duplicate `toggle` listeners with a `WeakSet`. The latter's message notes that `toggle` events on dialogs do not exist before Chrome 134 / Firefox 136, so the refresh runs on every observation as a safety net.
- `6f3d3239c2` PM-35399 (2026.5.0), a performance change that **narrowed** the opacity monitoring, see section 7.

---

## 6. The code today (main, `5a00c60`)

All in `apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts` unless noted.

| Defense | Where |
|---|---|
| Forced host styles (`all: initial; position: fixed; z-index: 2147483647`, important) | `customElementDefaultStyles` lines 68-73; applied by `updateCustomElementDefaultStyles` line 389 |
| Host tamper revert (style wiped, other attributes removed, `popover` forced back to `manual`) | `handleInlineMenuElementMutationObserverUpdate` lines 475-505 |
| Menu is a manual popover (top layer) | lines 330, 349, 360, 379; `showPopover()` lines 243, 259 |
| `html`/`body` attribute observers | `observePageAttributes` lines 421-439, filter `["style","hidden","popover","width","height"]` (line 424) |
| Page risk check | `checkPageRisks` lines 544-554: closes menu if `!pageIsOpaque \|\| !inlineMenuEnabled` |
| Opacity check | `getPageIsOpaque` lines 676-699: every `html, body` must have computed opacity > 0.6 |
| Check on container mutation (includes the menu being appended) | `processContainerElementMutation` lines 705-710 |
| Re-promote menu above other top-layer content | `refreshTopLayerPosition` lines 636-668; `getUnownedTopLayerItems` lines 582-596 |
| Backoff and kill switch with `alert()` | `experienceValidationBackoffThresholds` lines 23-32, `checkAndUpdateRefreshCount` lines 602-628 |
| Last-child override + `elementFromPoint` obscure check | lines 726-736, 768-814 |
| Excessive mutation circuit breaker (> 100 in 2 s) | `isTriggeringExcessiveMutationObserverIterations` lines 832-852 |
| Observe lifecycle | `startMonitoring()` / `stopMonitoring()` lines 102-130 (autofill lifecycle work, PM-37555) |

Other files:
- `.../iframe-content/autofill-inline-menu-iframe-element.ts` lines 13-71: closed shadow root plus pseudo-element reset stylesheet.
- `.../iframe-content/autofill-inline-menu-iframe.service.ts` lines 30-53 (iframe forced styles, `credentialless`), 283-343 (viewport check), and its own mutation observer.
- `apps/browser/src/autofill/services/collect-autofill-content.service.ts` lines 1486-1492 (`setupInitialTopLayerListeners`), 1741-1765 (`setupTopLayerCandidateListener`), 1772+ (`shouldListenToTopLayerCandidate`).
- `apps/browser/src/autofill/utils/event-security.ts`: `isEventTrusted`.

Container choice changed after the fix: `getInlineMenuContainerElement` (lines 303-321) returns a `<dialog>` that is `:modal`, **or an ARIA modal element** (`[role="dialog"][aria-modal="true"]`, added by PM-26503, `7cf20064b4`, 2026-07-30, browser 2026.8.0), otherwise `document.body`. See 7.5 for why this matters.

---

## 7. Residual risk: is the fix partial?

**Method.** For each claim "the code does not appear to check X", I grepped the inline-menu directory on current main (non-spec files), from `apps/browser/src/autofill/`:

```
grep -rn -e "<pattern>" overlay/inline-menu --include=*.ts | grep -v "\.spec\.ts"
```

Patterns run: `checkVisibility`, `elementsFromPoint`, `elementFromPoint`, `IntersectionObserver`, `visibilitychange`, `getComputedStyle`, `pointer-events|pointerEvents`, `clip-path|clipPath`, `\bmask\b|mask-image`, `scale(`, `\bfilter:|filter(`, `contentVisibility|content-visibility`, `z-index|zIndex`, `isTrusted`. Plus, for the whole autofill tree: `fullscreen|requestFullscreen|showModal`.

Results that matter:
- `checkVisibility`, `elementsFromPoint`, `IntersectionObserver`, `visibilitychange`, `mask`, `scale(`, `content-visibility`, `fullscreen`, `requestFullscreen`, `showModal`: **no matches** (non-spec).
- `elementFromPoint`: exactly one use, `autofill-inline-menu-content.service.ts:810`.
- `getComputedStyle`: exactly one real use, `:692`, reading **only `.opacity`**.
- `pointerEvents`/`clipPath`: only as values the extension **sets** on its own iframes (`iframe.service.ts:40`, `container.ts:55-56`), never as page checks.
- `filter`: only in the pseudo-element reset stylesheet.
- `checkPageRisks` has three callers: the html/body mutation handler, `processContainerElementMutation`, and the kill-switch path (verified with `grep -rn checkPageRisks`).

Everything below is **analysis from reading code; none of it was exploited or tested in a browser.**

### 7.1 Variant coverage table

| Variant | Covered? | Evidence and caveats |
|---|---|---|
| Opacity on the host element itself | Yes (already in 7.0) | Inline `!important` styles, host mutation observer reverts them |
| Opacity on `html` / `body` | Yes, with limits (7.2) | `getPageIsOpaque` |
| Opacity on a parent between `html` and `body` | Not applicable normally | The host is a direct child of `body`; `@TODO` notes this is not "definitive" |
| Other compositing effects on `html`/`body` (`filter`, `clip-path`, `mask`, `transform`, `visibility`, `content-visibility`, `mix-blend-mode`, `backdrop-filter`) | **Not checked** by code | Grep: only `.opacity` is read. Possible mitigation is that the menu is now in the top layer, which (to my understanding) renders independently of ancestors' effects; **untested, so I cannot say it fully neutralizes them**. Note the extension's own pseudo-element stylesheet explicitly resets `filter`, `transform`, `mix-blend-mode` for `:host::*` only |
| Overlay on top using z-index/DOM order | Largely yes | Menu is last-child (observer re-orders), z-index at the max, a persistent competing last child is z-index capped and, if it sits at the menu center, the menu closes. Structurally strengthened by being in the top layer |
| Overlay with `pointer-events: none` over the menu | **Weak in the legacy check** | `elementFromPoint` ignores `pointer-events: none` elements, so the "is something covering me" check cannot see such an overlay. What protects here is the top layer: a non-top-layer decoy cannot paint above a top-layer menu (general platform behaviour, untested here) |
| Partial overlay leaving a hole | **Not detected by that check** | The legacy check samples **one point** (the center) and only compares it to the container's `lastChild`. A decoy that leaves the center uncovered, or is not the last child, passes |
| Another top-layer element (popover, `<dialog>.showModal()`) above the menu | Yes (best-effort) | Re-promotion on `toggle`, with a 100 ms delay, plus a per-5 s backoff kill switch. Gaps: top-layer items that do not emit `toggle` (the repo's own commit message says dialogs lack `toggle` before Chrome 134 / Firefox 136, with a refresh on every observation as the fallback); the 100 ms window; **fullscreen** elements (no code references `requestFullscreen`/`fullscreen`; browsers usually require a user gesture for it) |
| Page rewrites `popover` attribute | Yes | PM-27797 |
| `::backdrop`, `::before`, `::after` of host | Yes | Reset stylesheet inside closed shadow root |
| Page calls `.click()`/`dispatchEvent` | Yes | `isTrusted` checks (PM-28831) |
| Menu outside viewport | Yes | PM-27798 |
| Notification bar (save/update prompt) | **No equivalent defenses** | See 7.6 |
| Passkey prompts | Different UI path | See 7.7 |

### 7.2 Opacity check details and gaps

1. **Threshold is 0.6.** Anything above 0.6 counts as opaque (`getPageIsOpaque`, `opacityThreshold = 0.6`). A page can keep `body` at, say, 0.65 and make the menu noticeably faded. Whether that is exploitable is a UX judgement, but it is not "fully opaque".
2. **Only `html` and `body`.** No `filter`/`clip-path`/`transform`/`mask`/`visibility` checks (see grep above).
3. **The trigger set was narrowed in 2026.** Commit `6f3d3239c2` (PM-35399, 2026-05-13, first in 2026.5.0) limited the html/body observers to:

```ts
// observePageAttributes, main line 424
// FIXME: find a more efficient means to monitor attribute changes so that indirect
// uses of attributes can be monitored without impacting the layout hot path.
const attributeFilter = ["style", "hidden", "popover", "width", "height"];
```

   The stated reason is performance (`getComputedStyle` forces layout). The consequence is that a change of `class`, `id`, `data-*`, `lang`, etc. on `html`/`body` (which can flip opacity through a stylesheet rule) no longer triggers a re-check, nor does inserting or modifying a `<style>`/`<link>`, a media-query flip, a `:hover`/`:has()` rule, or a **CSS animation or transition on opacity**. The code itself acknowledges this: "indirect uses of attributes" are not monitored.
4. **Checks happen at discrete times** (html/body attribute change, container child change), not continuously. There is no timer, no `visibilitychange`, no `IntersectionObserver`, no re-check on the click path. So an attacker who changes opacity after the post-append check through a non-observed route would not be caught in the code I read.

### 7.3 The legacy "obscured" check is narrow

`verifyInlineMenuIsNotObscured` runs only after the same non-menu element has been last-child three times, then waits 500 ms, then checks **one point** (center) against **that one element**. It cannot detect: overlays that are not the container's last child, overlays that leave the center clear, `pointer-events: none` overlays, or overlays in the top layer. Top-layer re-promotion (PM-25122) is the stronger mechanism for those.

### 7.4 Top-layer defenses are race-prone by design

The menu "wins" by being the most recent `showPopover()`. Re-promotion is event-driven with a 100 ms delay (`setTimeout(..., 100)`), and after more than 5 refreshes in 5 s it gives up and disables the menu (an availability trade-off: a hostile page can turn the inline menu off, which the `alert()` makes visible). During the delay a decoy could sit above the menu. The comment says the delay is "faster than a user's reaction", which is a probabilistic argument.

### 7.5 Container selection (post-fix change)

`getInlineMenuContainerElement` can append the menu inside a page-supplied ARIA modal element (PM-26503, 2026). `getPageIsOpaque` still only checks `html` and `body`, never that container or its ancestors, and the code's own `@TODO` says nodes between `html` and the container are not covered. Whether top-layer rendering makes this moot is **untested**. Worth raising as a question rather than a finding.

### 7.6 Notification bar (`overlay/notifications`)

`apps/browser/src/autofill/overlay/notifications/content/overlay-notifications-content.service.ts`:
- Host element has a **fixed tag name**, `bit-notification-bar-root` (line 252), closed shadow root (line 256), appended to `document.body` (line 182).
- Styles are applied to the inner `div` and iframe, not to the host. I found no host-style forcing, no `popover`, no MutationObserver, no `getComputedStyle`, no `elementFromPoint` in this file. `git log -S"getComputedStyle"` over `overlay/notifications` and `notification/` returned no commits; the `opacity` hits in history are animation-related (`a966e75576`, `7ce8d06315`, `5b4e4d8f1a`).
- The `isTrusted` rework (PM-28831) did cover the Lit notification components.
- Impact is lower by design: the bar's actions (`bgSaveCipher`, `bgOpenChangePasswordUrl`, `bgOpenAddEditVaultItemPopout`, `bgOpenViewVaultItemPopout`, `bgCloseNotificationBar`, `bgOpenAtRiskPasswords` in `notification/bar.ts`) do not fill page fields with vault secrets. A hijacked click could still save or edit data or open a popout. I did not audit each action for harm.

### 7.7 Passkey prompts

The inline-menu passkey path only resumes an already pending WebAuthn request for the same tab (section 3). The browser's own passkey UI and Bitwarden's popout windows are not injected into the page DOM, so this class of attack does not apply to them. I did not find any passkey-specific clickjacking changes in history (grep of `git log` for opacity/clickjack in `apps/browser` shows only the commits listed in this document and `c1d856430a`, an unrelated 2023 passkey feature).

### 7.8 Bottom line

The statement "the fix may be partial" is **fair**, with a more precise framing. The first shipped fix (2025.8.0) covered the root/parent opacity variant only. The overlay and top-layer variants were added across 2025.8.1, 2025.8.2 and 2025.12.0, and the structural change (top-layer menu) is a stronger defense than checks. Remaining theoretical gaps from the code: compositing effects other than opacity, opacity changes through non-observed routes after the 2026 narrowing, partial/`pointer-events` overlays against the legacy check (if the menu were not in the top layer), top-layer races, and the notification bar having no equivalent protection. None of these has been shown to be exploitable here.

---

## 8. Interview talking points

1. **The core idea:** extension UI that lives inside the attacker's page is subject to the attacker's CSS. Encapsulation (closed shadow root, cross-origin iframe, isolated world) protects **confidentiality of the contents**, not **integrity of presentation**. Opacity on an ancestor is the cleanest example because compositing effects flow down regardless of encapsulation.
2. **What was already good in 7.0:** `all: initial` + inline `!important` styles, mutation observers that undo tampering, an excessive-mutation circuit breaker, DOM order enforcement. The gap was one level up: `html`/`body`. Commit `e7059b790d` (main copy `e942645d44`, PM-24936) fixed it in browser 2025.8.0.
3. **Defense in depth over a single patch:** within about two weeks and three releases (2025.8.0, .1, .2) the team went from "close if opacity is low" (PM-24936), to "close if a popover is open" (PM-25025), to "put the menu in the top layer and keep re-promoting it" (PM-25122). That is a good example of replacing a blocklist check with a structural fix.
4. **Fail-safe vs usability:** the backoff in PM-27797 turns the feature off (with an `alert()`) if the page fights back. Be ready to discuss the trade-off: a hostile page can deny the feature, but it cannot trick a click.
5. **Detect, don't just reset:** opacity is not a property the extension can force on an ancestor, so the only options are to detect and close, or to move out of the ancestor's rendering scope (top layer). Know why `getComputedStyle` is used, and why it was later throttled (performance, PM-35399) and what that cost in coverage.
6. **`isTrusted` is not a clickjacking fix:** it stops script-driven clicks, but a clickjacked click is a genuine user event. Good to show you know the difference (PM-28831).
7. **Blast radius controls:** master-password reprompt items do not fill on click, passkey path needs an active request, and fills go through a background allowlist of commands with a per-session token. Also `credentialless` and the `sandbox` attribute on the nested iframe.
8. **Be honest about limits:** compositing effects other than opacity are not checked, the opacity threshold is 0.6, the notification bar has no equivalent defenses, and I did not dynamically test any of this. Offer how you would test: a local page with `html{filter:opacity(0)}`, `body{transform:scale(0)}`, a `pointer-events:none` decoy, and a CSS-animated opacity, loaded with the extension in a dev profile.
9. **Process point:** one fix produced several commits (main plus release-branch cherry-picks with different hashes). Always check which hash is in which tag.

---

## 9. How to explore it yourself

Set `H` to a clone with full history (the study clone is at `/tmp/claude-0/-home-user-clients/fc39e059-3baf-5a22-a886-d4e3622d4f46/scratchpad/bw-history`).

```bash
# The lead commit and its main-branch twin
git -C $H show --stat e7059b790d
git -C $H show e942645d44 -- apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts

# Which release first contained each
for c in e7059b790d e942645d44 4af4a863be b87cb2ba24 f921012080 8aba7757ab 7c4db701b9; do
  echo "$c: $(git -C $H tag --contains $c | grep '^browser-v' | sort -V | head -1)"; done

# All commits on the inline-menu code since the vulnerable release
git -C $H log --format='%h %ad %s' --date=short browser-v2025.7.0..HEAD -- \
  apps/browser/src/autofill/overlay/inline-menu/content \
  apps/browser/src/autofill/overlay/inline-menu/iframe-content

# Search by subject or by code
git -C $H log -i --grep=opacity --grep=PM-24936 --grep=PM-25025 --grep=PM-25122 --grep=PM-27797 --format='%h %ad %s' --date=short -- apps/browser
git -C $H log -S"getPageIsOpaque" --format='%h %ad %s' --date=short
git -C $H log -S"showPopover" --format='%h %ad %s' --date=short -- apps/browser/src/autofill
git -C $H log -S"isTrusted" --format='%h %ad %s' --date=short -- apps/browser/src/autofill

# The vulnerable code at the tag, and the fixed one
git -C $H show browser-v2025.7.0:apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts | less
git -C $H diff browser-v2025.7.0 browser-v2025.8.2 -- apps/browser/src/autofill/overlay/inline-menu/content/autofill-inline-menu-content.service.ts

# Current code
cd /home/user/clients/apps/browser/src/autofill
less overlay/inline-menu/content/autofill-inline-menu-content.service.ts
less overlay/inline-menu/iframe-content/autofill-inline-menu-iframe-element.ts
less overlay/inline-menu/iframe-content/autofill-inline-menu-iframe.service.ts
less overlay/inline-menu/pages/menu-container/autofill-inline-menu-container.ts
grep -n "fillInlineMenuCipher" -A 60 background/overlay.background.ts | less
less overlay/notifications/content/overlay-notifications-content.service.ts

# Tests that encode the intended behaviour
less overlay/inline-menu/content/autofill-inline-menu-content.service.spec.ts   # search "sufficiently opaque", "top layer", "popover"
```

Suggested exercise: build the extension, load it in a profile, open a local HTML page with a login form, and try each variant (`body{opacity:0}`, `html{filter:opacity(0)}`, `body{transform:scale(0)}`, `pointer-events:none` overlay, a `<dialog>` opened after focus, a CSS animation of body opacity) while watching whether the menu closes. That would settle the "untested" items above.

---

## Appendix A. External claims vs code (summary)

| Claim | Verdict |
|---|---|
| Technique makes injected UI invisible via opacity 0 | Partly. Opacity is one variant; coverage also names overlays, `pointer-events: none`, partial overlay and top layer. Code history shows Bitwarden's fixes addressed opacity first, then top-layer/overlay |
| Could expose credentials, 2FA codes, card details | Consistent with code: inline menu fills Login, Card and Identity, fills TOTP (`allowTotpAutofill: true`) and copies TOTP to the clipboard. Reprompt items are blocked |
| Bitwarden 2025.7.0 affected | Code supports that 2025.7.0 (and 2025.7.1) had no `html`/`body` opacity check. Exploitability was not tested here |
| Fix released in 2025.8.0 | Supported for the opacity variant: `e7059b790d` is first in `browser-v2025.8.0` (tag date 2025-08-19). The popover check shipped in 2025.8.1 and the top-layer menu in 2025.8.2 |
| "Fix merged in version 2025.7.2" (forum snippet) | **Not supported by git for the browser extension.** There is no `browser-v2025.7.2` tag; only `web-v2025.7.2` exists. Browser tags are 2025.7.0, 2025.7.1, 2025.8.0 |
| Fix may be partial because not all variants need opacity | Supported in spirit: the 2025.8.0 fix is opacity-only; later commits address popover/top layer; see section 7 for what remains |
