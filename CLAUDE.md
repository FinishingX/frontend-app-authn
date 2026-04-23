# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm start              # dev server on port 1999
npm run dev            # dev server at apps.local.openedx.io with MFE config API (requires Tutor)
npm test               # Jest with coverage
npm run lint           # ESLint
npm run lint:fix       # ESLint with auto-fix
npm run build          # production webpack build
npm run snapshot       # update Jest snapshots
npm run i18n_extract   # extract i18n strings to JSON
make validate          # full CI check: lint + test + build
```

Run a single test file:
```bash
npx fedx-scripts jest src/login/tests/LoginPage.test.jsx
```

## Architecture

This is an Open edX micro-frontend (MFE) for authentication: login, registration, forgot/reset password, progressive profiling, and post-registration course recommendations. It is built on React 18, Redux + redux-saga, and React Router 6. The platform adapter layer is `@edx/frontend-platform`.

### Route map (`src/MainApp.jsx`)

| Path | Component |
|---|---|
| `/login` | `Logistration` (login tab) |
| `/register` | `Logistration` (register tab) |
| `/register-embedded` | `RegistrationPage` (no chrome) |
| `/reset` | `ForgotPasswordPage` |
| `/password_reset_confirm/:token/` | `ResetPasswordPage` |
| `/welcome` | `ProgressiveProfiling` |
| `/recommendations` | `RecommendationsPage` |

### Feature-based ("ducks") module organization

Code is grouped by feature, not by type. Each feature directory follows this structure:

```
src/<feature>/
  index.js           # public interface — only import from here
  <FeaturePage>.jsx
  data/
    actions.js
    reducers.js       # exports reducer + storeName
    sagas.js          # exports default saga
    selectors.js
    service.js        # LMS API calls
  tests/
```

The root Redux store (`src/data/configureStore.js`) combines all feature reducers. The root saga (`src/data/sagas.js`) runs all feature sagas in parallel. Each feature's `index.js` exports `{ reducer, saga, storeName }` so the root can register them.

**Never import directly from a feature's internals** (e.g., `./login/data/reducers`). Always go through the feature's `index.js`.

### Shared context: Third-Party Auth (TPA)

`src/common-components/data/` fetches the shared auth context from `GET {LMS_BASE_URL}/api/mfe_context` on page load. This provides TPA provider configs (Google, Apple, SAML institutions), optional/required registration field descriptors, and CSRF tokens. All pages depend on this slice of the store via `tpaProvidersSelector` and `thirdPartyAuthContextSelector`.

### Logistration shell

`src/logistration/Logistration.jsx` is the unified login+register shell. It renders tab navigation and delegates to `LoginComponentSlot` (login) or `RegistrationPage` (register). Switching tabs dispatches `backupLoginForm` / `backupRegistrationForm` to preserve form state.

### Plugin slots

`src/plugin-slots/` wraps key components in `<PluginSlot>` from `@openedx/frontend-plugin-framework`. The login page is exposed as slot `org.openedx.frontend.authn.login_component.v1`. This allows operators to swap out or extend the login form without forking.

### BaseContainer and layouts

`src/base-container/` wraps every page and handles responsive layouts. Two layout modes exist:
- **Default layout** (`ENABLE_IMAGE_LAYOUT` unset/false): standard card layout at all breakpoints.
- **Image layout** (`ENABLE_IMAGE_LAYOUT=true`): side banner images at larger breakpoints, configured via `BANNER_IMAGE_*` env vars.

### Configuration

`src/config/index.js` defines all app-specific env vars that are merged into the platform config at startup via `mergeConfig`. Key feature flags:

| Variable | Effect |
|---|---|
| `ENABLE_DYNAMIC_REGISTRATION_FIELDS` | Enables backend-configured required fields beyond the defaults |
| `ENABLE_PROGRESSIVE_PROFILING_ON_AUTHN` | Redirects to `/welcome` after registration |
| `DISABLE_ENTERPRISE_LOGIN` | Hides enterprise SSO button |
| `ALLOW_PUBLIC_ACCOUNT_CREATION=false` | Forces login-only mode (hides register tab) |
| `SHOW_REGISTRATION_LINKS=false` | Hides the register tab without blocking the route |
| `ENABLE_IMAGE_LAYOUT` | Switches BaseContainer to image layout |

### LMS API endpoints

| Purpose | Method + Path |
|---|---|
| Login | `POST /api/user/v2/account/login_session/` |
| Register | `POST /api/user/v2/account/registration/` |
| Real-time field validation | `POST /api/user/v1/validation/registration` |
| TPA/MFE context | `GET /api/mfe_context` |
| Progressive profiling | `POST /api/user/v1/accounts/{username}` (PATCH) |

All requests go through `getAuthenticatedHttpClient()` from `@edx/frontend-platform/auth`. Registration and login POST bodies are `application/x-www-form-urlencoded`.

### Import ordering (ESLint enforced)

Imports must follow this order, alphabetically within each group:
1. React / react-dom / react-redux
2. Other external packages
3. Internal (`@edx/...`, `@openedx/...`)
4. Sibling/parent relative imports

### Query param preservation

`AUTH_PARAMS` (`course_id`, `enrollment_action`, `next`, etc.) are passed through all navigation calls via `updatePathWithQueryParams()` so downstream LMS enrollment flows survive the auth redirect.

### i18n

All user-visible strings use `react-intl` via `@edx/frontend-platform/i18n`. Each feature has a `messages.jsx` file. Translations are pulled from the OpenedX Atlas service via `make pull_translations`.

### Testing setup

Jest runs in `jsdom`. `src/setupTest.js` mocks `window.location`, `localStorage`, and `ResizeObserver`. Tests use `@testing-library/react`. Saga tests use `redux-saga/testing-utils`. Snapshots live alongside their test files in `__snapshots__/`.
