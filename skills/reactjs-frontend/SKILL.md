---
name: reactjs-frontend
description: Enforce high-performance, modular, and type-safe frontend architecture for React using Vite, TypeScript, Tailwind CSS, TanStack Query & Router, Zustand, Zod, Lucide React, Sonner, and Vitest. Use whenever designing, building, scaffolding, refactoring, or auditing React client applications, feature slices, components, state stores, or data-fetching layers.
---

# React Frontend: Production Architecture & Clean Code Standards

Build high-performance, scalable, modular, and bulletproof frontend applications powered by React 19, Vite, TypeScript, Tailwind CSS, TanStack Query v5, TanStack Router, Zustand, Zod, and Vitest.

```
Route (TanStack Router + Zod Search Params)
      ↓
Page View (src/app/routes/...)
      ↓
Feature Slice (src/features/[featureName]/)
┌─────────────────────────────────────────────────────────────┐
│  Component (Presentational UI + Lucide + Sonner)            │
│       ↓ (calls)                                             │
│  Custom Hook (Headless Logic + Handlers + Form State)       │
│       ↓ (calls)                                             │
│  TanStack Query (useQuery / useMutation + QueryKey Factory) │
│       ↓ (fetches)                                           │
│  API Client (src/lib/api.ts unwraps { ok, data } envelope)  │
└─────────────────────────────────────────────────────────────┘
      │ (global client state only)
      ▼
Zustand Store (src/stores/ — Auth / UI Modals)
```

Every component, custom hook, query, store, and route must adhere to uncompromising type safety, strict separation of concerns, and clean production hygiene.

---

## Operating Modes

### 1. Build Mode (Default)
When scaffolding new features, components, or hooks:
* Structure all domain features under `src/features/[featureName]/` with dedicated `api/`, `components/`, `hooks/`, `schemas/`, `types/`, and an `index.ts` public barrel export.
* Strictly enforce the separation of concerns: **Presentational Components $\rightarrow$ Headless Hooks $\rightarrow$ API / Query Hooks**. Never write raw business logic or inline data fetching inside JSX components.
* Route definitions live in `src/app/routes/` and are orchestrated by TanStack Router via `src/app/router.tsx`. Validate route parameters and search query params using Zod.
* Consume API endpoints exclusively through TanStack Query hooks (`queries.ts`, `mutations.ts`). Components never invoke `fetch` directly.
* Handle backend response envelopes (`{ ok: true, data: T }` / `{ ok: false, error: ... }`) automatically in `src/lib/api.ts`, returning typed payload data or throwing structured `ApiClientError`.
* **Zero `any`**: Ensure 100% type safety. `@ts-expect-error` is permitted only when strictly unavoidable, accompanied by an explanatory inline comment.
* **Component Testing Rule**: When a feature component, hook, or utility involves non-trivial branching, calculation, or user interaction, co-locate an automated test beside it (`[name].test.tsx` / `[name].test.ts`).
* **shadcn/ui Gate**: **STRICTLY PROHIBITED** by default. Do NOT install, generate, or import shadcn/ui components unless the user explicitly requests it. By default, build clean, semantic, accessible Tailwind CSS primitives in `src/components/ui/`.
* Provide visual feedback for asynchronous operations using `sonner` toasts and accessible loading skeletons.

### 2. Audit / Review Mode
When reviewing, refactoring, or debugging existing frontend code:
* **Gate 1 (Zero `useEffect` Fetching):** Flag and eliminate manual `fetch` or `axios` calls inside `useEffect`. Enforce TanStack Query (`useQuery` / `useMutation`).
* **Gate 2 (Component Cleanliness & Line Limit):** Ensure components remain focused on rendering and UI states. If a component exceeds 150 lines or contains inline complex state transformations, extract headless logic into a custom hook in `hooks/`.
* **Gate 3 (Strict Feature Encapsulation):** Verify that code outside `src/features/[featureName]/` only imports from that feature's root `index.ts`. Ban deep internal imports (e.g. `@/features/billing/components/internalWidget`).
* **Gate 4 (Type Safety & TS Hygiene):** Flag `any`, untyped props, unhandled nullable values, or `@ts-ignore`. Verify `@ts-expect-error` has clear rationale.
* **Gate 5 (State Hygiene & Re-render Prevention):** Ensure server state is never duplicated into Zustand or local `useState`. Verify Zustand stores use granular selectors (`useStore(s => s.item)`) to prevent unnecessary re-renders.
* **Gate 6 (Accessibility & Fallbacks):** Verify all interactive elements support keyboard navigation, ARIA attributes, and asynchronous views define loading skeletons and error boundaries.

