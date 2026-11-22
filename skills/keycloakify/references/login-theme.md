# Login theme (extension model)

Paths are relative to `src/`. Before editing a file, check that it's owned: it must not be listed as ignored in `src/.gitignore`. If it is, run `npx keycloakify own --path "<path>"` first.

## KcPage: providers only

`login/KcPage.tsx` sets up context and nothing else:

```tsx
import { useExclusiveAppInstanceEffect } from "@keycloakify/login-ui/tools/useExclusiveAppInstanceEffect";
import { KcClsxProvider } from "@keycloakify/login-ui/useKcClsx";
import type { ReactNode } from "react";
import { assert } from "tsafe/assert";
import { I18nProvider } from "./i18n";
import { type KcContext, KcContextProvider } from "./KcContext";
import { PageIndex } from "./pages/PageIndex";
import { useStyleLevelCustomization } from "./styleLevelCustomization";

export default function KcPage(props: { kcContext: KcContext }) {
  const { kcContext } = props;
  return (
    <KcContextProvider kcContext={kcContext}>
      <I18nProvider kcContext={kcContext}>
        <StyleLevelCustomization>
          <PageIndex />
        </StyleLevelCustomization>
      </I18nProvider>
    </KcContextProvider>
  );
}

function StyleLevelCustomization(props: { children: ReactNode }) {
  const { doUseDefaultCss, classes, loadCustomStylesheet, Provider } = useStyleLevelCustomization();

  useExclusiveAppInstanceEffect({
    effectId: "loadCustomStylesheet",
    isEnabled: loadCustomStylesheet !== undefined,
    effect: () => { assert(loadCustomStylesheet !== undefined); loadCustomStylesheet(); }
  });

  return (
    <KcClsxProvider doUseDefaultCss={doUseDefaultCss} classes={classes}>
      {Provider === undefined ? props.children : <Provider>{props.children}</Provider>}
    </KcClsxProvider>
  );
}
```

Add global providers (theme, toasts, query client) through the `Provider` returned by `styleLevelCustomization.tsx` (see [styling.md](styling.md)), not by nesting more wrappers here.

## PageIndex: routing by pageId

`login/pages/PageIndex.tsx` lazy-loads one folder per page:

```tsx
import { Suspense, lazy } from "react";
import { useKcContext } from "../KcContext";

const PageLogin = lazy(() => import("./login"));
const PageRegister = lazy(() => import("./register"));
// ... one per page id

export function PageIndex() {
  const { kcContext } = useKcContext();
  return (
    <Suspense>
      {(() => {
        switch (kcContext.pageId) {
          case "login.ftl": return <PageLogin />;
          case "register.ftl": return <PageRegister />;
          // ...
        }
      })()}
    </Suspense>
  );
}
```

- Folder name = pageId without `.ftl`: `login-reset-password.ftl` → `pages/login-reset-password/`.
- Each folder has:
  - `index.ts` (`export default Page`)
  - `Page.tsx`
  - `Page.stories.tsx`
  - optional parts such as `Form.tsx` and `Info.tsx`
- Keep lazy imports, so each page is its own chunk.

## Template: the shared frame

`login/components/Template/` renders everything around the page body. In the reference project that includes:
- the layout variant
- logo and realm name
- the "logging in as … / restart login" header
- alerts from `kcContext.message`
- the "required fields" note
- the info area, social providers and language switcher

It also has to:
- set `document.title` (`msgStr("loginTitle", realm.displayName || realm.name)`)
- apply `kcHtmlClass` and `kcBodyClass` via `useSetClassName` (from `keycloakify/tools/useSetClassName`)
- run `useInitializeTemplate()`, which inserts `kcContext.scripts` with `useInsertScriptTags` and runs the auth checker

Keep those behaviors when restyling.

Typical props, which pages pass to the Template:

```ts
type TemplateProps = {
  headerNode: ReactNode;          // page title
  displayMessage?: boolean;       // show kcContext.message alert (default true)
  displayInfo?: boolean;          // show infoNode below the card
  displayRequiredFields?: boolean;
  infoNode?: ReactNode;
  socialProvidersNode?: ReactNode;
  documentTitle?: string;
  bodyClassName?: string;
  children: ReactNode;
};
```

To change something on every page (logo, layout, alert style), edit the Template, not the individual pages.

## Forms: the contract with Keycloak

Keycloak processes plain HTML form posts, so a form must keep these:

