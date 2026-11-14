Analyzed test-share-options.mjs:L1-484
Analyzed test-unified-share-modal.mjs:L1-595
Analyzed test-yappout.mjs:L1-582
Analyzed verify-card-adjustments.mjs:L1-369
Ran run TypeScript type check · Exited with code 1
Ran run ESLint check · Exited with code 1
Ran check zip size and recent commits
Ran search symbol usage across src · Exited with code 1
Ran run tsc typecheck via node
Ran run eslint via node · Exited with code 1
Ran search symbol references via git grep
Ran audit type escapes, console, and raw HTML
Analyzed task-147.log:L1-97
Analyzed rootCogInfo.json:L1-7
Ran find all process.env usages in src
Ran check ui/select usage across src
Ran run pure unit test scripts · Exited with code 1
Ran run JSDOM component test scripts · Exited with code 1
Ran check Next.js 16 proxy docs
Analyzed proxy.md:L1-120
Ran run test-share-options script
Ran check package.json script and deps
Ran run npm audit vulnerability check · Exited with code 1
Canceled check npm audit and git tracked files
Analyzed postcss.config.mjs:L1-7
Analyzed rootCog.md:L1-34
Analyzed task-166.log
Checked task Check npm audit and git tracked files
Ran check gitignore, tracked files, and dep counts
Ran analyze src metrics, unused deps, and cycles · Exited with code 1
Ran run static import graph and metrics script
Checked task Check npm audit and git tracked files
Analyzed task-166.log
Killed task Check npm audit and git tracked files
Ran verify working tree untouched

# YETVAST — ENGINEERING STANDARDS & CODEBASE COMPLIANCE AUDIT: COGMIT

**Audit Scope:** Full static inspection of all 82 files (`12,758` lines) under `src/`, all 13 scripts under `scripts/`, `test-parsers.mjs`, `src/yappout/tests/yappout.test.mjs`, all configuration files, and all documentation under `docs/` and repository root.
**Mode:** Audit only (no files modified; `git status --short` verified clean).
**Verification Commands Run:**
- `node ./node_modules/typescript/bin/tsc --noEmit`: **PASSED** (exit code 0, Node v26.7.0).
- `node ./node_modules/eslint/bin/eslint.js .`: **FAILED** (exit code 1 — **38 problems: 2 errors, 36 warnings**).
- Direct execution of test scripts (`src/yappout/tests/yappout.test.mjs`, `scripts/test-yappout.mjs`, `scripts/test-rootcog-title-197.mjs`, `scripts/test-card-rendering-v2.mjs`, `scripts/test-share-options.mjs`, `scripts/test-card-footer-customization.mjs`, `scripts/test-font-rendering-fixes.mjs`, `scripts/test-expandable-card-options.mjs`): **ALL FAILED** (exit code 1) due to missing TS/JSX/alias test runner tooling and extensionless ESM imports.
- `npm.cmd audit --json`: **NOT VERIFIED** (hung/timed out waiting on registry response and was cancelled).
- `next build`: **NOT VERIFIED** (skipped to avoid generating/altering `.next/` build state).

---

## A. Executive Summary

Cogmit is a feature-rich prototype built on **Next.js 16.3.6**, **React 19.2.8**, **NextAuth v5 beta**, **Tailwind CSS v4**, and **Octokit** using GitHub repositories (`YVSApps_Data_Cogmit_Private` and `YVSApps_Data_Cogmit_Public`) as its persistence store. It demonstrates promising product domain discipline in a few targeted areas—most notably a centralized pure constants module ([`src/constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts)), an isolated TTS module ([`src/yappout/`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/README.md)), and deterministic SVG card rendering ([`src/lib/card-renderer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/card-renderer.ts)).

However, **Cogmit is not currently suitable as a reference implementation for the YetVast ecosystem**. It exhibits severe architectural drift, competing persistence and publishing pipelines, significant security gaps (including an unenforced user allowlist, open redirect on `/login`, and Host header injection on share URLs), fragile multi-step non-atomic GitHub writes with silent error swallowing, massive duplication in Git Tree operations, unrunnable and drifted test scripts, failing ESLint rules, and substantial client-bundle bloat (shipping seven Markdown parsers simultaneously).

---

## B. Architecture Assessment

### Current Observed Architecture
- **Framework & Rendering:** Next.js App Router (`src/app/`) with Server Components for data loading, Next.js 16 `src/proxy.ts` for corrupt-cookie cleanup, and Server Actions (`"use server"`) for GitHub mutations.
- **Auth:** Auth.js v5 beta (`next-auth@^5.0.0-beta.32`) with GitHub OAuth (`read:user user:email repo`) storing `accessToken` inside an encrypted JWE session cookie ([`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L16-L46)).
- **Persistence:** User-owned GitHub repositories accessed via `@octokit/rest` ([`src/lib/github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts)) on authenticated routes, and `raw.githubusercontent.com` + unauthenticated Octokit on public `/cogmits/*` routes.
- **Module Graph:** Acyclic (`0` circular imports detected across `src/`).