---

## Standard Directory Layout

```text
my-react-frontend/
├── public/                            # Static assets (favicons, robots.txt)
├── index.html                         # HTML root entry
├── vite.config.ts                     # Vite config & path aliases (@/ -> src/)
├── tsconfig.json                      # Strict TypeScript compiler options
├── package.json
├── .env.example                       # Environment template (VITE_API_BASE_URL, etc.)
│
├── tests/                             # End-to-End test suites
│   └── e2e/                           # Playwright / Vitest E2E integration specs
│       └── auth.spec.ts
│
└── src/
    ├── main.tsx                       # Entry point (DOM mount & global styles)
    │
    ├── app/                           # Application core & routing
    │   ├── App.tsx                    # Root application component
    │   ├── providers.tsx              # Consolidated providers (QueryClient, Toaster, Router)
    │   ├── router.tsx                 # TanStack Router instance & route tree export
    │   └── routes/                    # Route view components
    │       ├── index.tsx              # Home / Dashboard route
    │       ├── notFound.tsx           # 404 handler route
    │       └── ...
    │
    ├── config/                        # Runtime configurations
    │   ├── env.config.ts              # Fail-fast Zod validation for import.meta.env
    │   └── constants.ts               # Immutable constants, pagination defaults, limits
    │
    ├── lib/                           # Shared infrastructure singletons
    │   ├── api.ts                     # Type-safe HTTP client unwrapping { ok, data } envelopes
    │   ├── queryClient.ts             # TanStack Query client & query key factory utilities
    │   └── utils.ts                   # Class merging (cn) & common formatting helpers
    │
    ├── types/                         # Global ambient & API envelope contracts
    │   └── api.types.ts               # ApiResponse<T>, ApiError, PaginatedData<T>
    │
    ├── stores/                        # Global client-only state (Zustand)
    │   ├── auth.store.ts              # Session tokens, current user profile, auth status
    │   └── ui.store.ts                # Sidebar collapse, active modals, global drawer
    │
    ├── hooks/                         # Cross-cutting utility hooks
    │   ├── useDebounce.ts             # Value / function debounce hook
    │   ├── useMediaQuery.ts           # Responsive screen breakpoint listener
    │   └── useDisclosure.ts           # Modal / drawer open/close boolean controller
    │
    ├── components/                    # Shared presentation components
    │   ├── ui/                        # Atomic primitives (Button, Input, Dialog, Badge)
    │   │   ├── button.tsx
    │   │   ├── input.tsx
    │   │   └── dialog.tsx
    │   ├── feedback/                  # ErrorBoundary, EmptyState, LoadingSkeleton
    │   │   ├── errorBoundary.tsx
    │   │   ├── emptyState.tsx
    │   │   └── loadingSkeleton.tsx
    │   └── layout/                    # Shell layout (Header, Sidebar, PageContainer)
    │       ├── header.tsx
    │       ├── sidebar.tsx
    │       └── pageContainer.tsx
    │
    └── features/                      # Domain Vertical Slices (Self-contained)
        └── [featureName]/             # e.g., users, auth, billing, products
            ├── index.ts               # Public barrel export (only export what's needed)
            │
            ├── api/                   # Server communication (TanStack Query)
            │   ├── endpoints.ts       # Raw HTTP fetchers calling lib/api.ts
            │   ├── queries.ts         # useFeatureQuery, queryOptions
            │   └── mutations.ts       # useCreateFeatureMutation, optimistic updates
            │
            ├── components/            # Feature-specific UI components
            │   ├── featureList.tsx
            │   ├── featureCard.tsx
            │   ├── featureCard.test.tsx # Co-located component test (Vitest)
            │   └── featureForm.tsx
            │
            ├── hooks/                 # Headless business logic & form handlers
            │   ├── useFeatureState.ts
            │   └── useFeatureState.test.ts # Co-located hook test (Vitest)
            │
            ├── schemas/               # Zod schemas for forms and payload validation
            │   └── feature.schema.ts
            │
            └── types/                 # Feature-specific domain types & DTOs
                └── feature.types.ts
```

