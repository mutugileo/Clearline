# ADR 0002: Front-end architecture for Clearline

- **Status:** Proposed
- **Date:** 2026-10-07
- **Deciders:** Clearline founding team (to confirm before the pilot)
- **Depends on:** ADR 0001 (platform architecture)

## Context

The Clearline dashboard is used by a bank's security team, usually for long sessions on desktop machines inside the bank's network. It shows findings, scan status and scope, and it exports audit reports.

The bank's procurement and accessibility teams will check it against WCAG 2.2 AA. Users will be signed in through the bank's identity provider. The app must work offline from the bank's own servers, with no calls to external CDNs or services.

The design blueprint calls for a restrained, high-signal interface, with tables as the main working surface.

## Decision drivers

1. Accessible by default (WCAG 2.2 AA), with keyboard support and screen-reader labels.
2. Secure session handling: no tokens in browser storage.
3. Fits a single-page app served from the bank's network, with no server-side rendering needed.
4. Large tables (thousands of findings) stay responsive.
5. Types stay in sync with the API, so the front end cannot drift from the backend.
6. Components the team can own and audit, with no hidden vendor code.

## Options considered

| Option | Summary | Verdict |
|---|---|---|
| A. Next.js (server rendering) | Full-stack React framework | Rejected. Server rendering adds a second runtime to operate, and the app is behind login anyway. |
| B. React single-page app with Vite | Static build served by the bank's web server or the API | **Chosen.** |
| C. Angular | Full framework with built-in structure | Viable. Rejected because the team's current skills and available components are stronger in React. |
| D. Vue | Lighter framework | Viable. Same reasoning as Angular. |
| E. Component library with heavy theming (MUI, Ant Design) | Ready-made components | Rejected. Harder to get the restrained look, and the components are large to audit. |
| F. shadcn/ui on Radix UI primitives | Components copied into our repo, built on accessible primitives | **Chosen.** |

## Decision

1. **Framework:** React with TypeScript, built with Vite. Output is static files, served from the bank's web server or from the API container. No server-side rendering.
2. **Styling:** Tailwind CSS with design tokens defined as CSS variables. Colours, type scale and spacing live in one token file that the blueprint references.
3. **Components:** shadcn/ui. Components are copied into `src/components/ui` and owned by us. Keyboard and focus behaviour come from Radix UI primitives.
4. **Icons:** Lucide.
5. **Routing:** TanStack Router, with route-level data loading and typed route parameters.
6. **Server state:** TanStack Query for all API data, with caching, retries and loading and error states in one place. Global client state is avoided. Zustand is allowed only for state that never comes from the server.
7. **Tables:** TanStack Table for sorting, filtering and column state. TanStack Virtual for lists longer than about 200 rows.
8. **Forms:** React Hook Form with Zod schemas. Validation runs inline on blur and before submit. Errors state the problem and the fix in one sentence.
9. **Charts:** Recharts for the trend line and simple bars. Colours come from the same token file, and each chart has a text alternative in its table view.
10. **API types:** TypeScript types are generated from the backend's OpenAPI document using `openapi-typescript`. The build fails if the generated types are out of date.
11. **Authentication:** backend-for-frontend pattern. The API handles OpenID Connect or SAML with the bank's identity provider and keeps tokens on the server. The browser receives an HTTP-only, Secure, SameSite session cookie. Tokens are never stored in `localStorage` or `sessionStorage`.
12. **Session handling:** idle timeout and an on-screen warning before sign-out. The timeout is set by the bank's policy.
13. **Accessibility:** axe-core runs in the test suite on every main screen. Keyboard navigation and focus order are tested with Playwright. Colour is never the only signal for severity.
14. **Testing:** Vitest and React Testing Library for components, Playwright for end-to-end flows, and a visual check at phone and desktop widths.
15. **Internationalisation:** English only for the pilot. Strings are kept in one place so i18next can be added later without rewriting components.
16. **Build and delivery:** the front-end build is a container image, signed and included in the SBOM, like the back end. Dependencies are locked and scanned.

## Consequences

**Positive**
- Accessibility is checked on every change, not at the end.
- No tokens in the browser reduces the risk of session theft.
- Owned components make audits simpler: the bank can read every line the dashboard runs.
- Generated types stop the front end and back end drifting apart.

**Negative**
- The BFF adds a small amount of backend code to maintain.
- Copying components means we must update them by hand when upstream fixes bugs.
- No server rendering means the first page load depends on the JavaScript bundle. Keep the bundle small and split by route.

**Risks to watch**
- Long tables can slow the browser. The virtualisation threshold should be tested with 10,000 rows before the pilot.
- Screen-reader behaviour for charts needs a real user test, not only automated checks.

## Requirements this ADR must meet

- [ ] Passes axe-core with no serious or critical issues on every main screen
- [ ] No tokens in browser storage
- [ ] Works with no internet access
- [ ] Generated API types match the backend's OpenAPI document in CI
- [ ] Usable at 360 px width with no horizontal scrolling
- [ ] Table with 10,000 findings stays responsive when scrolled and sorted

## Open questions

1. Which identity provider does the first pilot bank use? This affects the BFF login flow.
2. Does the bank require a specific browser list? Confirm before the accessibility test plan is written.
3. Will the bank's security team need to read the front-end source during procurement? If yes, keep the repository clean from the first commit.

## Related

- ADR 0001: Platform architecture
