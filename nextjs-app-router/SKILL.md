---
name: nextjs-app-router
description: Build or audit a Next.js 15 App Router project — server vs client components, metadata, layout structure, fonts. Use when the user says "set up Next.js", "add a page", "fix a server/client error", or "add metadata".
---

You are building or auditing a Next.js 15 App Router project.

## Server vs client components

**Server components (default)** — run on the server, never sent to the client as JS:
- Can be async — fetch data directly, await promises
- Cannot use: `useState`, `useEffect`, event handlers, browser APIs
- Use for: layouts, pages, data fetching, static content, metadata

**Client components (`'use client'` at top of file)** — hydrated in the browser:
- Can use: `useState`, `useEffect`, event handlers, `window`, `localStorage`
- Cannot be async at the component level
- Use for: interactive UI, forms, animations, theme toggles, anything stateful

```tsx
// Server component — no directive needed
export default async function Page() {
  const data = await fetch('...')   // direct async is fine
  return <div>{data}</div>
}

// Client component
'use client'
import { useState } from 'react'
export default function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

**The boundary rule**: a server component can import a client component, but NOT the reverse. When you get "Event handlers cannot be passed to Client Component props" — the parent needs `'use client'`.

## File structure (App Router)

```
src/app/
  layout.tsx          ← root layout — wraps every page, load fonts here
  page.tsx            ← homepage (/)
  globals.css         ← global styles, CSS vars, keyframes
  about/
    page.tsx          ← /about route
  blog/
    [slug]/
      page.tsx        ← /blog/:slug dynamic route

src/components/
  layout/             ← Navbar, Footer
  sections/           ← page sections (HeroSection, etc.)
  ui/                 ← reusable primitives (Button, Card, etc.)
```

## Root layout

```tsx
// src/app/layout.tsx
import { Lora, DM_Sans } from 'next/font/google'
import './globals.css'

const lora = Lora({ subsets: ['latin'], variable: '--font-lora', display: 'swap' })
const dmSans = DM_Sans({ subsets: ['latin'], variable: '--font-dm-sans', display: 'swap' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${lora.variable} ${dmSans.variable}`} suppressHydrationWarning>
      <body suppressHydrationWarning>
        {children}
      </body>
    </html>
  )
}
```

`suppressHydrationWarning` on `<html>` and `<body>` prevents false hydration errors from browser extensions injecting attributes (Grammarly, etc.).

## Metadata

```tsx
// src/app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'Witly — Social Intelligence, Made Natural',
  description: '...',
  metadataBase: new URL('https://trywitly.com'),
  openGraph: {
    type: 'website',
    url: 'https://trywitly.com',
    title: '...',
    description: '...',
    images: [{ url: '/og-image.png', width: 1200, height: 630 }],
  },
  twitter: {
    card: 'summary_large_image',
    title: '...',
    images: ['/og-image.png'],
  },
}
```

Only export `metadata` from server components. Dynamic metadata uses `generateMetadata()`.

## Fonts

```tsx
// Load via next/font/google — zero layout shift, self-hosted automatically
const lora = Lora({
  subsets: ['latin'],
  variable: '--font-lora',    // exposes as CSS var
  display: 'swap',
  weight: ['400', '500', '600'],
  style: ['normal', 'italic'],
})

// Apply to <html> element, use in CSS:
// font-family: var(--font-lora);
```

## Wrapping client providers

Layout is a server component — can't directly use React context. Wrap with a client component:

```tsx
// src/components/Providers.tsx
'use client'
import { ThemeProvider } from '@/context/ThemeContext'
export default function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider>{children}</ThemeProvider>
}

// src/app/layout.tsx (server)
import Providers from '@/components/Providers'
// ...
<body><Providers>{children}</Providers></body>
```

## Common errors and fixes

| Error | Cause | Fix |
|-------|-------|-----|
| "Event handlers cannot be passed to Client Component props" | Server component has onClick etc. | Add `'use client'` |
| "useState is not a function" | Hook used in server component | Add `'use client'` |
| Hydration mismatch | Server HTML ≠ client render (extensions, theme) | `suppressHydrationWarning` |
| "Cannot find module './224.js'" | Stale `.next` cache | Delete `.next/`, rebuild |
| Fonts not applying | Variable not on `<html>` | Add `className={font.variable}` to `<html>` |

## Common mistakes

- Adding `'use client'` everywhere — defeats the purpose. Default to server, add client only when needed.
- Fetching data in client components when a server component could do it
- Putting metadata exports in client components — metadata only works in server components
- Hardcoding absolute URLs without `metadataBase` — OG images won't resolve correctly
