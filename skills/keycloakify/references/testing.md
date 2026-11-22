# Testing a theme

## 1. Storybook (fastest loop, mocked kcContext)

```bash
npx keycloakify add-story   # pick pages, e.g. login.ftl, register.ftl (re-run to add more)
npm run storybook
```

Generated story (React):

```tsx
import type { Meta, StoryObj } from "@storybook/react";
import { createKcPageStory } from "../KcPageStory";

const { KcPageStory } = createKcPageStory({ pageId: "register.ftl" });

const meta = {
  title: "login/register.ftl",
  component: KcPageStory
} satisfies Meta<typeof KcPageStory>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = { render: () => <KcPageStory /> };

export const InChinese: Story = {
  render: () => <KcPageStory kcContext={{ locale: { currentLanguageTag: "zh-CN" } }} />
};
```

- The `kcContext` prop is a **partial override** of the default mock.
- Write one story per state that matters:
  - errors, via `message` or `messagesPerField`
  - social providers (`social.providers`)
  - `realm.registrationAllowed`
  - other locales
- Angular stories use `decorators` and `globals: { pageId, kcContext }`.
- Svelte stories use `defineMeta` from `@storybook/addon-svelte-csf` with `args: { pageId: "login.ftl" }`.

## 2. Dev server with a mocked page

To work on a page in `npm run dev` without Storybook, set a mock in `src/main.tsx` temporarily:

```ts
import { getKcContextMock } from "./login/KcPageStory";

if (import.meta.env.DEV) {
  window.kcContext = getKcContextMock({ pageId: "register.ftl", overrides: {} });
}
```

Remove or comment out this block before building, because it bloats the bundle.

## 3. Real Keycloak (final check)

```bash
npx keycloakify start-keycloak   # prompts for a Keycloak version, runs it in Docker
```

- Requires Docker and Maven.
- Admin Console: http://localhost:8080.
- Preconfigured realm `myrealm` with test user `testuser` / `password123`.
- Test app: https://my-theme.keycloakify.dev. It sends you to your local Keycloak's login pages.
- The theme is rebuilt when you save. Config persists in `.keycloakify/`.

Options in `vite.config.ts`:

```ts
keycloakify({
  startKeycloakOptions: {
    dockerImage: "quay.io/keycloak/keycloak:26.0.7",
    port: 8081,
    realmJsonFilePath: "./path/to/realm.json",
    extensionJars: ["https://.../some-extension.jar"],
    dockerExtraArgs: [],
    keycloakExtraArgs: []
  }
})
```

Use `start-keycloak` especially for:
- Pages that work in Storybook but not in Keycloak.
- The Single-Page account theme, which has no Storybook support.
- Email flows.
- Realm-driven content: User Profile attributes, identity providers, localization overrides.
