# CSS Overrides System for AI Projects

Author: Vinayak Hegde  
Date: Feb 18, 2026

## Problem definition

The client needs a single `overrides.css` file to restyle any element in the app for pixel-perfect fidelity. This requires every meaningful DOM element to be uniquely and stably targetable via CSS selectors that won't collide with Tailwind utilities or bleed across components.

## The solution

`data-ai-*` attributes (not CSS classes)

Using data attributes instead of CSS classes because:

- No collision risk with Tailwind utility classes (`tailwind-merge` only merges Tailwind utilities, but data attributes are entirely separate)
- No style bleeding - attribute selectors `[data-ai-*]` are scoped by nature
- Dual purpose - can replace/supplement `data-testid` for E2E tests
- Higher specificity control - `[data-ai-*]` has consistent specificity `(0,1,0)`, easy to override
- Self-documenting - attributes are visible in DevTools, making the client's job easier

## 1. Naming Convention

`data-ai-{layer}-{component}[-{element}]`

### Layers

- `page` - Page root containers: `data-ai-page-contacts`, `data-ai-page-home`
- `layout` - Layout shell: `data-ai-layout-sidebar`, `data-ai-layout-navbar`
- `{domain}` - Domain components: `data-ai-contacts-table`, `data-ai-commerce-overview`
- `ui` - UI primitives: `data-ai-ui-button`, `data-ai-ui-card-header`

### Rules

- kebab-case only
- Max 4 segments (`layer-component-element-modifier`)
- Domain derived from file path (e.g., `components/contacts/` = domain `contacts`)
- Unique across the entire app (enforced by validation)
- Static strings only - `aiAttr()` arguments must be string literals, never variables or expressions (enforced by lint rule + generator)

### Repeated attributes (list like items)

Attributes like `data-ai-table-row` or `data-ai-list-item` will naturally repeat on every row in a table/list. This is by design - they identify the type of element, not the instance. The client targets specific instances using standard CSS pseudo-selectors:

```css
/* Style all rows */
[data-ai-contacts-table-row] {}

/* Style the 3rd row */
[data-ai-table-row]:nth-child(3) {}

/* Style the first row */
[data-ai-table-row]:first-child {}

/* Style rows on hover */
[data-ai-table-row]:hover {}
```

The generated `overrides.css` and the client-facing documentation will include a "Targeting Repeated Elements" section with these patterns.

Examples:

| Component            | NameOfTheComponent | Path                     |
| -------------------- | ------------------ | ------------------------ |
| Attribute            | Element            | Repeats                  |
| `data-ai-layout-sidebar` | DashboardSidebar root | No                    |
| `data-ai-layout-sidebar-nav-item` | SidebarNavItem | Yes             |

## 2. Helper Utility

A thin `aiAttr()` helper to generate data attributes, co-located with existing `cn()` in `lib/utils.ts`:

```ts
// Returns { "data-ai-layout-sidebar": "" } for JSX spread
function aiAttr(id: string): Record<string, string>
```

Usage in components:

```tsx
<aside {...aiAttr("layout-sidebar")} className={cn(...)}>
```

This keeps the convention centralised and greppable. The function validates format at dev time (kebab-case, max segments).

### Placement rule for Radix/library components

