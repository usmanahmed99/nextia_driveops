# 21 — Next Phase: Frontend Learner Module (Step 2)

This is the implementation plan for **Step 2** of the First Production Slice ([20-next-steps.md](./20-next-steps.md)). It closes the breaking change introduced when the API switched `GET /learners` to a paged response and exposed `/learners/{id}/workspace`, holds, and withdraw endpoints.

## Goal

Make the existing Learners list + detail pages work against the new backend, surface holds + withdraw + recommendations, and replace the placeholder "Register learner" button with a real wizard.

**Out of scope here**: B2C SPA wiring (Step 3) and the platform-admin Tenants console (Step 4). Those are separate phases.

## Why This Is The Right Next Phase

- The API ship in this slice changes the shape of `GET /api/v1/learners` from `Learner[]` to `{ items, nextCursor, pageSize }` — the deployed frontend will throw on the Learners page until this lands.
- It exercises every new backend capability end-to-end: paged list, workspace, holds, withdraw, idempotency.
- It keeps demo auth working (the frontend still calls `/auth/demo-login`), so we don't need B2C to be in place yet.

## Pre-Flight

Verify these exist after the API deploys before starting frontend work:

- `GET https://<api>/api/v1/learners?pageSize=5` returns `{ items, nextCursor, pageSize }`.
- `GET .../learners/{id}/workspace` returns the workspace shape.
- `POST .../learners/{id}/holds` accepts `{ type, reason }` and returns a `LearnerHold`.
- `POST .../learners/{id}/withdraw` accepts `{ reason }` and flips status to `withdrawn`.
- `POST /learners` with `Idempotency-Key: x` returns the same body on replay with header `Idempotency-Replay: true`.
- Suspending a tenant returns `423 { detail: { code: "tenant_suspended" } }`.

If any of these fail, fix the API first — don't paper over them in the adapter.

## Implementation Order

Do these in order. Each step is independently committable.

### 1. Contracts — [`dops-frontend/src/lib/contracts/learners.ts`](../dops-frontend/src/lib/contracts/learners.ts)

Add these types verbatim (matching the API schemas in [`dops-api/app/schemas/learners.py`](../dops-api/app/schemas/learners.py)):

```ts
export type LearnerStatus =
  | "draft_registration" | "pending_documents" | "pending_payment"
  | "ready_for_theory" | "in_theory" | "ready_for_practical" | "in_practical"
  | "ready_for_evaluation" | "road_test_ready" | "road_test_scheduled"
  | "certificate_ready" | "completed"
  | "on_hold" | "withdrawn" | "inactive" | "transferred"
  // Legacy aliases still emitted by older rows; remove after one release.
  | "active" | "behind_schedule" | "missing_documents" | "payment_due" | "waiting_for_practical";

export type LearnerHoldType = "payment" | "documents" | "behaviour" | "health" | "other";
export type LearnerHoldStatus = "active" | "resolved";

export interface LearnerHold extends TenantScopedEntity {
  learnerId: EntityId;
  type: LearnerHoldType;
  status: LearnerHoldStatus;
  reason?: string;
  createdByUserId?: EntityId;
  resolvedByUserId?: EntityId;
  resolvedAt?: ISODateTime;
}

export interface LearnerListFilters extends SortParams {
  status?: LearnerStatus;
  branchId?: EntityId;
  programId?: string;
  instructorId?: EntityId;
  language?: "en" | "fr";
  behindSchedule?: boolean;
  certificateReady?: boolean;
  paymentHold?: boolean;
  documentHold?: boolean;
  enrolledFrom?: ISODateTime;
  enrolledTo?: ISODateTime;
  search?: string;
  cursor?: string;
  pageSize?: number;
}

export interface PaginatedLearners {
  items: Learner[];
  nextCursor?: string;
  pageSize: number;
}

export interface LearnerWorkspaceUpcomingSession { /* matches API */ }
export interface LearnerWorkspaceInvoiceSummary { /* matches API */ }
export interface LearnerWorkspaceCertificateSummary { /* matches API */ }
export interface LearnerWorkspaceRecommendation {
  code: string; title: string; detail: string;
  severity: "info" | "success" | "warning" | "error";
}

export interface LearnerWorkspace {
  learner: LearnerDetail;
  upcomingSessions: LearnerWorkspaceUpcomingSession[];
  outstandingInvoices: LearnerWorkspaceInvoiceSummary[];
  certificates: LearnerWorkspaceCertificateSummary[];
  recommendations: LearnerWorkspaceRecommendation[];
}

export interface CreateLearnerHoldRequest { type: LearnerHoldType; reason?: string }
export interface ResolveLearnerHoldRequest { note?: string }
export interface WithdrawLearnerRequest { reason: string; retainDataUntil?: ISODateTime }
```

