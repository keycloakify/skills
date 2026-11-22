---
name: keycloakify
description: Build and customize Keycloak themes with Keycloakify, covering the login, registration, password reset, OTP, account console and email pages. Includes branding and styling with any CSS approach or component library, translations, runtime-configurable theme options, previewing pages, and building and deploying the theme to Keycloak. Use whenever the user wants to change how Keycloak's login or sign-up screens, account pages or emails look or behave, add a field or a custom page to the login flow, translate the login page, or is working in a project that uses Keycloakify, even if they only say "Keycloak theme" or "customize the Keycloak login page".
license: MIT
metadata:
  version: "1.1"
  keycloakify-version: "11"
---

# Keycloakify

Keycloakify lets you write Keycloak themes as a Vite + React app. It compiles them into a `.jar` that Keycloak loads as a theme provider.

At runtime, Keycloak injects `window.kcContext`. That is the server data for the current page: realm, URLs, form state, field errors and messages. Your app renders the page that matches `kcContext.pageId`.

## Step 0: identify the project's architecture

Read `package.json` and `src/login/` before changing anything. There are two architectures, and their code isn't interchangeable.

| Signal | Architecture | Follow |
|---|---|---|
| Depends on `@keycloakify/login-ui`; has a `src/.gitignore` "managed by Keycloakify"; has `src/login/pages/<page-id>/Page.tsx`; has a `"postinstall": "keycloakify sync-extensions"` script | **Extension model** (current, recommended) | This file plus the references |
| Imports from `keycloakify/login` (`DefaultPage`, `Template`); KcPage has props like `i18n` and `classes`; ejected pages are in `src/login/pages/Login.tsx` | **Legacy model** | [references/legacy-api.md](references/legacy-api.md) |

Unless the user says otherwise, new projects should use the extension model.

## The extension model in one minute

- **UI code lives in your repo.** The `@keycloakify/login-ui` npm package (and `keycloak-account-ui` for the account theme) ships the page source. `keycloakify sync-extensions` runs on postinstall and copies it into `src/`.
- **Ownership** is tracked in `src/.gitignore`, which Keycloakify manages (never edit it by hand):
  - **Not owned:** the file is git-ignored and gets overwritten on every sync. Never edit these files.
  - **Owned:** the file is committed and yours to change. Claim a file with:
    ```bash
    npx keycloakify own --path "login/pages/login/Page.tsx"           # claim one file
    npx keycloakify own --path "login/pages/login"                    # claim a folder
    npx keycloakify own --path "login/pages/login/Page.tsx" --revert  # give it back
    npx keycloakify own --path "login/css/login.css" --public         # file under public/keycloak-theme/
    ```
    Every synced file starts with a header comment showing the exact `own` command for it. `sync-extensions` also adds the extensions' peer dependencies to `package.json`.
- **Git is required.** It is how owned files are told apart from synced ones. Commit before running `init`, `own` or a dependency upgrade.
- **`src/kc.gen.tsx` is generated** (by `update-kc-gen`). It exports `ThemeName`, `themeNames`, `KcEnvName`, `kcEnvDefaults` and the root `KcPage` that switches on `kcContext.themeType`. Never edit it.

## Project layout (reference implementation)