---

## The 6 Banned Frontend Anti-Patterns (Clean Code Guardrails)

Reject these patterns unconditionally in any codebase:

### 1. Zero `useEffect` for Data Fetching or Derived State
* Never fetch data or store loading/error flags manually with `useEffect` + `useState`.
* Always use **TanStack Query** (`useQuery`, `useMutation`).
* Never use `useEffect` to sync props to state or compute derived data—compute derived values inline or with `useMemo` if computationally intensive.

### 2. Zero Monster Components
* Ban monster components (exceeding ~150 lines or combining rendering, state machines, and calculations).
* Keep components strictly presentational: extract form handling, calculation logic, and side effects into headless custom hooks inside `hooks/use[Feature]State.ts`.

### 3. Zero Private Cross-Feature Imports
* Never import from another feature's internal directory (e.g. `import { helper } from '@/features/billing/components/card'`).
* All inter-feature communication must pass through the public barrel export: `import { ... } from '@/features/billing'`. Anything not in `index.ts` is private to that feature.

### 4. Zero `any` or Unearned `@ts-ignore`
* Never use `any`, `as any`, or loose casting to silence TypeScript errors.
* Never use `@ts-ignore`. If a third-party library has broken types and suppression is strictly necessary, use `@ts-expect-error` accompanied by a comment explaining why.

### 5. Zero Premature shadcn/ui Adoption
* Never install or copy-paste shadcn/ui components unless the user explicitly requests shadcn/ui.
* Build semantic, accessible, bespoke Tailwind CSS primitives in `components/ui/` with clean ARIA attributes and keyboard navigation.

### 6. Zero Duplicate Server State in Client Stores
* Never store API query results in Zustand stores or local state caches.
* TanStack Query is the single source of truth for all server state. Zustand is strictly reserved for client-only state (e.g., active modal IDs, sidebar toggle, draft filters, theme).

---

## The 8 Core Engineering Pillars

### Pillar 1: Vertical Feature Slices & Barrel Encapsulation
Every domain capability lives in its own directory under `src/features/[featureName]/`.
* **Self-Contained**: Slices own their API layer, UI components, custom hooks, Zod validation, and TypeScript types.
* **Controlled Surface Area**: The `index.ts` file acts as the public contract of the feature. Only export components, hooks, or types intended for use by routes or other features.

```typescript
// src/features/users/index.ts
export { UserList } from './components/userList';
export { UserCard } from './components/userCard';
export { useUserState } from './hooks/useUserState';
export type { User, UserFilterParams } from './types/user.types';
```

### Pillar 2: Headless Logic Separation via Custom Hooks
Components must remain clean, declarative, and focused solely on visual output.
* Move event handlers, form coordination, modal states, and mutation triggering into `hooks/use[Feature]State.ts`.
* The component simply calls the hook and renders:

```tsx
// src/features/users/components/userForm.tsx
export function UserForm() {
  const { form, onSubmit, isSubmitting } = useUserState();

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
      <input {...form.register('name')} className="..." />
      {form.formState.errors.name && (
        <p className="text-xs text-rose-500">{form.formState.errors.name.message}</p>
      )}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save User'}
      </button>
    </form>
  );
}
```

### Pillar 3: Type-Safe Routing with TanStack Router
Use TanStack Router for 100% type-safe routing, search params validation, and route preloading.
* Parse and validate search parameters using Zod directly in route definitions.
* Code-split large page views using lazy route modules to maintain lightning-fast initial load times.

