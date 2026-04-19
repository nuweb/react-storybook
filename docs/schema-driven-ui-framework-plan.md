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

Each phase ends with a runnable artifact and all tests green.

### Phase 0 -- Foundations
- Add dependencies (Section 11).
- Configure Vitest + jsdom + RTL + `@testing-library/jest-dom`.
- Add MUI theme bridging existing design tokens (`src/tokens/design-tokens.css`) via `createTheme({ cssVariables: true })`.
- Wire `ThemeProvider` + `CssBaseline` in `src/main.tsx` and in `.storybook/preview.ts`.

**Done when:** `npm test` runs; MUI theme renders in Storybook; existing stories unaffected.

### Phase 1 -- Core + compile + registry
- Implement `core/types.ts`, `core/meta.ts`, `core/compile.ts`, `core/registry.ts`, `core/errors.ts`.
- Unit tests for all four.

**Done when:** `compile(SignupSchema)` returns a correct `FormSpec` snapshot; registry tests pass.

### Phase 2 -- Default field components
- Ship `TextField`, `NumberField`, `SelectField`, `CheckboxField`, `SwitchField`, `RadioGroupField`, `DateField`, `TextareaField` as MUI wrappers consuming `FieldBinding`.
- Each gets a Storybook story with `Default` + error + disabled + async-loading variants.

**Done when:** All stories render; a11y addon reports zero violations.

### Phase 3 -- TanStack engine + SchemaForm + LayoutRenderer
- Implement `TanStackEngine`.
- Implement `SchemaForm`, `LayoutRenderer`, `FieldRenderer`, `FallbackField`.
- Ship `signup.schema.ts` + a `Default` SchemaForm story.

**Done when:** The signup form renders, validates live, submits successfully, and shows errors inline. Engine capabilities populated.

### Phase 4 -- Nested + array fields
- Implement `ObjectField` and `ArrayField`.
- Ship `profile.schema.ts` and `survey.schema.ts` + stories.
- Add depth-limit compile tests.

**Done when:** Deeply nested and dynamic-array forms work end to end on TanStack.

### Phase 5 -- RHF engine
- Implement `RHFEngine` against the same interface.
- Add contract tests that run both engines over the same fixtures.

**Done when:** All contract tests pass for both engines.

### Phase 6 -- Engine switcher + demo dashboard
- Implement `useEngineSwitcher`.
- Build `Dashboard`, `Comparison`, `SchemaPicker`, `EngineToggle`, `StatePreview`, `ValidationPanel`.
- Add `SwitchingEngines` play-function story.

**Done when:** Demo app runs via `npm run dev`; switching engines mid-edit preserves values; comparison page shows both engines in sync.

### Phase 7 -- Polish
- Error boundary around each field.
- Form-level submit error surface (`errors['']`).
- Logger hook-up for fallback + error boundary events.
- `dependsOn` wiring with a `<form.Subscribe>`-style hook (engine-agnostic) for conditional rendering.
- Motion crossfade on engine swap.

**Done when:** All edge cases in Section 8 have explicit tests.

### Phase 8 -- Docs + examples (optional v1.1)
- Add `docs/schema-driven-ui-overview.md` quick-start.
- Auto-generate field docs from `get-documentation` MCP.

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

## Appendix A -- Packaging as an Nx Library

Short answer: **yes**, and by design. The folder layout in Section 3 was chosen so that promotion into an Nx workspace is a move + config operation, not a rewrite. This appendix specifies exactly how.

### A.1 Target monorepo layout

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

### A.2 Generate the library

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

### A.3 `project.json` targets

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

### A.4 `package.json` exports (granular, no barrels)

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

### A.5 Module boundary tags

Add these constraints to `eslint.config.mjs` (the skill already defines the structure; `schema-forms` slots in under `type:ui, scope:shared`):

```ts
{ sourceTag: 'type:ui', onlyDependOnLibsWithTags: ['type:ui', 'type:util'] },
{ sourceTag: 'scope:shared', onlyDependOnLibsWithTags: ['scope:shared'] },
```

Consequences:

- `@ibc/schema-forms` may import `@ibc/tokens` (`scope:shared, type:util`) but **not** from any `scope:shell` or `scope:agent` lib. This keeps the framework genuinely shared.
- Apps of any scope may depend on it, because apps have `type:app` which dep-constraints allow to consume shared `type:ui` libs.
- `@ibc/schema-forms` may import `@ibc/ui` (the atomic design system) if you want shared atoms as the fallback for custom field components. Keep that dep one-way and optional.

### A.6 Library-scoped Storybook

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

### A.7 Publishable vs. buildable

Pick one up front; switching later is a small config change but noisy in git.

| Mode | When to use | Effect |
|------|-------------|--------|
| **Buildable** | Monorepo-only consumption. Apps import via TS path alias; Nx caches the build output. | `nx build ibc-schema-forms` produces `dist/libs/ibc/schema-forms` used by app bundlers during dev and build. No version bumps. |
| **Publishable** | Cross-repo or open-sourcing. | Same as buildable, plus `release` target wiring (nx release / changesets) that publishes `@ibc/schema-forms` to a registry with semver versioning and a CHANGELOG. |

Start buildable. If a second repo needs it, promote to publishable with `nx g @nx/react:library ... --publishable --importPath=@ibc/schema-forms` applied as a patch to `package.json` + add `nx release` config.

### A.8 Consumer wiring (app side)

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

### A.9 Affected graph

Because the library is a first-class Nx project, the affected graph reacts correctly:

- A change in `libs/ibc/schema-forms/src/lib/core/compile.ts` invalidates `ibc-schema-forms`, any app importing it (`shell`, `agent`, `demo`), and every feature library that transitively depends on those apps.
- `nx affected -t lint test build` in CI runs only those projects.
- `nx graph` renders `ibc-schema-forms` as a node with edges to `ibc-tokens`, `shared-data-access`, and downstream consumers.

### A.10 Migration from the single-package scaffold

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

### A.11 Acceptance checklist for the Nx packaging

- [ ] `nx build ibc-schema-forms` emits a tree-shakeable ESM dist with `.d.ts` files.
- [ ] `nx test ibc-schema-forms` runs unit + engine-parity contract tests green.
- [ ] `nx storybook ibc-schema-forms` serves stories on :6007 with a11y addon clean.
- [ ] `nx lint ibc-schema-forms` passes with module-boundary constraints active.
- [ ] A consumer app can `import { SchemaForm } from '@ibc/schema-forms'` and `import { tanstackEngine } from '@ibc/schema-forms/engines/tanstack'` with zero barrel imports.
- [ ] Installing only one of the two form engines in the app still builds (peer meta `optional: true`).
- [ ] `nx graph` shows `ibc-schema-forms` as a shared node depended on by shell/agent/demo apps, with no inbound edges from scope-specific libs.
