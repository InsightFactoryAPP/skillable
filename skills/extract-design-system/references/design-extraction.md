# Design extraction: where to look

Read the files that exist; skip the rest. Prefer the source of truth (config, tokens) over compiled output.

## Tokens and themes

| Stack | Look in |
| --- | --- |
| Tailwind v4 | `@theme { … }` blocks in the global CSS (`app/globals.css`, `src/index.css`, `src/styles/*.css`), `@custom-variant dark` |
| Tailwind v3 | `tailwind.config.{js,ts,cjs,mjs}` → `theme.extend`, `darkMode` (`"class"`, `"media"`, `["class", "[data-theme=dark]"]`) |
| shadcn/ui | `components.json` (base color, CSS variables on/off), `:root` and `.dark` variable blocks in the global CSS |
| CSS variables | `:root`, `.dark`, `[data-theme="…"]`, `@media (prefers-color-scheme: dark)` in global stylesheets |
| Sass / Less | `_variables.scss`, `_tokens.scss`, `theme.scss`, `variables.less` |
| Design tokens | `tokens.json`, `style-dictionary` config, `*.tokens.json`, Figma token exports |
| MUI | `createTheme(...)` calls, `ThemeProvider`, `palette.mode` |
| Chakra UI | `extendTheme(...)` / `createSystem(...)`, `semanticTokens`, `ColorModeScript` |
| Mantine | `createTheme(...)`, `MantineProvider`, `colorScheme` |
| styled-components / Emotion | `theme.{ts,js}`, `ThemeProvider` usage, `lightTheme` / `darkTheme` objects |
| Vue / Nuxt | `nuxt.config` (`colorMode`), Vuetify `createVuetify({ theme })`, `assets/css` |
| Angular | `styles.scss`, Angular Material `mat.define-theme`, `theme.scss` |
| Flutter | `ThemeData`, `ColorScheme`, `darkTheme:` in `MaterialApp` |
| SwiftUI / iOS | `Assets.xcassets` color sets (Any / Dark appearances), `Color+Theme.swift` |
| Android | `res/values/colors.xml`, `res/values-night/`, `themes.xml`, Compose `MaterialTheme` |

How the theme is switched matters as much as the values: find the toggle component or provider (`next-themes`, `useColorMode`, `ThemeProvider`) and mirror its mechanism in the wireframe.

## Typography

- `next/font` imports in the root layout, `@font-face` rules, `<link>` tags to Google Fonts / Adobe Fonts in `index.html` or `_document`.
- `fontFamily` in the Tailwind config or theme object; heading components (`<H1>`, `Typography` variants).

## Layout shell

- Root layout and app frame: `app/layout.tsx`, `app/(app)/layout.tsx`, `src/layouts/*`, `App.vue`, `app.component.html`.
- Navigation: `Sidebar`, `AppNav`, `Header`, `Topbar`, `MobileNav`, route config files (`routes.ts`, `router/index.ts`) for real nav labels and order.
- Content container: max-width wrappers, page padding, page header components.

## Breakpoints

- Tailwind `screens` (defaults: sm 640, md 768, lg 1024, xl 1280, 2xl 1536) or `--breakpoint-*` in `@theme`.
- MUI/Chakra/Mantine `breakpoints` in the theme; Sass `$breakpoints` maps; `@media` queries in layout CSS.
- Note what each breakpoint changes in the shell (sidebar → drawer, header actions → overflow menu).

## Components and icons

- `components/ui/`, `src/components/`, design-system packages in the monorepo (`packages/ui`).
- Icon library from `package.json` (`lucide-react`, `@heroicons/react`, `@tabler/icons`, `react-icons`, Material Symbols). Use the same set from a CDN when possible, or inline the few SVGs needed.
- Empty, loading, and error idioms: search for `EmptyState`, `Skeleton`, `Spinner`, `ErrorBoundary`, `toast`.

## Domain data

- Types and schemas (`types/`, `*.d.ts`, Prisma / Drizzle schema, OpenAPI specs, GraphQL schema) for entity names and fields.
- Fixtures, seeds, storybook stories, and test factories for realistic sample values.

## Quick commands

```bash
# Theme and token sources
rg -l --hidden -g '!node_modules' -e '@theme' -e 'darkMode' -e 'createTheme' -e 'extendTheme' -e '\.dark\s*\{' -e 'data-theme' .

# Layout shell and navigation
rg -l -g '!node_modules' -i -e 'sidebar' -e 'navbar' -e 'topbar' -e 'AppShell' --type-add 'ui:*.{tsx,jsx,vue,svelte,html}' -t ui .

# Fonts
rg -n -g '!node_modules' -e 'next/font' -e '@font-face' -e 'fonts.googleapis' .
```