Update existing `Learner`, `LearnerDetail`, `CreateLearnerRequest`, `UpdateLearnerRequest`:

- `Learner.profile` keeps the same shape but `dateOfBirth?: string` is now exposed.
- `Learner.activeHoldTypes: LearnerHoldType[]` is new — used by the list to render hold badges without an extra fetch.
- `LearnerDetail.holds: LearnerHold[]` and `LearnerDetail.timeline: LearnerTimelineEvent[]` (timeline event now has optional `metadata`).
- `CreateLearnerRequest` adds `instructorId?`, `expectedCompletionAt?`, `requiredPracticalLessons?`.
- `UpdateLearnerRequest.status` typed to `LearnerStatus`; client must avoid sending `withdrawn`/`transferred` here (use withdraw endpoint).

### 2. Mappers — [`dops-frontend/src/lib/api/mappers/learners.ts`](../dops-frontend/src/lib/api/mappers/learners.ts)

The API now returns `profile`, `enrollment`, `progress` as nested objects (already camelCased by the global alias generator). Most of the existing mapper logic survives — the changes are:

- Read the new normalised fields off `profile`/`enrollment` (no longer scattered at the top of the JSON blob).
- New `mapLearnerHold(raw)` and `mapLearnerWorkspace(raw)` mappers.
- `mapLearner` to read `activeHoldTypes` and copy it through.
- `mapPaginatedLearners(raw): PaginatedLearners` that maps items via `mapLearner` and passes `nextCursor`/`pageSize` through.
- Status normaliser: map legacy values (`active`, `behind_schedule`, `missing_documents`, `payment_due`, `waiting_for_practical`) onto their lifecycle equivalents the same way the API's `_normalize_status` does. Keep this as `mapLearnerStatus(raw): LearnerStatus` so call sites have a single function.

### 3. HTTP adapter — [`dops-frontend/src/lib/api/adapters/httpAdapter.ts`](../dops-frontend/src/lib/api/adapters/httpAdapter.ts)

Concentrate the breaking change here.

- `listLearners(_tenantId, filters)`: build the query from `LearnerListFilters` using camelCase keys (the API accepts `pageSize`, `branchId`, `programId`, etc.); return `mapPaginatedLearners(await http.get("/learners", { query }))`.
- `getLearner(_tenantId, id)`: keep, but it now returns the new `LearnerDetail` with `timeline`, `documents`, `holds`. Drop the per-detail `documents` fetch — the API returns them inline.
- `getLearnerWorkspace(_tenantId, id)`: replace the six-call composition with a single `mapLearnerWorkspace(await http.get(`/learners/${id}/workspace`))`. Old keys (`conversations`, `invoices`, `sessions`, `certificates`) remain on the **client-shaped** workspace; map them from `outstandingInvoices`, `upcomingSessions`, `certificates`. Drop the conversation lookup — that belongs to the Automations rewrite later (Phase 3 in the roadmap).
- New methods:
  - `createLearnerHold(_tenantId, learnerId, payload)` → `mapLearnerHold(await http.post(`/learners/${learnerId}/holds`, payload, { headers: { "Idempotency-Key": newUuid() } }))`.
  - `listLearnerHolds(_tenantId, learnerId)` → `(await http.get(`/learners/${learnerId}/holds`)).map(mapLearnerHold)`.
  - `resolveLearnerHold(_tenantId, learnerId, holdId, payload)` → `mapLearnerHold(await http.patch(`/learners/${learnerId}/holds/${holdId}/resolve`, payload))`.
  - `withdrawLearner(_tenantId, learnerId, payload)` → `mapLearner(await http.post(`/learners/${learnerId}/withdraw`, payload, { headers: { "Idempotency-Key": newUuid() } }))`.
- `createLearner` & `withdrawLearner` send a fresh `Idempotency-Key` UUID per UI action. Generate with `crypto.randomUUID()`. If the request fails and the user retries, the same key is **not** reused — that's intentional. Idempotency exists to dedupe automatic retries, not manual ones.
- HTTP client error handling ([`src/lib/api/http.ts`](../dops-frontend/src/lib/api/http.ts)): when response is `423`, throw `new TenantSuspendedError(detail.status)` instead of the generic API error. Page-level boundaries handle this in step 7.

### 4. Mock adapter — [`dops-frontend/src/lib/api/adapters/mockAdapter.ts`](../dops-frontend/src/lib/api/adapters/mockAdapter.ts)

Mock mode must match the new contracts so mock-mode dev stays useful.

