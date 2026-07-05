# Cart free-gift / threshold progress bar — done right

A "spend ₹X, get a free gift" progress bar is a common ask. It's also a common way to accidentally ship a **serious revenue bug: customers checking out a ₹0 gift-only cart.** This is the pattern that avoids it.

## How it works

- Two tiers, each mapped to a $0 "gift" product variant. When the **qualifying subtotal** (cart total *excluding* gift lines) crosses a tier, auto-add that tier's gift via `/cart/add.js`. When it drops below, remove it via `/cart/change.js`.
- A bar UI is injected into `.cart-drawer__content` and re-injected on drawer re-render.
- Cart changes are detected theme-independently: intercept `window.fetch` **and** `XMLHttpRequest` for `/cart/(add|change|update|clear)`, plus listen to the theme's cart events and a `MutationObserver` on the drawer.

## The bug that strands a free gift

Naive implementations share **one debounce timer** and pass a `reconcile` flag (true = "add/remove gifts", false = "just repaint"). Sequence when a shopper deletes their only paid item:

1. The delete calls `/cart/change.js` → your fetch hook fires `debounced(true)` → schedules a reconcile that *would* remove the now-unqualified gift.
2. The theme re-renders the drawer → your `MutationObserver` fires `debounced(false)` → **clears the timer and reschedules with reconcile = false**.
3. The timer runs, repaints, and **never removes the gift.** Cart is now gift-only, total ₹0, checkout allowed.

## The three fixes (all required)

### 1. Sticky reconcile

Coalesced debounce calls must **OR** the reconcile intent — a later `debounced(false)` can't downgrade a pending `true`:

```js
var pendingReconcile = false, dTimer = null;
function debounced(reconcile) {
  if (reconcile) pendingReconcile = true;
  clearTimeout(dTimer);
  dTimer = setTimeout(function () {
    var r = pendingReconcile; pendingReconcile = false; refresh(r);
  }, 200);
}
```

### 2. Always-remove unqualified gifts (add only on reconcile)

Removing a gift the cart no longer qualifies for is **always safe** and must not depend on the reconcile flag. Only *adding* a gift is gated (so you don't re-add one a shopper manually deleted):

```js
var acts = [];
if (q < TIER1 && line1) acts.push(removeLine(line1.key));   // safety: every refresh
if (q < TIER2 && line2) acts.push(removeLine(line2.key));
if (reconcile) {
  if (q >= TIER1 && !line1) acts.push(addGift(GIFT1));
  if (q >= TIER2 && !line2) acts.push(addGift(GIFT2));
}
```

Remove by the **line-item `key`** (`{ id: line.key, quantity: 0 }`), not the variant id — robust when the gift line carries properties like `{_gift: 'true'}`.

### 3. Checkout guard (belt-and-suspenders)

Even with 1 & 2, guard against races and JS failures: disable checkout whenever the cart is **gifts-only** (qualifying subtotal ≤ 0 but items exist). Disable both the standard button and the express/accelerated buttons:

```js
function setCheckoutBlocked(blocked) {
  document.querySelectorAll('[name="checkout"], a[href^="/checkout"], .cart-drawer__checkout')
    .forEach(function (b) {
      if (blocked) { b.setAttribute('disabled','disabled'); b.style.pointerEvents='none'; b.style.opacity='.45'; b.setAttribute('data-blocked','1'); }
      else if (b.getAttribute('data-blocked')) { b.removeAttribute('disabled'); b.style.pointerEvents=''; b.style.opacity=''; b.removeAttribute('data-blocked'); }
    });
  document.querySelectorAll('shopify-accelerated-checkout, .additional-checkout-buttons, [class*="dynamic-checkout"]')
    .forEach(function (c) {
      if (blocked) { c.style.display='none'; c.setAttribute('data-blocked','1'); }
      else if (c.getAttribute('data-blocked')) { c.style.display=''; c.removeAttribute('data-blocked'); }
    });
}
```
Show a message on the bar too (e.g. "Add a product to your cart to check out") when gifts-only.

## Detecting cart changes theme-independently

```js
function isCartUrl(u){ return /\/cart\/(add|change|update|clear)/.test(u); }

var of = window.fetch;                                  // fetch
window.fetch = function () {
  var u = arguments[0] ? String(arguments[0]) : '', p = of.apply(this, arguments);
  if (isCartUrl(u)) p.then(function () { debounced(true); });
  return p;
};

var oOpen = XMLHttpRequest.prototype.open;              // XHR (some controls bypass fetch)
XMLHttpRequest.prototype.open = function (m, url) { this.__cart = isCartUrl(String(url||'')); return oOpen.apply(this, arguments); };
var oSend = XMLHttpRequest.prototype.send;
XMLHttpRequest.prototype.send = function () { if (this.__cart) this.addEventListener('loadend', function () { debounced(true); }); return oSend.apply(this, arguments); };
```

Plus listen for the theme's cart events (`cart:update`, `cart:refresh`, …) with `debounced(false)`, and a `MutationObserver` on the cart drawer with `debounced(false)` — the always-remove logic makes even these paint-only refreshes safe.

## Other guardrails

- Guard the whole IIFE with `if (window.__flag) return;` so it can't double-bind if loaded twice.
- After removing/adding, if on the `/cart` page reload; in the drawer, dispatch the theme's cart-refresh events so the drawer's own DOM reflects the change.
- Compute qualifying subtotal from `final_line_price` of non-gift lines (respects line discounts).
- **The truly bulletproof version** validates server-side (a Shopify Function on the cart/checkout) so it can't be bypassed with JS off. The JS pattern above is the pragmatic client-side floor; add a Function if the stakes justify it.
