# Images and fonts

> **In one line:** Serve the right image size in a modern format (AVIF/WebP), lazy-load what is below the fold but load the hero image early, and load fonts with `font-display: swap` and a preload for the one or two fonts the first screen needs.

## Key points
- **Responsive images:** `srcset` lists several widths and `sizes` tells the browser how wide the image will be shown, so a phone downloads a 400 px file instead of a 2000 px one. See [MDN: Responsive images](https://developer.mozilla.org/en-US/docs/Web/HTML/Responsive_images).
- **Modern formats:** AVIF and WebP are much smaller than JPEG/PNG at similar quality. Use `<picture>` with fallbacks. Use SVG for icons and logos.
- **Lazy loading:** `loading="lazy"` on images below the fold. Never on the LCP (hero) image; give that one `fetchpriority="high"` instead.
- **Prevent layout shift:** always set `width` and `height` (or CSS `aspect-ratio`) so space is reserved (better CLS).
- **Fonts:** `font-display: swap` shows fallback text right away and swaps in the web font when ready. Preload only the critical font files, self-host and subset them, and prefer WOFF2.

## Example
```html
<!-- Hero (LCP) image: modern formats, right size, high priority, NOT lazy -->
<picture>
  <source type="image/avif" srcset="/hero-480.avif 480w, /hero-960.avif 960w, /hero-1600.avif 1600w" sizes="100vw" />
  <source type="image/webp" srcset="/hero-480.webp 480w, /hero-960.webp 960w, /hero-1600.webp 1600w" sizes="100vw" />
  <img src="/hero-960.jpg" width="1600" height="800" alt="Market overview" fetchpriority="high" />
</picture>

<!-- Below the fold: lazy, with fixed dimensions to avoid layout shift -->
<img src="/broker-badge.webp" width="160" height="48" alt="SEBI registered" loading="lazy" decoding="async" />

<!-- Preload the one font used above the fold; crossorigin is required for fonts -->
<link rel="preload" href="/fonts/inter-latin-600.woff2" as="font" type="font/woff2" crossorigin />
```

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-latin-600.woff2') format('woff2');
  font-weight: 600;
  font-display: swap;            /* show fallback text immediately */
  unicode-range: U+0000-00FF;    /* latin subset only */
}

/* Numbers in price tables should not change width as they update */
.price { font-variant-numeric: tabular-nums; }
```

## When to use it
Marketing and stock detail pages with banners and logos, user avatars, and the app's brand font. In SvelteKit, the `@sveltejs/enhanced-img` plugin can generate AVIF/WebP and `srcset` at build time; see [SvelteKit: Images](https://svelte.dev/docs/kit/images). For a trading dashboard, fonts matter more than images: a slow font makes prices invisible or makes the layout jump.

## Likely questions
### How do `srcset` and `sizes` work?
`srcset` gives the browser a list of files with their real widths, like `hero-480.avif 480w`. `sizes` says how wide the image will display, like `(max-width: 600px) 100vw, 50vw`. The browser combines that with screen width and pixel density to pick the smallest file that still looks sharp. Use `<picture>` when you need different formats or different crops (art direction).

### WebP vs AVIF?
Both are modern formats with lossy and lossless modes and transparency. AVIF usually compresses better than WebP, but encodes slower. Browser support for both is now broad. I serve AVIF first and WebP or JPEG as fallback inside `<picture>`. See [web.dev: choose the right image format](https://web.dev/articles/choose-the-right-image-format).

### Should you lazy-load every image?
No. Lazy-load images below the fold. Lazy-loading the hero image delays LCP because the browser waits until layout to know it is in view. The LCP image should be in the HTML, eager, and ideally `fetchpriority="high"`. See [web.dev: lazy loading](https://web.dev/articles/browser-level-image-lazy-loading).

### What does `font-display: swap` do, and what is FOIT/FOUT?
FOIT is "flash of invisible text": the browser hides text while the font loads. FOUT is "flash of unstyled text": fallback font first, then a swap. `swap` chooses FOUT, so content is readable immediately, which is better for LCP. The swap can cause a small layout shift, which you reduce by choosing a similar fallback or using `size-adjust` on the fallback `@font-face`. `optional` is another choice: use the web font only if it arrives very quickly. See [MDN: font-display](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display).

### When should you preload a font?
Only for fonts used above the fold, because the browser otherwise discovers fonts late (after CSS is parsed and text is found). Preload one or two files at most; preloading too much competes with more important resources. Always add `crossorigin`, even for same-origin fonts, or the browser fetches it twice. See [web.dev: font best practices](https://web.dev/articles/font-best-practices).

## Common mistakes
- Lazy-loading the LCP image.
- Missing `width`/`height`, causing CLS.
- Preloading a font without `crossorigin`.
- Loading five weights of a Google Font when two are used.
- Shipping a 3000 px image and scaling it down in CSS.

## Resources
- [MDN: Responsive images](https://developer.mozilla.org/en-US/docs/Web/HTML/Responsive_images) - srcset, sizes, picture
- [web.dev: Choose the right image format](https://web.dev/articles/choose-the-right-image-format) - format trade-offs
- [web.dev: Browser-level lazy loading](https://web.dev/articles/browser-level-image-lazy-loading) - when to use loading="lazy"
- [web.dev: Best practices for fonts](https://web.dev/articles/font-best-practices) - preload, font-display, subsetting
- [web.dev: Fetch Priority](https://web.dev/articles/fetch-priority) - boosting the LCP image
