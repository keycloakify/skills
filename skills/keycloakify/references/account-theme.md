# Account theme

First ask whether an account theme is needed at all. Many needs, such as changing a password, email or profile, can be covered by login-theme pages like `login-update-profile.ftl` and `login-update-password.ftl`, triggered through required actions.

## Initialize

```bash
npx keycloakify initialize-account-theme   # prompts: Single-Page or Multi-Page
```

This creates `src/account/` and sets `accountThemeImplementation` in `vite.config.ts`. **Never edit that value by hand.** It also changes which jars `keycloakVersionTargets` can produce.

## Single-Page (account v3, the modern Keycloak UI)

- This is the default account UI since Keycloak 25. Per the docs it works back to Keycloak 19.
- React only. Storybook isn't supported, so preview with `npx keycloakify start-keycloak`.
- It is built on `@keycloakify/keycloak-account-ui`. Files are managed by Keycloakify until you take ownership of them:
  ```bash
  npx keycloakify own --path "account/KcPage.tsx"
  npx keycloakify own --path "account/root/Header.tsx"
  ```
  Each managed file shows its exact `own` command in a header comment.
- To change the logo:
  1. Own `src/account/root/Header.tsx`.
  2. Add the image to `src/account/assets/`.
  3. Change the import.
- PatternFly components are re-exported from `src/shared/@patternfly/...`. To swap a component:
  1. Own that index file.
  2. Write your component.
  3. Re-export it from there.

## Multi-Page (account v1 style, maintained by Keycloakify)

- Works like the login theme: a `KcPage` that switches on `kcContext.pageId` (for example `account.ftl`, `password.ftl`, `totp.ftl`), plus `classes`, `doUseDefaultCss`, `eject-page`, i18n and Storybook.
- Supports React, Angular and Svelte. Per the docs it is compatible with all Keycloak versions.
- It lacks some features of the modern UI. Add them yourself through the Keycloak Account REST API if needed.
- For i18n, import `i18nBuilder` from `keycloakify/account`.

## Enabling it

The account theme is set per realm only, under Realm Settings → Themes → Account theme. It can't be set per client.

Docs: https://docs.keycloakify.dev/theme-types/account-theme
