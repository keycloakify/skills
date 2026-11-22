# Email theme

There are two options. For React projects, prefer **A**.

## A. jsx-email templates with keycloakify-emails (recommended)

Install the packages:

```bash
pnpm add -D jsx-email keycloakify-emails
pnpm add i18next react-i18next
```

**Layout**

```
src/email/
  i18n.ts                  # i18next init + getMessages (Keycloak's own keys)
  locales/<locale>/translation.json   # one folder per locale (en, fr, ar, pt-BR, zh-CN…)
  layout.tsx               # shared EmailLayout (logo, colors, dir)
  rtl.ts                   # isRtlLocale(locale)
  theme/                   # colors resolved from env vars
  templates/
    password-reset.tsx     # one file per Keycloak template, named after the .ftl
    email-verification.tsx
    executeActions.tsx
    event-*.tsx
    assets/                # images bundled into the email theme resources
```

**A template**

Each template exports `getTemplate`, `getSubject`, `templateName` and `previewProps`:

```tsx
import i18n, { type TFunction } from "i18next";
import { Button, Text, render } from "jsx-email";
import type { GetSubject, GetTemplate, GetTemplateProps } from "keycloakify-emails";
import { createVariablesHelper } from "keycloakify-emails/variables";
import { EmailLayout } from "../layout";

type TemplateProps = Omit<GetTemplateProps, "plainText"> & { t: TFunction };

export const templateName = "Password Reset";
export const previewProps: TemplateProps = { t: i18n.getFixedT("en"), locale: "en", themeName: "vanilla" };

const { exp } = createVariablesHelper("password-reset.ftl"); // typed FreeMarker expressions

export const Template = ({ locale, t }: TemplateProps) => (
  <EmailLayout preview={t("password-reset.subject")} locale={locale}>
    <Text>{t("password-reset.message", { realmName: exp("realmName") })}</Text>
    <Button href={exp("link")} width={200} height={40}>{t("password-reset.resetButton")}</Button>
    <Text>{t("password-reset.linkExpiration", { expiration: exp("linkExpirationFormatter(linkExpiration)") })}</Text>
  </EmailLayout>
);

export const getTemplate: GetTemplate = async props =>
  render(<Template {...props} t={i18n.getFixedT(props.locale)} />, { plainText: props.plainText });

export const getSubject: GetSubject = async props => i18n.getFixedT(props.locale)("password-reset.subject");
```

- Use `exp("...")` for every dynamic value: `link`, `realmName`, `user.firstName`, `linkExpirationFormatter(linkExpiration)`, and so on. The values are filled in by Keycloak at send time, so never hard-code them.
- `src/email/i18n.ts` initializes i18next from the JSON files. It also exports `getMessages: GetMessages`, which maps Keycloak's built-in keys (`requiredAction.*`, `linkExpirationFormatter.timePeriodUnit.*`) to your translations.
- For RTL locales, set `dir="rtl"` in the layout and flip alignments (for example `align={isRtlLocale(locale) ? "right" : "left"}`).

**Build wiring**

Build the emails in `postBuild` in `vite.config.ts`:

```ts
import { buildEmailTheme } from "keycloakify-emails";

keycloakify({
  postBuild: async buildContext => {
    await buildEmailTheme({
      templatesSrcDirPath: path.join(buildContext.themeSrcDirPath, "email", "templates"),
      assetsDirPath: path.join(buildContext.themeSrcDirPath, "email", "templates", "assets"),
      i18nSourceFile: path.join(buildContext.themeSrcDirPath, "email", "i18n.ts"),
      themeNames: buildContext.themeNames,
      keycloakifyBuildDirPath: buildContext.keycloakifyBuildDirPath,
      locales: ["en", "fr", "ar" /* must match src/email/locales/* folders */],
      esbuild: { jsx: "automatic" },
      cwd: import.meta.dirname,
      environmentVariables: buildContext.environmentVariables // theme env vars usable in emails
    });
  }
});
```

**Scripts**

```json
"emails:preview": "email preview ./src/email/templates",
"emails:check": "email check ./src/email/templates"
```

`emails:check` reports client-compatibility problems.

**Branding through env vars**

Expose email colors and logo as theme env vars (for example `MYTHEME_EMAIL_PRIMARY_COLOR` and `MYTHEME_EMAIL_LOGO_URL`) and resolve them into a theme object:
- At build time they become FreeMarker expressions.
- In previews they use static defaults.

## B. Native FreeMarker

```bash
npx keycloakify initialize-email-theme   # choose "native"
```

- Edit the `.ftl` templates and the `messages/*.properties` bundles under `src/email/`.
- Put assets in `src/email/resources/` and reference them with `${url.resourcesUrl}/file.png`.

## Testing and enabling

- Select the theme under Realm Settings → Themes → **Email theme**.
- Configure SMTP under Realm Settings → Email. Use a local catcher such as Mailpit in development.
- Trigger the flows from `npx keycloakify start-keycloak`: forgot password, verify email, required actions.

Links:
- https://github.com/timofei-iatsenko/keycloakify-emails
- https://jsx.email
