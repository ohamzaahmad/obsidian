is_verfied -> employees
roles, verfied, tracking, teams and search by name, email or department
-------------------------------------------------
add the drag and drop in the workflow creation




# ORTS Frontend V2 — Full Architecture Audit

> React 19 · Vite · Tailwind 3 · TypeScript · No state management library · No data-fetching library

---

## 🔹 Sub-Part Analysis

---

### 📁 Part 1: `App.tsx` — Router + Auth Shell

**🔍 Issues:**
- **Hand-rolled string router** (`ViewState` union type) instead of React Router. Every new page requires a new union variant, `case` in `handleNavigation`, and a conditional render block.
- **13+ useState calls** at the root component managing: navigation signals, dept filters, user edit targets, nav signal counters, OTP email, session state. This is the app's entire "store".
- **`sessionStorage` used as a message bus** between components (`dept-requests:initial-status`, `manage-dept:requests:filters`, `viewRequestId`, `viewRequestFrom`, `sidebar:selectedWorkflow`, `sidebar:openPopoverAfterNav`). This is extremely fragile and untestable.
- **`window.dispatchEvent(CustomEvent)` used for cross-component communication** (line 238-244). This is an anti-pattern — components shouldn't communicate via DOM events.
- **`setTimeout(() => {...}, 0)` inside navigation** (Layout.tsx L245-248). Used to defer navigation to avoid "update during render" errors — signals a design flaw, not a fix.
- **`initialMode` + `requestsNavSignal`** pair: two redundant mechanisms to do the same thing (force Requests to reset). Indicates the router/state coupling is broken.
- `handleLoginResponse` and `handleOtpVerify` contain identical async IIFE patterns — duplicated code.
- `APP:profile` and `APP:notifications` reuse `<Dashboard>` as a shell wrapper with children injected — mixing routing with layout.

**⚠️ Risks:**
- Breaking change to any page requires touching `App.tsx`, `handleNavigation`, and the `ViewState` type — high coupling.
- `sessionStorage` keys are magic strings — typo = silent bug, impossible to trace in dev tools.
- Custom events create invisible dependencies between unrelated files.

**✅ Optimizations:**
- Replace with React Router v6 (`createBrowserRouter` + `<Outlet>` layouts).
- Replace `sessionStorage` messaging with URL params (`?status=pending&dept=X`) or a tiny Zustand store.
- Deduplicate login/OTP async handlers into a single `handleAuthSuccess` function.

**💡 Refactor:**
```typescript
// Before: 13 useState + sessionStorage bus
// After: React Router URL is the source of truth
<Route path="/requests" element={<Requests />}>
  <Route path=":id" element={<RequestDetails />} />
</Route>
// Filter state lives in URL: /requests?status=pending&workflow=3
```

---

### 📁 Part 2: `pages/Requests.tsx` — Monolith (2,394 lines, 146 KB)

**🔍 Issues:**

**State explosion (28+ useState at top level):**
- `viewMode`, `isLoading`, `tickets`, `statusFilter`, `workflowFilter`, `workflowFilterId`, `selectedWorkflowFilterId`, `priorityFilter`, `assignedFilter`, `departmentFilter`, `searchInput`, `search`, `createdAfter`, `createdBefore`, `historyUser`, `datePickerOpen`, `filtersOpen`, `assignedCompleted`, `showEscalatedOnly`, `workflows`, `createWorkflows`, `selectedWorkflowId`, `selectedWorkflowDetails`, `isLoadingSelectedWorkflow`, `formData`, `isSubmitting`, `allowComments`, `filePreviewUrls`, `assignableUsers`, `providers`, `billingSources`, `assignedToUsers`, `isLoadingAssignedToUsers`, `firstStageDeptName`, `multiSelectOpen`, `multiSearch`, `selectedTicket`, `detailWorkflow`, `historyOpen`, `historyEntries`, `historyTitle`, `isEditingDetails`, `isClosing`, `editedDetails`, `scope`.

