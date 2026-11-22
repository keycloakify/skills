# Login theme

## KcPage: the router

`src/login/KcPage.tsx` gets `kcContext` and switches on `kcContext.pageId`, for example `"login.ftl"`, `"register.ftl"` or `"login-reset-password.ftl"`. Any page you haven't ejected falls through to `DefaultPage`.

```tsx
import { Suspense, lazy } from "react";
import type { ClassKey } from "keycloakify/login";
import type { KcContext } from "./KcContext";
import { useI18n } from "./i18n";
import DefaultPage from "keycloakify/login/DefaultPage";
import Template from "keycloakify/login/Template";
import "./main.css";

const UserProfileFormFields = lazy(() => import("keycloakify/login/UserProfileFormFields"));
const Login = lazy(() => import("./pages/Login")); // ejected page

const doMakeUserConfirmPassword = true;

export default function KcPage(props: { kcContext: KcContext }) {
  const { kcContext } = props;
  const { i18n } = useI18n({ kcContext });

  return (
    <Suspense>
      {(() => {
        switch (kcContext.pageId) {
          case "login.ftl":
            return (
              <Login
                {...{ kcContext, i18n, classes }}
                Template={Template}
                doUseDefaultCss={true}
              />
            );
          default:
            return (
              <DefaultPage
                kcContext={kcContext}
                i18n={i18n}
                classes={classes}
                Template={Template}
                doUseDefaultCss={true}
                UserProfileFormFields={UserProfileFormFields}
                doMakeUserConfirmPassword={doMakeUserConfirmPassword}
              />
            );
        }
      })()}
    </Suspense>
  );
}

const classes = {} satisfies { [key in ClassKey]?: string };
```

Starter files can differ slightly. Edit the existing file and don't rewrite it from this snippet.

## CSS customization

- Most elements carry a `kc*` class (for example `kcLabelClass` or `kcButtonPrimaryClass`) plus Patternfly `pf-*` classes. Target the `kc*` classes.
- Put global CSS in `src/login/main.css` and add `import "./main.css";` in `KcPage.tsx`.
- To remove or replace Patternfly classes, map the `kc*` keys in `classes`:
  ```ts
  const classes = {
    kcButtonClass: "",          // was pf-c-button
    kcButtonPrimaryClass: "",   // was pf-m-primary long-pf-btn
    kcButtonBlockClass: "",
    kcButtonLargeClass: "",
    kcLabelClass: "form-label col-form-label" // e.g. Bootstrap / Tailwind classes
  } satisfies { [key in ClassKey]?: string };
  ```
- To drop all default styles, set `doUseDefaultCss={false}`. This also skips the global Patternfly stylesheet. You can do it for every page, or only in a specific `case`.
- For per-page CSS, import a stylesheet dynamically based on `kcContext.pageId`, for example `import("./pages/login.css")` in a `useEffect`.
- In ejected components, removing `kcClsx(...)` calls also removes the hooks your `kc*` CSS relies on.
- Angular: `getKcPage(pageId)` returns `{ PageComponent, TemplateComponent, UserProfileFormFieldsComponent, doMakeUserConfirmPassword, doUseDefaultCss, classes }`. Customize there.
- Svelte: KcPage computes `doUseDefaultCss` and `classes` per `pageId` in the same way.

## Ejecting pages and components

```bash
npx keycloakify eject-page   # interactive: pick a page (e.g. login.ftl) or Template.tsx / UserProfileFormFields.tsx
```

1. The command copies the source into `src/login/pages/` (or `src/login/` for the template).
2. Add the `case "<pageId>":` in `KcPage.tsx` that renders the ejected component. Without that case, the ejected file is never used.
3. Eject `Template.tsx` to change the shared layout (logo, header, language selector, alert banner), then pass your `Template` to every page.
4. Keep the form `action={url.loginAction}`, the field `name` attributes and the hidden inputs exactly as they were. Keycloak's server-side processing depends on them.

## i18n

Text changes go in `src/login/i18n.ts` through `i18nBuilder().withCustomTranslations({...})`, using inline literals only. Use `msg`/`msgStr` for known keys and `advancedMsg` for server-defined keys.

See **[i18n.md](i18n.md)** for the full guide: overriding and adding keys, extra languages, per-variant wording, realm overrides, previewing languages, and the email and account themes.

## Custom register / profile fields

The recommended path uses no code:

1. In Keycloak: Realm Settings → **User Profile** → add an attribute (for example `favourite_pet`). Set its validators, annotations (`inputType`, `inputOptionLabels`, ...) and permissions, and enable it for the register context.
2. `UserProfileFormFields` renders it automatically on `register.ftl`, `login-update-profile.ftl` and similar pages.
3. Set the display name in Realm Settings → Localization (`profile.attributes.favourite_pet`), and set the attribute's display name to `${profile.attributes.favourite_pet}`. It is rendered with `advancedMsg`.
4. For custom rendering, eject `UserProfileFormFields.tsx` (or the register page) and branch on `attribute.name`.

To preview the attribute in Storybook, add it to the story's `kcContext` override under `profile.attributesByName`. For details, search the docs for "user profile" or "registration page".

## Custom kcContext data / custom pages

- Extend the `KcContext` type in `src/login/KcContext.ts` for extra properties, such as env vars exposed through `environmentVariables`.
- Custom `.ftl` pages provided by Keycloak extensions are also declared there.
- Follow the docs for the exact type helpers, because they are version-specific.
