# Account theme

First ask whether you need an account theme at all. Profile, password and email changes can often be handled by login-theme pages (`login-update-profile.ftl`, `login-update-password.ftl`, `update-email.ftl`) triggered through required actions.

## Initialize

```bash
npx keycloakify init   # or: npx keycloakify initialize-account-theme
```

- Choose **Single-Page** (the modern Keycloak account console) or **Multi-Page** (the classic account UI, maintained by Keycloakify).
- This sets `accountThemeImplementation` in `vite.config.ts`. **Don't edit that value by hand.** It also controls which `keycloakVersionTargets` you can use.

## Single-Page (recommended for Keycloak 25+)

- It is built from the `@keycloakify/keycloak-account-ui` extension, plus `@keycloakify/keycloak-ui-shared` for shared components. The sources sync into `src/account/` and `src/shared/`.
- `src/.gitignore` tracks ownership. Claim only what you change:
  ```bash
  npx keycloakify own --path "account/root/Header.tsx"
  npx keycloakify own --path "account/personal-info"            # a whole folder
  npx keycloakify own --path "account/root/Header.tsx" --revert
  ```
  Each owned file starts with a header comment that names the version it was claimed from and the exact `--revert` command.
- `account/KcPage.tsx` runs oidc-spa's `oidcEarlyInit` before loading the UI, because the account console is an OIDC SPA. Keep that block when you own the file:
  ```tsx
  const { shouldLoadApp } = oidcEarlyInit({
    BASE_URL: kcContext.baseUrl.path,
    sessionRestorationMethod: import.meta.env.DEV ? "full page redirect" : "auto",
    securityDefenses: { ...browserRuntimeFreeze({ excludes: ["fetch"] }), ...DPoP({ mode: "auto" }) }
  });
  if (!shouldLoadApp) return null;
  return <KcAccountUiLoader kcContext={kcContext} KcAccountUi={KcAccountUi} />;
  ```
- **Restyling:** own `account/root/*` (Header, PageNav, Root) and the page components (`personal-info/`, `account-security/`, `applications/`, …).
  - You can replace PatternFly with your own design system (for example the shadcn components in `src/components/ui/`, shared with the login theme).
  - Keep the data hooks and API calls from `account/api/` (`getPersonalInfo`, `savePersonalInfo`, …).
- **Typing:** extend `account/KcContext.ts` so it includes your env vars: `properties: Record<KcEnvName, string>`.
- **Translations:**
  - The Single-Page account UI uses `.properties` bundles from the extension, plus i18next.
  - Override keys in a separate override file, as described in the header of each synced translation file.
  - Don't own the base bundles.
- **Preview:** there is no Storybook support by default. Use `npx keycloakify start-keycloak`, or build a custom preview harness with a mocked context (the reference project has `account/preview/`).
- Some pages depend on Keycloak features. For example, Verifiable Credentials needs `--features=oid4vc-vci`. Enable them through `startKeycloakOptions.keycloakExtraArgs`.

## Multi-Page

- Works like the legacy login theme: a `KcPage` that switches on `pageId` (`account.ftl`, `password.ftl`, `totp.ftl`, …).
- Supports Storybook and `classes` / `doUseDefaultCss`.
- Works with every Keycloak version, but lacks the newer features.

## Enabling

Set it under Realm Settings → Themes → **Account theme**. It is realm-level only.

Docs: https://docs.keycloakify.dev/theme-types/account-theme