```
src/
  main.tsx              # window.kcContext ? import("./main-kc") : import("./main-kc.dev")
  main-kc.tsx           # renders <KcPage kcContext={window.kcContext}/> from kc.gen
  main-kc.dev.tsx       # renders with getKcContextMock({ pageId: "login.ftl" }) for `vite dev`
  kc.gen.tsx            # GENERATED
  kc-env.ts             # typed theme env vars: defaults, parse/serialize (see environment-variables.md)
  components/ui/        # shared design-system components (whatever library you use)
  lib/                  # utils (cn, resolveAssetUrl, color scheme…)
  login/
    KcPage.tsx          # providers: KcContextProvider → I18nProvider → StyleLevelCustomization → PageIndex
    KcContext.ts        # ExtendKcContext<KcContextExtension, KcContextExtensionPerPage>
    i18n.ts             # i18nBuilder → { I18nProvider, useI18n }
    styleLevelCustomization.tsx   # doUseDefaultCss, classes, global CSS import, theme Provider
    useKcEnv.ts         # typed access to kcContext.properties
    index.css           # global styles entry
    pages/
      PageIndex.tsx     # switch (kcContext.pageId) → lazy(() => import("./<page-id>"))
      login/            # one folder per page id (login.ftl → login/)
        index.ts        # export default Page
        Page.tsx        # page shell: <Template headerNode=… > <Form/> </Template>
        Form.tsx        # page-specific parts split out
        Page.stories.tsx
    components/
      Template/         # shared frame: layout, logo, alerts, language switcher
      UserProfileFormFields/
    mocks/
      getKcContextMock.ts   # createGetKcContextMock(...) with extension defaults
      KcPageStory.tsx       # createKcPageStory({ pageId })
  account/              # Single-Page account UI (owned files only)
  email/                # jsx-email templates (keycloakify-emails)
public/keycloak-theme/  # static assets served with the theme (login/, account/), partially managed
```