- `listLearners`: return `{ items, nextCursor: undefined, pageSize: filters.pageSize ?? 25 }` filtered against the mock array using the new filter set.
- `getLearnerWorkspace`: return a `LearnerWorkspace` shape backed by the mock dataset (mock has all the bits already, just need to reshape).
- New mock implementations for `createLearnerHold` / `listLearnerHolds` / `resolveLearnerHold` / `withdrawLearner` that mutate the in-memory dataset.
- Add an `activeHoldTypes` derivation on mock learners so badges work.

### 5. Hooks — [`dops-frontend/src/lib/api/hooks.ts`](../dops-frontend/src/lib/api/hooks.ts)

- `useLearners(filters)` becomes a `useInfiniteQuery`:
  ```ts
  useInfiniteQuery({
    queryKey: queryKeys.learners.list(tenantId, filters),
    queryFn: ({ pageParam }) =>
      apiClient.learnerService.listLearners(tenantId, { ...filters, cursor: pageParam as string | undefined }),
    initialPageParam: undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
  ```
- Add a thin convenience helper `useLearnersFlat(filters)` that flattens `data.pages.flatMap(p => p.items)` for the page that doesn't care about pages.
- `useLearnerWorkspace(id)` unchanged signature; under the hood now hits the new endpoint.
- New mutations: `useCreateLearner`, `useUpdateLearner`, `useWithdrawLearner`, `useCreateLearnerHold`, `useResolveLearnerHold`. Each invalidates `queryKeys.learners.list(tenantId)` and `queryKeys.learners.workspace(tenantId, learnerId)` on success.

### 6. Pages

**[`dops-frontend/src/features/learners/LearnersPage.tsx`](../dops-frontend/src/features/learners/LearnersPage.tsx)**

- Remove the `useMemo` client filter. Filters become controlled state that feeds into `useLearners(filters)`.
- Filter chips drive `filters.status` (lifecycle) + `filters.paymentHold` / `filters.documentHold` / `filters.certificateReady` / `filters.behindSchedule`. Map UI chip → typed filter; don't pass arbitrary strings.
- Search input becomes debounced (300 ms) and sets `filters.search`.
- Below the list, render a **Load more** button when `hasNextPage`; calling `fetchNextPage()` advances. Mobile: auto-load on intersection-observer trigger.
- Each list card shows `activeHoldTypes` as small red/amber chips.
- "Register learner" button opens the wizard (step 8).

**[`dops-frontend/src/features/learners/LearnerDetailPage.tsx`](../dops-frontend/src/features/learners/LearnerDetailPage.tsx)**