**10+ useEffect calls:**
1. Initial load (workflows + dropdowns)
2. Session deep-link navigation check
3. Initial mode sync
4. Nav signal re-apply initial mode (duplicate of #3)
5. `manage-dept-filters-changed` window event listener
6. Navigation path filter reset
7. Search debounce (300ms)
8. **Main data-fetch effect** — over-specified dep array that will cascade re-renders
9. Filter-change → page-reset effect (interacts with effect #8 = double-render)
10. Detail workflow fetch
11. Selected workflow fetch (for create form)
12. File preview URL generation / cleanup
13. Click-outside for multi-selects
14. `refreshRef` sync
15. Keyboard shortcut F5/Ctrl+R
16. Escape cancel edit

**Dual client+server filtering (line 732-747):** `filteredTickets` performs client-side filtering on server-paginated data. The server is already filtering via params — this produces incorrect counts.

**`isFetchingRef` as a mutex:** Using a ref to prevent overlapping fetches is a pattern usually handled by AbortController alone. The ref can get stuck to `true` if any code path throws before resetting it.

**`loadData` is a `useCallback` with 7 deps** but reads `search`, `departmentFilter`, `createdAfter`, `createdBefore`, `historyUser` from closure. If `search` changes, `loadData` rebuilds, which triggers the fetch effect, which calls `loadData` — creating an infinite loop risk mitigated only by `isFetchingRef`.

**`pendingWorkflowFilterRef`** is set/read across two different effects — invisible coupling that's impossible to test.

**⚠️ Risks:**
- Effect cascade: filter change → page reset (effect A) → data fetch (effect B) = two fetches per filter change.
- `isFetchingRef.current = false` in every `catch` branch — any missed path leaves the component locked.
- File preview `URL.createObjectURL` leaks if the cleanup effect runs before URLs are created (async race).
- 2,394 lines makes debugging extremely difficult — any engineer editing this will break something unrelated.

**✅ Optimizations:**
- Extract to 4+ smaller components: `<RequestsList>`, `<RequestFilters>`, `<CreateRequestForm>`, `<RequestDetailView>`.
- Replace all data-fetching with React Query (`useQuery`/`useMutation`) — eliminates ~60% of the effect code.
- Replace filter state with URL search params + a single `useFilterState` hook.
- Remove client-side `filteredTickets` — trust the server.

**💡 Refactor:**
```typescript
// Target structure
<Requests>
  ├── <RequestsFilters />         // Filter bar state only
  ├── <RequestsList />            // Table + pagination
  ├── <RequestCreateDrawer />     // Slide-over form
  └── <RequestDetailPage />       // Separate route
```
Each gets ~200-300 lines. `useRequests()` hook handles fetching.

---

### 📁 Part 3: `components/requests/RequestDetails.tsx` — Presentation Monolith (1,507 lines, 101 KB)

**🔍 Issues:**
- **Mixed concerns**: stage navigation, issue fetching, assignee modal, file lightbox, edit mode, sub-request creation, escalation — all in one component.
- **15+ useState** including: `issues`, `issuesLoading`, `assignModalOpen`, `assignMode`, `selectedAssignees`, `moveLoading`, `closeLoading`, `reopenLoading`, `completeLoading`, `modalUsers`, `modalUsersLoading`, `targetDeptName`, `editModeUsers`, `isLoadingEditUsers`, `lightboxUrl`, `lightboxLoading`, `lightboxError`, `lightboxZoomed`, `escalateOpen`.
- **Business logic in render**: `openAssignModal` triggers an API call, `confirmMove` contains the full stage-transition logic.
- **Inline `resolveSingle` function at line ~759** (inside `.map()` in render JSX) — recreated on every render for every field.
- **`idxNext`, `idxPrev`, `effectiveIndex` computed inline in render** without `useMemo`.
- `api.request()` called directly with a raw path string (`/requests/${id}/move_to_next_stage/`) — bypasses the typed API layer.
- Props interface has 20 fields — component consumed via heavy prop drilling from `Requests.tsx`.

**⚠️ Risks:**
- Performance: large re-renders when any of the 15+ states change, with inline expensive computations.
- `resolveSingle` runs linear search over `users`, `providers`, `billingSources` arrays for every field on every render.
- Any change to stage transition logic requires editing a 1,507-line file.

**✅ Optimizations:**
- `useMemo` on `sortedStages`, `effectiveIndex`, `idxNext`, `idxPrev`, `completionPercent`.
- Extract `resolveSingle` to a pure utility (already partially done in `requestUtils.ts`).
- Extract `<StageNavigationModal>`, `<AttachmentLightbox>`, `<IssuesList>` as independent components.
- Use `useCallback` for `openAssignModal`, `confirmMove`, `toggleAssignee`.

**💡 Refactor:**
```typescript
// Extract hook
function useRequestActions(ticket, onRefresh) {
  // confirmMove, handleClose, handleReopen, handleComplete
}
// Extract components
<StageNavigation ticket={ticket} workflow={workflow} onMove={...} />
<AttachmentViewer fields={fields} />
```

---

### 📁 Part 4: `services/api.ts` — API Layer (1,018 lines)

**🔍 Issues:**
- **Duplicated query-string builder** (copy-pasted ~7 times across `getRequests`, `getCreatedByMeRequestsWithParams`, `getAssignedToMeRequestsWithParams`, `getRecentCreatedRequests`, `getAuditLogs`, etc.). The pattern `params → URLSearchParams → qs → path` is in every function.
- **FormData construction duplicated** between `createRequest`, `updateRequest`, `createUser`, `updateUser` — identical loops for file/array/boolean handling.
- **In-flight deduplication only for GET** (`inFlightGetRequests` Map) — deduplication key uses the raw URL, not a semantic cache key.
- **`fetchTickets()` alias** (line 267) — exists only for backwards compat. Should be removed.
- `getMyIssues`, `getCreatedIssues`, `getIssues` use a different query-string builder pattern than `getRequests` — inconsistency.
- `exportICD10Codes` / `exportProvidersAdmin` bypass the `request()` helper and duplicate auth header logic.
- **Default export `api` object** alongside named exports — consumers use both patterns inconsistently.

**⚠️ Risks:**
- Any change to query-string handling (e.g., empty string filtering) must be made in 7+ places.
- The deduplication map never has a TTL — a stuck request will block all subsequent identical GET calls permanently (until navigation).

**✅ Optimizations:**
- Extract `buildQueryString(params)` utility.
- Extract `buildFormData(payload)` utility.
- Replace with React Query — eliminates all manual caching, deduplication, loading states.
- Remove `fetchTickets` alias.
- Centralize export download logic in one `downloadFile(endpoint)` helper.

**💡 Refactor:**
```typescript
function buildQueryString(params: Record<string, any>): string { ... }
function buildFormData(payload: Record<string, any>): FormData { ... }

// getRequests becomes:
export function useRequests(params) {
  return useQuery(['requests', params], () => request(`requests/?${buildQueryString(params)}`));
}
```

---

### 📁 Part 5: `hooks/useNotifications.ts` — Polling Hook

**🔍 Issues:**
- **`fetchUnreadCount` depends on `isPanelOpen`** (line 96 dep array), which means the function reference changes every time the panel opens/closes → the polling `useEffect` (dep: `fetchUnreadCount`) re-subscribes, clearing and resetting the interval on every panel toggle.
- **`abortControllerRef` is misused**: A new `AbortController` is created on every `fetchUnreadCount` call but nothing is actually passed as a `signal` to `api.getUnreadNotificationCount()` — so abort does nothing.
- **Sound file hardcoded** to `/sounds/notification.mp3` — no configuration option.
- `totalNotifications` is exposed from state but `useNotifications.ts` doesn't export it (line 241-252 return object). The state exists but is unused externally.

**⚠️ Risks:**
- Panel open/close → interval reset → one missed poll tick per toggle.
- Memory: `Audio` element created every mount; `removeEventListener('canplay')` after `audioRef.current = null` still has a reference — works by coincidence.

**✅ Optimizations:**
- Separate `isPanelOpen` from the fetch logic — `fetchUnreadCount` should have no dependencies.
- Use React Query's `useQuery` with `refetchInterval` instead of manual `setInterval`.
- Actually pass the `AbortSignal` to the API call, or remove the dead code.

---

### 📁 Part 6: `hooks/usePagination.ts` — Pagination Hook ✅

**Assessment: Well-designed.** Proper use of `useMemo`, `useCallback`. `externalTotal` vs `internalTotal` split is clean. No issues found.

**Minor note:** `reset()` calls `onPageChange?.(1, limit)` but doesn't reset `internalTotal` — minor semantic gap.

---

### 📁 Part 7: `utils/requestUtils.ts` — Utilities (724 lines) ✅

**Assessment: Mostly good.** Properly centralized. `normalizeTicket`, `normalizeDetailsForEdit`, `normalizeOptions` are real value.

**Issues:**
- `normalizeTicket` creates `new Date()` instances and calls `.toLocaleString()` — expensive if called on every row in a 100-item list on every render.
- `getBackendBase()` is defined inline calling `import.meta.env` on every call — should be a module-level constant.
- `normalizeStatus` / `normalizePriority` use `.includes()` substring matching → `'incomplete'` would match `'complete'`.

---

### 📁 Part 8: `components/Layout.tsx` — Sidebar (1,271 lines)

**🔍 Issues:**
- **1,271 lines for a sidebar** — mobile sidebar, desktop sidebar, collapsed mode, popovers, keyboard shortcuts, profile menu, managed-dept menu all in one component.
- `setTimeout(() => {...}, 0)` inside `toggleMenu` (line 245-248) to defer navigation — architectural smell.
- `useEffect` fires on `[collapsedPopover, collapsedProfilePopover]` to run `document.addEventListener` — this adds/removes DOM listeners on every popover state change.
- Desktop and mobile sidebars duplicate ~300 lines of nav item rendering logic.
- `workflowFilters` prop passed from `Requests.tsx` all the way to `Layout` — that's prop drilling through pages.

**✅ Optimizations:**
- Split into `<Sidebar>`, `<MobileSidebar>`, `<NavItem>`, `<WorkflowSubmenu>`, `<ProfileMenu>`.
- Pass `workflowFilters` via Context instead of prop drilling.
- Move keyboard shortcut logic to a `useKeyboardShortcuts` hook.

---

### 📁 Part 9: `types.ts` — Type System

**🔍 Issues:**
- **`RequestTicket.status` is typed as `any`** (line 77). The most important field in a "request tracking system" has no type safety.
- **`RequestTicket.priority` is typed as `any`** (line 78).
- `RequestTicket` has duplicate camelCase/snake_case pairs: `current_stage` + `currentField`, `next_stage` + `nextStage`, `assignedTo` + `assigned_to`, `closedAt` + `closed_date` etc. — 8+ aliased fields.
- `User.role` is a discriminated union of `string | object | null` — callers must do defensive `typeof` checks everywhere (evidenced by `roleUtils.ts`).
- Heavy use of `any[]` for `activityLog`, `comments`, `all_stages` — no benefit from TypeScript.

**✅ Optimizations:**
```typescript
// Before
status: any;
// After
type RequestStatus = 'pending' | 'in process' | 'completed' | 'rejected' | 'closed';
status: RequestStatus;
```
- Normalize to snake_case only (matches backend) and use a mapper at the boundary.
- Type `Comment`, `ActivityLogEntry` properly.

---

## 🔸 Chunk Summary — Repeated Anti-Patterns

| Anti-Pattern | Locations |
|---|---|
| `any` type annotations | `types.ts` (status, priority, activityLog, allStages), `api.ts` (updateRequest payload), `RequestDetails.tsx` (issues state) |
| `sessionStorage` as IPC bus | `App.tsx`, `Layout.tsx`, `Requests.tsx` (5+ keys) |
| `window.dispatchEvent(CustomEvent)` for state | `App.tsx` → `Requests.tsx` |
| `setTimeout(..., 0)` to escape re-render | `Layout.tsx` (toggleMenu), `App.tsx` (filter dispatch) |
| Duplicated FormData construction | `api.ts` (createRequest, updateRequest, createUser, updateUser) |
| Duplicated query-string builder | `api.ts` (7+ functions) |
| Functions defined inside render / `.map()` | `Requests.tsx` (`fmt`), `RequestDetails.tsx` (`resolveSingle`) |
| `useEffect` dep array over-specification causing cascades | `Requests.tsx` (effects 8+9 double-fetch) |
| Missing `useMemo` on expensive inline computations | `RequestDetails.tsx` (stage indices, completion%) |
| Missing `React.memo` on list items | `RequestsList.tsx`, `RequestDetails` field rows |

---

## 🎨 UI/UX Review (First Chunk)

### Workflow Clarity
- **Positive**: Stage progress bar with percentage and color coding is clear.
- **Negative**: Status values are raw strings from backend (`in process`, `in_process`, `In Process`) that normalize inconsistently — users may see different capitalizations across views.
- **Negative**: The `__fromAssigned` internal flag (a runtime duck-type annotation on objects) controls which UI buttons appear for a user. This is fragile and invisible to TypeScript.

### Loading & Feedback
- **Positive**: Skeleton loader (`TableSkeleton`) used on list views.
- **Negative**: No global error boundary shown in the component tree — a thrown error in any sub-component will crash the full app.
- **Negative**: `setIsLoading(false)` must be called in every catch/finally branch manually — inconsistent handling (e.g., `loadData` in Requests.tsx has 3 different paths resetting loading state).
- **Negative**: Toast notifications are fire-and-forget — there's no toast queue deduplication (clicking "Refresh" 3x = 3 identical error toasts).

### Form Usability
- **Positive**: Dynamic fields (`DynamicField.tsx`) with field-type-aware rendering (user pickers, ICD10, file uploads) is impressive.
- **Negative**: Form validation errors are not displayed inline — the form only shows toast on submit failure.
- **Negative**: Create form resets `formData: {}` when the workflow changes (correct) but doesn't focus the first field (UX gap).
- **Negative**: Multi-select dropdowns (`multiSelectOpen` state per field) are managed in the parent page component with per-key object state — this causes the entire 2,394-line component to re-render on every dropdown toggle.

### Accessibility
- Most interactive elements use `<button>` (correct) but lack `aria-expanded`, `aria-controls` on dropdowns.
- No `role="alert"` on toast messages.
- No skip-navigation link.
- Dark sidebar has sufficient contrast.

### Responsiveness
- Tailwind responsive prefixes (`sm:`, `lg:`, `md:`) are used consistently — good.
- Mobile sidebar exists — good.
- Detail view `col-span-12 lg:col-span-8` grid is responsive — good.

---

## 📊 Final Verdict

---

### ⚛️ Frontend Architecture Verdict

> **The codebase is functionally complete but architecturally overloaded.** The absence of a router, state management library, and data-fetching library has led to all three being hand-implemented inside page components, resulting in files that are 10-30× larger than they should be. The system currently works but is at the edge of maintainability — adding one more request type or workflow state will push key files past 3,000 lines.

---

### 🚨 Critical Problems

1. **No router** — `App.tsx` is a 395-line switch statement masquerading as a router. URL doesn't reflect app state. Browser back/forward don't work. Deep links require sessionStorage hacks.
2. **`Requests.tsx` is 2,394 lines** — single file contains list view, create form, detail view, filter state, pagination state, API calls, keyboard handlers, and scroll restoration. Untestable as a unit.
3. **`sessionStorage` + `CustomEvent` as IPC** — produces invisible, cross-file state dependencies with no type safety and no debugging support.
4. **`status: any` and `priority: any`** — the two most important fields in the domain model are untyped.
5. **No error boundaries** — any unhandled runtime error crashes the entire SPA.
6. **Double-fetch on filter change** — the `page-reset` effect triggers after the `data-fetch` effect both react to the same filter state changes.

---

### ⚡ Performance Bottlenecks

1. **`filteredTickets` (client-side filter on server-paginated data)** — does nothing useful but runs on every render.
2. **`resolveSingle` inside `.map()` in render** — linear search over `users/providers/billingSources` for every custom field on every render of `RequestDetails`. For a ticket with 10 fields and 100 users, this is 1,000 comparisons per render.
3. **`normalizeTicket` creating `Date` objects** on every row in the list (every pagination + filter change).
4. **`useNotifications` polling interval resets** on every panel open/close — worst case: user opens bell 10 times = 10 interval resets.
5. **`useEffect` dep array in `Requests.tsx` (line 333)** depends on 15 values including `viewMode` — entering detail view triggers a data fetch attempt (guarded by an `if` check, but the effect still runs).
6. **No `React.memo` on list row components** — entire list re-renders when any parent state changes (e.g. `datePickerOpen: true`).
7. **No virtualization** on `RequestsList` — rendering 100 rows fully in DOM.

---

### 🧠 State Management Strategy Fix

**Recommended stack:**

| Concern | Current | Recommended |
|---|---|---|
| Routing | Custom string union + `useState` | React Router v6 |
| Server state | Manual `useState` + `useEffect` + fetch | TanStack Query (React Query) |
| Client/UI state | 28+ `useState` in page components | Zustand (small global store) |
| URL/filter state | `sessionStorage` keys | URL search params (`useSearchParams`) |
| Cross-component events | `window.dispatchEvent` | React Query cache invalidation / Zustand actions |

React Query alone would eliminate ~70% of the `useEffect` and loading/error state code in `Requests.tsx`.

---

### 🧱 Refactoring Plan (Step-by-Step)

**Phase 1 — Foundation (1-2 weeks)**
- [ ] Install `react-router-dom` v6, `@tanstack/react-query`, `zustand`
- [ ] Create `QueryClientProvider` + `RouterProvider` in `index.tsx`
- [ ] Define typed routes: `/`, `/requests`, `/requests/:id`, `/requests/new`, `/dept/:deptName`, etc.
- [ ] Fix `types.ts`: type `status` and `priority` as string unions, eliminate `any[]` on known arrays, consolidate duplicate fields
- [ ] Add `ErrorBoundary` at root and per-page level

**Phase 2 — API Layer (1 week)**
- [ ] Extract `buildQueryString(params)` and `buildFormData(payload)` utilities to `services/helpers.ts`
- [ ] Wrap each API function in a React Query `useQuery`/`useMutation` hook in a new `hooks/queries/` directory
- [ ] Remove `inFlightGetRequests` Map (React Query handles deduplication)
- [ ] Remove `fetchTickets` alias

**Phase 3 — Page Decomposition: `Requests.tsx` (2 weeks)**
- [ ] Extract `<RequestFilters />` component with its own state (statusFilter, workflowFilter, etc.)
- [ ] Extract `<RequestsList />` using the shared `usePagination` hook
- [ ] Extract `<CreateRequestForm />` as a `<Dialog>` or separate route
- [ ] Move `loadData` to `useQuery(['requests', filters], fetchFn)`
- [ ] Replace `filteredTickets` client-side filter with URL params → server filter
- [ ] Replace `sessionStorage` deep-link with URL params (`/requests?view=12345`)

**Phase 4 — Component Decomposition: `RequestDetails.tsx` (1 week)**
- [ ] Extract `<StageNavigationModal />` with its own fetch hook
- [ ] Extract `<AttachmentLightbox />` 
- [ ] Extract `<RequestActionButtons />` (edit/close/reopen/escalate)
- [ ] `useMemo` on `sortedStages`, `effectiveIndex`, `completionPercent`
- [ ] Move API mutations to `useMutation`

**Phase 5 — Layout & Navigation (3-4 days)**
- [ ] Split `Layout.tsx` into `<Sidebar>`, `<MobileSidebar>`, `<NavItem>`, `<ProfileMenu>`
- [ ] Remove `workflowFilters` prop from `Layout` — use a context or query instead
- [ ] Extract keyboard shortcuts to `useKeyboardShortcuts` hook

**Phase 6 — Polish (ongoing)**
- [ ] Add `React.memo` to `RequestRow`, `NotificationItem`, `IssueRow`
- [ ] Replace `window.addEventListener('keydown')` in Requests with a single global handler
- [ ] Add inline form validation (error messages per field)
- [ ] Add `aria-expanded`, `aria-controls` to navigation dropdowns

---

*Audit performed: March 30, 2026 | Scope: Full codebase (d:\frontend\orts-frontend-v2)*





from now on use this for the export with filters if applied: export users: GET /api/users/export_users/


Break ->color blue
idle->color red
Computer change to->updated field Online(NEW)->color green
new field Offline(NEW)-> grey


all, online, offline, idle

on users list view show {report to} person in the place of status column

as for the issue export: make call with this param to get the export file :issue export: GET any enpoint of issues add ?format=csv in front




