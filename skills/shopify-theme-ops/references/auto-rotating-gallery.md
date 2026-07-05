# Auto-rotating product-card galleries (progressive, dependency-free)

A recurring "make the cards feel alive" build: each product card cross-fades through **all** of that product's images. Done naively it's a performance disaster (eager-loads every image on every card). This is the correct, light version — vanilla JS, no libraries, images loaded on demand.

## Where the images come from

Shopify exposes every product's images at the AJAX endpoint **`/products/{handle}.js`** → JSON with an `images` array (protocol-relative CDN urls). No Admin API, no theme data — the storefront can fetch its own product images. Get the handle from the card's `a[href*="/products/"]`.

## The three things that keep it fast & correct

1. **Lazy build via IntersectionObserver.** Only fetch `/products/{handle}.js` and inject layers for cards **near** the viewport (`rootMargin: '250px'`). Cards never scrolled to never cost anything.
2. **Progressive src loading.** Inject the extra `<img class="kx-layer">` with the url in **`data-src`, not `src`** — so nothing downloads yet. Set `.src = dataset.src` only for the layer about to show (and preload the next). Warm just the *first* alt image per built card for a smooth first fade. Result: a 20-card grid downloads a handful of images, not 80.
3. **Pause when hidden.** Gate the rotation ticker on `!document.hidden` (also skip on `prefers-reduced-motion`). Note this makes it look static in an automation tab — it rotates for real focused visitors (see `live-verification-and-gotchas.md`).

## Structure

- Inject layers into the card's media wrapper (`.product-media`; set it `position:relative; overflow:hidden`). The base `img.product-media__image` stays as layer 0; `.kx-layer`s stack `position:absolute; inset:0; object-fit:cover; opacity:0; transition:opacity .85s`. Active layer gets `opacity:1`.
- Cap extras (`images.slice(1, 4)` → 3 total) so heavy PDPs don't rotate forever.
- One shared `setInterval` (~220 ms tick) advances each visible card on a per-card phase offset (`i * 500 ms`) so they don't all flip in unison; cadence ~3 s.
- A `MutationObserver` re-scans for cards Horizon hydrates/re-renders (quick-add, infinite scroll).

## Sketch

```js
function initProductRotator() {
  if (matchMedia('(prefers-reduced-motion: reduce)').matches) return;
  const cache = {}, rotators = [], INTERVAL = 3000;
  const sized = (u,w) => (u.startsWith('//')?'https:'+u:u) + (u.includes('?')?'&':'?') + 'width='+w;
  const load  = l => { if (l && l.dataset.src) { l.src = l.dataset.src; delete l.dataset.src; } };

  function build(card) {
    if (card.__kx) return; card.__kx = true;
    const media = card.querySelector('.product-media');
    const a = card.querySelector('a[href*="/products/"]');
    const h = a && a.getAttribute('href').match(/\/products\/([^/?#]+)/)?.[1];
    if (!media || !h) return;
    (cache[h] ??= fetch(`/products/${h}.js`).then(r=>r.ok?r.json():null).then(p=>p?.images||[]).catch(()=>[]))
      .then(imgs => {
        if (imgs.length < 2) return;
        const layers = imgs.slice(1,4).map(src => {
          const im = document.createElement('img');
          im.className='kx-layer'; im.loading='lazy'; im.decoding='async';
          im.setAttribute('fetchpriority','low'); im.dataset.src = sized(src,600);
          media.appendChild(im); return im;
        });
        const total = layers.length + 1; let idx = 0;
        card.__next = () => {
          idx = (idx + 1) % total;
          if (idx>0) load(layers[idx-1]); const n=(idx+1)%total; if (n>0) load(layers[n-1]);
          layers.forEach((l,i)=>l.classList.toggle('is-active', i+1===idx));
        };
        load(layers[0]); rotators.push(card);
      });
  }
  const vis = new IntersectionObserver(es=>es.forEach(e=>e.target.__vis=e.isIntersecting));
  const setup = new IntersectionObserver((es,o)=>es.forEach(e=>{ if(e.isIntersecting){build(e.target);o.unobserve(e.target);} }), {rootMargin:'250px'});
  const scan = () => document.querySelectorAll('product-card,.product-card').forEach(c=>{ if(c.__seen)return; c.__seen=true; vis.observe(c); setup.observe(c); });
  setInterval(() => { if(document.hidden) return; const t=Date.now();
    rotators.forEach((c,i)=>{ if(!c.__vis||!c.__next)return; c.__t ??= t + i*500; if(t>=c.__t){ c.__next(); c.__t=t+INTERVAL; } }); }, 220);
  scan(); new MutationObserver(scan).observe(document.body,{childList:true,subtree:true});
}
```

Ship it as a footer/`theme.liquid` asset (`references/horizon-architecture.md` → loading global JS). Optional: tiny `.kx-dots` image-count indicator that shows on card hover.
