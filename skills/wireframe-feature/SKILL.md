---
name: wireframe-feature
description: >-
  Turn a complex feature idea into one or more interactive, responsive wireframes
  published as artifacts (self-contained HTML pages) that reuse the current
  project's layout shell, design tokens, typography, and light/dark themes, and
  behave like the real app — working navigation, forms, modals, and loading,
  empty, and error states. Generates several distinct variants on the first pass
  so ideas can be compared, then iterates on the chosen one. Use whenever the
  user wants to wireframe, mock up, prototype, or visualize a feature before
  building it, explore several UX directions for an idea, or see how a feature
  would look and feel inside their app. Also triggers on /wireframe-feature with
  the feature idea and an optional variant count.
argument-hint: "<feature-idea> [variants=3]"
---

# Wireframe Feature

Goal: take a feature idea that is too complex to judge from a description and make it clickable — inside a faithful replica of the project's own app shell, in its own visual language, in both themes, at every breakpoint. Wireframes are throwaway exploration: they live as artifacts, never as commits in the project.

## When to use

- "Wireframe / mock up / prototype this feature before we build it"
- "Show me 3 different ways the onboarding / billing / bulk-edit flow could work"
- "What would X look like in our app?" for a multi-screen or stateful feature
- `/wireframe-feature <idea> [variants=N]`
- Not for shipping production UI (implement the feature instead) or for a standalone marketing page (use build-landing-page).

## Inputs

| Input | Default | Notes |
| --- | --- | --- |
| Feature idea | required | From the arguments or the conversation. If absent, ask one short question. |
| Variants | 3 | First pass only. Clamp 1–5. "one", "a few", "5 ideas" all parse. |
| Target screen | inferred | Where the feature lives (new route, panel in an existing page, modal…). |

Do not run a questionnaire. Fill gaps with sensible assumptions and list them in the report so the user can correct them on the next iteration.

## 1. Read the project's design language

Before drawing anything, extract what makes the app look and feel like itself. See [references/design-extraction.md](references/design-extraction.md) for where to look per stack. Capture, into a short working note:

1. **Tokens** — colors (as the project names them: `--background`, `primary-500`, `$brand`…), radii, spacing scale, shadows, borders, z-layers.
2. **Themes** — whether light and dark both exist, and the switching mechanism (`class="dark"`, `[data-theme]`, `prefers-color-scheme`, a theme provider). Record the token values for *each* theme. If the app has only one theme, ship only that one.
3. **Typography** — font families (and where they load from), sizes, weights, line heights, heading styles.
4. **Layout shell** — the chrome every page sits in: sidebar vs. top nav, header height, content max-width, page padding, breadcrumbs, the real nav labels and icons.
5. **Breakpoints and responsive behavior** — the project's breakpoint values and what changes at each (sidebar collapses to a drawer, tables become cards, bottom tab bar on mobile…).
6. **Component idioms** — how buttons, inputs, cards, tables, tabs, dialogs, toasts, empty states, and skeletons look; the icon set; motion conventions.
7. **Domain vocabulary** — entity names, sample data shapes, and existing copy tone, so placeholder content reads like the real product.

Read real source files (theme config, global CSS, layout components, a couple of representative pages) rather than guessing. If the project has no UI yet, say so and derive a neutral system from its framework defaults or brand assets, and note it in the report.

## 2. Model the feature

Before designing variants, write down (briefly, for yourself):

- The **jobs** the feature serves and the **primary flow** step by step.
- Every **screen and state**: default, empty, loading, error, success, permission-denied, long-content, many-items — whichever the feature actually has.
- **Entry points** from the existing app (nav item, button on an existing page, deep link) and exit points.
- **Edge cases** that would change the design (bulk actions, partial failure, undo, concurrency).

## 3. Diverge: design distinct variants

On the first pass, produce N variants that differ in **concept**, not in color or spacing. Vary the axes that matter for this feature, for example:

- Information architecture: dedicated page vs. side panel vs. modal wizard vs. inline editing.
- Interaction model: step-by-step wizard vs. single form vs. direct manipulation vs. command palette.
- Density and disclosure: everything visible vs. progressive disclosure vs. power-user shortcuts.