### Strengths
1. **Pure Shared Constants SSOT:** [`src/constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L1-L237) keeps client/server-safe values in one zero-dependency file.
2. **Modular Feature Boundary (`yappout`):** [`src/yappout/`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts) separates TTS state, DOM range tracking, scrolling, and text normalization from Cogmit storage concerns.
3. **Zero Circular Dependencies:** Static import graph analysis across all `src/**/*.ts(x)` files found zero import cycles.
4. **Next.js 16 Proxy Adoption:** [`src/proxy.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/proxy.ts#L6-L39) uses the Next.js 16 `proxy` convention rather than deprecated `middleware`.

### Weaknesses & Architectural Drift
1. **No Data-Access / Repository Layer Separation:** GitHub Octokit calls and JSON schema knowledge (`cogsIndex.json`, `cogmitsIndex.json`, `projectData.json`) are scattered directly across Server Components ([`myCogs/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/page.tsx#L105-L141), [`deletedCogs/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/deletedCogs/page.tsx#L57-L90), [`editCog/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/page.tsx#L66-L80), [`viewCog/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/viewCog/page.tsx#L74-L88)) and UI-colocated Server Actions (`src/app/myCogs/actions.ts`, `src/app/myCogs/newCog/actions.ts`, `src/app/myCogs/[cogId]/editCog/actions.ts`).
2. **Read Functions Performing Hidden Mutations:** [`getCogmitDataStatus`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L413-L485) silently creates a private repository (`L437-L442`) and runs [`reconcileArchitecture`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L338-L364) (5–7 sequential `getContent` / `createOrUpdateFileContents` API calls) on every read/page/action invocation.
3. **Competing Publishing Implementations & Data Drift:**
   - **Flow A (Active UI):** [`EditCogClient.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L161-L472) orchestrates 4 client-driven Server Action calls (`createNewRootCog`/`saveRootCog(..., false)` → [`writePublishedCogmitMarkdown`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/publish-actions.ts#L56-L186) → [`updateCogmitsIndex`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/publish-actions.ts#L188-L300) → [`updateSourceCogsIndexStatus`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/publish-actions.ts#L302-L421)) targeting `YVSApps_Data_Cogmit_Public` and removing `slug`.
   - **Flow B (Legacy/Competing):** [`createNewRootCog`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L135-L157) and [`saveRootCog`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L117-L141) still accept `isPublic: boolean` and write `cogmit_${timestamp}` + `slug` into `cogsIndex.json` without publishing to `YVSApps_Data_Cogmit_Public`.
4. **Severe Documentation-to-Code Drift:**
   - Repo name in [`CogmitDataProvisioning.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogmitDataProvisioning.tsx#L134), [`src/lib/github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L581-L628), [`src/app/cogmits/[authorId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/page.tsx#L10), and `docs/` still references `YVSApps_Cogmit_Data`, while actual code uses `YVSApps_Data_Cogmit_Private` / `YVSApps_Data_Cogmit_Public`.
   - [`docs/architecture.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/docs/architecture.md#L120-L127) & [`README_CONTENT`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L661) describe `rootCogInfo.json`, yet actual code never creates, reads, or updates `rootCogInfo.json`.
   - [`docs/2026-09-29/cogmit_deadcode_audit.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/docs/2026-09-29/cogmit_deadcode_audit.md#L12) claims zero unused dependencies, which is false (`remark-html`, `ui/select.tsx`, etc.).
5. **Inverted Dependency Boundaries:** Shared `src/components/` imports from route-specific `src/app/` paths ([`CogOptionsModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogOptionsModal.tsx#L6), [`DeleteCogButton.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/DeleteCogButton.tsx#L6), [`RestoreCogButton.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/RestoreCogButton.tsx#L6)).
6. **God Components / God Files:** [`src/lib/github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts) (`917` lines), [`src/lib/card-renderer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/card-renderer.ts) (`875` lines), [`EditCogClient.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx) (`797` lines), [`ShareCogmitModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ShareCogmitModal.tsx) (`723` lines), [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts) (`659` lines), and [`ViewCog.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx) (`569` lines) concentrate disparate concerns.

---

## C. Engineering Standards Compliance Matrix

| Standard | Current Implementation | Evidence | Compliance | Severity | Recommendation |
|---|---|---|---|---|---|
| **Auth Allowlist Enforcement** | `.env.example` defines `ALLOWED_GITHUB_USERS`, `sitemap.ts` reads it, but `auth.ts` never checks it in `signIn` callback. | [`.env.example`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/.env.example#L14-L15), [`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L16-L46), [`sitemap.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/sitemap.ts#L11) | NON-COMPLIANT | CRITICAL | Enforce `ALLOWED_GITHUB_USERS` in `NextAuth.callbacks.signIn` (checking GitHub `profile.login`) or remove the misleading config. |
| **Open Redirect Prevention** | `/login` reads `searchParams.callbackUrl` and passes it raw into `redirect(callbackUrl)` when authenticated. | [`src/app/login/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/login/page.tsx#L13-L18) | NON-COMPLIANT | HIGH | Validate `callbackUrl` is a single string starting with `/` (not `//` or protocol) before calling `redirect()`. |
| **Host Header Trust** | Public Cogmit page constructs `shareUrl` directly from unvalidated `x-forwarded-host` / `host` headers. | [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L231-L245) | NON-COMPLIANT | HIGH | Derive `shareUrl` from trusted env config (`NEXT_PUBLIC_APP_URL` / `VERCEL_PROJECT_PRODUCTION_URL`) or strict host allowlist. |
| **Auth Identity Consistency** | `isAuthor` checks `session?.user?.name` (display name) against `authorId` (GitHub handle); `auth.ts` doesn't store `login`. | [`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L31-L45), [`cogmit page`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L246-L247), [`aboutCogmit`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/aboutCogmit/page.tsx#L34-L36) | NON-COMPLIANT | HIGH | Persist GitHub `profile.login` into JWT & `session.user.login` in `auth.ts`; eliminate extra `getAuthenticatedUser()` call in `GlobalNav`. |
| **Client/Server Rendering Separation & Bundle Discipline** | `ViewCog.tsx` (`"use client"`) imports 6 client HTML parsers + `react-markdown`, relies on transitive `dompurify` (`import("dompurify")`) and `dangerouslySetInnerHTML`. | [`ViewCog.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L4-L97), [`L189-L194`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L189-L194), [`L539`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L539), [`package.json`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/package.json#L16-L38) | NON-COMPLIANT | HIGH | Standardize on a single Markdown pipeline (e.g., `react-markdown` + `remark-gfm` or unified w/ `rehype-sanitize`), remove experimental `v1–v6` switcher and unused parsers. |
| **Input & Path Validation** | `projectName`, `destProjectName`, and `NPPCogPage` `cogId` are interpolated into GitHub paths without rejecting `..`, `//`, leading/trailing `/`, or control chars; `saveRootCog` doesn't `.trim()` before length check. | [`newCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L41-L42), [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L438-L447), [`NPPCogPage`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/cog/%5BcogId%5D/page.tsx#L18-L19) | NON-COMPLIANT | HIGH | Introduce central path-segment validator rejecting `..`, empty segments, `\`, invalid characters, and validating trimmed lengths uniformly. |
| **Data Consistency & Lifecycle Sync** | Deleting a Cog leaves published copy & `cogmitsIndex.json` intact; `moveCogToDestination` writes `"No Parent Project"` literal into `indexCog.project`; competing `isPublic` logic exists. | [`myCogs/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/actions.ts#L20-L98), [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L609), [`L117-L129`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L117-L129) | NON-COMPLIANT | CRITICAL | Unpublish/mark deleted in `cogmitsIndex.json` on delete; store `undefined`/`null` for NPPCogs uniformly; remove dead `isPublic` branch in `createNewRootCog`/`saveRootCog`. |
| **Persistence Atomicity & Concurrency** | Multi-step publish writes 3 separate commits across repos; `cogsIndex.json` read-modify-write lacks SHA conflict retry or schema validation; `saveRootCog` swallows read failures. | [`EditCogClient.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L244-L435), [`publish-actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/publish-actions.ts#L56-L421), [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L66-L100) | NON-COMPLIANT | CRITICAL | Encapsulate publish into a server-side transactional workflow with Git Tree API per-repo commits, conflict retry, and explicit error propagation. |
| **Read/Write Side-Effect Separation** | `getCogmitDataStatus()` creates GitHub repos and runs `reconcileArchitecture()` writes on every read/status check. | [`src/lib/github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L413-L485) | NON-COMPLIANT | HIGH | Make `getCogmitDataStatus()` strictly read-only; run reconciliation only on explicit init/repair mutations. |
| **Error Handling & Propagation** | Empty `catch {}` blocks in `saveRootCog`; mixed auth-error contracts (`redirect` vs `{success:false}` vs `throw Error`); raw `alert()` in client buttons; silent empty content on raw fetch failure. | [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L75), [`L99`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L99), [`DeleteCogButton.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/DeleteCogButton.tsx#L22), [`RestoreCogButton.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/RestoreCogButton.tsx#L20), [`cogmit page`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L52-L56) | NON-COMPLIANT | HIGH | Eliminate bare `catch {}`; standardize Server Action result envelopes `{ ok, data } | { ok: false, code, message }`; replace `alert()` with inline/accessible error UI. |
| **Design Tokens & YVS Visual Language** | `globals.css` diverges from `YVS-AGENTS.md` v1.2.2 tokens (`muted`, `border`, `destructive`, `secondary`, missing `success`/`warning`/`brand-border`); raw hexes scattered in TSX. | [`YVS-AGENTS.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/YVS-AGENTS.md#L257-L309), [`globals.css`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/globals.css#L53-L124), [`ShareCogmitModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ShareCogmitModal.tsx#L497-L667), [`CogActionRow.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogActionRow.tsx#L102-L143) | PARTIALLY COMPLIANT | MEDIUM | Align `globals.css` CSS variables with YVS Guidelines v1.2.2 and remove hardcoded hex classes from UI components. |
| **TypeScript Strictness** | `strict: true`, `tsc --noEmit` passes (0 errors), minimal suppressions (1 `@ts-expect-error` in `ViewCog.tsx:15`); however `JSON.parse` outputs are untyped/unvalidated. | [`tsconfig.json`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/tsconfig.json#L7), [`ViewCog.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L15), [`github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L223) | PARTIALLY COMPLIANT | MEDIUM | Add runtime schema validation (`zod` or typed guards) around `JSON.parse` for `cogsIndex.json` and `cogmitsIndex.json`. |
| **Linting & Static Quality** | ESLint fails with 2 React Hook errors (`set-state-in-effect`) and 36 warnings (unused vars/imports, missing hook deps). | [`ResponsiveActionGroups.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ResponsiveActionGroups.tsx#L54), [`useYappout.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts#L130) | NON-COMPLIANT | HIGH | Fix both React 19 `set-state-in-effect` errors and clean all 36 warnings; enforce ESLint in CI. |
| **Testing & Test Reliability** | No `test` script in `package.json`; all 8 executed scripts in `scripts/` & `src/yappout/tests/` fail out of the box; zero automated tests for auth or GitHub mutations; stale assertions in `scripts/`. | [`package.json`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/package.json#L5-L10), [`scripts/`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/scripts/test-card-rendering-v2.mjs#L288-L290) | NON-COMPLIANT | HIGH | Adopt a real test runner (`vitest` + `@testing-library/react`), add `"test"` script, fix/delete stale scripts, test Server Actions with mocked Octokit. |
| **Accessibility (a11y)** | Clickable `<div>` tree rows in `ProjectHierarchyModal`, 5 custom modals missing `Escape` handling, `role="dialog"`, `aria-modal`, and focus traps; `<Link href="#">` when disabled. | [`ProjectHierarchyModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ProjectHierarchyModal.tsx#L135-L283), [`CogActionRow.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogActionRow.tsx#L77-L83), [`CogOptionsModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogOptionsModal.tsx#L32-L60) | NON-COMPLIANT | MEDIUM | Replace custom `div` overlays with a shared accessible Dialog primitive (`@base-ui/react` Dialog); use `<button>` for interactive tree items. |
| **Performance & GitHub Rate Limits** | N+1 `listCommits` per cog on `/myCogs` & `/deletedCogs`; `GlobalNav` calls `getAuthenticatedUser()` on every page; `RootLayout` forces dynamic rendering via `checkSessionStatus()`. | [`myCogs/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/page.tsx#L116-L140), [`GlobalNav.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/GlobalNav.tsx#L26-L33), [`layout.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/layout.tsx#L37-L38) | NON-COMPLIANT | HIGH | Move `login` into JWT session, eliminate `getCogmitDataStatus()` side-effect calls, batch/defer or bound commit fetches, avoid forcing dynamic layout on static/public routes. |
| **Dependency Hygiene** | `shadcn` CLI in runtime `dependencies`; obscure `"cn": "^0.4.0"` instead of `clsx`+`tailwind-merge`; unused `remark-html`; missing direct `dompurify`/`jsdom` deps; `next-auth` beta pinned via `^`; `Cogmit.zip` tracked in Git. | [`package.json`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/package.json#L11-L39), `Cogmit.zip` (`257,930` bytes tracked in git) | NON-COMPLIANT | MEDIUM | Audit and prune dependencies, move CLI/test deps to `devDependencies` (or remove), remove `Cogmit.zip` from Git. |
| **RootCog 197-Char Title Invariant** | `MAX_ROOTCOG_TITLE_LENGTH = 197` defined once in `constants` and enforced in client forms, actions, publish, card SVG, and OG metadata. | [`constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L117), [`EditCogClient.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L592), [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L43) | PARTIALLY COMPLIANT | LOW | Apply `.trim()` consistently before checking length on Server Actions and check `!title.trim()` in `EditCogClient.submit()`. |
| **Dependency Vulnerability Scan** | `npm audit` timed out / hung before producing output. | N/A | NOT VERIFIED | INFORMATIONAL | Run `npm audit` in CI with registry access and address any critical/high advisories. |

---

## D. Security Assessment

1. **[CRITICAL] Unenforced `ALLOWED_GITHUB_USERS` Access Control:** [`.env.example`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/.env.example#L14-L15) documents `ALLOWED_GITHUB_USERS` to restrict login, and [`src/app/sitemap.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/sitemap.ts#L11) reads it, but [`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L16-L46) lacks a `signIn` callback. Any arbitrary GitHub user can authenticate to the app.
2. **[HIGH] Open Redirect in `/login`:** [`src/app/login/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/login/page.tsx#L14-L18) executes `redirect((params.callbackUrl as string) || "/")` directly when a session exists, allowing an attacker to craft `/login?callbackUrl=https://phishing.example` or `//evil.example`.
3. **[HIGH] Host Header Injection on Public Share URLs:** [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L232-L245) trusts raw `x-forwarded-host` and `host` headers to construct `shareUrl` passed into the UI and Share modal.
4. **[HIGH] Client-Side XSS Reliance via `allowDangerousHtml` + Undeclared `dompurify`:** [`ViewCog.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L58-L72) explicitly sets `html: true` / `allowDangerousHtml: true` across multiple parsers and passes output into [`dangerouslySetInnerHTML`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L539) relying on [`await import("dompurify")`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L189), even though `dompurify` is not a declared direct dependency in `package.json` (only `isomorphic-dompurify` is). If `dompurify` fails to load or is hoisted differently by a package manager, rendering fails or degrades. Additionally, [`src/lib/card-renderer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/card-renderer.ts#L81) returns `opts.font` unescaped into SVG `<style>` and `style=""` attributes (safe today only because `ShareCogmitModal` uses a static `<select>`, but unsafe as a reusable library function).
5. **[MEDIUM] Broken `isAuthor` Identity Check:** [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L246-L247) and [`src/app/aboutCogmit/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/aboutCogmit/page.tsx#L34-L36) compare `session?.user?.name` (editable GitHub Display Name, not unique) to `authorId` (GitHub handle).
6. **[MEDIUM] Unvalidated Repo Path Segments:** [`createProjectAction`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L38-L69), [`createProjectAndMoveCog`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L161-L224), [`moveCogToDestination`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L428-L495), and [`NPPCogPage`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/cog/%5BcogId%5D/page.tsx#L18-L19) do not reject `..`, `//`, `\`, or traversal segments when constructing GitHub repository file paths.
7. **[MEDIUM] Unconditional `trustHost: true`:** [`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L17) hardcodes `trustHost: true` rather than configuring explicit trusted origins/hosts.
8. **[LOW] Broad GitHub OAuth `repo` Scope:** [`src/constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L159) requests full `repo` scope across all user private/public repositories (inherent to GitHub OAuth Apps; requires a GitHub App migration to narrow).
9. **[LOW] Excessive Diagnostic Logging & Verbose Error Reflection:** [`src/app/cogmits/[authorId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/page.tsx#L71) and [`EditCogClient.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L162) emit `console.log` debug statements; [`src/lib/github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L125-L135) attaches `console.debug`/`console.info` directly to Octokit. No secrets were observed in logs or tracked `.env` files (`.env.local` is ignored by `.gitignore:34`).
10. **[INFORMATIONAL] Dependency Vulnerabilities:** `npm audit` could not complete (`NOT VERIFIED`).

---

## E. Reliability and Testing Assessment

### Observed Verification Results
- **TypeScript (`tsc --noEmit`):** Passed with `0` errors.
- **ESLint (`eslint .`):** Failed with **2 errors** and **36 warnings**:
  - **Errors (`react-hooks/set-state-in-effect`):** [`ResponsiveActionGroups.tsx:54`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ResponsiveActionGroups.tsx#L52-L56) and [`useYappout.ts:130`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts#L128-L133).
  - **Warnings:** 36 warnings across unused imports/variables and `react-hooks/exhaustive-deps` ([`ResponsiveActionGroups.tsx:49`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ResponsiveActionGroups.tsx#L49)).
- **Automated Test Execution:** **0% of existing test scripts execute cleanly under Node:**
  - `src/yappout/tests/yappout.test.mjs` & `scripts/test-yappout.mjs` fail immediately on `ERR_MODULE_NOT_FOUND` (`../../constants/index` without `.ts` extension in [`speedController.ts:4`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/speedController.ts#L4)).
  - `scripts/test-rootcog-title-197.mjs`, `scripts/test-card-rendering-v2.mjs`, `scripts/test-share-options.mjs` fail immediately on `Cannot find package '@/constants'`.
  - All 8 `.tsx`-importing scripts fail immediately on `ERR_UNKNOWN_FILE_EXTENSION: ".tsx"`.
  - `package.json` defines **no `"test"` script**, **no test runner** (`vitest`/`jest`/`playwright`), and **no `jsdom` dependency** (relying accidentally on transitive `jsdom` inside `isomorphic-dompurify`).

### Quality & Coverage Gaps
1. **Drifted / Self-Referential Tests:** Even inspecting the source of the test scripts reveals broken assertions against obsolete UI/renderer behavior:
   - [`test-card-rendering-v2.mjs`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/scripts/test-card-rendering-v2.mjs#L232-L355) asserts `49px`/`45px` font sizes and `"View & Listen full @ "` footer text, both removed in `card-renderer.ts` (`47px`/`41px` and custom footer).
   - [`test-generate-card-transition.mjs`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/scripts/test-generate-card-transition.mjs#L84-L99) and [`test-unified-share-modal.mjs`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/scripts/test-unified-share-modal.mjs#L112-L125) assert pre-refactor Share modal DOM structure.
   - [`test-e2e-share-public-route.mjs`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/scripts/test-e2e-share-public-route.mjs#L139-L147) hits live external GitHub (`yvnow/yvnow_2026oct090558`) without mocking, violating test isolation.
   - [`test-yappout.mjs`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/scripts/test-yappout.mjs#L171-L290) defines local fake helper functions (`simulateSpeak`, `evaluateTap`, `shouldScroll`) and tests those fakes instead of importing `playbackController.ts`, `doubleTapDetector.ts`, or `scrollController.ts`!
2. **Test Coupling in Production UI:** [`ShareCogmitModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ShareCogmitModal.tsx#L612-L623) embeds a hidden `<button id="toggle-with-footer" className="sr-only hidden">` solely to satisfy a legacy test selector.
3. **Zero Coverage on Critical Paths:** Authentication (`auth.ts`, `proxy.ts`), GitHub repository initialization/reconciliation (`github.ts`), all Server Actions (`createNewRootCog`, `saveRootCog`, `deleteRootCog`, `restoreRootCog`, `moveCogToDestination`, `publish-actions.ts`), and all App Router pages have **0 automated tests**.

### Runtime Failure & Reliability Risks
- **GitHub Rate Limiting:** `/myCogs` and `/myCogs/deletedCogs` execute `N + 2` GitHub calls per page load (`getCogmitDataStatus` reconciliation + index + `listCommits` per cog). Unauthenticated `/cogmits/[authorId]` and `/sitemap.ts` hit GitHub's 60 req/hr IP limit directly.
- **History Truncation:** [`MyCogsPage`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/page.tsx#L124) and [`DeletedCogsPage`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/deletedCogs/page.tsx#L75) omit `per_page` (defaulting to 30 commits) and compute `createdAt` from `commits[length - 1]`—meaning any Cog with >30 commits reports an incorrect creation timestamp.
- **Broken Navigation & Placeholder Controls:** [`/repo/[owner]/[name]/explore`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/explore/page.tsx#L57) links to nonexistent `/repo/.../project/...` (404); `Pin` ([`CogActionRow.tsx:57`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogActionRow.tsx#L57)) and `Like` ([`ViewCog.tsx:147`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L147)) mutate ephemeral client state with no persistence.

---

## F. Duplication Assessment

| Duplicated Pattern | Locations | Consolidation Justified? |
|---|---|---|
| **Git Tree API Move + `cogsIndex.json` Commit Boilerplate** (~120–140 lines × 4) | [`deleteRootCog`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/actions.ts#L56-L90), [`restoreRootCog`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/actions.ts#L136-L168), [`createProjectAndMoveCog`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L229-L372), [`moveCogToDestination`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L500-L645) | **YES.** Identical mechanical Git Tree operations (`getRef` → `getCommit` → `dirData` blob move → `createBlob` → `createTree` → `createCommit` → `updateRef`). Also `createProjectAndMoveCog` is a strict subset of `moveCogToDestination(..., true)`. |
| **`stripMarkdown` Implementations (× 3)** | [`src/lib/excerpt.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/excerpt.ts#L18-L97), [`src/yappout/utils/markdownExtractor.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/utils/markdownExtractor.ts#L9-L40), [`src/yappout/utils/textNormalizer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/utils/textNormalizer.ts#L54-L85) | **PARTIALLY.** The two inside `src/yappout/utils/` are 100% verbatim duplicates and must be consolidated immediately. `src/lib/excerpt.ts` has different semantics (skips matching H1 title, collapses whitespace to spaces vs preserving `\n` for TTS segmentation), so keep `excerpt.ts` separate from `yappout`'s `stripMarkdown`. |
| **Repository Bootstrap (`ensureFile` vs `createCommit`)** | [`reconcileRepoStructure`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L366-L411) vs [`initializeCogmitRepository`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L743-L812) | **YES.** Both run identical `getContent` → 404/409 → `createOrUpdateFileContents` file seeding and are both called sequentially during `initiateCogmitData()`. |
| **RootCog SSR Page Loader** | [`editCog/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/page.tsx#L13-L86) vs [`viewCog/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/viewCog/page.tsx#L18-L97) | **YES.** Identical `auth()` + `getCogmitDataStatus()` + `getRootCogIndexEntry()` + `getRepositoryContent(rootCog.md)` + `listCommits()` + `getProjects()` loader. |
| **6 Markdown Parsers + `react-markdown`** | [`ViewCog.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L34-L97) vs [`L543`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L543) | **YES.** Consolidate onto one Markdown rendering pipeline. |
| **Horizontal Marquee Overflow Rows (× 2)** | [`MarqueeContent.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/MarqueeContent.tsx) vs [`CogMetadataRow.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogMetadataRow.tsx) | **YES.** Both implement overflow detection + `animate-marquee` + hover popover. |
| **Custom Overlay Modals (× 6)** | `ShareCogmitModal`, `PublishCogmitModal`, `ProjectHierarchyModal`, `CogOptionsModal`, `DeleteCogButton`, `HistoryModal` | **YES.** Extract one shared accessible `<Modal>` wrapper (focus trap, Escape, backdrop, ARIA attributes) while keeping modal contents distinct. |
| **Duplicate Repo Status 404 Retry** | [`cogmits/[authorId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/page.tsx#L29-L44) & [`[cogmitId]/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogmitId%5D/page.tsx#L83-L99) | **YES.** Retries the exact same `COGMIT_REPO_PUBLIC` call twice inside `catch (404)`. |
| **Duplicate `yappout.css` vs `globals.css`** | [`src/yappout/styles/yappout.css`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/styles/yappout.css) vs [`src/app/globals.css`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/globals.css#L307-L335) | **YES.** Import `yappout.css` or keep one copy; they have already drifted (`.yappout-segment-active` missing in `globals.css`). |
| **Duplicate Base URL Resolution** | [`layout.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/layout.tsx#L16-L19), [`card-renderer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/card-renderer.ts#L476-L493), [`cogmit page`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogmitId%5D/page.tsx#L179-L182), [`robots.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/robots.ts#L5), [`sitemap.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/sitemap.ts#L17) | **YES.** Extract a shared `getAppOrigin()` helper (note `card-renderer.ts` falls back to `cogmit.vercel.app` while `layout.tsx`/`robots.ts`/`sitemap.ts` fall back to `cogmit.com`!). |
| **Duplicate Sign-In Pages** | [`src/app/login/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/login/page.tsx) vs [`src/app/auth/signin/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/auth/signin/page.tsx) | **YES.** Two distinct sign-in routes with inconsistent styling and redirect behavior. |
| **Playback Recursion in Yappout** | [`speakSegmentBySegment`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/playbackController.ts#L23-L72) vs [`speakWordByWord`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/playbackController.ts#L77-L119) | **OPTIONAL.** `speakWordByWord` is unused outside barrel exports; remove dead mode rather than abstracting it. |

---

## G. Standards Worth Preserving

1. **Zero-Dependency Client/Server-Safe Constants SSOT:** [`src/constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L1-L14) enforces an explicit invariant: pure values only, zero runtime Node/browser dependencies, zero secrets.
2. **Session-Guarded Async Playback:** [`useYappout.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts#L160-L186) and [`playbackController.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/playbackController.ts#L34) use monotonic `sessionIdRef` checks to eliminate stale Web Speech API callback races on rapid seek/stop/unmount.
3. **DOM-Non-Mutating Range Highlight Overlay:** [`wordTracker.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/wordTracker.ts) and [`YappoutHighlight.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/components/YappoutHighlight.tsx) compute `Range.getClientRects()` overlays instead of mutating sanitized HTML DOM trees.
4. **Unicode-Safe Boundary Truncation:** [`src/lib/excerpt.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/excerpt.ts#L114-L146) splits via `Array.from(stripped)` to avoid slicing UTF-16 surrogate pairs and enforces hard length invariants with post-checks.
5. **Typed Octokit 401 Interception & NextAuth JWE Token Segregation:** [`src/lib/github.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L138-L143) hooks Octokit errors into `AuthenticationRequiredError`, while [`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L39-L44) keeps `accessToken` out of client-exposed `session` objects.
6. **AI Governance & Evidence Taxonomy:** [`YVS-AGENTS.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/YVS-AGENTS.md) and [`docs/AI-DEVELOPMENT.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/docs/AI-DEVELOPMENT.md) establish a disciplined *Inspect → Search → Reuse → Implement → Verify* workflow and `OBSERVED / DOCUMENTED / TESTED / INFERRED / UNKNOWN` reporting contract.
7. **Deterministic XML-Escaped SVG Generation + Source Hashing:** [`src/lib/card-renderer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/card-renderer.ts#L126-L134) and `computeCardSourceHash` (`L318-L339`) make card outputs deterministic and cache-invalidated on option changes.

---

## H. Standards Violations (Material Violations Index)

1. **Unenforced `ALLOWED_GITHUB_USERS` allowlist:** [`.env.example:15`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/.env.example#L14-L15), [`src/auth.ts:16-46`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L16-L46).
2. **Open Redirect via `callbackUrl`:** [`src/app/login/page.tsx:14-18`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/login/page.tsx#L14-L18).
3. **Host Header Injection in `shareUrl`:** [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx:232-245`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L232-L245).
4. **Read Function Mutating GitHub State (`getCogmitDataStatus`):** [`src/lib/github.ts:413-458`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L413-L458).
5. **Unreachable Repo Fallback Logic (`COGMIT_REPO_PUBLIC === COGMIT_REPO_PUBLISHED`):** [`src/constants/index.ts:177-179`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L177-L179), [`src/lib/github.ts:428-448`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L428-L448).
6. **Non-Atomic Multi-Commit Publishing & Dual Competing `isPublic` Flows:** [`EditCogClient.tsx:244-435`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L244-L435), [`publish-actions.ts:56-421`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/publish-actions.ts#L56-L421), [`newCog/actions.ts:135-157`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L135-L157), [`editCog/actions.ts:117-141`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L117-L141).
7. **Orphaned Public Cogmits on Delete:** [`src/app/myCogs/actions.ts:20-98`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/actions.ts#L20-L98).
8. **Data Corruption on Move to No Parent Project (`"No Parent Project"` stored in index):** [`src/app/myCogs/[cogId]/editCog/actions.ts:609`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L609) vs [`newCog/actions.ts:142`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L142).
9. **Silently Swallowed Errors (`catch {}`) Before Writes:** [`src/app/myCogs/[cogId]/editCog/actions.ts:75,99`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L65-L100).
10. **Silent Empty Content Fallback on Raw Fetch Error:** [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx:52-56`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogmitId%5D/page.tsx#L52-L56).
11. **6 Client Markdown Parsers + Undeclared `dompurify` Import:** [`src/components/ViewCog.tsx:4,14-97,189,539`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L14-L97), [`package.json:16-38`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/package.json#L16-L38).
12. **ESLint Failing Errors (`react-hooks/set-state-in-effect`):** [`ResponsiveActionGroups.tsx:54`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ResponsiveActionGroups.tsx#L54), [`useYappout.ts:130`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts#L130).
13. **Broken `isAuthor` Identity Comparison & Per-Request `users.getAuthenticated()` Call:** [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx:246-247`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L246-L247), [`src/app/aboutCogmit/page.tsx:34-36`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/aboutCogmit/page.tsx#L34-L36), [`GlobalNav.tsx:28`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/GlobalNav.tsx#L28).
14. **Broken Link (`/repo/[owner]/[name]/project/...`):** [`src/app/repo/[owner]/[name]/explore/page.tsx:57`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/explore/page.tsx#L57).
15. **Missing Path-Traversal Checks on Project/Cog Paths:** [`newCog/actions.ts:38-69`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L38-L69), [`editCog/actions.ts:438-447`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L438-L447), [`repo/.../cog/[cogId]/page.tsx:18-19`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/cog/%5BcogId%5D/page.tsx#L18-L19).
16. **Inconsistent Validation & Server Action Contracts:** [`EditCogClient.tsx:163 vs 477`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L477); `mode === "new"` `onMoveProject` no-op at [`EditCogClient.tsx:637-658`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L637-L658); mixed `redirect()` vs `{success:false}` vs `throw Error` across Server Actions.
17. **Dead Code, Test Hooks, & Leftover Logs in Production:** Hidden test button [`ShareCogmitModal.tsx:613-623`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ShareCogmitModal.tsx#L613-L623); debug logs [`EditCogClient.tsx:162`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L162), [`cogmits/[authorId]/page.tsx:71`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/page.tsx#L71); unused files (`ui/select.tsx`, `seekController.ts`, `speechSynthesisAdapter.ts`, `cogmitContentAdapter.ts`, `YappoutSegment.tsx`, `YappoutWord.tsx`, `yappout/index.ts`, `test-parsers.mjs`, `Cogmit.zip`); dead `published = false` in [`myCogs/page.tsx:120`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/page.tsx#L120); unused `addOrUpdateSteps` in [`EditCogClient.tsx:143`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L143); unused `refreshSession`/`handleAuthFailure` wiring in [`AuthProvider.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/AuthProvider.tsx#L71-L123).
18. **Token & Font Naming Drift:** [`globals.css:53-124`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/globals.css#L53-L124) vs [`YVS-AGENTS.md:259-309`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/YVS-AGENTS.md#L259-L309); `--font-geist-mono` for JetBrains Mono in [`layout.tsx:12`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/layout.tsx#L12); `.markdown-content code` bypassing `--font-mono` in [`globals.css:224`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/globals.css#L224); `PAGE_CONTENT_MAX_WIDTH = 1400` contradicting 90% width in [`constants/index.ts:102`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L102).
19. **Blocking `alert()` Error Dialogs & Inaccessible Modal/Tree Controls:** [`DeleteCogButton.tsx:22`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/DeleteCogButton.tsx#L22), [`RestoreCogButton.tsx:20`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/RestoreCogButton.tsx#L20), [`ProjectHierarchyModal.tsx:135-283`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ProjectHierarchyModal.tsx#L135-L283).

---

## I. Proposed YetVast Engineering Standards

### A. Universal Engineering Standards (All YetVast Applications)
1. **Strict Read/Write Separation:** Read/status functions must never mutate external state or create resources as side effects.
2. **One Source of Truth per Domain Concept:** Never keep competing live & legacy write/publish flows or duplicate constants/token tables in parallel.
3. **Standardized Mutation Envelope:** All backend actions/endpoints return discriminated typed results (`{ ok: true, data } | { ok: false, code, message }`) rather than mixing `throw`, `redirect()`, and `{ success }`.
4. **Zero Silent Error Swallowing:** `catch {}` is forbidden unless explicitly checking expected domain conditions (e.g., HTTP `404` on existence checks) and rethrowing all other failures.
5. **Security Boundaries & Validation at Every Entry Point:** Enforce auth allowlists inside auth callbacks, sanitize `callbackUrl`/redirect targets to relative paths, validate external headers (`Host`), and validate untrusted JSON/external payloads with runtime schemas (e.g., `zod`) instead of bare `JSON.parse` casts.
6. **Zero Dead/Test Code in Production:** Never embed hidden DOM elements for tests, leftover `console.log` debug statements, or unreferenced dependencies/archives (`*.zip`) in tracked application code.
7. **CI Quality Gates:** Typecheck (`tsc --noEmit`), Lint (`eslint` with 0 errors), and an actual runnable `"test"` script executing in isolation (no live external network dependencies) must pass before merge.
8. **YVS Visual Language Token Discipline:** Semantic CSS design tokens defined in `YVS-AGENTS.md` / YVS Guidelines live in the theme layer only; raw brand hex values must not be repeated inline across components.

### B. Technology-Specific Standards (Next.js / React / TS / Tailwind)
1. **Layer Dependency Direction:** `src/app/` → `src/components/` → `src/lib/` → `src/constants/`. Shared `src/components/` must never import from `src/app/`.
2. **Server/Client & Bundle Hygiene:** Keep heavy parsing/Markdown transformations on the server where possible; never ship redundant parser libraries in `"use client"` components; avoid `cookies()`/`auth()` in root layouts if public routes should remain static/cacheable.
3. **React 19 Hook Compliance:** Zero synchronous `setState` inside `useEffect` (`react-hooks/set-state-in-effect`); derive values via `useMemo` or compute during render/event handlers.
4. **Accessible UI Primitives:** Use headless dialog/menu primitives (`@base-ui/react` / Radix) for focus trapping, `Escape` closing, and ARIA compliance; never use clickable `<div>` elements for interactive actions.
5. **Direct Dependency Rule:** Every module imported directly in `src/` must be declared explicitly in `package.json` (never rely on transitive sub-dependencies like `dompurify` or `jsdom`).

### C. Product-Specific Architectural Invariants (Cogmit Only — Do Not Generalize)
- Using user-owned GitHub repositories (`YVSApps_Data_Cogmit_Private` / `YVSApps_Data_Cogmit_Public`) as the persistence store.
- Directory hierarchy rules: `projects/<Project>/projectData.json`, `NPPCogs/<RootCogId>/`, `deletedCogs/<RootCogId>/`, no `projects/<Project>/cogs/` subfolder, flat `cogsIndex.json` navigation index, `57`-day deletion lifecycle concept.
- Cogmit-specific bounds: Project name max `57` chars, RootCog title max `197` chars, card description max `597` chars.
- Cogmit `90%` desktop page-content shell ratio vs header width (`--page-content-max-width`).
- Yappout Web Speech API + DOM `Range` overlay module.

### D. Existing Good Patterns Worth Adopting Across YetVast
- Pure, zero-dependency, client/server-safe [`src/constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts) pattern.
- Session-ID fencing pattern for async browser APIs ([`playbackController.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/playbackController.ts)).
- Explicit AI governance & evidence classification (`OBSERVED / DOCUMENTED / TESTED / INFERRED / UNKNOWN`) in [`YVS-AGENTS.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/YVS-AGENTS.md) and [`docs/AI-DEVELOPMENT.md`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/docs/AI-DEVELOPMENT.md).
- Surrogate-pair-safe truncation helpers ([`src/lib/excerpt.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/excerpt.ts)).

### E. Existing Bad Patterns That Must Not Propagate
- Mutating external state inside read/status check functions (`getCogmitDataStatus`).
- Orchestrating multi-step distributed repository writes from a React Client Component (`EditCogClient` calling 4 sequential actions).
- Leaving abandoned A/B library experiments (6 Markdown parsers) in production bundles.
- Writing standalone ad-hoc `.mjs` test scripts that test mock helper functions rather than real code and lack a working `"test"` runner.
- Duplicating long low-level API boilerplate (4× Git Tree commit sequences) across route-local `actions.ts` files.
- Ignoring `.env.example` security flags (`ALLOWED_GITHUB_USERS`) in auth callbacks.

---

## J. Prioritized Remediation Roadmap

### 1. Critical Fixes

1. **Enforce `ALLOWED_GITHUB_USERS` in Auth.js `signIn` Callback**
   - **Problem:** `ALLOWED_GITHUB_USERS` is documented as an access control restriction in `.env.example` but never enforced during login.
   - **Evidence:** [`.env.example:15`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/.env.example#L15), [`src/auth.ts:16-46`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L16-L46).
   - **Proposed correction:** Implement `callbacks.signIn({ profile })` in `src/auth.ts` rejecting logins whose GitHub `profile.login` (case-insensitive) is not in `process.env.ALLOWED_GITHUB_USERS` when configured.
   - **Expected benefit:** Closes unauthorized user login vulnerability when restricted access is expected.
   - **Risk of changing existing behavior:** Users not listed in `ALLOWED_GITHUB_USERS` (when set) will be denied sign-in.
   - **Dependencies:** None.

2. **Eliminate Competing Publish Paths, Non-Atomic Client Orchestration, and Orphaned Public Cogmits on Delete**
   - **Problem:** Publishing is split across client-orchestrated calls in `EditCogClient.tsx` (`publish-actions.ts`) and obsolete `isPublic` branches in `createNewRootCog`/`saveRootCog`; deleting a RootCog leaves its published copy live in `YVSApps_Data_Cogmit_Public`.
   - **Evidence:** [`EditCogClient.tsx:244-435`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L244-L435), [`newCog/actions.ts:135-157`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L135-L157), [`editCog/actions.ts:117-141`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L117-L141), [`myCogs/actions.ts:20-98`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/actions.ts#L20-L98).
   - **Proposed correction:** Move multi-step publication into a single server-side orchestrator using one Git Tree commit per repository, remove `isPublic` branches from `createNewRootCog`/`saveRootCog`, and update `deleteRootCog` to remove/unpublish the public entry in `cogmitsIndex.json` (and markdown artifact).
   - **Expected benefit:** Prevents partial publish corruption, ID/slug schema drift, and deleted private Cogs staying publicly exposed.
   - **Risk of changing existing behavior:** Step-by-step progress UI in `PublishCogmitModal` must transition to server-driven status or single pending/completed state unless streamed.
   - **Dependencies:** Shared GitHub service extraction.

3. **Fix Silent Error Swallowing and Index Corruption in `saveRootCog` & `moveCogToDestination`**
   - **Problem:** `saveRootCog` swallows all exceptions (`catch {}`) when reading `rootCog.md` and `cogsIndex.json`; `moveCogToDestination` writes `"No Parent Project"` into `indexCog.project` instead of `undefined`.
   - **Evidence:** [`editCog/actions.ts:65-100`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L65-L100), [`editCog/actions.ts:609`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L609).
   - **Proposed correction:** Only ignore explicit `404` where creation is valid; rethrow/return structured error on all non-404 failures; set `indexCog.project = destProjectName !== NO_PARENT_PROJECT ? destProjectName : undefined` (and add 409 SHA retry + runtime schema validation).
   - **Expected benefit:** Prevents silent overwriting of `cogsIndex.json` or `rootCog.md` and preserves NPPCog data integrity.
   - **Risk of changing existing behavior:** Existing corrupted records with `project: "No Parent Project"` should be normalized on read (`project === NO_PARENT_PROJECT ? undefined : project`).
   - **Dependencies:** Shared index helper + validator.

---

### 2. High-Priority Fixes

4. **Fix Open Redirect (`/login`) and Host Header Injection (`/cogmits/[authorId]/[cogmitId]`)**
   - **Problem:** Unsanitized `callbackUrl` redirects externally; untrusted `host` / `x-forwarded-host` builds `shareUrl`.
   - **Evidence:** [`src/app/login/page.tsx:14-18`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/login/page.tsx#L14-L18), [`src/app/cogmits/[authorId]/[cogmitId]/page.tsx:231-245`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L231-L245).
   - **Proposed correction:** Restrict `callbackUrl` to relative paths (`/^\/(?!\/)/`); derive `shareUrl` from canonical configured origin (`getAppOrigin()`).
   - **Expected benefit:** Prevents phishing redirects and poisoned share links.
   - **Risk of changing existing behavior:** Localhost/preview share URLs need `NEXT_PUBLIC_APP_URL` or `VERCEL_URL` configured properly.
   - **Dependencies:** Shared `getAppOrigin()` helper.

5. **Remove 6 Redundant Markdown Parsers from `ViewCog.tsx` & Fix `dompurify` Dependency**
   - **Problem:** `ViewCog.tsx` ships 7 markdown renderers to the client, enables raw HTML across them, and dynamically imports undeclared `dompurify`.
   - **Evidence:** [`ViewCog.tsx:4,14-97,189,539`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L14-L97), [`package.json:16-38`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/package.json#L16-L38).
   - **Proposed correction:** Render both private and public views via one secure pipeline (`react-markdown` + `remark-gfm` without `rehype-raw`, or server-sanitized HTML), remove the `v1–v6` parser toggle, and prune the 10+ unused markdown/type packages from `package.json`.
   - **Expected benefit:** Dramatic client JS bundle reduction, consistent GFM rendering across private/public views, and elimination of `allowDangerousHtml` + undeclared dependency risk.
   - **Risk of changing existing behavior:** Removes the experimental `v1–v6` parser preview button in public Cogmit mode.
   - **Dependencies:** Confirm if `v1–v6` parser switcher was temporary developer tooling (unresolved product question if intended for users).

6. **Separate Reads from Mutations in `getCogmitDataStatus()` & Eliminate N+1 Commit Fetches**
   - **Problem:** Every call to `getCogmitDataStatus()` runs `reconcileArchitecture()` (and may create repos); `/myCogs` and `/deletedCogs` fire `listCommits` without `per_page` for every Cog; `GlobalNav` calls `getAuthenticatedUser()` on every render.
   - **Evidence:** [`github.ts:413-485`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L413-L485), [`myCogs/page.tsx:116-140`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/page.tsx#L116-L140), [`GlobalNav.tsx:26-33`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/GlobalNav.tsx#L26-L33).
   - **Proposed correction:** Store GitHub `login` in Auth.js JWT/session (`src/auth.ts`); make `getCogmitDataStatus()` read-only; run `reconcileArchitecture()` only during explicit repo initialization/repair; fix `COGMIT_REPO_PUBLISHED` unreachable logic; evaluate caching/storing `createdAt`/`modifiedAt` or paginating commit lookups cleanly.
   - **Expected benefit:** Cuts GitHub API calls per navigation by >80%, prevents unexpected repo side-effects during GETs, and fixes `isAuthor` matching.
   - **Risk of changing existing behavior:** Auto-healing missing `.gitkeep` files will only happen during init/repair unless explicitly invoked; note `cogsIndex.json` timestamp caching must respect `Cogmit_in_Github_invariants.md` Section 21 (avoiding duplicating Git history unnecessarily — unresolved design trade-off to confirm).
   - **Dependencies:** Auth.js JWT callback update.

7. **Validate Repository Paths and Trim Titles Consistently**
   - **Problem:** Project/Cog path builders lack traversal (`..`, `\`, empty segment) guards; `saveRootCog` checks `.length > 197` before trimming; `EditCogClient.submit` checks `!title` instead of `!title.trim()`.
   - **Evidence:** [`newCog/actions.ts:38-89`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/newCog/actions.ts#L38-L89), [`editCog/actions.ts:40-45,438-447`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts#L438-L447), [`EditCogClient.tsx:477`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L477).
   - **Proposed correction:** Add `validateProjectPath()` / `validateSafeSegment()` helpers in a shared validation module and enforce identical trimmed validation on client and server.
   - **Expected benefit:** Prevents malformed repo paths and inconsistent validation errors.
   - **Risk of changing existing behavior:** Rejects edge-case project names containing `..` or `//`.
   - **Dependencies:** None.

8. **Establish a Working Test Harness (`vitest` + React Testing Library) & Fix Failing ESLint Rules**
   - **Problem:** ESLint fails with 2 `react-hooks/set-state-in-effect` errors; all 8 tested scripts in `scripts/` & `src/yappout/tests/` crash on startup; zero tests cover auth, Server Actions, or error states.
   - **Evidence:** ESLint output ([`ResponsiveActionGroups.tsx:54`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ResponsiveActionGroups.tsx#L54), [`useYappout.ts:130`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts#L130)), Node test execution output.
   - **Proposed correction:** Refactor the two `set-state-in-effect` hooks to derive state or handle resets via refs/callbacks; install `vitest` + `@vitejs/plugin-react` + `jsdom` in `devDependencies`; add `"test"` script to `package.json`; migrate/rewrite tests against real imports with mocked Octokit; delete obsolete scripts.
   - **Expected benefit:** Working green lint + unit/integration CI checks and real regression coverage for critical data mutations.
   - **Risk of changing existing behavior:** Low (internal test setup + hook state refactor).
   - **Dependencies:** User authorization to add `vitest` devDependency.

9. **Standardize Server Action Error Contracts & Fix Silent Fallback on Public Cogmit Read**
   - **Problem:** Server Actions mix `redirect()`, `{success:false}`, and raw `throw Error` (whose messages are stripped in Next.js production builds); `getPublishedCogmit` returns empty `content: ""` when `raw.githubusercontent.com` fails on the `.md` file; `cogmits/[authorId]` skips raw fast-path and hits 60/hr Octokit limit.
   - **Evidence:** [`publish-actions.ts:65-78`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/publish-actions.ts#L65-L78), [`cogmits/[authorId]/[cogmitId]/page.tsx:52-56`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/%5BcogId%5D/page.tsx#L52-L56), [`cogmits/[authorId]/page.tsx:28-83`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/cogmits/%5BauthorId%5D/page.tsx#L28-L83).
   - **Proposed correction:** Return typed result objects `{ ok, ... }` from all Server Actions; fall back to Octokit if `contentRes` on raw GitHub is non-OK; add raw GitHub fetch + `revalidate` to `/cogmits/[authorId]`.
   - **Expected benefit:** Reliable user-facing error messages in production and resilient public reads.
   - **Risk of changing existing behavior:** Client callers (`EditCogClient`, `HeaderControls`, etc.) must check `result.ok` and route to `/login` cleanly on `UNAUTHORIZED`.
   - **Dependencies:** Standard error envelope type.

---

### 3. Medium-Priority Improvements

10. **Extract Shared Git Tree Commit Service & Consolidate `createProjectAndMoveCog` / `moveCogToDestination` / SSR Loaders**
    - **Problem:** 4× duplicated 120-line Git Tree commit flows; duplicated SSR loading in `editCog/page.tsx` and `viewCog/page.tsx`; inverted `src/components/` → `src/app/` imports.
    - **Evidence:** [`myCogs/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/actions.ts), [`editCog/actions.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/actions.ts), [`CogOptionsModal.tsx:6`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogOptionsModal.tsx#L6).
    - **Proposed correction:** Create a repository data service in `src/lib/cog-repository.ts` encapsulating index read/write + atomic directory move; move shared actions/modals out of `src/app/` into `src/lib/` and `src/components/`.
    - **Expected benefit:** Eliminates 400+ lines of duplicate code and restores clean layer boundaries.
    - **Risk of changing existing behavior:** Low if signatures stay compatible during transition.
    - **Dependencies:** Remediation #2 and #3.

11. **Fix `EditCogClient.tsx` Bugs & `ExplorePage` Dead Link**
    - **Problem:** `EditCogClient` `mode === "new"` ignores selecting an existing project (`onMoveProject` returns `{success:true}` without calling `setProject`); `/repo/[owner]/[name]/explore` links to missing `/project/...` route; `NPPCogPage` captures stale SSR `sha`.
    - **Evidence:** [`EditCogClient.tsx:637-658`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/myCogs/%5BcogId%5D/editCog/EditCogClient.tsx#L637-L658), [`explore/page.tsx:57`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/explore/page.tsx#L57), [`NPPCogPage:28,47`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/repo/%5Bowner%5D/%5Bname%5D/cog/%5BcogId%5D/page.tsx#L28).
    - **Proposed correction:** Call `setProject(name)` in `mode === "new"` `onMoveProject`; resolve whether legacy `/repo/...` routes should be removed (as noted in `docs/2026-09-29/cogmit_deadcode_audit.md`) or fixed.
    - **Expected benefit:** Fixes project assignment on new Cogs and removes 404 navigation.
    - **Risk of changing existing behavior:** Legacy `/repo/...` explorer removal affects users connecting arbitrary repos (unresolved product decision).
    - **Dependencies:** Product decision on `/repo/[owner]/[name]/*` and `/connect`.

12. **Unify Design Tokens (`YVS-AGENTS.md` v1.2.2), Modal Primitive, and Marquee Component**
    - **Problem:** `globals.css` tokens differ from `YVS-AGENTS.md`; components hardcode `#1E0038`, `#2D0052`, `#4B0084`, `#FFFF11`; 6 custom modals lack keyboard/focus accessibility; `CogMetadataRow` duplicates `MarqueeContent`; `DeleteCogButton`/`RestoreCogButton` use `alert()`.
    - **Evidence:** [`globals.css:53-124`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/globals.css#L53-L124), [`ShareCogmitModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ShareCogmitModal.tsx#L497-L667), [`ProjectHierarchyModal.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ProjectHierarchyModal.tsx#L135-L283), [`CogMetadataRow.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogMetadataRow.tsx).
    - **Proposed correction:** Align `globals.css` to YVS v1.2.2 tokens, build a single accessible `<Modal>` wrapper using `@base-ui/react` Dialog, convert clickable `<div>` rows to `<button>`, standardize on `MarqueeContent`, and replace `alert()` with inline status UI.
    - **Expected benefit:** Consistent YVS theming, keyboard accessibility, and removal of duplicate UI primitives.
    - **Risk of changing existing behavior:** Visual color adjustments where `globals.css` differed from YVS spec.
    - **Dependencies:** None.

13. **Consolidate Duplicate `stripMarkdown` and `yappout.css` + Split Large Modules**
    - **Problem:** Two identical `stripMarkdown` functions in `src/yappout/utils/`; `yappout.css` copied into `globals.css`; `useYappout` runs unnecessarily in `ViewCog` private mode; `EditCogClient`, `ShareCogmitModal`, `github.ts`, `card-renderer.ts` exceed 700–900 lines.
    - **Evidence:** [`markdownExtractor.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/utils/markdownExtractor.ts), [`textNormalizer.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/utils/textNormalizer.ts), [`ViewCog.tsx:156-169`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L156-L169).
    - **Proposed correction:** Export `stripMarkdown` from one Yappout utility file, import `yappout.css` directly rather than duplicating in `globals.css`, gate `useYappout` work when `mode !== "cogmit"`, and decompose `ShareCogmitModal` / `EditCogClient` into focused subcomponents/hooks.
    - **Expected benefit:** Eliminates CSS/utility drift and improves readability.
    - **Risk of changing existing behavior:** Low.
    - **Dependencies:** None.

---

### 4. Low-Priority Cleanup

14. **Prune Dead Code, Unused Dependencies, Debug Logs, and `Cogmit.zip`**
    - **Problem:** 36 ESLint unused-var warnings; dead `ui/select.tsx`, `addOrUpdateSteps`, `published = false`, `repoName`, `slug`, `ToolbarItem`, `test-parsers.mjs`, `src/yappout/index.ts`, `seekController.ts`, `speechSynthesisAdapter.ts`, `cogmitContentAdapter.ts`, `YappoutSegment.tsx`, `YappoutWord.tsx`, `#toggle-with-footer` hidden button; `shadcn` & unused `remark-html` in `dependencies`; `"cn": "^0.4.0"` wrapper; `Cogmit.zip` committed to Git; `console.log` debug statements; `--font-geist-mono` misnomer in `layout.tsx`.
    - **Evidence:** ESLint output, `git ls-files`, [`ShareCogmitModal.tsx:613-623`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ShareCogmitModal.tsx#L613-L623), [`package.json`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/package.json).
    - **Proposed correction:** Remove verified dead files/symbols/logs/`Cogmit.zip`, replace `"cn"` with standard `clsx` + `tailwind-merge` (or verify `cn` package necessity), rename `--font-geist-mono` to `--font-mono`, and fix `.markdown-content code` font family to `var(--font-mono)`.
    - **Expected benefit:** Cleaner repo, smaller dependency tree, 0 ESLint warnings.
    - **Risk of changing existing behavior:** Ensure external scripts/consumers don't import `yappout/index.ts` before pruning it.
    - **Dependencies:** Remediation #8 (so test cleanup aligns).

15. **Reconcile Docs, Routes, and Ephemeral UI Placeholders**
    - **Problem:** `docs/` and `README_CONTENT` reference old repo names (`YVSApps_Cogmit_Data`), `rootCogInfo.json`, and `children.json`; `/login` vs `/auth/signin` duplicate sign-in pages; `Pin`, `Like`, `Add to Reads`, `+ Child Cog`, and private `YappOut` exist as non-persistent or permanently disabled controls.
    - **Evidence:** [`src/lib/github.ts:581-701`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/github.ts#L581-L701), [`src/app/auth/signin/page.tsx`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/auth/signin/page.tsx), [`CogActionRow.tsx:57-72`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/CogActionRow.tsx#L57-L72), [`ViewCog.tsx:147,403-439`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/components/ViewCog.tsx#L147).
    - **Proposed correction:** Consolidate `/auth/signin` and `/login`; update `README_CONTENT` and `docs/` to match `YVSApps_Data_Cogmit_*` and current schema; hide or mark deferred buttons clearly per `docs/product.md` Section 7.
    - **Expected benefit:** Accurate onboarding/architecture docs and honest UI capabilities.
    - **Risk of changing existing behavior:** None on core persistence.
    - **Dependencies:** Product decision on placeholder UI buttons and legacy `/connect` + `/repo/*` routes.

---

### 5. Long-Term Architectural Improvements

16. **Resolve RootLayout Dynamic Auth Coupling & Public Caching Strategy**
    - **Problem:** `RootLayout` awaits `checkSessionStatus()`, invoking `cookies()` across every route including public Cogmit reader pages and `/aboutCogmit`, preventing static/ISR page caching and coupling public reading latency to session decoding.
    - **Evidence:** [`src/app/layout.tsx:37-49`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/app/layout.tsx#L37-L49).
    - **Proposed correction:** Move session-dependent header rendering (`HeaderControls` / `GlobalNav` auth state) into client hydration via `/api/auth/session-status` or route-group layouts (`(app)` authenticated vs `(public)` cacheable), keeping public Cogmit views cacheable at the edge.
    - **Expected benefit:** Sub-50ms edge caching for public Cogmit reader links and zero session overhead for anonymous readers.
    - **Risk of changing existing behavior:** Brief client auth-UI hydration state on public pages unless handled with skeleton/default header state.
    - **Dependencies:** Product decision on public vs private route shell rendering.

17. **Evaluate GitHub App Migration for Scoped Repository Permissions**
    - **Problem:** GitHub OAuth `repo` scope grants read/write access to all of a user's repositories rather than only `YVSApps_Data_Cogmit_*`.
    - **Evidence:** [`src/constants/index.ts:159`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts#L159), [`src/auth.ts:26`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts#L26).
    - **Proposed correction:** When Cogmit scales beyond single-user/hobby scope, migrate from a GitHub OAuth App to a GitHub App with fine-grained single-repository installation permissions.
    - **Expected benefit:** Principle of least privilege for user GitHub accounts.
    - **Risk of changing existing behavior:** Major onboarding UX change (GitHub App installation flow vs simple OAuth).
    - **Dependencies:** Requires architectural decision per `docs/architecture.md` Section 9.

---

## K. Overall Engineering Scorecard

| Category | Score (/10) | Reasoning |
|---|---|---|
| **Architecture** | **4.5 / 10** | Clean Next.js App Router foundation and isolated `yappout` module with zero import cycles, but severely undermined by competing publish pipelines, mutating read functions (`getCogmitDataStatus`), inverted `components → app` imports, and missing repository/service boundaries. |
| **Code organization** | **5.0 / 10** | Good separation of `src/constants/`, `src/lib/`, `src/yappout/`, and `src/components/`, offset by `newCog` vs `new` route mismatch, actions imported across unrelated routes, `Cogmit.zip` in root, and 700–900-line multi-concern files. |
| **Naming consistency** | **5.5 / 10** | Domain constants follow clear prefixes (`ERROR_*`, `ROUTE_*`, `DIR_*`, `FILE_*`), but naming drifts across `--font-geist-mono` (holding JetBrains Mono), `YVSApps_Cogmit_Data` vs `YVSApps_Data_Cogmit_*`, `CogmitPage` vs `ViewCog`, and mixed route casing (`/aboutCogmit`, `/myCogs`, `/cogmits`). |
| **Type safety** | **6.5 / 10** | `strict: true` and `tsc --noEmit` pass cleanly with only one `@ts-expect-error`; however, `JSON.parse` index payloads (`cogsIndex.json`, `cogmitsIndex.json`) lack runtime validation and rely on structural casts/unchecked optional indexing. |
| **Security** | **4.0 / 10** | Good practice keeping OAuth `accessToken` out of client `session` and XML-escaping SVG text, but compromised by unenforced `ALLOWED_GITHUB_USERS`, `/login` open redirect, Host header injection on `shareUrl`, missing path-traversal checks, and `allowDangerousHtml` + undeclared `dompurify`. |
| **Data integrity** | **3.5 / 10** | Critical defects: `saveRootCog` swallows `cogsIndex.json` read errors (`catch {}`), `deleteRootCog` leaves published Cogmits orphaned in the public repo, `moveCogToDestination` writes `"No Parent Project"` into index, non-atomic 3-commit publish flow, and truncated `createdAt` (>30 commits). |
| **Error handling** | **4.0 / 10** | Typed `AuthenticationRequiredError` and `RootCogNotFoundError` exist, but sibling Server Actions mix 3 incompatible failure contracts, bare `catch {}` hides GitHub failures, raw `alert()` is used in UI buttons, and raw fetch failures silently return empty Markdown. |
| **Reusability** | **5.0 / 10** | `src/constants/`, `CogWorkspaceShell`, `excerpt.ts`, and `card-renderer.ts` are cleanly reusable, yet Git Tree move logic is duplicated 4×, `stripMarkdown` 3×, modals 6×, marquee rows 2×, and base URL resolution 5×. |
| **Testing** | **1.5 / 10** | While 14 test files exist in `scripts/` and `src/yappout/tests/`, **zero execute successfully** via `node`, there is no `"test"` script or test runner, several assertions target removed UI/specs or test fake local helpers, and auth/GitHub actions have 0% coverage. |
| **Documentation** | **5.0 / 10** | `YVS-AGENTS.md`, `docs/AI-DEVELOPMENT.md`, and `Cogmit_in_Github_invariants.md` show strong intent, but `docs/architecture.md`, `README.md` (still default `create-next-app`), `README_CONTENT`, and prior audits have drifted significantly from actual code. |
| **Maintainability** | **4.5 / 10** | Failing ESLint (`2` errors, `36` warnings), 7 bundled Markdown parsers, dead components/scripts, stale test expectations, and 600–900-line client components make safe iteration slow and error-prone. |
| **Overall engineering maturity** | **4.5 / 10** | Functional personal prototype with thoughtful domain invariants and modular pockets (`constants`, `yappout`, `excerpt`), but lacking the reliability, security hardening, single-pipeline discipline, and automated verification needed for production or ecosystem reference status. |

---

## Final Summary: Standard vs. Correction vs. Permanent YetVast Rule

### 1. Already a Good Engineering Standard in Cogmit
- Pure, zero-dependency, secret-free [`src/constants/index.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/constants/index.ts) SSOT module.
- Session-ID fencing for async stateful browser APIs ([`playbackController.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/core/playbackController.ts), [`useYappout.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/yappout/hooks/useYappout.ts)).
- Keeping OAuth access tokens server-only in JWTs rather than client session payloads ([`src/auth.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/auth.ts)).
- Unicode-safe truncation with explicit post-condition invariant verification ([`src/lib/excerpt.ts`](file:///c:/Users/yviyv/yvscodes/YetVastSolutions/Cogmit/src/lib/excerpt.ts)).
- Strict `YVS-AGENTS.md` evidence taxonomy (`OBSERVED / DOCUMENTED / TESTED / INFERRED / UNKNOWN`) and *Inspect → Search → Reuse → Implement → Verify* workflow.

### 2. Needs Immediate Correction in Cogmit (Do Not Propagate)
- Enforce `ALLOWED_GITHUB_USERS` in `auth.ts`, fix `/login` open redirect, fix `shareUrl` Host header trust, and add `login` to session for `isAuthor`.
- Remove side-effecting writes (`reconcileArchitecture` / repo creation) from `getCogmitDataStatus()`.
- Consolidate publishing into one atomic server workflow, sync `deleteRootCog` with `cogmitsIndex.json`, fix `moveCogToDestination` `"No Parent Project"` bug, and remove bare `catch {}` blocks.
- Strip the 6 redundant client Markdown parsers + `allowDangerousHtml` + undeclared `dompurify` in `ViewCog.tsx`.
- Fix both ESLint `set-state-in-effect` errors, clean all 36 warnings, remove dead code/`Cogmit.zip`, and replace broken `.mjs` scripts with an actual runnable `vitest` suite.
- Consolidate the 4× duplicated Git Tree commit boilerplate, 6× custom modals, 2× `stripMarkdown` inside `yappout`, and drifted YVS CSS tokens.

### 3. Permanent YetVast-Wide Rules to Adopt Across All Products
1. **No Mutating Reads:** Status/query functions must have zero write side effects.
2. **One Pipeline Per Responsibility:** Never merge a new workflow while leaving an old competing path active in production code.
3. **Strict Layer Dependencies:** `app` → `components` → `lib` → `constants`; zero `components → app` imports.
4. **Uniform Mutation Result Contract & Zero Bare `catch {}`:** Every Server Action/API mutation returns a typed discriminated union; only expected error codes (e.g., `404`) may be handled conditionally.
5. **Security Boundary Defaults:** Mandatory relative-path checks on redirect parameters, trusted origin config instead of `Host` headers, auth allowlist enforcement at `signIn`, and runtime schema validation (`zod`) on all external/parsed JSON.
6. **Direct Dependencies & Single Parser Rule:** Never import transitive packages; never ship competing libraries for the same concern (e.g., multiple Markdown engines or UI icon sets).
7. **Green CI Triad Before Merge:** `tsc --noEmit` + `eslint` (0 errors) + `npm test` (real runner, isolated from external networks) must all pass cleanly.
8. **Design Tokens Live in Theme CSS Only:** All YVS applications consume semantic YVS token variables (`YVS-AGENTS.md` v1.2.2) in CSS rather than raw hex values inside components.