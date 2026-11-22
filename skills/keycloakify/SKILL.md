---
name: keycloakify
description: Build, customize, test and deploy custom Keycloak themes (login, account, email, admin) with React, Angular or Svelte using Keycloakify v11. Use when the user mentions Keycloakify, a Keycloak theme or login/register page customization, kcContext, KcPage, pageId like "login.ftl", eject-page, i18n/translations (i18nBuilder, msg, adding a language), keycloak-theme jar, dist_keycloak, or runs `npx keycloakify` commands.
license: MIT
metadata:
  version: "1.0"
  keycloakify-version: "11"
---

# Keycloakify

Keycloakify lets you write Keycloak themes as a normal Vite app (React, Angular or Svelte). Then it compiles them into a `.jar` that Keycloak loads as a theme provider. At runtime Keycloak injects a `kcContext` object (the server-side data for the current page: realm, URLs, messages, form state, and so on). Your app renders the right page from `kcContext.pageId`.

## Mental model

```
src/login/KcPage.tsx  ── switch (kcContext.pageId) ──►  your page | <DefaultPage/>
        ▲                                                       │
 kcContext (injected by Keycloak in prod,                       ▼
 mocked by getKcContextMock / Storybook in dev)     npm run build-keycloak-theme
                                                                │
                                         dist_keycloak/keycloak-theme-for-kc-*.jar
                                                                │
                                              /opt/keycloak/providers/  →  select theme in realm
```

- **Login theme** (`src/login/`): login, register, reset-password, OTP and other pages. This is the most common target.
- **Account theme** (`src/account/`): optional. It's added with `npx keycloakify initialize-account-theme`.
- **Email theme** (`src/email/`): optional. It's added with `npx keycloakify initialize-email-theme`.
- **Admin theme**: React only, and rarely needed.

## Before you change anything

1. Check the project setup. Look for `keycloakify` in `package.json` and `keycloakify({...})` in `vite.config.ts`. Note the framework (React, Angular or Svelte).
2. Always run the CLI as `npx keycloakify <command>`. `npx keycloakify --help` lists the commands.
3. Never hand-edit generated or managed things:
   - `kc.gen.*` files
   - `public/keycloakify-dev-resources`
   - the `accountThemeImplementation` value
   - files in an initialized account theme you haven't "owned" yet (use `npx keycloakify own --path "<file>"`)
4. Building the jar needs **Maven** (`mvn` on PATH) and a JDK. `start-keycloak` also needs **Docker**.

## Pick the lightest customization that works

Try these in order:

1. **CSS only**:
   - Target the `kc*` classes in `src/login/main.css`, imported at the top of `KcPage.tsx`.
   - Remap or empty the Patternfly classes through the `classes` object.
   - Set `doUseDefaultCss={false}` to drop the Patternfly styles entirely.
2. **Text only**: change or add messages through `i18nBuilder().withCustomTranslations(...)` in `src/login/i18n.ts`. See [references/i18n.md](references/i18n.md).
3. **Override one page**: run `npx keycloakify eject-page`, choose the page (for example `login.ftl`), then add a `case "login.ftl":` to the switch in `KcPage.tsx`.
4. **Override the layout**: eject `Template.tsx` to change the shared frame (header, logo, language switcher) for every page.
5. **Server-side data or fields**: configure Keycloak itself, for example realm User Profile attributes for register fields or realm localization. Do this before writing client code.

Details: [references/login-theme.md](references/login-theme.md)

## Core CLI

| Command | Use |
|---|---|
| `npx keycloakify add-story` | Add Storybook stories for chosen pages (fastest dev loop) |
| `npm run storybook` | Preview pages with mocked `kcContext` |
| `npx keycloakify start-keycloak` | Run a real Keycloak in Docker with the theme and hot rebuild |
| `npx keycloakify eject-page` | Copy a default page or component into `src/` to customize it |
| `npx keycloakify initialize-account-theme` | Add an account theme (Single-Page or Multi-Page) |
| `npx keycloakify initialize-email-theme` | Add an email theme (`native` = FreeMarker) |
| `npx keycloakify own --path "<file>"` | Take ownership of a managed account-theme file |
| `npm run build-keycloak-theme` | `vite build && keycloakify build` → jars in `dist_keycloak/` |

## Workflow for a typical request

1. Reproduce the page in Storybook. Use `npx keycloakify add-story` if no story exists yet, then `npm run storybook`.
2. Make the change at the lightest level from the list above.
3. Add story variants for edge cases: errors, another locale, social providers. Pass a partial `kcContext` to `<KcPageStory kcContext={{...}} />`.
4. Check the theme in a real Keycloak with `npx keycloakify start-keycloak`. Use the test user `testuser` / `password123` in the `myrealm` realm.
5. Build with `npm run build-keycloak-theme` and deploy the jar that matches the target Keycloak version.

## Pitfalls

- **Translations must be inline literals** in `withCustomTranslations`. The build analyzes the code statically, so you can't import them from JSON files.
- **Server-defined keys** aren't type-checked. This covers User Profile display names and `termsText`. Render them with `advancedMsg(...)`, not `msg(...)`.
- **Sanitize HTML.** Wrap server-provided HTML with `kcSanitize` (from `keycloakify/lib/kcSanitize`) before using `dangerouslySetInnerHTML`.
- **Don't customize the `master` realm** for apps. Create a dedicated realm and set the theme under Realm Settings → Themes, or per client.
- **Jar choice.** `keycloak-theme-for-kc-22-to-25.jar` is for Keycloak 22–25. `keycloak-theme-for-kc-all-other-versions.jar` is for Keycloak 11–21 and 26+. The exact names depend on `keycloakVersionTargets`, so check `dist_keycloak/`.
- **Storybook vs. Keycloak.** If a page works in Storybook but not in Keycloak, the real `kcContext` differs from the mock. Test with `start-keycloak` and read the Keycloak logs.
- **Angular**: Keycloakify can't be added to an existing Angular app. Use the dedicated starter, either standalone or as a monorepo subproject.

## Reference files

Read only the ones the task needs:

- [references/setup.md](references/setup.md): starters, prerequisites, `vite.config.ts` options, project layout
- [references/login-theme.md](references/login-theme.md): KcPage, CSS, `classes`, eject-page, Template, register fields
- [references/i18n.md](references/i18n.md): translations (override or add keys, extra languages, per-variant text, realm overrides, `msg` vs `advancedMsg`, previewing locales)
- [references/account-theme.md](references/account-theme.md): Single-Page vs Multi-Page account themes
- [references/email-theme.md](references/email-theme.md): native FreeMarker vs jsx-email
- [references/testing.md](references/testing.md): Storybook, `getKcContextMock`, `start-keycloak`, `startKeycloakOptions`
- [references/build-deploy.md](references/build-deploy.md): build output, Docker, bare metal, Helm, enabling the theme
- [references/troubleshooting.md](references/troubleshooting.md): common errors and the v10 → v11 migration

If something isn't covered here or looks outdated, fetch the official docs index at https://docs.keycloakify.dev/llms.txt and read the relevant page. Don't guess.
