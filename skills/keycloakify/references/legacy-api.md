# Legacy API (`keycloakify/login`)

This file is for projects that import `DefaultPage`, `Template` and `i18nBuilder` from `keycloakify/login`, and customize pages with `npx keycloakify eject-page`. For new work, prefer the extension model described in SKILL.md.

## KcPage

`src/login/KcPage.tsx` switches on `kcContext.pageId` and passes everything down as props:

```tsx
import { Suspense, lazy } from "react";
import type { ClassKey } from "keycloakify/login";
import type { KcContext } from "./KcContext";
import { useI18n } from "./i18n";
import DefaultPage from "keycloakify/login/DefaultPage";
import Template from "keycloakify/login/Template";
import "./main.css";

const UserProfileFormFields = lazy(() => import("keycloakify/login/UserProfileFormFields"));
const Login = lazy(() => import("./pages/Login")); // ejected

export default function KcPage(props: { kcContext: KcContext }) {
  const { kcContext } = props;
  const { i18n } = useI18n({ kcContext });

  return (
    <Suspense>
      {(() => {
        switch (kcContext.pageId) {
          case "login.ftl":
            return <Login {...{ kcContext, i18n, classes }} Template={Template} doUseDefaultCss={true} />;
          default:
            return (
              <DefaultPage
                kcContext={kcContext}
                i18n={i18n}
                classes={classes}
                Template={Template}
                doUseDefaultCss={true}
                UserProfileFormFields={UserProfileFormFields}
                doMakeUserConfirmPassword={true}
              />
            );
        }
      })()}
    </Suspense>
  );
}

const classes = {} satisfies { [key in ClassKey]?: string };
```

## Customizing

- **CSS:**
  - Target the `kc*` classes in `main.css`.
  - Remap or empty the Patternfly classes through `classes`, for example `kcButtonClass: ""` or `kcLabelClass: "form-label"`.
  - `doUseDefaultCss={false}` drops Patternfly entirely, for every page or only in one `case`.
- **Eject:** run `npx keycloakify eject-page` and pick a page, `Template.tsx` or `UserProfileFormFields.tsx`. Then add a `case "<pageId>":` in `KcPage.tsx`, otherwise the ejected file is unused.
- **`kcClsx`:** if you remove the `kcClsx(...)` calls from an ejected component, `kc*` CSS no longer applies to it.

## i18n (legacy)

```ts
import { i18nBuilder } from "keycloakify/login"; // or "keycloakify/account"
import type { ThemeName } from "../kc.gen";

const { useI18n, ofTypeI18n } = i18nBuilder
  .withThemeName<ThemeName>()
  .withCustomTranslations({ en: { loginAccountTitle: "Log in to ACME" } })
  .build();

type I18n = typeof ofTypeI18n;
export { useI18n, type I18n };
// usage: const { i18n } = useI18n({ kcContext }); then pass i18n as a prop
```

Everything else in [i18n.md](i18n.md) still applies: inline literals, `withExtraLanguages` and per-variant objects.

## Storybook (legacy)

```tsx
const { KcPageStory } = createKcPageStory({ pageId: "register.ftl" });
export const Default: Story = { render: () => <KcPageStory /> };
export const French: Story = { render: () => <KcPageStory kcContext={{ locale: { currentLanguageTag: "fr" } }} /> };
```

## Migrating to the extension model

1. Commit everything.
2. Follow the official migration guide (https://docs.keycloakify.dev). It installs `@keycloakify/login-ui` and runs `npx keycloakify init`.
3. Port each ejected page into `src/login/pages/<page-id>/Page.tsx`:
   - Replace `props.kcContext` / `props.i18n` with `useKcContext()` / `useI18n()`, plus `assert(kcContext.pageId === ...)`.
   - Replace `Template` props with the owned `components/Template`.
4. Move the `classes` / `doUseDefaultCss` settings into `styleLevelCustomization.tsx`.

## Migration v10 → v11 (legacy API)

- **i18n:** `createUseI18n(...)` becomes `i18nBuilder.withThemeName<ThemeName>().withCustomTranslations({...}).build()`.
- **Ejected `Template.tsx`:**
  - `currentLanguageTag` becomes `currentLanguage` plus `enabledLanguages`.
  - The `html.lang` effect was removed.
  - Use `useInitialize({ kcContext, doUseDefaultCss })` from `keycloakify/login/Template.useInitialize`.
- **Sanitizing:** use `kcSanitize` from `keycloakify/lib/kcSanitize` for `dangerouslySetInnerHTML`.
- **Webpack only:** `keycloakify-resources` was renamed to `keycloakify-dev-resources`.
- **`keycloakVersionTargets`:** the keys changed if you came from 10.1.4 or earlier.