A complete real-world example is [keycloakify-editor/packages/shadcn-theme](https://github.com/Oussemasahbeni/keycloakify-editor/tree/feature/editor/packages/shadcn-theme): every login page, a Single-Page account theme and jsx-email templates (it happens to use Tailwind and shadcn/ui). Mirror its structure and practices when the user's project has no conventions of its own, but keep the user's styling stack.

## Writing a page: the canonical pattern

```tsx
// src/login/pages/login/Page.tsx
import { assert } from "tsafe/assert";
import { useKcContext } from "#/login/KcContext.ts";
import { useI18n } from "#/login/i18n.ts";
import { Template } from "../../components/Template";
import { Form } from "./Form";

export function Page() {
  const { kcContext } = useKcContext();
  assert(kcContext.pageId === "login.ftl"); // narrows kcContext to this page's type

  const { msg } = useI18n();

  return (
    <Template
      headerNode={msg("loginAccountTitle")}
      displayMessage={!kcContext.messagesPerField.existsError("username", "password")}
      displayInfo={kcContext.realm.password && kcContext.realm.registrationAllowed && !kcContext.registrationDisabled}
      infoNode={<Info />}
    >
      <Form />
    </Template>
  );
}
```

Rules:
- **Use hooks, not props.** Read data with `useKcContext()` and `useI18n()`, and narrow with `assert(kcContext.pageId === "...")` in each component that needs page-specific fields.
- **Keep the Keycloak form contract**:
  - `action={kcContext.url.loginAction}` (or the page's own action URL, such as `registrationAction`) with `method="post"`
  - the original `name` attributes (`username`, `password`, `rememberMe`, `credentialId`, …) and the hidden inputs
  - the submit button's `name`/`value`

  Change the markup and styling freely, but Keycloak parses those fields on the server.
- **Show field errors** with `messagesPerField.existsError(...)` and `getFirstError(...)`, plus `aria-invalid` and `aria-live="polite"`. Wrap server HTML with `kcSanitize` (`@keycloakify/login-ui/kcSanitize`) before using `dangerouslySetInnerHTML`.
- **Prevent double submits** by disabling the submit button in `onSubmit`, then `return true`.
- **Split big pages.** `Page.tsx` is the shell, and the parts go in sibling files (`Form.tsx`, `Info.tsx`, `SocialProviders.tsx`).
- **Every page gets a `Page.stories.tsx`** with one story per meaningful state. See [references/testing.md](references/testing.md).

## Pick the lightest customization that works

1. **Theme-wide look**: global styles and providers in `styleLevelCustomization.tsx`, using the project's CSS approach or component library. See [references/styling.md](references/styling.md).
2. **Text**: `withCustomTranslations` in `src/login/i18n.ts`. See [references/i18n.md](references/i18n.md).
3. **Runtime-configurable branding** (logo, colors, layout) without a rebuild: theme environment variables. See [references/environment-variables.md](references/environment-variables.md).
4. **The frame shared by every page** (logo, header, alerts, language switcher): own and edit `login/components/Template/`.
5. **A single page**: own `login/pages/<page-id>/` and edit it.
6. **Fields**: configure realm User Profile attributes first, then style `UserProfileFormFields`. See [references/login-theme.md](references/login-theme.md).
7. **Pages added by Keycloak extensions** (magic link, captcha, and so on): extend `KcContext` and add a page folder. See [references/login-theme.md](references/login-theme.md).

## Core CLI

| Command | Use |
|---|---|
| `npx keycloakify init` | (beta) Set up a theme type (login, account, admin or email) in a project: picks extensions, adds stories |
| `npx keycloakify own --path "<path>" [--revert]` | Claim (or give back) a synced file or folder |
| `npx keycloakify sync-extensions` | Re-sync extension sources (runs on postinstall) |
| `npx keycloakify add-story` | Add Storybook stories for pages |
| `npx keycloakify update-kc-gen` | Regenerate `src/kc.gen.tsx` (normally automatic) |
| `npx keycloakify start-keycloak` | Run a real Keycloak in Docker with the theme and hot rebuild |
| `npm run build-keycloak-theme` | `tsc -b && vite build && keycloakify build` → jars in `dist_keycloak/` |

`npx keycloakify --help` lists everything. Always call the CLI through `npx`, `pnpm exec` or the equivalent.

## Workflow for a typical request

1. Check the architecture (Step 0) and whether the target file is owned. If it isn't, run `npx keycloakify own --path ...` first.
2. Reproduce the page in Storybook: `npm run storybook`, story `login/<page-id>`.
3. Make the change using the lightest option from the list above.
4. Add or extend stories for the new states.
5. Run `tsc -b` (or `npm run build`) and lint.
6. Check in a real Keycloak with `npx keycloakify start-keycloak` (realm `myrealm`, user `testuser` / `password123`).
7. Build with `npm run build-keycloak-theme`. See [references/build-deploy.md](references/build-deploy.md).

## Pitfalls

- **Editing a file you don't own.** The next `pnpm install` silently reverts it. Check `src/.gitignore` first.
- **Translations must be inline literals** in `withCustomTranslations`, because the build analyzes the code statically.
- **Server-defined labels** (User Profile display names, `messageHeader`) must be rendered with `advancedMsg(...)`.
- **`kcContext.properties` values are always strings.** Parse them in one place (`useKcEnv`). Don't sprinkle `=== "true"` checks through the code.
- **Don't use the `master` realm** for apps.
- **Missing `case` in `PageIndex.tsx`.** A new page folder does nothing until `PageIndex.tsx` has a `case` for its pageId.
- **Works in Storybook but not in Keycloak.** The mock differs from the real `kcContext`. Reproduce with `start-keycloak`.

## Reference files

Read only the ones the task needs:

- [references/setup.md](references/setup.md): starting a project, `init`, `vite.config.ts` options, aliases, scripts
- [references/login-theme.md](references/login-theme.md): KcPage and providers, PageIndex, Template, forms, User Profile fields, custom KcContext and extension pages
- [references/styling.md](references/styling.md): `styleLevelCustomization`, choosing a styling stack (plain CSS, Tailwind, component libraries, CSS-in-JS), dark mode, RTL, assets
- [references/environment-variables.md](references/environment-variables.md): typed theme env vars (`kc-env.ts`, `useKcEnv`)
- [references/i18n.md](references/i18n.md): translations, extra languages, per-variant text, realm overrides, language switcher
- [references/testing.md](references/testing.md): Storybook stories, mocks, `start-keycloak`
- [references/account-theme.md](references/account-theme.md): Single-Page and Multi-Page account themes
- [references/email-theme.md](references/email-theme.md): jsx-email via keycloakify-emails, native FreeMarker
- [references/build-deploy.md](references/build-deploy.md): `keycloakVersionTargets`, `postBuild`, jars, Docker, enabling the theme
- [references/troubleshooting.md](references/troubleshooting.md): common errors
- [references/legacy-api.md](references/legacy-api.md): projects on `keycloakify/login` (DefaultPage, eject-page) and the v10 → v11 migration

If something isn't covered or looks outdated, read the official docs (index: https://docs.keycloakify.dev/llms.txt) and the source the extension synced into `src/`. Don't guess.