Give each variant a short name ("A · Inline drawer", "B · Guided wizard", "C · Command-first") and a one-sentence thesis. Every variant still uses the same app shell and tokens — the project's look is the constant; the idea is the variable.

## 4. Build each wireframe

One self-contained HTML page per variant. Requirements:

**Fidelity to the project**
- Recreate the app shell (nav, header, real nav labels) around the feature so it reads as "this app, plus the feature". Show neighboring nav items as inert but present.
- Define the extracted tokens as CSS custom properties on `:root` using the project's own token names, with the second theme's values under the same mechanism the project uses (e.g. `.dark { … }` or `[data-theme="dark"] { … }`), and also follow `prefers-color-scheme` by default.
- Add a small floating **prototype toolbar** (clearly outside the app chrome) with a theme toggle and, when useful, a state switcher (empty / loading / error / populated). Keep it collapsible so it does not hide the design.
- Use the project's fonts when they are loadable from a public CDN or Google Fonts; otherwise pick the closest available match and say so.

**Behaves like the app**
- Navigation between the feature's screens works (hash routing or in-page state); back/cancel return where the real app would.
- Forms validate, disable submit while "saving", show inline errors, and land on a success state.
- Modals, drawers, popovers, tabs, menus, toasts, and keyboard basics (Esc closes, Enter submits, focus moves sensibly) work.
- Simulate latency for async actions (300–1200 ms) with the project's skeleton/spinner idiom, and give one path that fails so the error state is reachable.
- Realistic, domain-specific sample data in plain JS objects — no lorem ipsum, no network calls, no real customer data.

**Responsive**
- Use the project's breakpoints and reproduce its responsive behavior at each: test mentally at ~375 px, ~768 px, and ≥1280 px. No horizontal page scroll on mobile; touch targets ≥ 40 px.

**Wireframe, not production**
- Plain HTML/CSS/vanilla JS (or a pinned CDN library the project already uses, e.g. React + Tailwind via CDN) in one file. No build step.
- Keep it honest: mark anything invented (an endpoint, a permission, a metric) with a subtle dotted underline or a `?` tooltip so it is not mistaken for an existing capability.

## 5. Publish

- **Artifact-capable agent** (e.g. Claude Code with an artifact tool): publish each variant as its own artifact so each has a URL to compare, share, and comment on. Title them `<Feature> <letter>` (e.g. "Bulk Edit A"), with the thesis as the description.
- **Otherwise:** write the files outside the project's tracked tree (a scratch/temp directory) as `wireframe-<feature>-<letter>.html` and open them locally (`open` / `xdg-open`).
- Never commit wireframes to the project, create branches, or open PRs for them unless the user explicitly asks.

## 6. Report the first pass

```markdown
**<Feature>** · <N> variants · themes: light + dark · breakpoints: <list>

| Variant | Thesis | Best for | Trade-off | Link |
| --- | --- | --- | --- | --- |
| A · Inline drawer | … | … | … | <url or path> |

**Try:** <the 2–3 interactions that best show the difference between variants>
**Assumptions:** <bullets — things guessed about the feature or the design system>
**Pick one, mix (e.g. "A's layout with C's shortcuts"), or ask for another round.**
```

## 7. Iterate

- **"Go with B" / "refine B"** → keep improving variant B in place (republish to the same artifact URL or overwrite the same file) instead of creating a new one, so links stay stable.
- **"Mix A and C"** → a new variant D that names what it took from each.
- **"More ideas" / "another round"** → N fresh variants that explicitly avoid concepts already shown.
- **Feedback on details** (copy, a state, a breakpoint) → apply it to the chosen variant and list what changed in one or two lines.
- When the user is happy, offer a short **handoff note**: screens and states, which existing components each part maps to, new components needed, and open questions — so the implementation can start from it.

## Quality bar before reporting

- [ ] Shell, tokens, fonts, and nav are recognisably the project's, in every theme it supports
- [ ] Theme toggle switches every surface (no hard-coded colors left behind)
- [ ] Every state listed in step 2 is reachable by clicking or via the state switcher
- [ ] Works at mobile, tablet, and desktop widths with the project's responsive patterns
- [ ] Variants differ in concept, not just styling
- [ ] No network calls, secrets, or real user data in the pages
