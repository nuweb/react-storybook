# Schema-Driven UI Framework -- Implementation Plan

A production-ready plan for a **form-library-agnostic, schema-driven UI framework** that renders dynamic forms from Zod schemas with pluggable form engines (TanStack Form primary, React Hook Form secondary), built on React 18 + MUI v6 and aligned with the enterprise frontend architecture defined in `.cursor/skills/frontend-architecture/SKILL.md`.

This document is the blueprint. It is intentionally detailed enough that an engineer or cloud agent can execute it without re-deriving architecture decisions.

---

## 1. Goals and Non-Goals

### Goals

- **Single source of truth.** A Zod schema defines types, validation, *and* UI metadata. Derived form state, API validation, and renderers all read from the same schema.
- **Engine-agnostic rendering.** A stable `FormEngine` interface decouples field components from the underlying form library. Switching the engine must not require changes in field, layout, or page components.
- **Extensible field registry.** Field types map to React components via a registry. Consumers can override any field (e.g. replace the default `text` with a branded one) or register new ones.
- **Responsive, token-driven layout.** Layout metadata converts to responsive MUI Grid layouts consuming design tokens -- no inline styles.
- **Graceful failure.** Unknown field types render a visible fallback with a developer-facing warning. Missing metadata uses sensible defaults. Engine mid-edit switch preserves values.
- **Demo-ready.** A live dashboard showcases multiple schemas, both engines side-by-side, inline validation, async validation, and a JSON preview of current form state.

### Non-Goals (explicitly out of scope for v1)

- Drag-and-drop schema builder UI.
- Visual form editor.
- Server-side rendering of forms.
- Multi-step wizard orchestration (leave hooks for it; do not build it).
- Non-Zod schema adapters (e.g. Yup, Valibot). Design keeps the door open but v1 ships Zod only.

---

## 2. Conceptual Architecture

```
                ┌────────────────────────────────────┐
                │          Zod Schema + Meta         │   single source of truth
                └───────────────┬────────────────────┘
                                │  compile()
                                ▼
                ┌────────────────────────────────────┐
                │          Compiled FormSpec         │   normalized, engine-agnostic
                │ { fields[], layout, validators }   │
                └───────────────┬────────────────────┘
                                │
          ┌─────────────────────┼──────────────────────┐
          ▼                     ▼                      ▼
   ┌────────────┐        ┌────────────┐        ┌────────────┐
   │ FormEngine │        │  Layout    │        │   Field    │
   │  Adapter   │◀──────▶│  Renderer  │◀──────▶│  Registry  │
   │(TanStack / │        │ (Grid from │        │  (MUI      │
   │  RHF)      │        │   spec)    │        │  fields)   │
   └────────────┘        └────────────┘        └────────────┘
          │                     │                      │
          └─────────────────────┼──────────────────────┘
                                ▼
                  ┌────────────────────────────┐
                  │  <SchemaForm schema=… />   │
                  └────────────────────────────┘
```

Four decoupled concerns:

1. **Schema** -- Zod + UI metadata, author-facing.
2. **Spec** -- compiled, normalized intermediate representation (IR).
3. **Engine** -- a hook + bindings contract; one implementation per form library.
4. **Renderer** -- layout + field registry; consumes IR and engine bindings.

The IR is the stable contract. Engines and renderers do not talk to each other directly; they both talk to the IR.

---

## 3. Package / Folder Structure

Following the atomic-design + scope-first conventions in the skill, even though this repo is currently single-package. Structure the framework as if it were an Nx library so the promotion path is trivial.

```
src/
  framework/                              # the schema-driven UI framework
    contracts/                            # Zod schemas authored by consumers
      user.schema.ts
      signup.schema.ts
      survey.schema.ts

    core/
      types.ts                            # FieldSpec, FormSpec, LayoutNode, FieldMeta
      meta.ts                             # ui() helper; attaches metadata to Zod nodes
      compile.ts                          # zodSchema → FormSpec
      registry.ts                         # FieldRegistry + createRegistry()
      errors.ts                           # UnknownFieldTypeError, CompileError

    engines/
      types.ts                            # FormEngine interface
      tanstack/
        TanStackEngine.tsx                # default engine
        useTanStackForm.ts
      rhf/
        RHFEngine.tsx                     # secondary engine
        useRHFForm.ts
      switcher.ts                         # engine state preservation helper

    renderer/
      SchemaForm.tsx                      # top-level orchestrator
      LayoutRenderer.tsx                  # grid / sections / dividers
      FieldRenderer.tsx                   # registry lookup, error surfacing
      FallbackField.tsx                   # unknown field type UI

    fields/                               # default MUI-backed field components
      TextField/TextField.tsx
      NumberField/NumberField.tsx
      SelectField/SelectField.tsx
      CheckboxField/CheckboxField.tsx
      SwitchField/SwitchField.tsx
      RadioGroupField/RadioGroupField.tsx
      DateField/DateField.tsx
      TextareaField/TextareaField.tsx
      ObjectField/ObjectField.tsx         # nested objects
      ArrayField/ArrayField.tsx           # repeatable groups

    hooks/
      useSchemaForm.ts                    # thin facade: engine + compile + state
      useFieldBinding.ts                  # adapter-agnostic field hook

    testing/
      renderWithProviders.tsx
      mockSchemas.ts

    index.ts                              # public surface (no barrels below)

  demo/                                   # live showcase app
    pages/
      Dashboard.tsx                       # schema picker + engine toggle
      Comparison.tsx                      # side-by-side TanStack vs RHF
    components/
      SchemaPicker.tsx
      EngineToggle.tsx
      StatePreview.tsx                    # JSON view of current form state
      ValidationPanel.tsx                 # live error list
    App.tsx

  tokens/design-tokens.css                # existing; framework consumes these
```

All components are co-located with their `*.stories.tsx` file for Storybook coverage. No barrel `index.ts` files inside `framework/` subfolders -- only the top-level `framework/index.ts` re-exports the public API.

---

## 4. Schema Definition System

### 4.1 Authoring model

Zod is the schema language. UI metadata attaches via a small helper that stores metadata in `schema._def.description` using a reserved JSON envelope, so it survives Zod transforms and does not require a custom Zod build.

```ts
import { z } from 'zod'
import { ui } from '@/framework/core/meta'

export const SignupSchema = z.object({
  email: ui(z.string().email(), {
    type: 'text',
    label: 'Email',
    placeholder: 'you@example.com',
    autoComplete: 'email',
    col: { xs: 12, md: 6 },
  }),
  password: ui(z.string().min(8), {
    type: 'password',
    label: 'Password',
    helperText: 'At least 8 characters.',
    col: { xs: 12, md: 6 },
  }),
  role: ui(z.enum(['admin', 'member', 'viewer']), {
    type: 'select',
    label: 'Role',
    options: [
      { value: 'admin', label: 'Admin' },
      { value: 'member', label: 'Member' },
      { value: 'viewer', label: 'Viewer' },
    ],
    col: { xs: 12 },
  }),
  acceptsTerms: ui(z.boolean().refine((v) => v === true, 'You must accept the terms'), {
    type: 'checkbox',
    label: 'I accept the terms of service',
    col: { xs: 12 },
  }),
})

export type Signup = z.infer<typeof SignupSchema>
```

### 4.2 `ui()` helper contract

```ts
type FieldMeta = {
  type: FieldType                  // 'text' | 'number' | 'select' | … | string (registry extensible)
  label?: string
  helperText?: string
  placeholder?: string
  hidden?: boolean
  readOnly?: boolean
  autoComplete?: string
  col?: ResponsiveCol              // { xs?, sm?, md?, lg?, xl? } 1–12
  options?: Array<{ value: string | number; label: string; disabled?: boolean }>
  asyncValidate?: AsyncValidatorRef  // reference to a named async validator
  dependsOn?: string[]             // dot-paths; re-renders when these change
  componentProps?: Record<string, unknown>  // passthrough to custom field components
}

function ui<T extends z.ZodTypeAny>(schema: T, meta: FieldMeta): T
```

`ui()` serializes meta as JSON into `schema.describe(...)` so it round-trips through `z.infer`, `.optional()`, `.nullable()`, and `.default()` without loss. `compile()` reads it back with a deserialize guard that rejects malformed metadata with a precise `CompileError`.

### 4.3 Compilation (`compile.ts`)

`compile(zodSchema) → FormSpec` walks the schema once and produces a normalized IR:

```ts
interface FormSpec {
  fields: Record<string, FieldSpec>      // dot-path → spec
  order: string[]                         // author order, used when no layout given
  layout: LayoutNode                      // tree of sections/rows/fields
  defaults: Record<string, unknown>       // default values from z.default() + type-based fallback
  validators: {
    whole: ZodSchema                      // original schema for submit-time validation
    byField: Record<string, ZodSchema>    // derived per-field schema for live validation
  }
}

interface FieldSpec {
  path: string                            // e.g. 'address.city' or 'hobbies[0].name'
  type: FieldType
  meta: FieldMeta
  zod: ZodSchema                          // the exact sub-schema at that path
  isArray: boolean
  isObject: boolean
  depth: number
}
```

Rules enforced during compile:

- If no `type` in meta, infer from Zod node (`ZodString`→`text`, `ZodNumber`→`number`, `ZodBoolean`→`checkbox`, `ZodEnum`/`ZodUnion` of literals→`select`, `ZodArray`→`array`, `ZodObject`→`object`, `ZodDate`→`date`).
- If no `label`, derive from the last segment of the field path (camel→Title Case).
- If no `col`, default to `{ xs: 12 }` (full width mobile, stacks on all breakpoints until overridden).
- Nested objects produce a sub-tree with `depth + 1`; a maximum depth of `5` is enforced, emitting a `CompileError` beyond that.