`aiAttr()` must be spread on the innermost wrapper that holds semantic meaning, not on an outer shell `<div>`. This ensures the attribute co-locates with library-owned `data-*` attributes (e.g., Radix's `data-state`, `data-side`), enabling the client to combine selectors for state-aware styling:

```css
/* Style the open state of a dialog */
[data-ai-ui-dialog-content][data-state="open"] {}

/* Style a sidebar item when active */
[data-ai-layout-sidebar-nav-item][data-active="true"] {}

/* Style a disabled button */
[data-ai-ui-button]:disabled {}
[data-ai-ui-button][data-disabled] {}
```

The generated CSS will include a "Combining with Library State Attributes" reference section listing common Radix `data-*` attributes and how to compose them.

### Portal-rendered elements

Example component: Dialogs, Popovers, Tooltips, Sheets.

Radix Portals render at the end of `<body>`, outside the component tree. This means nested CSS selectors will not work for portal content:

```css
/* WILL NOT WORK -- Dialog is not a DOM child of the page container */
[data-ai-page-contacts] [data-ai-ui-dialog-content] {}

/* CORRECT -- target the portal element directly */
[data-ai-ui-dialog-content] {}

/* CORRECT -- combine with state, not ancestry */
[data-ai-ui-dialog-content][data-state="open"] {}
```

The `OVERRIDES-GUIDE.md` will include a dedicated "Portal Elements" warning section listing all portal-rendered components (Dialog, Sheet, Popover, Tooltip, DropdownMenu, ContextMenu, AlertDialog, HoverCard, Command/CommandDialog) and explicitly stating they must be targeted as top-level selectors, never nested under a page or layout attribute.

### Static strings only

No dynamic values to `aiAttr()` calls:

```tsx
// ALLOWED -- static string literal
<div {...aiAttr("contacts-table")} />

// FORBIDDEN -- variable or expression
<div {...aiAttr(someVariable)} />

// SHOULD WE ALLOW TERNERIES??- check with ts-morph lib compatibility for this
<div {...(isAdmin ? aiAttr("admin-page") : aiAttr("user-page"))} />
<div {...aiAttr(isAdmin ? "admin-page" : "user-page"))} />
```

Both the generator script and the static validation test will reject any `aiAttr()` call whose argument is not a plain string literal. This constraint ensures the ts-morph parser never misses an attribute and the generated CSS is always complete.

### Escape hatch: Attribute stacking for contextual differentiation

A generic Card component will always carry `data-ai-ui-card`. But when the same Card is used in multiple places, (e.g., "User Profile" and "Billing Summary"), the client needs to tell them apart. Since shadcn components spread `...props`, the parent can stack a second `aiAttr()` on top:

```tsx
// Inside Card (generic, always present)
<div {...aiAttr("ui-card")} className={cn(...)} {...props} />

// Parent adds context (specific, stacks onto the generic)
<Card {...aiAttr("billing-card")} />
<Card {...aiAttr("profile-card")} />
```

DOM output:

```html
<div data-ai-ui-card data-ai-billing-card>...</div>
<div data-ai-ui-card data-ai-profile-card>...</div>
```

The client can now target at any granularity:

```css
[data-ai-ui-card] {}                              /* all cards */
[data-ai-billing-card] {}                         /* just billing */
[data-ai-ui-card][data-ai-billing-card] {}        /* billing card specifically */
```

This works because:

- Both attributes are static string literals at their respective call sites (component + parent)
- The generator picks up both - no dynamic expressions needed
- No new prop API (`aiId`) to maintain - it uses the same `aiAttr()` + spread pattern
- Works with any component that forwards `...props` (all shadcn components do)

## 3. Automated Attribute Injection (Codemod)

Assuming we have a CodeMod, developers should never manually add `aiAttr()` calls. A codemod script at `scripts/ai-inject.ts` handles all injection automatically using ts-morph.

### 3a. Naming Algorithm

The codemod computes the `data-ai-*` name from the file path + component export name.

#### Step 1 - Determine layer from file path

| Path pattern                 | Layer      | Example            |
| --------------------------- | ---------- | ------------------ |
| `components/ui/{comp}/`     | `ui`       | `ui-button`        |
| `components/layout/{comp}/` | `layout`   | `layout-sidebar`   |
| `components/{domain}/{comp}/` | `{domain}` | `contacts-content` |
| `app/**/page.tsx`           | `page`     | `page-contacts`    |

#### Step 2 - Derive element name from export

- Convert PascalCase to kebab-case: `CardHeader` -> `card-header`
- Strip redundant path-component prefix: `DashboardSidebar` in `layout/dashboard-sidebar/` -> `sidebar` (the `dashboard-` prefix is redundant since the folder already scopes it)
- Multi-export files: each export gets its own name (e.g., `Card`, `CardHeader`, `CardTitle` in `card.tsx` -> `ui-card`, `ui-card-header`, `ui-card-title`)

#### Step 3 - Combine: `{layer}-{element-name}`

Examples of auto-generated names:

| File                                     | Export            | Generated ID          |
| ---------------------------------------- | ----------------- | --------------------- |
| `components/ui/button/button.tsx`        | `Button`          | `ui-button`           |
| `components/ui/card/card.tsx`            | `Card`            | `ui-card`             |
| `components/ui/card/card.tsx`            | `CardHeader`      | `ui-card-header`      |
| `components/ui/dialog/dialog.tsx`        | `DialogContent`   | `ui-dialog-content`   |
| `components/layout/dashboard-sidebar/...`| `DashboardSidebar`| `layout-sidebar`      |
| `app/contacts/[id]/objects/.../page.tsx` | `default`         | `page-contacts-list`  |

### 3b. Root Element Detection

For each exported component function, the codemod finds the injection target:

- Locate the return statement (or arrow-function body)
- If the root is a JSX Fragment (`<>...</>`), descend to the first concrete child element
- If the root is a Radix primitive (e.g., `DialogPrimitive.Content`), inject there (co-locates with `data-state`)
- If the root is a Slot or `asChild` pattern, inject `aiAttr()` on the Slot itself - Radix merges those props onto the immediate child, so this is cleaner than trying to find the "real" child element via AST traversal
- For `forwardRef` (in theory for react 19, we should not be using forwardRef) components, find the render callback's return
- If the root return is a `React.cloneElement()` call (no JSX root), inject `aiAttr()` spread into the props argument: `React.cloneElement(child, { ...aiAttr("name"), ...existingProps })`. Note: the current codebase has zero `cloneElement` usage (it uses Radix Slot instead), but this handles future additions defensively

### 3c. Idempotency and Manual Override

- Skip rule: If the root element already has `aiAttr(...)` in its props, the codemod skips it entirely
- This allows developers to manually override the auto-generated name by placing their own `aiAttr()` call before running the codemod
- Re-running the codemod on the same file produces zero changes (clean git diff)

### 3d. Execution Cadence

- Pre-commit: Runs `ai-inject` on staged (`.tsx/.ts`) files only. Injects missing attributes and re-stages the files.
- CI: Runs `ai-inject --check` (dry-run mode) on all files. If any component is missing an attribute, CI fails(?) with a clear error listing the components
- Manual: `pnpm ai:inject` runs the full scan and injects everywhere. Used after large refactors or pulling new code.

Scripts added to `package.json`:

```json
"ai:inject": "tsx scripts/ai-inject.ts",
"ai:inject:check": "tsx scripts/ai-inject.ts --check"
```

## 4. Generator Script and Two-File Architecture

### Problem

If the generator writes a single file and the client has already added CSS rules inside the `{ }` brackets, re-running the generator would clobber their work.

### Solution

Split into two files:

| File               | Purpose                                                         |
| ------------------ | --------------------------------------------------------------- |
| `ai-targets.css`   | Read-only - regenerated on every run; complete selector list   |
| `ai-overrides.css` | Writable - never touched by generator; client CSS lives here   |

### The generator script

At `scripts/generate-overrides-css.ts`

Uses ts-morph to parse all `.tsx` files

- Finds all `aiAttr("...")` calls via AST (rejects non-string-literal arguments)
- Extracts the string argument (the attribute ID)
- Groups by layer/domain
- Formats output with Prettier (via API call) so it reads like hand-written CSS, not machine-generated output
- Writes `styles/ai-targets.css` (always fully regenerated):

```css
/* ===========================================================
   Available Override Targets
   Generated: 2026-02-20 T 14:30:00Z
   Hash: a1b2c3d4

   READ-ONLY: This file is regenerated by `pnpm generate:overrides`.
   Write your custom styles in ai-overrides.css, not here.

   All selectors have specificity (0,1,0).
   See OVERRIDES-GUIDE.md for usage patterns.
   ===========================================================*/

/* --- Layout --- */
[data-ai-layout-sidebar] {}
[data-ai-layout-sidebar-nav-item] {}
[data-ai-layout-navbar] {}

/* --- Page: Contacts --- */
[data-ai-page-contacts] {}

/* --- Domain: Contacts --- */
[data-ai-contacts-content] {}
[data-ai-contacts-table] {}
[data-ai-contacts-table-header] {}

/* --- UI Primitives --- */
[data-ai-ui-button] {}
[data-ai-ui-card] {}
[data-ai-ui-card-header] {}
```

If `styles/ai-overrides.css` does not exist, creates a starter file:

```css
/* ===========================================================
   Custom Style Overrides

   YOUR FILE: Edit freely. This file is never overwritten by the generator.
   Reference ai-targets.css for all available selectors.
   ===========================================================*/
```

Script added to `package.json`:

```json
"generate:overrides": "tsx scripts/generate-overrides-css.ts"
```

## 5. Validation Tests

### 5a. Static Validation (Vitest/Jest)

- runs in `pnpm test`
- File: `lib/validation/__tests__/ai-attributes.test.ts`

Parses all `.tsx` files with ts-morph and extracts all `aiAttr()` string arguments

- Uniqueness test: No duplicate attribute IDs across the entire codebase (with an explicit allowlist for intentionally repeated attributes like `ui-button`, `ui-card`, list rows, etc.)
- Format test: All IDs match `/^[a-z]+(-[a-z]+){1,3}$/`
- Coverage test: Every component directory in `components/` has at least one `aiAttr()` call
- Static-only test: Every `aiAttr()` call uses a plain string literal argument -- rejects variables, template literals, ternaries, or any expression. This is the guardrail that keeps the ts-morph generator reliable.

### 5b. Runtime Validation (Playwright)

- runs in `pnpm test:e2e`
- File: `app/e2e/specs/overrides/targetability.spec.ts`

- Navigates to each major page route
- Queries all `[data-ai-*]` elements
- Targetability: Every visible interactive element (buttons, inputs, links, table rows) has a `data-ai-*` attribute
- Repeated-attribute awareness: List/table items (rows, nav items, etc.) are expected to share the same `data-ai-*` value across siblings. The test validates that repeated attributes only occur on sibling elements of the same type (not unrelated elements on the page).
- No orphans: Every `data-ai-*` value found in the DOM exists in the generated `overrides.css`

## 6. Cadence and CI Integration

### Pre-commit (via Husky)

- `ai-inject` on staged (`.tsx/.ts`) files only. Injects missing attributes and re-stages the files.
- Static validation test (uniqueness + format)
- The developer never needs to think about `aiAttr()`. They write a component, commit, and the hook handles injection.

### CI

- `ai:inject:check` - dry-run on all files; fails if any component is missing (catches cases where someone skipped the hook)
- Full static validation suite (including E2E targetability tests)
- `generate:overrides` - regenerates `ai-targets.css`, commits if changed

### Pre-release

`pnpm generate:overrides` regenerates the final `ai-targets.css` deliverable.

## 7. Loading the Override Files

Both files live in `styles/` (project root, alongside `app/`) and are imported in `app/globals.css` as the last two imports, in order:

```css
@import 'tailwindcss';
/* ... existing theme variables ... */

/* Available selectors (read-only, regenerated) */
@import '../styles/ai-targets.css';

/* Client custom styles (never overwritten) */
@import '../styles/ai-overrides.css';
```

Import order matters: `ai-overrides.css` comes last so the client's rules always win when specificity is equal.

Why `styles/` instead of `public/`:

- Both files pass through the PostCSS/Tailwind build pipeline - autoprefixed and minified automatically
- No extra network requests - bundled into the main CSS output
- The build pipeline catches syntax errors before they reach the browser
- The raw (unminified, commented) versions remain in the repo for the client to read

Specificity vs. Tailwind `!important`:

All `[data-ai-*]` selectors have specificity `(0,1,0)`. This beats plain Tailwind utilities `(0,1,0 for classes, but the override file loads later)`. However, Tailwind's `!` modifier (e.g., `!bg-blue-500`) generate `!important` declarations, which cannot be beaten by specificity alone.

If the client encounters a Tailwind `!`-prefixed utility in the source, they must escalate:

```css
/* Normal override -- works against regular Tailwind utilities */
[data-ai-ui-button] {
  background-color: red;
}

/* Escalated override -- required when source uses !bg-blue-500 */
[data-ai-ui-button] {
  background-color: red !important;
}
```

The `OVERRIDES-GUIDE.md` will include a specificity comparison table and explicitly call out this edge case.

## 8. Client Documentation (shipped in styles/)

A `styles/OVERRIDES-GUIDE.md` ships alongside the two CSS files with:

- Quick start - how the two-file system works (`ai-targets.css` = reference, `ai-overrides.css` = your file)
- Selector reference - every available `[data-ai-*]` selector, grouped by domain (auto-generated section, kept in sync with `ai-targets.css`)
- Targeting repeated elements - how to use `:nth-child()`, `:first-child`, `:last-child`, `:hover`, `:focus` with `data-ai-*` selectors
- Attribute stacking - how parent components add contextual attributes (e.g., `data-ai-billing-card`) on top of generic ones (`data-ai-ui-card`)
- Combining with component state - reference table of common Radix `data-*` attributes (`data-state`, `data-side`, `data-orientation`, `data-disabled`) and how to compose them with `data-ai-*`
- Portal elements warning - explicit list of portal-rendered components (Dialog, Sheet, Popover, Tooltip, DropdownMenu, etc.) that must be targeted as top-level selectors, never nested under page/layout attributes
- Specificity guide - comparison table: attribute selectors `(0,1,0)`, combined attributes `(0,2,0)`, Tailwind utilities `(0,1,0)`, Tailwind `!`-prefixed utilities (`!important`), and when the client must use `!important` in their overrides
- Versioning - the targets file includes a generation timestamp and content hash so the client knows when it was last updated
- DevTools tip - "In Chrome/Edge/Safari, press Ctrl+F (Cmd+F on Mac) in the Elements panel and type `[data-ai-` to instantly highlight every targetable element on the current page."

## Files to Create/Modify

| File | Action |
| ---- | ------ |
| `lib/utils.ts` | Add `aiAttr()` helper |
| `scripts/ai-inject.ts` | codemod that auto-injects `aiAttr()` into all components |
| `components/**/*.tsx`, `app/**/page.tsx` | Auto-modified by codemod (no manual edits) |
| `scripts/generate-overrides-css.ts` | generator (ts-morph + Prettier formatting) |
| `lib/validation/__tests__/ai-attributes.test.ts` | static validation |
| `app/e2e/specs/overrides/targetability.spec.ts` | E2E validation |
| `styles/ai-targets.css` | generated read-only selector reference |
| `styles/ai-overrides.css` | client-editable overrides (never overwritten) |
| `app/globals.css` | Import both `ai-targets.css` and `ai-overrides.css` last |
| `package.json` | Add `ai:inject`, `ai:inject:check`, `generate:overrides` scripts, add husky |
| `.husky/pre-commit` | runs `ai-inject` + lint-staged + attribute validation |
| `docs/decisions/ai-override-attributes.md` | ADR documenting the convention |
| `styles/OVERRIDES-GUIDE.md` | client-facing guide (pseudo-selectors, state attrs) |