| Keep | Example |
|---|---|
| `action` + `method="post"` | `action={kcContext.url.loginAction}`, `url.registrationAction`, `url.loginRestartFlowUrl`… |
| Field `name`s | `username`, `password`, `rememberMe`, `credentialId` (hidden), `password-new`, `password-confirm`, `otp`, `selectedCredentialId`… |
| Submit `name`/`value` | `<Button name="login" type="submit" value={msgStr("doLogIn")}>` |
| Known ids used by Keycloak scripts | `kc-form-login`, `kc-login`, `kc-register-form`, `username`, `password` |

The pattern used in the reference project, with shadcn `Field`/`Input`:

```tsx
const hasError = kcContext.messagesPerField.existsError("username", "password");

<Field>
  <FieldLabel htmlFor="username">{msg("username")}</FieldLabel>
  <Input id="username" name="username" defaultValue={kcContext.login.username ?? ""}
         autoComplete={kcContext.enableWebAuthnConditionalUI ? "username webauthn" : "username"}
         aria-invalid={hasError} autoFocus />
  {hasError && (
    <FieldError>
      <span id="input-error-username" aria-live="polite"
        dangerouslySetInnerHTML={{ __html: kcSanitize(kcContext.messagesPerField.getFirstError("username", "password")) }} />
    </FieldError>
  )}
</Field>
```

- Use realm flags to show or hide features:
  - `realm.password`
  - `realm.rememberMe`
  - `realm.resetPasswordAllowed`
  - `realm.registrationAllowed`
  - `realm.loginWithEmailAllowed`
  - `registrationEmailAsUsername`
  - `usernameHidden`
- For the password reveal toggle, use `useIsPasswordRevealed({ passwordInputId })` from `keycloakify/tools/useIsPasswordRevealed`.
- For WebAuthn, passkeys and recaptcha, keep the synced `useScript.tsx` and `WebAuthnConditionalUi` logic. Restyle only the markup.

## User Profile fields (register, update-profile, idp-review)

Fields come from the realm's **User Profile** configuration, not from code:

1. Add or modify the attribute in Keycloak: Realm Settings → User Profile. Set validators, required/permissions, group, and annotations such as `inputType`, `inputOptionLabels`, `inputHelperTextBefore`/`After`, `inputTypePlaceholder`.
2. `components/UserProfileFormFields/` renders the attributes with `useUserProfileForm({ kcContext, i18n, doMakeUserConfirmPassword })` from `@keycloakify/login-ui/useUserProfileForm`. It handles validation and multi-valued fields, and reports `isFormSubmittable` back to the page.
3. Render labels and help text with `advancedMsg(attribute.displayName ?? "")`, because they're server-defined.
4. For per-attribute customization, pass `BeforeField` / `AfterField` render props, or branch on `attribute.name` in `InputFieldByType`.
5. Put placeholders for dynamic fields in User Profile annotations, not in code.

The register page wires it like this:

```tsx
<form id="kc-register-form" action={kcContext.url.registrationAction} method="post">
  <UserProfileFormFields onIsFormSubmittableValueChange={setIsFormSubmittable} />
  {kcContext.termsAcceptanceRequired && <TermsAcceptance ... />}
  <Button type="submit" disabled={!isFormSubmittable || (kcContext.termsAcceptanceRequired && !areTermsAccepted)}>
    {msgStr("doRegister")}
  </Button>
</form>
```

## Extending KcContext and supporting extension pages

`login/KcContext.ts`:

```ts
import { type ExtendKcContext, createUseKcContext } from "@keycloakify/login-ui/KcContext";
import type { KcEnvName, ThemeName } from "../kc.gen";

export type KcContextExtension = {
  themeName: ThemeName;
  properties: Record<KcEnvName, string> & {};   // theme env vars (see environment-variables.md)
  // extra global fields your Keycloak setup injects
};

/** Pages not in base Keycloak, contributed by server extensions. */
export type KcContextExtensionPerPage = {
  "otp-form.ftl": {
    auth: { attemptedUsername: string };
    url: { loginRestartFlowUrl: string; loginAction: string };
  };
};

export type KcContext = ExtendKcContext<KcContextExtension, KcContextExtensionPerPage>;
export const { useKcContext, KcContextProvider } = createUseKcContext<KcContext>();
```

To support a new extension page:
1. Declare its fields in `KcContextExtensionPerPage`.
2. Add mock values in `mocks/getKcContextMock.ts`, under `kcContextExtensionPerPage`.
3. Create `pages/<page-id>/` (`index.ts`, `Page.tsx`, `Page.stories.tsx`).
4. Add the `case` in `PageIndex.tsx`.
5. Add its custom message keys to `i18n.ts`.

If a value you need is missing from `kcContext`, see the docs FAQ "Some values you need are missing from kcContext". The usual fix is `kcContextExclusionsFtl` or an extension that adds the value.
