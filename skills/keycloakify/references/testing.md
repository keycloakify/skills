# Testing a theme

## 1. Storybook (main dev loop)

`.storybook/main.ts`:

```ts
import type { StorybookConfig } from "@storybook/react-vite";
const config: StorybookConfig = {
  stories: ["../src/**/*.mdx", "../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"],
  framework: { name: "@storybook/react-vite", options: {} },
  staticDirs: ["../public"]
};
export default config;
```

### Mocks

`src/login/mocks/getKcContextMock.ts` builds a realistic `kcContext` for any pageId. Put your extension defaults here once:

```ts
import { createGetKcContextMock } from "@keycloakify/login-ui/KcContext/getKcContextMock";
import { kcEnvDefaults, themeNames } from "../../kc.gen";
import type { KcContextExtension, KcContextExtensionPerPage } from "../KcContext";

const kcContextExtension: KcContextExtension = {
  themeName: themeNames[0],
  properties: { ...kcEnvDefaults }
};

const kcContextExtensionPerPage: KcContextExtensionPerPage = {
  // mock data for every extension page declared in KcContext.ts
};

export const { getKcContextMock } = createGetKcContextMock({
  kcContextExtension,
  kcContextExtensionPerPage,
  overrides: {},                 // applied to every page (e.g. locale: { currentLanguageTag: "de" })
  overridesPerPage: {}           // e.g. { "otp-form.ftl": { auth: { showUsername: true } } }
});
```

`src/login/mocks/KcPageStory.tsx`:

```tsx
import type { DeepPartial } from "@keycloakify/login-ui/tools/DeepPartial";
import type { KcContext } from "../KcContext";
import KcPage from "../KcPage";
import { getKcContextMock } from "./getKcContextMock";
export type { Meta, StoryObj } from "../../kc.gen";

export function createKcPageStory<PageId extends KcContext["pageId"]>(params: { pageId: PageId }) {
  function KcPageStory(props: { kcContext?: DeepPartial<Extract<KcContext, { pageId: PageId }>> }) {
    return <KcPage kcContext={getKcContextMock({ pageId: params.pageId, overrides: props.kcContext })} />;
  }
  return { KcPageStory };
}
```

### Stories: one file per page, one story per state

`src/login/pages/login/Page.stories.tsx`:

```tsx
import { createKcPageStory, type Meta, type StoryObj } from "../../mocks/KcPageStory";

const { KcPageStory } = createKcPageStory({ pageId: "login.ftl" });

const meta = { title: "login/login.ftl", component: KcPageStory } satisfies Meta<typeof KcPageStory>;
export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {};

export const WithInvalidCredential: Story = {
  args: {
    kcContext: {
      login: { username: "johndoe" },
      messagesPerField: {
        // other messagesPerField helpers derive from these two
        existsError: (field: string, ...others: string[]) =>
          [field, ...others].some(f => f === "username" || f === "password"),
        get: (field: string) => (field === "username" || field === "password" ? "Invalid username or password." : "")
      }
    }
  }
};

export const WithoutRegistration: Story = { args: { kcContext: { realm: { registrationAllowed: false } } } };
export const WithSocialProviders: Story = {
  args: {
    kcContext: {
      social: {
        displayInfo: true,
        providers: [{ loginUrl: "google", alias: "google", providerId: "google", displayName: "Google", iconClasses: "fa fa-google" }]
      }
    }
  }
};
export const WithErrorMessage: Story = {
  args: { kcContext: { message: { type: "error", summary: "The time allotted for the connection has elapsed." } } }
};
export const Arabic: Story = { args: { kcContext: { locale: { currentLanguageTag: "ar" } } } };
```

- Pass overrides through `args.kcContext`. It's a `DeepPartial` that is merged into the mock.
- Cover these states on each page:
  - default
  - field errors and global messages (`message.type`: error, warning, success, info)
  - realm flags off: `registrationAllowed`, `rememberMe`, `resetPasswordAllowed`, `password`
  - email-as-username
  - a preset or immutable username (`auth.showUsername`, `usernameHidden`)
  - 0, 1, 2 and many social providers
  - an RTL locale
- For global switches (locale, layout, color preset, dark mode), prefer Storybook **toolbar globals** plus a decorator that injects them into `kcContext`. That's better than copying stories.

Run `npm run storybook`. Add stories for new pages with `npx keycloakify add-story`, or copy an existing `Page.stories.tsx`.

## 2. Dev server

`pnpm dev` renders `main-kc.dev.tsx`. Change the `pageId` or `overrides` there to work on a page outside Storybook.

## 3. Real Keycloak

```bash
npx keycloakify start-keycloak   # pick a Keycloak version; runs it in Docker
```

- Admin Console: http://localhost:8080.
- Realm `myrealm`, user `testuser` / `password123`.
- Test app: https://my-theme.keycloakify.dev.
- The theme rebuilds when you save. The realm config persists in `.keycloakify/`.

```ts
keycloakify({
  startKeycloakOptions: {
    dockerImage: "quay.io/keycloak/keycloak:26.0.7",
    port: 8080,
    keycloakExtraArgs: ["--features=oid4vc-vci"],        // enable preview features you depend on
    dockerExtraArgs: ["-e", "MYTHEME_LAYOUT=centered-card"], // theme env vars
    extensionJars: ["https://github.com/p2-inc/keycloak-magic-link/releases/download/vX/keycloak-magic-link-X.jar"],
    realmJsonFilePath: "./.keycloakify/realm.json"
  }
})
```

Always do a final check in real Keycloak for:
- extension pages
- WebAuthn and passkeys
- recaptcha and Turnstile
- the Single-Page account theme
- emails
- realm-driven content (User Profile, identity providers, localization)
