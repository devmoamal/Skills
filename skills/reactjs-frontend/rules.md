# React Frontend Architecture & Clean Code Rules

Apply these strict architectural standards and clean code guardrails when building, modifying, or refactoring this React frontend application.

---

## 1. Directory & Feature Structure
- Group by domain features in `src/features/[featureName]/`:
  - `api/`: TanStack Query hooks (`queries.ts`, `mutations.ts`) and direct HTTP fetchers (`endpoints.ts`).
  - `components/`: Feature-specific UI components and co-located unit/component tests (`*.test.tsx`).
  - `hooks/`: Headless custom hooks managing form state, calculations, and UI coordination (`*.test.ts` co-located).
  - `schemas/`: Zod schemas for forms, mutations, and search parameters.
  - `types/`: Feature domain types and API DTO definitions.
  - `index.ts`: Public barrel export. Outside modules MUST only import from this file.
- Global client-only state lives in `src/stores/` (Zustand stores with granular selectors).
- Shared infrastructure lives in `src/lib/`:
  - `api.ts`: Type-safe HTTP client unwrapping `{ ok: true, data: T }` envelopes and handling JWT auth.
  - `queryClient.ts`: TanStack Query client configuration and query key factories.
  - `utils.ts`: Utility helpers (e.g. `cn` class merger).
- UI Primitives live in `src/components/ui/` (bespoke semantic Tailwind components).
- End-to-end integration tests live in `tests/e2e/` parallel to `src/`.

---

## 2. The 6 Banned Anti-Patterns (Clean Code Guardrails)
1. **Zero `useEffect` for Data Fetching or Derived State**:
   - Never fetch data or manage loading/error flags with `useEffect`. Always use TanStack Query (`useQuery`, `useMutation`).
   - Compute derived state inline or wrap in `useMemo` if computationally heavy.
2. **Zero Monster Components**:
   - Ban components exceeding ~150 lines or combining rendering with business workflows.
   - Extract handlers, form coordination, and calculations into headless hooks (`hooks/use[Feature]State.ts`).
3. **Zero Private Cross-Feature Imports**:
   - Never import from another feature's internal directories.
   - All inter-feature communication must pass through the target feature's `index.ts`.
4. **Zero `any` or Unearned `@ts-ignore`**:
   - Ensure 100% strict type safety.
   - Never use `as any` or `@ts-ignore`. If a broken third-party type forces suppression, use `@ts-expect-error` with an explanatory comment.
5. **Zero Premature shadcn/ui Adoption**:
   - Strictly prohibited unless explicitly requested by the user.
   - Build clean, accessible, bespoke Tailwind CSS primitives in `components/ui/`.
6. **Zero Duplicate Server State in Client Stores**:
   - TanStack Query is the single source of truth for server state.
   - Never copy query data into Zustand or local state caches. Zustand is strictly for client-only state (modals, active drawer, sidebar, theme).

---

## 3. Server State & TanStack Query v5
- Always define query keys using the **Query Key Factory** pattern:
  ```typescript
  export const featureKeys = {
    all: ['features'] as const,
    lists: () => [...featureKeys.all, 'list'] as const,
    list: (filters: Filters) => [...featureKeys.lists(), filters] as const,
    detail: (id: string) => [...featureKeys.all, 'detail', id] as const,
  };
  ```
- Always configure `staleTime` and `gcTime` in `src/lib/queryClient.ts` to prevent redundant network requests.
- Trigger explicit cache invalidation in `onSuccess` handlers of mutations.
- Keep mutation callbacks clean: handle user feedback via `sonner` toasts (`toast.success()`, `toast.error()`).

---

## 4. TanStack Router & Routing Hygiene
- Define type-safe route trees using TanStack Router in `src/app/router.tsx`.
- Validate all route parameters and search query params using Zod schemas.
- Use route lazy loading (`.lazy.tsx`) for heavy page components to minimize bundle sizes.

---

## 5. API Client & Backend Contract Alignment
- Use `src/lib/api.ts` for all HTTP communication.
- Native digestion of the backend envelope:
  - `{ ok: true, data: T }` $\rightarrow$ Unwraps and returns `data: T`.
  - `{ ok: false, error: ... }` or HTTP $\ge$ 400 $\rightarrow$ Throws typed `ApiClientError`.
- Automatically injects `Authorization: Bearer <token>` from `useAuthStore`.

---

## 6. Testing & Quality Standards
- **Co-located Unit & Component Tests**:
  - Test files must sit directly alongside the file under test (e.g. `userCard.test.tsx` next to `userCard.tsx`).
  - Use Vitest and `@testing-library/react`.
  - Test user interactions, accessible labels, validation errors, and conditional renders.
- **E2E Tests in `tests/e2e/`**:
  - Keep end-to-end integration flows (auth journey, multi-step checkout, settings) in root `tests/e2e/`.

---

## 7. UI, Icons & Accessibility
- **Icons**: Exclusively use `lucide-react`. Maintain consistent icon dimensions (`size-4`, `size-5`) and stroke width (`strokeWidth={1.75}`).
- **Notifications**: Trigger feedback with `sonner` toasts.
- **Accessibility**: Enforce semantic HTML (`<button>`, `<dialog>`, `<nav>`), keyboard navigation handlers (`onKeyDown`), and ARIA attributes (`aria-expanded`, `aria-label`).