Compilation is memoized per schema reference so re-renders don't recompile.

---

## 5. Form Engine Abstraction

### 5.1 Interface

```ts
export interface FormEngine<TValues extends Record<string, unknown> = Record<string, unknown>> {
  name: 'tanstack' | 'rhf'

  // One-time setup: returns a stable handle used by renderers.
  useForm(args: {
    spec: FormSpec
    defaultValues: Partial<TValues>
    onSubmit: (values: TValues) => void | Promise<void>
  }): FormHandle<TValues>
}

export interface FormHandle<TValues> {
  values: TValues                             // current values snapshot (subscribable)
  errors: Record<string, string[]>            // keyed by dot-path
  isSubmitting: boolean
  isDirty: boolean
  isValid: boolean

  submit: () => Promise<void>
  reset: (next?: Partial<TValues>) => void

  // Field-level binding; used by FieldRenderer.
  getFieldProps: (path: string) => FieldBinding
  subscribe: (paths: string[], cb: (snapshot: unknown) => void) => Unsubscribe
}

export interface FieldBinding {
  value: unknown
  onChange: (next: unknown) => void
  onBlur: () => void
  error?: string                              // first error for the field
  name: string                                // dot-path, for `name` attr + testing
  ref?: React.Ref<HTMLElement>
}
```

The interface is intentionally minimal. Everything else (arrays, async validation, cross-field dependencies) is layered on top using `subscribe` plus engine-specific opt-in APIs exposed through a capability probe:

```ts
interface FormHandle<TValues> {
  capabilities: {
    fieldArrays: boolean
    asyncValidation: boolean
  }
  // Optional, guarded by capabilities.
  arrayOps?: {
    push: (path: string, value: unknown) => void
    remove: (path: string, index: number) => void
    move: (path: string, from: number, to: number) => void
  }
}
```

Field components call `arrayOps` only after checking `capabilities.fieldArrays`. If an engine lacks a capability, the renderer falls back to an imperative pattern using `reset` with a re-computed value tree.

### 5.2 TanStack Form adapter (primary)

- Uses `@tanstack/react-form` with `@tanstack/zod-form-adapter`.
- `useForm({ defaultValues, validatorAdapter: zodValidator(), onSubmit })` returns TanStack's API.
- `getFieldProps` wraps `form.Field` by calling the hook-form-free imperative helpers TanStack exposes: `form.getFieldValue(path)`, `form.setFieldValue(path, v)`, `form.getFieldMeta(path)` plus a subscription registered in an effect.
- `byField` validators attach per field via `validators.onChange: spec.validators.byField[path]`.
- `onChangeAsyncDebounceMs` is wired when `meta.asyncValidate` is present.
- Array ops use `form.pushFieldValue`, `form.removeFieldValue`, `form.moveFieldValue` -- capability flag `fieldArrays: true`.

### 5.3 React Hook Form adapter (secondary)

- Uses `react-hook-form` with `@hookform/resolvers/zod`.
- `useForm({ defaultValues, resolver: zodResolver(spec.validators.whole) })` for submit validation; per-field live validation uses `trigger(path)` plus a manual Zod parse of `spec.validators.byField[path]` to report precise errors.
- `getFieldProps` adapts `register(path)` + `getFieldState(path)` into the common `FieldBinding` shape.
- Array ops use `useFieldArray`. Capability flag `fieldArrays: true`.
- Async validation uses `register(path, { validate: async (v) => … })`.

### 5.4 Engine switching with state preservation

`switcher.ts` exports a hook:

```ts
function useEngineSwitcher<T>(initial: EngineName): {
  engine: FormEngine<T>
  setEngine: (name: EngineName) => void
  snapshot: Partial<T>           // last known values, preserved across switches
}
```

Mechanism:

1. The active engine's `FormHandle` emits its `values` into a `useRef` on every change via `subscribe`.
2. When the user toggles, the hook:
   - Calls `handle.submit` in a *dry-run* mode? No -- simpler: snapshots `handle.values`, unmounts the old engine, mounts the new one with `defaultValues: snapshot`.
   - Errors and touched state are intentionally *not* preserved (they're engine-specific); values are.
3. `SchemaForm` keys its internal engine subtree by `engine.name` so React unmounts the old tree and remounts cleanly. A short `<AnimatePresence>` crossfade avoids flicker.

This is why the engine interface exposes `values` but not internal error state: preserving values is portable; preserving errors is not.

---

## 6. Layout Renderer

### 6.1 Layout IR

```ts
type LayoutNode =
  | { kind: 'row'; children: LayoutNode[]; gap?: number }
  | { kind: 'section'; title?: string; description?: string; children: LayoutNode[] }
  | { kind: 'field'; path: string; col: ResponsiveCol }
  | { kind: 'divider' }
```

`compile()` produces a default layout: one section containing one row per field, each field spanning `meta.col`. Authors override by providing an explicit layout in meta:

```ts
ui(SignupSchema, {
  layout: {
    kind: 'section',
    title: 'Account',
    children: [
      { kind: 'row', children: [
        { kind: 'field', path: 'email', col: { xs: 12, md: 6 } },
        { kind: 'field', path: 'password', col: { xs: 12, md: 6 } },
      ]},
      { kind: 'field', path: 'role', col: { xs: 12 } },
      { kind: 'divider' },
      { kind: 'field', path: 'acceptsTerms', col: { xs: 12 } },
    ],
  },
})
```

### 6.2 Rendering

`LayoutRenderer` translates each node to MUI primitives:

- `row` → `<Grid container spacing={node.gap ?? 2}>`
- `field` → `<Grid size={node.col}>` wrapping `<FieldRenderer path={node.path} />`
- `section` → `<Stack gap>` with `Typography` heading + `Divider`
- `divider` → `<Divider />`

Responsive behavior flows from MUI's Grid v2, which accepts `size={{ xs, sm, md, lg, xl }}` matching `ResponsiveCol`. No custom CSS; token-driven spacing inherited from the MUI theme.

### 6.3 Ordering and unused fields

If a custom layout omits fields, a development-only `console.warn` lists them and the renderer appends them in author order at the end of the top-level section, so forms cannot silently drop inputs.

---

## 7. Field Registry and Field Renderer

### 7.1 Registry

```ts
export type FieldComponent = React.ComponentType<{
  spec: FieldSpec
  binding: FieldBinding
  form: FormHandle<unknown>   // for dependent fields / array ops
}>

export interface FieldRegistry {
  get(type: FieldType): FieldComponent | undefined
  register(type: FieldType, component: FieldComponent): void
  extend(partial: Record<FieldType, FieldComponent>): FieldRegistry
}

export function createDefaultRegistry(): FieldRegistry
```

A default registry ships with MUI-backed implementations for `text`, `password`, `email`, `number`, `textarea`, `select`, `checkbox`, `switch`, `radio`, `date`, `object`, `array`. Each is a thin wrapper around the corresponding MUI component, reading from `binding` for value/error/onChange and from `spec.meta` for label/helperText/options.

Consumers override per-project:

```tsx
<SchemaForm
  schema={SignupSchema}
  registry={defaultRegistry.extend({
    'rich-text': MyRichTextField,
    text: MyBrandedTextField,
  })}
/>
```

### 7.2 FieldRenderer

`FieldRenderer` is where most edge-case handling lives:

```tsx
function FieldRenderer({ path }: { path: string }) {
  const spec = useSpec(path)
  const registry = useRegistry()
  const form = useFormHandle()
  const binding = form.getFieldProps(path)

  const Component = registry.get(spec.type)
  if (!Component) {
    return <FallbackField spec={spec} binding={binding} />
  }

  if (spec.meta.hidden) return null

  return (
    <ErrorBoundary fallback={<FieldErrorFallback path={path} />}>
      <Component spec={spec} binding={binding} form={form} />
    </ErrorBoundary>
  )
}
```

### 7.3 Fallback field (unknown types)

`FallbackField` renders an MUI `Alert` at severity `warning` with the field label, the attempted type, and the current value serialized. It still wires `value` / `onChange` to a generic `TextField` so the form is not broken -- users can still type a value. The warning also logs via the project logger with component stack.

### 7.4 Nested rendering (objects and arrays)

- `ObjectField` recursively renders its sub-spec through `LayoutRenderer` (if author provided a layout for the sub-object) or a default row-per-child fallback.
- `ArrayField` renders each element as a group, driven by `form.arrayOps` when available, falling back to `reset`-based mutations otherwise. It enforces `depth ≤ 5` (compiled into the spec) so circular or runaway structures fail loudly.

---

## 8. Edge Case Handling -- Specification

| Edge case | Behavior |
|-----------|----------|
| Unknown field type | `FallbackField` renders `Alert` + generic text input. Warning logged. Form still submits. |
| Missing `type` in meta | `compile()` infers from Zod node. If not inferrable (e.g. `z.any()`), defaults to `text`. |
| Missing `label` | Derived from field path via camel-to-title-case. |
| Missing `col` | Defaults to `{ xs: 12 }`. |
| Validation errors | Rendered inline as MUI `FormHelperText` with `error` state. Live on change after first blur; always on submit. |
| Async validation in flight | Field shows `CircularProgress` adornment; submit button disabled via `isValid` + `isSubmitting`. |
| Deeply nested objects | Enforced max depth 5. Compile fails loudly beyond that. |
| Empty form (no fields) | Renders `Alert` with copy: "No fields defined. Add a Zod schema in `contracts/`." with link to the schema docs. |
| Engine switch mid-edit | `useEngineSwitcher` snapshots `values`, remounts with them as `defaultValues`. Errors reset; values preserved. |
| Server-side submit error | `onSubmit` rejection propagates to `FormHandle.errors[''] ` (form-level) and is rendered as an `Alert` above the form. |
| Dependent field that hides mid-edit | Hidden fields retain their last value unless meta has `clearOnHide: true`. |

---

## 9. Public API (the `<SchemaForm>` facade)

```tsx
<SchemaForm
  schema={SignupSchema}
  engine="tanstack"                       // 'tanstack' | 'rhf'
  registry={customRegistry}               // optional
  defaultValues={{ role: 'member' }}      // optional
  onSubmit={async (values) => await api.signup(values)}
  onChange={(values) => …}                // optional; throttled
  className={…}
/>
```

Under the hood:

```tsx
function SchemaForm(props) {
  const spec = useMemo(() => compile(props.schema), [props.schema])
  const engine = useEngine(props.engine)
  const handle = engine.useForm({ spec, defaultValues: props.defaultValues, onSubmit: props.onSubmit })

  return (
    <FormProvider spec={spec} handle={handle} registry={props.registry ?? defaultRegistry}>
      <form onSubmit={(e) => { e.preventDefault(); handle.submit() }} noValidate>
        <LayoutRenderer node={spec.layout} />
        <FormActions />
      </form>
    </FormProvider>
  )
}
```

`FormActions` is a default submit/reset bar; consumers can pass `actions={…}` to override.

---

## 10. Live Demo Dashboard

### 10.1 Pages

- **Dashboard (`/`)** -- Schema picker dropdown on the left; engine toggle (TanStack / RHF) at the top; rendered form in the middle; two side panels on the right:
  - `StatePreview`: live `JSON.stringify(values, null, 2)` of the form state.
  - `ValidationPanel`: list of current errors with field paths.
- **Comparison (`/compare`)** -- Two `<SchemaForm>` instances side-by-side on the same schema, one per engine. Values sync bidirectionally via a shared controlled `defaultValues` derived from a parent `useState`. Demonstrates that the abstraction holds.

### 10.2 Example schemas to ship

Place each in `src/framework/contracts/`:

1. `signup.schema.ts` -- simple flat schema with string, enum, boolean, async email uniqueness check.
2. `profile.schema.ts` -- nested object (`address: { street, city, zip }`), demonstrates `ObjectField`.
3. `survey.schema.ts` -- dynamic array of question/answer pairs, demonstrates `ArrayField` + dependent fields (question type changes answer field type).
4. `billing.schema.ts` -- discriminated union on `paymentMethod` (card vs invoice); demonstrates `dependsOn` and conditional rendering.

### 10.3 Interactions wired

- Edit any field → live validation and `StatePreview` update.
- Toggle engine → values preserve, errors reset, same JSON in `StatePreview`.
- Swap schema → form remounts with new defaults.
- Click "Submit" → disabled while invalid/submitting; async uniqueness check shows loading adornment; success toast on fulfilled submit.

---

## 11. Dependencies to Add

```
@tanstack/react-form            ^1        # primary engine
@tanstack/zod-form-adapter      ^0.4+     # Zod bridge
react-hook-form                 ^7        # secondary engine
@hookform/resolvers             ^3
zod                             ^3
@mui/material                   ^6
@mui/icons-material             ^6
@emotion/react @emotion/styled  ^11       # peer deps of MUI
motion                          ^11       # crossfade on engine switch
```

Dev-only:

```
@testing-library/react          ^16
@testing-library/user-event     ^14
vitest                          ^2
jsdom                           ^25
msw                             ^2
@storybook/addon-themes         ^8
@storybook/test                 ^8
@axe-core/playwright            ^4        # later, when E2E is added
```

Pin versions when scaffolding; do not hand-write them in JSON.

---

## 12. Testing Strategy

### 12.1 Unit (Vitest + RTL)

- `core/compile.test.ts` -- schema → spec invariants (defaults, inferred types, layout fallback, depth limits).
- `core/meta.test.ts` -- `ui()` round-trips through `.optional()`, `.nullable()`, `.default()`.
- `core/registry.test.ts` -- `get`, `register`, `extend`, unknown-type behavior.
- `engines/tanstack/useTanStackForm.test.ts` -- values, validation, submit, array ops, capabilities.
- `engines/rhf/useRHFForm.test.ts` -- same coverage via the same interface so the test file can share fixtures.
- `engines/switcher.test.ts` -- value preservation across engine switches.
- `renderer/FieldRenderer.test.tsx` -- unknown type fallback, hidden fields, error display.

### 12.2 Integration (Storybook + play functions)

- One story per default field component (typing, validation errors, async validation).
- `SchemaForm` stories per demo schema:
  - `Default`, `WithValidationErrors`, `WithAsyncValidation`, `Submitting`, `SwitchingEngines` (play function toggles engine mid-edit and asserts preserved values).
- `addon-a11y` runs axe-core on every story; zero violations required.

### 12.3 Contract tests (engine parity)

Critical. A single test suite iterates both engines over the same `FormSpec` fixture set and asserts identical observable behavior:

```ts
describe.each(['tanstack', 'rhf'] as const)('FormEngine parity: %s', (name) => {
  it('preserves values across submit', …)
  it('surfaces per-field Zod errors', …)
  it('supports array push/remove/move', …)
  it('runs async validation with debounce', …)
})
```

This is the only guarantee that swapping engines is safe.

### 12.4 E2E (Playwright, optional for v1)

Cover the demo dashboard: pick schema → fill fields → validation states → submit → success toast. One test per demo schema. Run axe-core on each rendered page.

---

## 13. Phased Delivery

Execution is broken into small, sequential phases. Each phase is sized to fit in a single PR, ends with a runnable artifact, and has a clear entry gate (what must be true before starting) and exit gate (what must be true before merging). Phases in **v1** are required; phases marked **v1.1** are deferred per the simplicity review in Appendix A.

### Map of phases

| # | Phase | Release | Depends on | Rough size |
|---|-------|---------|------------|------------|
| 0 | Foundations (deps, theme, test runner) | v1 | -- | Small |
| 1 | Core types + `ui()` helper | v1 | 0 | Small |
| 2 | `compile()` (schema → FormSpec) | v1 | 1 | Medium |
| 3 | Field registry + FallbackField | v1 | 1 | Small |
| 4 | Default field components (flat types) | v1 | 0, 3 | Medium |
| 5 | TanStack engine adapter | v1 | 2 | Medium |
| 6 | `SchemaForm` + `LayoutRenderer` + `FieldRenderer` | v1 | 2, 3, 4, 5 | Medium |
| 7 | Nested (`ObjectField`) and arrays (`ArrayField`) | v1 | 6 | Medium |
| 8 | Demo dashboard (one page, 4 schemas, state + validation panels) | v1 | 6, 7 | Small |
| 9 | Polish: error boundaries, form-level errors, docs, quickstart | v1 | 6, 7, 8 | Small |
| 10 | Nx promotion (Appendix B) | v1 or later | 9 | Medium |
| 11 | RHF adapter + parity contract tests | **v1.1** | 5, 6 | Medium |
| 12 | Engine switcher + comparison page | **v1.1** | 11 | Small |
| 13 | Advanced meta: `asyncValidate`, `dependsOn`, `hidden`, `readOnly` | **v1.1** | 6 | Medium |

Phases 0-9 in order give a junior dev a shippable v1. Phase 10 (Nx promotion) can happen at any time from 9 onward.

### Phase 0 -- Foundations

*Outcome:* the repo is ready to build schema-driven forms.

- Entry gate: the existing app still runs (`npm run dev`, `npm run storybook`).
- Add dependencies from Section 11 (Zod, MUI, Emotion, TanStack Form + zod adapter, motion; dev: Vitest, jsdom, Testing Library, MSW).
- Configure Vitest with jsdom + `@testing-library/jest-dom` matchers. One smoke test file that imports React and asserts `1 + 1 === 2`.
- Create `src/framework/tokens/mui-theme.ts` bridging `src/tokens/design-tokens.css` into `createTheme({ cssVariables: true })`.
- Wrap `src/main.tsx` with `<ThemeProvider theme={theme}><CssBaseline />…</ThemeProvider>`.
- Wrap Storybook via `.storybook/preview.ts` using `withThemeFromJSXProvider`.
- Exit gate:
  - `npm test` runs and passes the smoke test.
  - `npm run dev` and `npm run storybook` still work.
  - MUI Button renders with tokens-derived palette in a throwaway story.
- PR contents: `package.json` diff, `vitest.config.ts`, `src/framework/tokens/mui-theme.ts`, theme provider wiring, one smoke test, one temporary Storybook check.

### Phase 1 -- Core types and `ui()` helper

*Outcome:* a schema author can attach UI metadata to any Zod node.

- Entry gate: Phase 0 merged.
- Create `src/framework/core/types.ts` with `FieldMeta`, `FieldSpec`, `FormSpec`, `LayoutNode`, `ResponsiveCol`, `FieldType` (v1 meta = `type`, `label`, `helperText`, `placeholder`, `options`, `col`, `componentProps`).
- Create `src/framework/core/meta.ts` exporting `ui<T extends z.ZodTypeAny>(schema: T, meta: FieldMeta): T`. Serializes meta as JSON into `schema.describe(...)`.
- Create `src/framework/core/errors.ts` with a single exported `SchemaFormError extends Error` class that carries `{ path: string }`.
- Unit tests in `src/framework/core/__tests__/meta.test.ts`:
  - `ui()` round-trips meta through `.optional()`, `.nullable()`, `.default()`, `.describe()`.
  - Malformed envelopes throw `SchemaFormError` with the path.
- Exit gate: `npm test` green; no renderer or engine code yet.
- PR contents: the four files above plus one test file.

### Phase 2 -- `compile()` (schema → FormSpec)

*Outcome:* any Zod schema can be normalized to a `FormSpec` the renderer will consume.

- Entry gate: Phase 1 merged.
- Create `src/framework/core/compile.ts` exporting `compile(schema): FormSpec`.
- Implementation rules (from Section 4.3):
  - Walk the Zod tree once; build `fields` map keyed by dot-path, `order`, `defaults`, `validators.byField`, `validators.whole`.
  - Infer `type` from Zod node if meta doesn't set it (`ZodString`→`text`, `ZodNumber`→`number`, `ZodBoolean`→`checkbox`, `ZodEnum`→`select`, `ZodArray`→`array`, `ZodObject`→`object`, `ZodDate`→`date`, literal `true`→`checkbox`).
  - Derive `label` from field name (camel→Title Case) if meta omits it.
  - Default `col` to `{ xs: 12 }`.
  - Enforce depth limit of 5 -- throw `SchemaFormError`.
  - Default layout = one row per field in author order.
- Memoize via `WeakMap` keyed on the schema reference.
- Unit tests in `src/framework/core/__tests__/compile.test.ts`:
  - Flat schema snapshot.
  - Inference covers every Zod node type.
  - Nested object depth produces correct paths (`address.city`).
  - Array produces correct paths (`hobbies[0].name`).
  - Depth > 5 throws.
  - Missing meta falls back to defaults.
- Exit gate: snapshot tests green for flat + nested + array schemas.
- PR contents: `compile.ts` + tests + a `__fixtures__/` folder of small schemas.

### Phase 3 -- Field registry and FallbackField

*Outcome:* the renderer has a lookup mechanism for field types and a graceful fallback for unknown ones.

- Entry gate: Phase 1 merged (can run in parallel with Phase 2).
- Create `src/framework/core/registry.ts` with `createDefaultRegistry()`, `FieldRegistry` type, and `extend()` helper.
- Registry is seeded empty for now; Phase 4 will populate it.
- Create `src/framework/renderer/FallbackField.tsx` -- an MUI `Alert` (severity `warning`) + generic `TextField` wired to `binding.value` / `binding.onChange`. In dev, logs via `console.warn`.
- Unit tests:
  - `get(type)` returns registered component; returns `undefined` for unknown.
  - `extend()` does not mutate the base registry.
- Storybook: one story for `FallbackField` showing an unknown-type warning.
- Exit gate: tests green; fallback renders in Storybook.

### Phase 4 -- Default field components (flat types only)

*Outcome:* the 8 flat MUI-backed field components exist and are documented in Storybook.

- Entry gate: Phase 3 merged.
- Build, one component per commit inside the phase PR if possible:
  - `TextField` (text, password, email -- variants via `type`)
  - `NumberField`
  - `TextareaField`
  - `SelectField`
  - `CheckboxField`
  - `SwitchField`
  - `RadioGroupField`
  - `DateField` (native `type="date"` via MUI `TextField` for v1; upgrade to `@mui/x-date-pickers` if needed later)
- Each component accepts `{ spec, binding, form }`, reads `label` / `helperText` / `placeholder` / `options` from `spec.meta`, wires `binding.value` / `binding.onChange` / `binding.onBlur`, surfaces `binding.error` via MUI `error` + `helperText`.
- Every component co-located with a `*.stories.tsx` file showing `Default`, `WithError`, `Disabled`. No `ObjectField` or `ArrayField` yet.
- Exit gate: all stories render; `@storybook/addon-a11y` reports zero violations; component-level unit tests exist for each field (3-5 lines each, checking label + error surfacing).
- PR contents: 8 component folders, 8 stories, 8 test files. Register all into the default registry created in Phase 3.

### Phase 5 -- TanStack Form engine adapter

*Outcome:* a working `FormEngine` built on TanStack Form.

- Entry gate: Phases 2 and 4 merged.
- Create `src/framework/engines/types.ts` with `FormEngine`, `FormHandle`, `FieldBinding` (v1 shape; no `capabilities` yet).
- Create `src/framework/engines/tanstack/TanStackEngine.ts` and `useTanStackForm.ts`.
  - `useForm({ spec, defaultValues, onSubmit })` wraps `@tanstack/react-form`.
  - `getFieldProps(path)` returns `{ value, onChange, onBlur, error, name }` by subscribing to the TanStack field meta.
  - Per-field validators attach via `validators.onChange: spec.validators.byField[path]`.
  - Array operations exposed via `arrayOps: { push, remove, move }` backed by TanStack's array helpers.
- Unit tests using `@testing-library/react` + a tiny test harness:
  - Typing into a field updates `values`.
  - Invalid values produce the expected Zod error.
  - `submit()` resolves when valid; rejects when invalid.
  - Array push/remove/move mutate correctly.
- Exit gate: adapter tests green; no `SchemaForm` yet.
- PR contents: 3 files + 1 test file.

### Phase 6 -- `SchemaForm` + LayoutRenderer + FieldRenderer

*Outcome:* a consumer can call `<SchemaForm schema={…} onSubmit={…} />` and get a working form.

- Entry gate: Phases 2, 3, 4, 5 merged.
- Create `src/framework/renderer/`:
  - `FormContext.ts` -- React context for `{ spec, handle, registry }`.
  - `LayoutRenderer.tsx` -- translates `LayoutNode` tree into MUI `Grid` v2 tree.
  - `FieldRenderer.tsx` -- resolves `spec.type` via registry, renders component or `FallbackField`.
  - `SchemaForm.tsx` -- top-level orchestrator (compile spec, call engine `useForm`, wrap in provider, render layout).
  - `FormActions.tsx` -- default submit/reset bar with `disabled={!isValid || isSubmitting}`.
- Public API from `src/framework/index.ts`:
  - `SchemaForm`, `ui`, `defaultRegistry` only. Nothing else exported from the root.
- Create `src/framework/contracts/signup.schema.ts` with the schema from Section 4.1.
- Storybook: `SchemaForm/SchemaForm.stories.tsx` with `Default` and `WithValidationErrors` (play function fills invalid values and asserts inline errors).
- Exit gate:
  - Signup form renders in Storybook with tokens-derived MUI styling.
  - Live validation works (type invalid email → error appears after blur or change).
  - Submit with valid values calls `onSubmit` with typed `z.infer<typeof SignupSchema>`.
  - a11y addon reports zero violations.
  - 80%+ statements coverage on the renderer files.

### Phase 7 -- Nested objects and arrays

*Outcome:* nested schemas and dynamic arrays render correctly.

- Entry gate: Phase 6 merged.
- Create `src/framework/fields/ObjectField/ObjectField.tsx` -- recursively renders sub-specs via `LayoutRenderer`.
- Create `src/framework/fields/ArrayField/ArrayField.tsx` -- uses `form.arrayOps` to add/remove rows, each row is a sub-layout of the item schema.
- Register both in the default registry.
- Create `src/framework/contracts/profile.schema.ts` (nested `address`) and `src/framework/contracts/survey.schema.ts` (array of question/answer).
- Storybook stories demonstrating both.
- Tests: `ArrayField` push/remove behavior via Testing Library; depth-limit violation path in `compile.test.ts`.
- Exit gate: both stories render, validate, submit; depth-limit test green.

### Phase 8 -- Demo dashboard

*Outcome:* a single page a junior can run (`npm run dev`) that showcases the framework.

- Entry gate: Phase 7 merged.
- Build `src/demo/App.tsx` with a router route:
  - `/` = `Dashboard` page with:
    - `SchemaPicker` (dropdown listing 4 schemas: signup, profile, survey, contact)
    - `<SchemaForm>` rendered for the chosen schema
    - `StatePreview` panel showing `JSON.stringify(currentValues, null, 2)` (subscribes to the form via a hook exposed by the renderer)
    - `ValidationPanel` listing current errors
- Wire existing demo `package.json` scripts to point `dev` at `src/demo/App.tsx`.
- One simple Playwright-less smoke test is sufficient: a Storybook play function for each schema.
- Exit gate:
  - `npm run dev` loads the dashboard on :5173.
  - Switching schemas in the picker remounts the form with fresh defaults.
  - Editing fields updates `StatePreview` live.
  - Invalid values show in `ValidationPanel`.
  - No engine toggle in v1 (deferred to Phase 12).

### Phase 9 -- Polish, quickstart, and v1 docs

*Outcome:* v1 is ready for juniors.

- Entry gate: Phase 8 merged.
- Wrap `FieldRenderer` in an `ErrorBoundary` per field so one broken field doesn't crash the form.
- Add form-level error surface: when `onSubmit` rejects, render an MUI `Alert` at the top of the form with the error message; store as `errors['']` on the handle.
- Write `docs/schema-forms-quickstart.md` (~200 lines) based on Appendix A.4's "contact form in 10 minutes" flow. Include: add a field, add validation, style one field (via `componentProps`), test a form with `renderWithProviders`.
- Write `docs/schema-forms-cookbook.md` with 6 copy-paste recipes.
- Add JSDoc with one example each on `ui()`, `SchemaForm`, and every default field component.
- Verify Storybook Docs tab is populated for every field.
- Exit gate: all Appendix A.8 sanity checks pass. A volunteer (a new engineer, not the author) can build the contact form from the quickstart in under 10 minutes.

### Phase 10 -- Nx library promotion (optional timing)

*Outcome:* the framework lives in `libs/ibc/schema-forms/` as an Nx library and consumer apps import from `@ibc/schema-forms`.

- Follow Appendix B steps 1-10 exactly. No code is rewritten; everything is a move or a config addition.
- Exit gate: Appendix B.11 acceptance checklist all green.

### Phase 11 -- RHF adapter + parity contract tests (v1.1)

*Outcome:* a second engine exists, proving the abstraction holds.

- Reintroduce `FormHandle.capabilities` in the engine types.
- Implement `src/framework/engines/rhf/RHFEngine.ts` + `useRHFForm.ts` against the same interface.
- Build `src/framework/engines/__tests__/parity.test.ts`:
  ```ts
  describe.each(['tanstack', 'rhf'] as const)('engine parity: %s', (name) => {
    it('runs Zod validators per-field', …)
    it('supports array push/remove/move', …)
    it('resolves submit with typed values', …)
  })
  ```
- Add `engine` prop to `SchemaForm` (default `'tanstack'`); dynamic `import()` of engines so only the active one ships.
- Exit gate: parity suite runs against both engines and is green.

### Phase 12 -- Engine switcher + comparison demo (v1.1)

*Outcome:* the demo showcases the abstraction with live engine switching.

- Implement `useEngineSwitcher` per Section 5.4.
- Add an `EngineToggle` control to the dashboard.
- Add a `/compare` route that renders the same schema with both engines side-by-side, sharing `defaultValues`.
- Add a `SwitchingEngines` Storybook play function that toggles engines mid-edit and asserts preserved values.

### Phase 13 -- Advanced meta (v1.1)

*Outcome:* async validation, conditional fields, and hidden/readonly fields are first-class.

- Extend `FieldMeta` with `asyncValidate`, `dependsOn`, `hidden`, `readOnly`, `clearOnHide`.
- Implement the async validator registry (named validators referenced by string so schemas stay JSON-safe).
- Implement `dependsOn` via a re-render hook that subscribes to specific paths.
- Implement hidden/readonly behavior in `FieldRenderer`.
- Add cookbook examples for each.

### Tracking and visibility

Each phase is a single PR using the Conventional Commits format (`feat(schema-forms): phase 3 -- field registry and fallback`). A junior can open `docs/schema-driven-ui-framework-plan.md` Section 13, scroll to the current phase, and know exactly what files to touch, what tests to write, and what the exit gate is. No phase depends on more than three earlier phases, so the graph is small.

Project managers can track v1 completion as "Phases 0-9 green". Phase 10 (Nx) is independent and can land whenever the monorepo is ready. v1.1 work (Phases 11-13) is triggered only by a real user need.

---

## 14. Acceptance Criteria (v1 -- matches spec in the request)

1. **Schema Definition System** -- Authoring a Zod schema with `ui()` metadata renders a working, validated form. TS types flow from schema to `onSubmit`.
2. **Form Engine Abstraction** -- `SchemaForm` supports `engine="tanstack"` and `engine="rhf"` with identical observable behavior (contract tests green). No field component imports from either library.
3. **Layout Renderer** -- Forms adapt responsively across `xs/sm/md/lg`. Author layout overrides are honored; missing fields fall through to a warning + append. No CSS authored outside tokens + MUI theme.
4. **Field Registry** -- All shipped field types render. Unknown types produce the `FallbackField` with warning log. Consumers can register custom types.
5. **Live Demo Dashboard** -- Four schemas, two engines, engine-switching preserves values, `StatePreview` + `ValidationPanel` reflect state in real time.
6. **Edge cases** -- All entries in Section 8 covered by unit or story tests.

---

## 15. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| TanStack Form API churn (pre-1.0 when adopted) | Pin to a specific minor; wrap every API call inside `TanStackEngine` so upgrades touch one file. |
| RHF `resolver` validation granularity differs from TanStack's per-field model | Engine adapter re-parses `spec.validators.byField[path]` manually on change; contract tests enforce parity. |
| Zod metadata lost through transforms (`.optional()`, `.default()`, etc.) | Use `.describe()` JSON envelope which survives all Zod combinators; contract test each combinator. |
| Infinite render loops from `dependsOn` | Compile a static dependency graph, detect cycles at compile time, fail with `CompileError`. |
| `useFieldArray` vs TanStack array semantics differ | Normalize through `FormHandle.arrayOps` capability; fallback path uses `reset(nextValues)`. |
| Engine switch loses ref / focus state | Document that focus is not preserved; focus the first invalid field on switch to give a sensible UX. |
| Bundle bloat from shipping both engines | Engines are imported from `engines/tanstack/*` and `engines/rhf/*` behind a dynamic `import()` inside `useEngine`, so only the active engine ships in the primary chunk. |
| Schema meta schema drift (author writes wrong meta) | `ui()` validates meta against an internal Zod schema at authoring time in dev (`process.env.NODE_ENV !== 'production'`). |

---

## 16. Out-of-Band Questions for the Product Owner

None blocking. Decisions assumed in this plan:

- **Default engine = TanStack** (per user preference stated in the request).
- **Zod v3** (v4 is not stable as of authoring; upgrade path is a one-line adapter change).
- **MUI Grid v2** for layout.
- **CSS Modules** are *not* used inside the framework (MUI theme + `sx`) but remain available for custom non-MUI field components per the architecture skill.
- **No router integration** in the framework itself; the demo uses `react-router-dom` v7 already in the repo.

---

## 17. Quick Reference

| Command | Purpose |
|---------|---------|
| `npm run dev` | Launch demo dashboard on :5173 |
| `npm run storybook` | Launch Storybook on :6006 |
| `npm test` | Run Vitest suite (unit + contract tests) |
| `npm run build` | Type-check + production build of demo |
| `npm run build-storybook` | Static Storybook export |

---

## 18. Tree of Deliverables (Cheat Sheet)

```
framework/
  core/                    ← compile, meta, registry, types, errors
  engines/
    tanstack/              ← primary
    rhf/                   ← secondary
    switcher.ts
  renderer/
    SchemaForm.tsx         ← public entry
    LayoutRenderer.tsx
    FieldRenderer.tsx
    FallbackField.tsx
  fields/                  ← MUI-backed default field set
  hooks/
  testing/
  index.ts                 ← public surface

contracts/                 ← consumer-authored schemas (4 demo schemas)

demo/                      ← dashboard + comparison pages
```

This is the complete plan. Execution follows the phases in Section 13; every phase ends with runnable artifacts and green tests, so progress is observable without a final big-bang integration.

---

## Appendix A -- Simplicity Review and Junior-Dev Quickstart

This plan optimizes for a capability ceiling that the request describes (two engines, registry, compiled IR, mid-edit engine switching). Not every ceiling needs to be in v1, and not every concept needs to be visible to a junior dev at the call-site. This appendix calls out where the design was over-built, trims the v1 scope, and codifies the API shape a new developer should actually see.

### A.1 Is this over-engineered?

Honestly, yes -- in specific places:

| Concept in plan | Verdict | Recommendation |
|---|---|---|
| Two form engines + parity contract tests | **Defer.** Requested, but ~40% of the code and the main complexity source. | Ship **TanStack only in v1**. Keep the `FormEngine` interface as the seam, but don't build the RHF adapter or engine-switcher until a real app asks for it. The demo "engine comparison" tab becomes a v1.1 story. |
| `useEngineSwitcher` with mid-edit value preservation | **Defer.** Demo-only concern; production apps pick an engine and stay. | Move to v1.1 alongside the RHF adapter. Not in public API for v1. |
| `FormHandle.capabilities` probe | **Cut.** With one engine shipped, capabilities is dead code. | Re-introduce when the second adapter lands. |
| Dynamic `import()` of engines for code-splitting | **Cut.** Premature optimization. One engine means nothing to split. | Plain static imports. |
| Static `dependsOn` graph + cycle detection at compile time | **Cut for v1.** Real cycles are rare; runtime recursion guard is enough. | Simple `<form.Subscribe>`-style re-render when referenced paths change. |
| Max-depth compile error (`depth > 5`) | **Keep but simplify.** It's a few lines and it prevents a confusing failure mode. | Throw a plain `Error` with the offending path; skip the custom `CompileError` class. |
| Granular `exports` map, no barrels | **Keep.** Tree-shaking matters even in v1. | No change. |
| Compiled IR (`FormSpec`) | **Keep, but hide.** It is the thing that makes engine-agnosticism actually work. | Keep it internal. No consumer code imports `FormSpec`; they pass a Zod schema and get a form. |
| Field registry | **Keep.** This is the extension point juniors and seniors both need. | Ship a narrow default registry (10 field types). Override via one prop. |
| Layout DSL (`LayoutNode`) with rows/sections/dividers | **Keep, but make optional.** 95% of forms need no layout authoring -- just `col` per field. | Default layout = one row per field, responsive from `col`. Only authors who want sections or custom grids reach for `LayoutNode`. |
| `ui()` meta with `autoComplete`, `hidden`, `readOnly`, `componentProps`, etc. | **Keep, but start small.** Ship the minimum first; extend on demand. | v1 meta = `type`, `label`, `helperText`, `placeholder`, `options`, `col`. Add the rest as real schemas request them. |

Net effect: v1 is roughly the plan minus the RHF adapter, the switcher, the capability probe, dynamic imports, and the static dependency-graph analyzer. The public API and folder layout stay the same so v1.1 is additive.

### A.2 The API a junior dev sees

Two touchpoints, that's it.

**(1) Authoring a schema.** Same Zod they already use, with one extra helper:

```ts
// libs/shared/data-access/schemas/signup.ts
import { z } from 'zod'
import { ui } from '@ibc/schema-forms'

export const SignupSchema = z.object({
  email:    ui(z.string().email(),            { label: 'Email',   col: { xs: 12, md: 6 } }),
  password: ui(z.string().min(8),             { label: 'Password', col: { xs: 12, md: 6 } }),
  role:     ui(z.enum(['admin', 'member']),   { label: 'Role' }),   // select inferred
  terms:    ui(z.literal(true),               { label: 'I accept the terms' }),
})
```

Notice what is *not* required: no `type:` (inferred from the Zod node in 90% of cases), no layout, no registry. A junior writes Zod + labels and the form works.

**(2) Rendering a form.** One component:

```tsx
// apps/shell/src/pages/SignupPage.tsx
import { SchemaForm } from '@ibc/schema-forms'
import { SignupSchema } from '@ibc/shared/data-access/schemas/signup'

export function SignupPage() {
  return (
    <SchemaForm
      schema={SignupSchema}
      onSubmit={async (values) => await api.signup(values)}
    />
  )
}
```

That's the end of the mandatory surface area. `SchemaForm` handles submit, reset, validation, error display, and layout. A junior shipping a form knows exactly these two things.

### A.3 Progressive disclosure (when they need more)

Advanced knobs exist but stay opt-in. A junior only meets them when the task actually calls for it:

| Need | Additional concept | Example |
|---|---|---|
| Custom input look for one field | `componentProps` passthrough | `ui(z.string(), { label: 'Bio', componentProps: { multiline: true, rows: 4 } })` |
| Brand your own Input across the app | Registry override | `<SchemaForm registry={defaultRegistry.extend({ text: OurInput })} … />` |
| Two fields side-by-side on desktop | `col` per field | `col: { xs: 12, md: 6 }` |
| Sections, dividers, or custom grouping | Author a `LayoutNode` | Covered in Section 6.1; not needed for typical forms |
| Async uniqueness check | `asyncValidate` in meta | `ui(z.string().email(), { label: 'Email', asyncValidate: 'checkEmail' })` |
| Conditional field | `dependsOn` + Zod `discriminatedUnion` | Covered in Section 10.2's billing example |
| Array of repeating items | Nothing extra -- `z.array(z.object(…))` just works | `ArrayField` handles it by default |

Each of these is a single prop or a single line of meta. The mental model never requires a junior to know there is an IR, an engine adapter, or a registry lookup happening.

### A.4 The 80% form -- from zero to screen

This is the literal recipe a junior gets in the README. It should fit on one screen.

```tsx
// 1. Define the schema
import { z } from 'zod'
import { ui } from '@ibc/schema-forms'

export const ContactSchema = z.object({
  name:    ui(z.string().min(1),       { label: 'Name' }),
  email:   ui(z.string().email(),      { label: 'Email' }),
  message: ui(z.string().min(10),      { label: 'Message', componentProps: { multiline: true, rows: 4 } }),
})

// 2. Render it
import { SchemaForm } from '@ibc/schema-forms'

export function ContactPage() {
  return <SchemaForm schema={ContactSchema} onSubmit={send} />
}

async function send(values: z.infer<typeof ContactSchema>) {
  await fetch('/api/contact', { method: 'POST', body: JSON.stringify(values) })
}
```

Three imports. One schema. One component. Types flow from the schema to `send()` automatically. Validation, error display, submit handling, and accessibility are all included.

### A.5 What a junior will hit first, and how the library handles it

| First surprise | How the library responds |
|---|---|
| Forgot to add `ui(...)` wrapper | Field still renders with type inferred from Zod + label derived from the field name (camel→Title Case). `console.warn` in dev tells them the convention. |
| Used a `type` the registry doesn't know | `FallbackField` renders a visible warning Alert + a plain text input; the form still submits. Not a crash. |
| Typo in `col` breakpoint key | TypeScript catches it (`col: { xs, sm, md, lg, xl }` is a typed interface). |
| Wants to test the form | `renderWithProviders` from `@ibc/schema-forms/testing` wraps with MUI theme + QueryClient; they import it and their test works. |
| Needs to reset after submit | `<SchemaForm resetOnSuccess />` one-prop behavior. |
| Needs a custom submit button | `<SchemaForm actions={<MyActions />} />` children slot. |
| Wants to prefill from URL / server | `defaultValues={{ email: session.email }}` prop. |

The library owns the graceful-degradation edge cases so the junior doesn't have to think about them.

### A.6 Documentation we commit to shipping for juniors

Docs are not a nice-to-have; for a framework with this much abstraction, they are the actual UX. As part of v1:

- `docs/schema-forms-quickstart.md` -- the recipe in Section B.4, expanded to ~200 lines with "add a field", "add validation", "add async validation", "style one field", "test a form".
- `docs/schema-forms-cookbook.md` -- 6-8 copy-paste examples: contact form, signup, profile, nested address, repeating line items, conditional billing form, async email check, multi-step-lite (two `SchemaForm` instances).
- Inline JSDoc on `ui()`, `SchemaForm`, and each default field component, with one example each. Shows up on hover in VS Code.
- Storybook `Docs` tab for every default field with props table + "Try it" controls.
- Storybook MCP integration (already in this repo per `AGENTS.md`) means a junior's AI assistant can call `get-documentation` before using any field.

### A.7 Revised v1 scope

Taking the trims above, v1 ships:

- Core: `compile()`, `ui()`, `FieldRegistry`, `FallbackField`, errors.
- Engine: **TanStack Form only**. `FormEngine` interface present; second adapter deferred.
- Renderer: `SchemaForm`, `LayoutRenderer` (default layout for 95% of forms), `FieldRenderer`.
- Fields: `TextField`, `NumberField`, `SelectField`, `CheckboxField`, `SwitchField`, `RadioGroupField`, `DateField`, `TextareaField`, `ObjectField`, `ArrayField`.
- Meta: `type`, `label`, `helperText`, `placeholder`, `options`, `col`, `componentProps`.
- Demo: one dashboard page with four schemas, a `StatePreview` JSON panel, and a `ValidationPanel`. **No engine toggle in v1.**
- Docs: quickstart + cookbook + JSDoc + Storybook autodocs.
- Nx packaging per Appendix B.

v1.1 (when a real user need appears):

- RHF adapter + `FormEngine` capability probe.
- `useEngineSwitcher` and engine-toggle in demo.
- `asyncValidate`, `dependsOn` full graph analysis, `hidden`, `readOnly`, `clearOnHide`.
- Dynamic `import()` engine splitting if bundle analysis shows it matters.

This shaves the plan by roughly a third while keeping every promise to the consumer intact. Most importantly, **the API a junior dev sees in v1 is the same API they see in v1.1** -- their call-sites do not change when we add the second engine, because the engine choice is an optional prop that defaults to `tanstack`.

### A.8 Sanity checks before we call v1 "junior-friendly"

Run these before shipping:

- [ ] A junior who has never seen the library can build the contact form in Section B.4 in under 10 minutes with only the quickstart open.
- [ ] Their schema has **no `type`** properties -- inference covers every field they used.
- [ ] They never import from `@ibc/schema-forms/core`, `@ibc/schema-forms/engines/*`, or any internal path. Default surface is enough.
- [ ] Every default field has a Storybook page with working Controls they can copy args from.
- [ ] When they break the schema (e.g. pass an unsupported Zod combinator), the dev-mode error names the field and tells them what to do.
- [ ] The public TypeScript surface of `@ibc/schema-forms` has fewer than 20 exported names. (IR types, engine types, and internal helpers are not exported.)

If any of those fails, the abstraction leaked and needs sanding before release.

---

## Appendix B -- Packaging as an Nx Library

Short answer: **yes**, and by design. The folder layout in Section 3 was chosen so that promotion into an Nx workspace is a move + config operation, not a rewrite. This appendix specifies exactly how.

### B.1 Target monorepo layout

Assuming the enterprise layout from `.cursor/skills/frontend-architecture/SKILL.md` (scope-first, type-tagged):

```
apps/
  shell/                               # consumer app that renders forms
  demo/                                # optional: the live dashboard from Section 10

libs/
  ibc/
    schema-forms/                      # ← the framework lives here
      src/
        lib/
          core/                        # compile, meta, registry, types, errors
          engines/
            tanstack/
            rhf/
            switcher.ts
          renderer/
          fields/                      # MUI-backed default field set
          hooks/
          testing/
        index.ts                       # public surface, no sub-barrels
      .storybook/                      # library-scoped Storybook (Section A.6)
      project.json
      package.json                     # exports map (Section A.4)
      vite.config.ts                   # buildable library config
      README.md
    ui/                                # existing atomic design system (optional, separate lib)

  shared/
    tokens/                            # design tokens + MUI theme (dependency)
    data-access/                       # optional: Zod schemas for API + forms (dependency)
```

The library name is `@ibc/schema-forms` (tag it `type:ui, scope:shared`). It depends on `@ibc/tokens` (for the MUI theme bridge) and peer-depends on `react`, `@mui/material`, `zod`, and optionally `@tanstack/react-form` / `react-hook-form`. Consumer apps install whichever engine(s) they use.

### B.2 Generate the library

```bash
# Scaffold a buildable React library inside the scope-first tree
nx g @nx/react:library libs/ibc/schema-forms \
  --bundler=vite \
  --unitTestRunner=vitest \
  --buildable \
  --importPath=@ibc/schema-forms \
  --tags="type:ui,scope:shared" \
  --component=false \
  --directory=libs/ibc/schema-forms
```

Flags that matter:

- `--bundler=vite` aligns with the rest of the stack and produces an ESM-first output with type declarations.
- `--buildable` creates a `build` target. Use `--publishable --importPath=@ibc/schema-forms` instead if you plan to publish to npm or a private registry.
- `--component=false` suppresses the default `<SchemaForms>` stub -- the framework exposes `<SchemaForm>` from its own folder.
- `--tags` are enforced by `@nx/enforce-module-boundaries` so apps can import the library only when their scope permits.

### B.3 `project.json` targets

```jsonc
{
  "name": "ibc-schema-forms",
  "$schema": "../../../node_modules/nx/schemas/project-schema.json",
  "sourceRoot": "libs/ibc/schema-forms/src",
  "projectType": "library",
  "tags": ["type:ui", "scope:shared"],
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "options": {
        "outputPath": "dist/libs/ibc/schema-forms",
        "main": "libs/ibc/schema-forms/src/index.ts",
        "tsConfig": "libs/ibc/schema-forms/tsconfig.lib.json",
        "assets": ["libs/ibc/schema-forms/*.md"]
      }
    },
    "lint":   { "executor": "@nx/eslint:lint" },
    "test":   { "executor": "@nx/vite:test",
                "options": { "config": "libs/ibc/schema-forms/vite.config.ts" } },
    "storybook": {
      "executor": "@nx/storybook:storybook",
      "options": { "port": 6007, "configDir": "libs/ibc/schema-forms/.storybook" }
    },
    "build-storybook": {
      "executor": "@nx/storybook:build",
      "options": { "outputDir": "dist/storybook/ibc-schema-forms",
                   "configDir": "libs/ibc/schema-forms/.storybook" }
    },
    "typecheck": { "executor": "nx:run-commands",
                   "options": { "command": "tsc -p libs/ibc/schema-forms/tsconfig.lib.json --noEmit" } }
  }
}
```

### B.4 `package.json` exports (granular, no barrels)

Per Section 1 of the architecture skill, expose specific entry points instead of a single barrel. This keeps tree-shaking intact even though the library ships both engines and the full default field set:

```jsonc
{
  "name": "@ibc/schema-forms",
  "version": "0.1.0",
  "sideEffects": false,
  "type": "module",
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "@mui/material": "^6.0.0",
    "@emotion/react": "^11.0.0",
    "@emotion/styled": "^11.0.0",
    "zod": "^3.22.0"
  },
  "peerDependenciesMeta": {
    "@tanstack/react-form": { "optional": true },
    "@tanstack/zod-form-adapter": { "optional": true },
    "react-hook-form": { "optional": true },
    "@hookform/resolvers": { "optional": true }
  },
  "exports": {
    ".":                  "./src/index.ts",                       // <SchemaForm>, types, default registry
    "./core":             "./src/lib/core/index.ts",              // compile, ui(), errors, types
    "./registry":         "./src/lib/core/registry.ts",
    "./engines/tanstack": "./src/lib/engines/tanstack/index.ts",  // opt-in
    "./engines/rhf":      "./src/lib/engines/rhf/index.ts",       // opt-in
    "./fields":           "./src/lib/fields/index.ts",            // default MUI fields
    "./testing":          "./src/lib/testing/index.ts"            // renderWithProviders, mock schemas
  }
}
```

Consumers pull what they need:

```ts
import { SchemaForm, ui } from '@ibc/schema-forms'
import { tanstackEngine } from '@ibc/schema-forms/engines/tanstack'
import { MyBrandedTextField } from '@ibc/schema-forms/fields'
```

The two engines are behind their own subpath exports so if an app only installs `@tanstack/react-form` it never pulls `react-hook-form` into its bundle, and vice versa. The `peerDependenciesMeta` block marks both engines optional so `npm install @ibc/schema-forms` does not warn when only one is installed.

### B.5 Module boundary tags

Add these constraints to `eslint.config.mjs` (the skill already defines the structure; `schema-forms` slots in under `type:ui, scope:shared`):

```ts
{ sourceTag: 'type:ui', onlyDependOnLibsWithTags: ['type:ui', 'type:util'] },
{ sourceTag: 'scope:shared', onlyDependOnLibsWithTags: ['scope:shared'] },
```

Consequences:

- `@ibc/schema-forms` may import `@ibc/tokens` (`scope:shared, type:util`) but **not** from any `scope:shell` or `scope:agent` lib. This keeps the framework genuinely shared.
- Apps of any scope may depend on it, because apps have `type:app` which dep-constraints allow to consume shared `type:ui` libs.
- `@ibc/schema-forms` may import `@ibc/ui` (the atomic design system) if you want shared atoms as the fallback for custom field components. Keep that dep one-way and optional.

### B.6 Library-scoped Storybook

Each library gets its own Storybook instance (the skill endorses Storybook composition across the monorepo):

```
libs/ibc/schema-forms/.storybook/
  main.ts       # stories glob: '../src/**/*.stories.@(ts|tsx)'
  preview.ts    # imports @ibc/tokens CSS + MUI ThemeProvider + MSW loader
  theme.ts
```

Root workspace `.storybook/main.ts` composes library Storybooks via `refs`, so `nx storybook shell` opens a combined UI showing `ibc/ui` + `ibc/schema-forms` + feature Storybooks.

Run isolated during development:

```bash
nx storybook ibc-schema-forms         # port 6007
nx run ibc-schema-forms:test          # vitest watch
nx run ibc-schema-forms:build         # emit dist/
```

### B.7 Publishable vs. buildable

Pick one up front; switching later is a small config change but noisy in git.

| Mode | When to use | Effect |
|------|-------------|--------|
| **Buildable** | Monorepo-only consumption. Apps import via TS path alias; Nx caches the build output. | `nx build ibc-schema-forms` produces `dist/libs/ibc/schema-forms` used by app bundlers during dev and build. No version bumps. |
| **Publishable** | Cross-repo or open-sourcing. | Same as buildable, plus `release` target wiring (nx release / changesets) that publishes `@ibc/schema-forms` to a registry with semver versioning and a CHANGELOG. |

Start buildable. If a second repo needs it, promote to publishable with `nx g @nx/react:library ... --publishable --importPath=@ibc/schema-forms` applied as a patch to `package.json` + add `nx release` config.

### B.8 Consumer wiring (app side)

In the consuming app (`apps/shell`), installation is transparent because Nx links the workspace package:

```tsx
// apps/shell/src/pages/SignupPage.tsx
import { SchemaForm } from '@ibc/schema-forms'
import { SignupSchema } from '@ibc/shared/data-access/schemas/signup'

export function SignupPage() {
  return (
    <SchemaForm
      schema={SignupSchema}
      engine="tanstack"
      onSubmit={async (values) => api.signup(values)}
    />
  )
}
```

`@ibc/shared/data-access` hosts the Zod schemas that back both forms (via `@ibc/schema-forms`) and API response validation (via `@ibc/shared/data-access`). Section 8 of the skill already enforces this "one schema, shared for form + API" pattern; `schema-forms` just consumes whatever schemas live there.

### B.9 Affected graph

Because the library is a first-class Nx project, the affected graph reacts correctly:

- A change in `libs/ibc/schema-forms/src/lib/core/compile.ts` invalidates `ibc-schema-forms`, any app importing it (`shell`, `agent`, `demo`), and every feature library that transitively depends on those apps.
- `nx affected -t lint test build` in CI runs only those projects.
- `nx graph` renders `ibc-schema-forms` as a node with edges to `ibc-tokens`, `shared-data-access`, and downstream consumers.

### B.10 Migration from the single-package scaffold

If Phase 0-8 of Section 13 have already been executed in the current single-package repo and you later want to promote the code into an Nx monorepo:

1. `npx create-nx-workspace@latest ibc --preset=apps` in a sibling directory.
2. `nx g @nx/react:library libs/ibc/schema-forms --importPath=@ibc/schema-forms --buildable --bundler=vite --unitTestRunner=vitest --tags=type:ui,scope:shared --component=false`.
3. Move `src/framework/*` → `libs/ibc/schema-forms/src/lib/*` (one-to-one; no restructuring required).
4. Update relative imports: inside the lib use relative paths; across libs use `@ibc/*` path aliases from `tsconfig.base.json`.
5. Copy `src/framework/*.stories.tsx` in place; wire the library Storybook from Section A.6.
6. Port `src/demo/*` to `apps/demo/src/` (or fold into `apps/shell` as a showcase route).
7. Convert the `SignupSchema` / `ProfileSchema` etc. to live in `libs/shared/data-access/schemas/` so both forms and APIs consume them.
8. Update `package.json` exports per Section A.4.
9. Add tags + boundary constraints; run `nx lint` and fix violations until clean.
10. `nx affected -t lint test build storybook` green → done.

Because the single-package scaffold mirrors the lib's internal folder layout (core/engines/renderer/fields/hooks/testing), step 3 is a `git mv` tree, not a rewrite. No import paths above the `framework/` boundary need to change: `@/framework/*` becomes `@ibc/schema-forms`.

### B.11 Acceptance checklist for the Nx packaging

- [ ] `nx build ibc-schema-forms` emits a tree-shakeable ESM dist with `.d.ts` files.
- [ ] `nx test ibc-schema-forms` runs unit + engine-parity contract tests green.
- [ ] `nx storybook ibc-schema-forms` serves stories on :6007 with a11y addon clean.
- [ ] `nx lint ibc-schema-forms` passes with module-boundary constraints active.
- [ ] A consumer app can `import { SchemaForm } from '@ibc/schema-forms'` and `import { tanstackEngine } from '@ibc/schema-forms/engines/tanstack'` with zero barrel imports.
- [ ] Installing only one of the two form engines in the app still builds (peer meta `optional: true`).
- [ ] `nx graph` shows `ibc-schema-forms` as a shared node depended on by shell/agent/demo apps, with no inbound edges from scope-specific libs.

---

## Appendix C -- MUI v9 Compatibility and Upgrade Path

Short answer: **yes, the design is compatible with MUI v9**, and the plan is structured so the upgrade from v6/v7 to v9 is a contained migration rather than a rewrite. This appendix covers what changes, what stays, and the concrete steps.

### C.1 What MUI v9 actually ships (the parts that touch this framework)

MUI v9 (released April 8, 2026, re-synchronized with MUI X v9) is primarily a polish + breaking-deprecation release. The items that intersect our design:

| v9 change | Impact on schema-forms |
|---|---|
| Removal of deprecated `component` and `componentsProps` props across the library | Low. Our field wrappers don't use these -- they use `slots` + `slotProps` or the `as` prop on polymorphic helpers. Audit needed in Phase 4 review. |
| Removal of deprecated system props from layout components | Low. We use Grid v2 (`size={…}`) and `sx`, both supported. |
| `disableEscapeKeyDown` removed from `Dialog` / `Modal` | None. Framework doesn't own Dialog/Modal; if consumers wrap us in one, it's their call. |
| New `NumberField` primitive from Base UI | **Opportunity**. When on v9, our default `NumberField` should wrap MUI's new `NumberField` (better a11y + keyboard handling). v6/v7 path keeps using our `TextField type="number"` wrapper. |
| New `Menubar` | Not used by the framework. Available to consumers if they build custom page chrome. |
| CSS variables + `color-mix()` derived colors | Neutral-positive. Our theme bridge already enables `cssVariables: true`; derived colors work automatically when consumers upgrade. |
| `TableCell` border `color-mix` + `nativeColor` + `cssVariables` interaction fix | None for the framework itself. |
| Autocomplete `root` slot + full slots for indicators | Positive. If we ever add a combobox field, we'll use the new slot API. Not in v1 scope. |
| Roving TabIndex across Stepper / Tabs / MenuList | Positive. Our `SelectField` and `RadioGroupField` benefit automatically. |
| `aria-hidden` removed from `Backdrop` by default | None for the framework. |
| Theme typing: `MuiTouchRipple` removed | Check theme augmentation in `tokens/mui-theme.ts`; we don't override `MuiTouchRipple` so we are clean. |
| Bundle size ~3% smaller; `sx` up to 30% faster in heavy usage | Free win. |
| Future: Emotion dependency will be removed ("What's next" section of v9 blog) | **Track**. When Emotion is dropped in a post-v9 minor, our install instructions change (drop `@emotion/react`, `@emotion/styled` peer deps). Non-breaking to our API. |

Nothing in v9 changes the mental model or the public API of `SchemaForm`, `ui()`, or the field registry. It's our field-component internals + peer-dep declarations that move.

### C.2 Why the design absorbs this cleanly

Three choices in Sections 5-7 make version churn absorbable:

1. **We don't re-export MUI components.** Every default field is a *wrapper* around MUI primitives. Consumers import `TextField` from `@ibc/schema-forms/fields`, not `@mui/material`. A major MUI bump touches ~10 files in `src/framework/fields/*`, not the consuming app's 400 call-sites.
2. **Styling is theme + `sx`, never `@mui/system` layout props.** The deprecated system layout props (the thing v9 removes) are not used anywhere in the framework. Consumers who follow the skill's rules are also safe.
3. **The `FieldBinding` contract is MUI-free.** `{ value, onChange, onBlur, error, name }` is a plain shape. Swapping the rendering layer (for example, replacing MUI with Joy or Base UI directly in v1.2) doesn't touch the engine adapters, the compile step, or the registry.

### C.3 Version matrix

Commit to supporting a window, not a point release:

| MUI line | React | Zod | TanStack Form | Framework status |
|---|---|---|---|---|
| v6.x | 18.x | 3.x | 0.x - 1.x | v1 ships here |
| v7.x | 18.x or 19.x | 3.x | 1.x | Supported (minor audit) |
| v9.x | 19.x | 3.x or 4.x | 1.x | **Supported via v1.1 minor**; adds optional `NumberField` Base UI backend |

Skip v8 entirely -- MUI itself did (v7 → v9 to align with MUI X). The React 19 move coincides with MUI v9 and is addressed in Appendix D-like future work if and when the consuming apps are ready.

### C.4 Peer-dep declaration (forward-compatible)

In `libs/ibc/schema-forms/package.json` (Appendix B.4), widen the MUI peer range so consumers can upgrade without our explicit release:

```jsonc
{
  "peerDependencies": {
    "react":         ">=18.0.0 <20.0.0",
    "react-dom":     ">=18.0.0 <20.0.0",
    "@mui/material": ">=6.0.0 <10.0.0",
    "@emotion/react":   ">=11.0.0 <13.0.0",
    "@emotion/styled":  ">=11.0.0 <13.0.0",
    "zod":              ">=3.22.0 <5.0.0"
  },
  "peerDependenciesMeta": {
    "@emotion/react":   { "optional": true },
    "@emotion/styled":  { "optional": true },
    "@tanstack/react-form":    { "optional": true },
    "@tanstack/zod-form-adapter": { "optional": true },
    "react-hook-form": { "optional": true },
    "@hookform/resolvers": { "optional": true }
  }
}
```

Emotion is *optional* in the v9+ world (MUI has signaled it will remove the hard dependency), so we mark it optional now. On v6/v7 installs, npm will warn if it's missing; that's the correct behavior because on v6/v7 Emotion is still required.

### C.5 What to audit when v9 upgrade happens (checklist)

Treat the upgrade as a single PR guarded by the existing test suite (unit + Storybook play functions + a11y addon):

- [ ] Bump `@mui/material`, `@mui/system`, `@mui/icons-material` to v9.
- [ ] Run MUI's codemods: `npx @mui/codemod@latest v9.0.0/preset-safe src/framework/fields`.
- [ ] Grep `src/framework/fields/**` for `component=`, `componentsProps=` -- replace with `slots` + `slotProps`. These should already be absent if Phase 4 was implemented to the skill.
- [ ] Grep `src/framework/**` for deprecated system layout props (`display=`, `alignItems=`, `justifyContent=`, etc. applied directly to `<Box>` / `<Grid>`). Move to `sx`.
- [ ] Check `tokens/mui-theme.ts` for any `MuiTouchRipple` theme overrides -- remove (removed from theme types in v9).
- [ ] Re-run Storybook + a11y addon across the full story set. Zero new violations.
- [ ] Re-run Vitest unit tests. All snapshots still match or are intentionally updated.
- [ ] (v9-only enhancement, optional) Refactor `NumberField/NumberField.tsx` to wrap MUI's new `NumberField` primitive. Behind a feature flag or behind a `peerDependencies` check -- ship only when the consumer is on v9+.
- [ ] Update `docs/schema-forms-quickstart.md` install command if Emotion drop has happened.

If the checklist is green, the upgrade is done. Expected size: a few files + a codemod pass, not a redesign.

### C.6 Where the plan would have to change if MUI v9 had been a bigger break

For completeness, here's what *would* have forced a design change -- none of which happened:

- If MUI had dropped CSS variables support → our theme bridge (Section 5) would need rewrite. **It didn't; variables are now the preferred path.**
- If Grid v2 had been deprecated → `LayoutRenderer` would need to migrate to `Stack` + manual breakpoints. **It wasn't.**
- If the `sx` prop had been removed → our per-field `componentProps` would lose its main escape hatch. **It wasn't; sx got 30% faster.**
- If MUI had adopted a different React form-control signature → our `FieldBinding` adapter in each field wrapper would need updates. **It didn't.**

The design absorbs MUI v9 because the v9 release is evolutionary. If a future major were revolutionary, the affected surface is still limited to `src/framework/fields/*` and `tokens/mui-theme.ts` -- engines, compile, registry, renderer, and the public API stay fixed.

### C.7 Summary

- **v1 ships on MUI v6 (or v7).** Works unchanged on v9 after a small audit + codemod PR.
- **The framework's public API does not change with MUI version.** Consumers' call-sites are isolated by our field wrappers.
- **Upgrading is one PR, not a project.** Peer ranges are wide, Emotion is optional, and the test suite catches regressions.
- **v9-only goodies (NumberField Base UI primitive, improved Roving TabIndex) land automatically or behind a tiny optional refactor.**
