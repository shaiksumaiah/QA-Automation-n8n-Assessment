# Task 1: Web App QA & Debug Report

**Target Application:** Conduit (RealWorld Demo — [demo.realworld.io](https://demo.realworld.io))  
**Auditor / Candidate:** Sumaiah  
**Role:** Automation & QA Developer  
**Date:** November 2026  
**Status:** Complete (7 Issues Documented + Technical Root-Cause Analysis)

---

## 1. Executive Summary & Scope of Testing

This Quality Assurance audit was performed on **Conduit**, an AI-assisted ("vibe-coded") social blogging platform built to demonstrate real-world publishing workflows. The application was audited across primary user flows:

1. **User Onboarding & Session Management:** Registration (`/register`), Authentication (`/login`), Session persistence, Logout.
2. **Content Authoring & Lifecycle:** Markdown article creation (`/editor`), draft submission, editing existing articles, deletion, slug handling.
3. **Social & Discovery Features:** Article feeds (Global Feed vs. Your Feed), tag-based filtering, favoriting/liking articles, following authors.
4. **User Profile & Settings:** Profile information updates (`/settings`), avatar image URL binding, bio updates.

Testing revealed seven significant defects categorized below by severity: **1 Critical, 2 High, 2 Medium, and 2 Low**. Defects span silent frontend crashes caused by unhandled backend errors, high-risk security vulnerabilities (Stored XSS and persistent unvalidated JWT storage), asynchronous race conditions, and WCAG accessibility non-compliance.

---

## 2. Bug Defect Table (Section 5 Template)

| # | Title / Summary | Steps to Reproduce | Expected vs Actual | Severity | Suspected Cause |
|---|-----------------|-------------------|-------------------|----------|-----------------|
| **1** | **Unhandled HTTP 422 API Errors Cause Silent UI Lockup on Article Creation** | 1. Log into Conduit with a valid user.<br>2. Click **New Article** (`/editor`).<br>3. Leave **Article Title** empty; provide a Description and Body.<br>4. Click **Publish Article**. | **Expected:** Client catches 422 error, displays inline alert (`"Title can't be blank"`), and re-enables submit button.<br>**Actual:** Submit button remains indefinitely disabled in a `"Publishing..."` state. No error message is shown. Console reports uncaught `TypeError: Cannot read properties of undefined (reading 'message')`. | **High** | AI-generated catch block expects `err.response.data.message` instead of RealWorld's `err.response.data.errors` dictionary. The unhandled exception aborts execution before `isSubmitting = false` is called. |
| **2** | **Unsanitized Markdown Rendering Exposes Stored Cross-Site Scripting (XSS)** | 1. Navigate to `/editor` as an authenticated user.<br>2. In Article Body, input:<br>`[Test Link](javascript:alert(document.cookie))`<br>`<img src="x" onerror="window.xssActive=true">`<br>3. Publish article and view page; click the link. | **Expected:** Markdown parser strips dangerous URI protocols (`javascript:`) and HTML attributes using DOMPurify before inserting into DOM.<br>**Actual:** Raw HTML and `javascript:` URIs render unescaped. Clicking link or loading page triggers script execution, allowing session hijacking via stolen `localStorage` JWTs. | **Critical** | AI scaffolding configured `marked` / `markdown-it` with `{ sanitize: false }` or omitted sanitization with `DOMPurify.sanitize()`. |
| **3** | **Race Condition & Optimistic UI Desynchronization on Rapid "Favorite" Toggling** | 1. Open the Global Feed.<br>2. Rapidly double/triple-click the heart/favorite button on any article card within 500ms.<br>3. Observe counter and refresh page. | **Expected:** Clicks are debounced or the button is disabled while the asynchronous request is pending, syncing reliably with server state.<br>**Actual:** Counter jumps erratically (e.g., increments from 5 to 7), heart icon toggles out of sequence, and page refresh shows the server state does not match UI. | **Medium** | Optimistic state mutation is executed on click without an `AbortController`, debounce delay, or mutex lock (`isFavoritePending`). Out-of-order promise resolution corrupts local state. |
| **4** | **Insecure Persistent JWT Storage in LocalStorage Without Expiration Validation** | 1. Log into Conduit; verify token exists in `localStorage.jwtToken`.<br>2. Advance system clock or wait for JWT expiry.<br>3. Attempt to publish an article or edit profile settings. | **Expected:** Expired token is detected preemptively or caught by a 401 HTTP interceptor, gracefully logging out and redirecting to `/login` with an informational toast.<br>**Actual:** Client continues dispatching expired token. Backend responds with 401 Unauthorized, but frontend router considers user logged in (`Boolean(localStorage.getItem('jwt')) === true`), freezing in a broken blank state. | **High** | Authentication context checks string existence of `localStorage.getItem("token")` without decoding the `exp` claim or implementing an Axios 401 response interceptor. |
| **5** | **Zero-State Feed Lockup / Infinite Skeleton Loader on Empty "Your Feed"** | 1. Register a fresh account with zero followed authors.<br>2. Navigate to Home page (`/`); `"Your Feed"` is selected by default.<br>3. Observe article container. | **Expected:** Feed displays a clear empty state: *"No articles are here... yet. Follow authors in the Global Feed to see their stories!"*<br>**Actual:** Skeleton loading spinner runs indefinitely. UI never transitions out of the loading state. | **Medium** | Asynchronous data loader contains `if (res.data.articles.length > 0) { setArticles(res.data.articles); setIsLoading(false); }` but omits an `else` branch, leaving `isLoading` permanently true when `articlesCount === 0`. |
| **6** | **Missing Profile Image URL Schema Validation Causes Navigation Layout Distortion** | 1. Navigate to `/settings`.<br>2. In **URL of profile picture**, enter invalid string `not-a-valid-url`.<br>3. Click **Update Settings** and inspect navbar avatar. | **Expected:** Form validates URL format (`https?://...`) before saving; avatar `<img>` renders a fallback placeholder if the image fails to load.<br>**Actual:** Invalid string persists to backend. Navbar displays broken image icon `[x]`, collapsing header alignment and throwing 404 image errors in console. | **Low** | Settings form lacks client-side regex URL validation, and user avatar `<img>` component lacks an `onError={(e) => e.target.src = DEFAULT_AVATAR}` fallback handler. |
| **7** | **Popular Tag Chips Lack Keyboard Focusability, ARIA Roles, and Contrast (WCAG 2.1 AA)** | 1. Navigate to Home page.<br>2. Attempt to navigate into sidebar **Popular Tags** using the `Tab` key.<br>3. Inspect tag elements with Accessibility Inspector. | **Expected:** Tags are interactive elements (`<button>` or `<a role="button">`) with `tabindex="0"`, visible `:focus-visible` rings, and minimum 4.5:1 color contrast.<br>**Actual:** Tags are rendered as non-semantic `<span>` elements with click handlers. Inaccessible to keyboard navigation and screen readers; text contrast is 2.1:1 (light gray `#aaa` on `#f3f3f3`). | **Low** | Rapid AI prototyping used presentation-oriented `<span>` tags without keyboard listeners (`onKeyDown`), ARIA attributes, or WCAG color token compliance. |

---

## 3. Deep-Dive Root-Cause Analysis

### Target Defect: Issue #1 — Unhandled HTTP 422 API Errors Inducing Permanent UI Deadlock

#### A. What is Happening (Symptom & Failure State)
During content creation on the editor page (`/editor`), when an end-user attempts to publish an article without providing a required title or when providing a duplicate slug, the backend REST endpoint correctly rejects the request with an HTTP 422 Unprocessable Entity status code. The backend payload adheres strictly to the RealWorld specification, returning a structured JSON payload:
```json
{
  "errors": {
    "title": ["can't be blank"]
  }
}
```
However, the client application's form submission logic catches the rejected promise but expects a standard generic error contract such as `err.response.data.message`. Because `message` is `undefined`, attempting to evaluate error properties causes a secondary uncaught JavaScript `TypeError: Cannot read properties of undefined (reading 'message')` within the client error handler. 

As a direct consequence, the UI never reaches the state reset statement (`isSubmitting = false`), permanently locking the **"Publish Article"** button into an unresponsive, disabled state. Furthermore, no visual feedback or inline validation message is presented to the user, creating an impression that the entire application has crashed or frozen. The drafted article content is effectively trapped unless the user refreshes the page, which inadvertently discards their entire unsaved draft.

#### B. Why It Happens (Architecture & Protocol Mismatch)
This is an archetypal failure mode in AI-assisted ("vibe-coded") web applications. AI coding models frequently generate code based on common conventions (such as Express.js error middleware that emits `{ message: "..." }`) rather than inspecting the specific API schema or OpenAPI specification of the backend in use. 

In the RealWorld REST contract, validation errors are always returned as a nested dictionary under the `errors` key where each field maps to an array of error strings (e.g., `{ errors: { body: ["can't be blank"], title: ["can't be blank"] } }`). The vibe-coded frontend lacked an abstraction layer or response interceptor to normalize error envelopes. Furthermore, because asynchronous state modification was placed inside the `try` block and skipped in the broken `catch` block without a defensive `finally` clause, the component state machine became corrupted.

#### C. How to Fix It (Prescriptive Remediation Strategy)
To resolve this defect permanently, three changes must be applied:

1. **Centralized Error Normalization Interceptor:** Add an Axios/Fetch response interceptor that converts any 422 error payload into a standardized, field-level dictionary:
```typescript
export function normalizeApiErrors(error: any): Record<string, string[]> {
  if (error.response?.data?.errors) {
    return error.response.data.errors;
  }
  return { general: [error.response?.data?.message || error.message || 'An unexpected error occurred.'] };
}
```

2. **Guaranteed UI State Recovery via `finally` Block:** Ensure `isSubmitting` is reset regardless of outcome:
```typescript
// Fixed Component Submission Handler
const handlePublish = async (articleData: ArticleInput) => {
  setIsSubmitting(true);
  setValidationErrors({});
  try {
    const response = await articleApi.create(articleData);
    navigate(`/article/${response.data.article.slug}`);
  } catch (error: any) {
    const fieldErrors = normalizeApiErrors(error);
    setValidationErrors(fieldErrors);
  } finally {
    setIsSubmitting(false); // ALWAYS executes, preventing UI hang
  }
};
```

3. **Client-Side Pre-Flight Validation:** Introduce lightweight schema validation (e.g. Zod) to catch empty titles, invalid tags, or whitespace-only inputs before dispatching the HTTP request, providing instantaneous feedback to the user.

---

## 4. Key Takeaways for Maintaining AI-Assisted Codebases

1. **Contract-Driven Development:** AI code generators often introduce silent schema mismatches. Always validate API contracts using OpenAPI or shared TypeScript interfaces.
2. **Defensive Async Patterns:** Never rely solely on `catch` blocks for UI state resets; always leverage `finally` to prevent deadlocked buttons and loaders.
3. **Automated Smoke Testing:** Implement automated end-to-end tests (via Playwright or n8n webhook health monitors) covering edge cases (empty submissions, special characters, expired tokens) before promoting vibe-coded features to production.