- Drive entirely from `useLearnerWorkspace(learnerId)`. No additional `useInvoices/useCertificates` calls.
- Replace the hard-coded "Recommended next actions" card with `workspace.recommendations` rendered via severity → tone mapping.
- New **Holds** card: list `workspace.learner.holds`, with an "Add hold" CTA (Dialog) that picks type + reason and calls `useCreateLearnerHold`. Each active hold has a "Resolve" button → `useResolveLearnerHold`.
- New **Danger zone** section at the bottom: "Withdraw learner" → confirmation modal capturing reason → `useWithdrawLearner`. Successful withdraw navigates back to `/learners` with a toast.
- The legacy WhatsApp preview block stays for now (it'll be reworked in the automations phase).

### 7. Tenant suspension boundary

**[`dops-frontend/src/components/layout/AppShell.tsx`](../dops-frontend/src/components/layout/AppShell.tsx)** + new `TenantStatusBoundary.tsx`.

- New ErrorBoundary-style component that listens for queries throwing `TenantSuspendedError` (use TanStack Query's `useQueryClient().getQueryCache().subscribe(...)` or a global `onError` on `QueryClient`).
- On detection, render a full-page state: "Your school account is `<status>` — contact support" with an "Open billing" CTA pointing to a placeholder route.
- Place the boundary as a sibling to `<Outlet />` inside `AppShell`.

### 8. Registration wizard

**New: [`dops-frontend/src/features/learners/RegisterLearnerWizard.tsx`](../dops-frontend/src/features/learners/RegisterLearnerWizard.tsx)**

Six steps, controlled with `react-hook-form` + Zod per step:

1. **Identity** — first/last name, DOB, preferred language.
2. **Contact** — email + phone with format validators. (Real OTP verification is out of scope.)
3. **Branch + Program** — branch select from `useBranches`, program select (static `Class 5 Complete Program` for now until programs are a real entity).
4. **Pickup** — pickup address, pickup notes. Stored on the learner via the existing `notes` column (or skipped if backend doesn't accept yet — currently it doesn't, so just collect and discard).
5. **Documents** — placeholder card "Upload after registration" (the real upload flow is documents Phase 5).
6. **Confirmation** — review screen, **Register** button calls `useCreateLearner` with all fields.

On success: invalidate the list query, navigate to `/learners/{id}` and toast "Learner registered".

Hook the wizard up from `LearnersPage.tsx`'s "Register learner" button (Radix Dialog or full-page route — either works; full-page route is easier to test).

### 9. Translation strings — [`dops-frontend/src/i18n/{en,fr}/common.json`](../dops-frontend/src/i18n/en/common.json)

Add keys for:
- `learners.holds.title`, `learners.holds.add`, `learners.holds.resolve`, hold-type labels per language.
- `learners.lifecycle.<status>` for each new lifecycle status (16 keys × 2 languages).
- `learners.wizard.<step>` titles + descriptions.
- `learners.danger.withdraw`, `learners.danger.confirm`.
- `tenantStatus.suspended`, `tenantStatus.cancelled`, `tenantStatus.archived` for the boundary.

Old legacy status keys (`active`, `behindSchedule`, etc.) stay until backend stops emitting them.

### 10. Verification

Before opening the PR:

```bash
cd dops-frontend
npm run lint
npm run typecheck
npm run build
```

Manual:

- Mock mode (`VITE_DATA_MODE=mock`): list, detail, holds, withdraw, wizard all work.
- API mode pointing at deployed dev API: same flows work, plus tenant suspension boundary catches a manually-suspended tenant.
- Network inspector: confirm `/learners/{id}/workspace` is one request, not six.
- Idempotency: open dev tools, force-retry the create POST — second response carries `Idempotency-Replay: true` and the same body.

## Risks / Tripwires

- **Legacy status fall-through.** Some seeded learners still have `status: in_practical` mapped from legacy. The UI's chip filter must use canonical lifecycle values; if a user filters by "in_practical" and the backend returns the same row that was previously surfaced as "behind_schedule", the row should appear under the new chip but not under the old one. Avoid showing both.
- **Cursor pagination + filter changes.** Changing a filter must reset the cursor — make the `useInfiniteQuery` `queryKey` include all filters so TanStack Query naturally invalidates.
- **`Idempotency-Key` reuse on Save-and-edit-and-Save-again.** Generate a fresh UUID per mutation invocation, not per component mount. Re-using a key with an edited payload returns `409` — make sure the mutation hook minted a new one before the second submit.
- **Workspace endpoint perf.** The backend implementation does several SELECTs in one request. Watch p95 on the deployed API; if it crosses 800 ms, the backend needs joined eager-loading. Frontend can't fix this.
- **Tenant boundary placement.** If we wrap the whole router in the boundary, the boundary itself can't render the login page (which doesn't have a tenant). Keep it scoped to `AppShell`'s child routes only.

## Estimate

- Contracts + mappers + adapters: half a day.
- Mock-adapter parity: half a day.
- Hooks + LearnersPage rewrite: 1 day.
- LearnerDetailPage rewrite + holds UI + withdraw: 1 day.
- Registration wizard: 1 day.
- Tenant boundary + i18n strings + verification: half a day.

**Total: ~4 working days** for one full-stack engineer.

## What Comes After This Phase

In priority order, with brief notes:

### Step 3 — Frontend B2C wiring (~3 days)

- Add `@azure/msal-browser` + `@azure/msal-react`.
- On app load, call `GET /auth/login-config`. If `provider === "azure_b2c"` and `demoEnabled === false`, render only the MSAL redirect button. Otherwise keep the demo buttons (dev path).
- Replace `loginAsDemoRole` as the default with `loginWithMsal()`.
- HTTP client gets a 401 interceptor that calls `acquireTokenSilent` then retries; on hard failure, redirects to login.
- Gated by infra (Step 5) being done so there's a real B2C tenant to point at — but the code can ship beforehand and stay inert.

### Step 4 — Platform-admin Tenants console (~2 days)

- New `/platform/tenants` route guarded by `platform.tenants.read`.
- Page lists tenants from `GET /platform/tenants`; row click opens a side drawer with status editor (`PATCH /platform/tenants/{id}/status`) and subscription editor.
- Tenant switcher in the app shell becomes real (X-Tenant-ID change) for users with the cross-tenant permission.

### Step 5 — Infra & secrets (~2 days)

- Bicep parameters for B2C authority/client-id/audience/JWKS URL.
- Key Vault entries: `DOPS-AZURE-B2C-CLIENT-ID`, `DOPS-AZURE-B2C-AUDIENCE`, `DOPS-AZURE-B2C-AUTHORITY`, `DOPS-AZURE-B2C-JWKS-URL`.
- API and frontend Container App env wiring.
- CORS allow-list update if B2C uses a separate sign-in subdomain.

### Phase 2 — Scheduling Engine

Real per-learner schedule generation. The largest remaining backend slice; see [06-scheduling.md](./06-scheduling.md) and [07-school-schedule-templates.md](./07-school-schedule-templates.md). The data-model groundwork (templates, programs, curricula) is the entry point.