### Pillar 4: TanStack Query v5 Server State & Query Key Factories
* **Query Key Factories**: Standardize query keys in `api/queries.ts` or `lib/queryClient.ts` to prevent cache desynchronization.
* **Cache Invalidation**: Trigger explicit invalidation in `onSuccess` handlers of mutations.
* **Stale Time**: Configure a default `staleTime` (e.g., 1 minute) in `queryClient.ts` to avoid redundant network spam.

```typescript
// Query key factory pattern
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilterParams) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};
```

### Pillar 5: Envelope-Aware API Client & Unified Errors
`src/lib/api.ts` provides a typed wrapper over native `fetch` that natively digests the standard backend envelope (`{ ok: true, data: T }` and `{ ok: false, error: ... }`):
* Automatically unwraps `response.data` on `ok: true`.
* Automatically attaches Authorization Bearer tokens from `auth.store.ts`.
* Automatically throws a structured `ApiClientError` containing status code, error code, and error messages when `ok: false` or HTTP status >= 400.

### Pillar 6: Atomic Zustand Stores with Granular Selectors
Client-only global state is managed with Zustand in `src/stores/`.
* **Granular Selectors**: Always select state slices specifically (`const user = useAuthStore(s => s.user)`) rather than pulling the entire store, preventing unnecessary component re-renders.
* **Actions in Store**: Colocate action methods inside the store definition for clean mutations.

### Pillar 7: Co-located Vitest Testing & E2E Strategy
* **Co-located Unit & Component Tests**: Place test files right next to the code under test (e.g., `userCard.test.tsx` next to `userCard.tsx`).
* **Test What Matters**: Test user interactions, form validation error rendering, conditional display logic, and custom hook state transitions.
* **E2E in `tests/e2e/`**: Maintain critical journey workflows (authentication, checkout, CRUD pipelines) inside the top-level `tests/e2e/` folder.

### Pillar 8: Semantic Tailwind UI, Accessibility & Feedback
* **Lucide Icons**: Standardize on `lucide-react` for iconography. Maintain consistent `size` (e.g. `size-4`, `size-5`) and `strokeWidth={1.75}` across the UI.
* **Sonner Toasts**: Trigger accessible notifications via `toast.success()` and `toast.error()` inside mutation callbacks.
* **Semantic Accessibility**: Use proper HTML tags (`<button>`, `<dialog>`, `<nav>`), keyboard navigation (`tabIndex`, `onKeyDown`), and ARIA labels (`aria-expanded`, `aria-describedby`).

---

## Production Quality Checklist

Before finalizing any frontend task, verify against this checklist:
- [ ] **Feature Isolation:** Feature code strictly lives inside `src/features/[featureName]/`.
- [ ] **Public Barrel:** Outer layers only import from `features/[featureName]/index.ts`. No deep internal imports.
- [ ] **Headless Separation:** Complex logic and handlers extracted to `hooks/use[Feature]State.ts`. Components stay under ~150 lines.
- [ ] **Type Safety:** 100% strict TypeScript. Zero `any`. Zero unearned `@ts-ignore` (only `@ts-expect-error` with comment).
- [ ] **Zero `useEffect` Fetching:** All server state managed with TanStack Query v5.
- [ ] **Query Key Factories:** All queries use structured query key factories to avoid cache bugs.
- [ ] **Router & Zod:** Route paths and search parameters defined with TanStack Router and validated with Zod.
- [ ] **Zustand Discipline:** Zustand only holds client state (modals, auth session, sidebar). Never mirror server data.
- [ ] **shadcn/ui Policy:** Never used unless explicitly requested by the user. Custom semantic Tailwind primitives used by default.
- [ ] **Co-located Testing:** Co-located tests (`*.test.tsx` / `*.test.ts`) written beside components or hooks with non-trivial logic.
- [ ] **E2E Tests:** Critical end-to-end flows placed in root `tests/e2e/`.

---

## Supporting Resources
* **[Production Reference Templates](./references/templates.md)**: Full copy-pasteable implementations of the API client, Query setup, feature vertical slice, Zustand store, and tests.
* **[Project Rules](./rules.md)**: Portable rules to drop into `.cursorrules`, `.windsurfrules`, or workspace configurations.

