# `src/app/layout.tsx` — Root Application Layout

**Tags:** `#next-js` `#layout` `#seo` `#observability`  
**See also:** [[relic-types]]

---

## Purpose

Next.js App Router root layout. Wraps every page with:

- Google Fonts (Geist Sans + Geist Mono)
- CSS custom property variables for fonts
- Sentry client-side initialization
- SEO metadata

---

## Fonts

```ts
const geistSans = Geist({ variable: "--font-geist-sans", subsets: ["latin"] });
const geistMono = Geist_Mono({ variable: "--font-geist-mono", subsets: ["latin"] });
```

Applied as CSS variables on `<html>`:

```tsx
<html className={`${geistSans.variable} ${geistMono.variable} h-full antialiased`}>
```

---

## SEO Metadata

```ts
export const metadata: Metadata = {
  title: {
    default: "Relic Ring Protocol — Stack Kings",
    template: "%s | Relic Ring Protocol",
  },
  description: "Relic Ring Protocol — low-latency routing simulation...",
  metadataBase: new URL(process.env.NEXT_PUBLIC_SITE_URL ?? "https://relic.inusha.me"),
  openGraph: { ... },
  twitter: { card: "summary_large_image", ... },
  icons: { icon: "/icon" },
}
```

Child pages override the title using the `%s | Relic Ring Protocol` template.

---

## Body

```tsx
<body className="min-h-full flex flex-col">
  <SentryClientInit /> ← triggers Sentry client SDK initialization
  {children}
</body>
```

`SentryClientInit` is an empty client component that exists purely to run the Sentry client-side setup as a React side effect.

---

## Notes

- `NEXT_PUBLIC_SITE_URL` env var should be set in deployment (Vercel) for correct OG image URLs.
- `h-full` + `min-h-full flex flex-col` on `html`/`body` allows full-viewport-height layouts without overflow hacks.

---

Back to [[00 - Index]]
