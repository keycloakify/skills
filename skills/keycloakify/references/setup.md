# Setup

## Prerequisites

- Node.js 18+ and a package manager (pnpm, npm, yarn or bun).
- **Git**. The extension model uses it to track which files you own.
- **Maven** plus a JDK, to build the jar.
  - macOS: `brew install maven`
  - Debian/Ubuntu: `sudo apt-get install maven`
  - Windows: `choco install openjdk maven`
  - Check with `mvn --version`.
- **Docker**, only for `npx keycloakify start-keycloak`.

## Starting a project

**Option A: official starter**

```bash
git clone https://github.com/keycloakify/keycloakify-starter my-theme
cd my-theme && pnpm install
```

There are also starters for Angular (`keycloakify-starter-angular-vite`) and Svelte (`keycloakify-starter-svelte`). React gets the most complete support.

**Option B: new Vite app plus `init`**

```bash
pnpm create vite my-theme --template react-ts
cd my-theme
pnpm add keycloakify
git init && git add -A && git commit -m "init"
npx keycloakify init          # choose theme types (login/account/email), install stories
```

**Option C: start from a published theme package** that ships as a Keycloakify extension:

```bash
pnpm add keycloakify @oussemasahbeni/keycloakify-login-shadcn
npx keycloakify init
```

Its files sync into `src/`. Own only the files you change.

After any of these:
- add `"postinstall": "keycloakify sync-extensions"` to `package.json` if it isn't already there
- commit the result

## vite.config.ts

```ts
import tailwindcss from "@tailwindcss/vite";
import react from "@vitejs/plugin-react";
import { keycloakify } from "keycloakify/vite-plugin";
import { defineConfig } from "vite";
import { kcEnvironmentVariables } from "./src/kc-env";

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
    keycloakify({
      themeName: "my-theme",                       // or an array for theme variants
      accountThemeImplementation: "none",          // "none" | "Single-Page" | "Multi-Page" — managed by init, don't hand-edit
      keycloakVersionTargets: {                    // which jars to emit (see build-deploy.md)
        "22-to-25": false,
        "all-other-versions": "my-theme.jar"
      },
      environmentVariables: kcEnvironmentVariables, // see environment-variables.md
      kcContextExclusionsFtl: `<@addToXKeycloakifyMessagesIfMessageKey str="welcomeMessage" />`, // see i18n.md
      startKeycloakOptions: { /* see testing.md */ },
      postBuild: async buildContext => { /* prune resources, build emails – see build-deploy.md */ }
    })
  ]
});
```

## package.json essentials

```json
{
  "type": "module",
  "imports": {
    "#/components/*": "./src/components/*.tsx",
    "#/lib/*": "./src/lib/*.ts",
    "#/hooks/*": "./src/hooks/*.ts",
    "#/*": "./src/*"
  },
  "scripts": {
    "postinstall": "keycloakify sync-extensions",
    "dev": "vite",
    "build": "tsc -b && vite build",
    "build-keycloak-theme": "pnpm run build && keycloakify build",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  }
}
```

- The `#/` subpath imports give stable aliases with no Vite or TypeScript alias config.
- Add `lint` / `fmt` scripts (ESLint or oxlint, Prettier or oxfmt), and run them before building.

## Entry points

```tsx
// src/main.tsx — split real vs dev so the mock never ships in the Keycloak bundle path
if (window.kcContext !== undefined) {
  void import("./main-kc");
} else {
  void import("./main-kc.dev");
}
```

```tsx
// src/main-kc.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { KcPage } from "./kc.gen";

if (!window.kcContext) throw new Error("No Keycloak context");
createRoot(document.getElementById("root")!).render(
  <StrictMode><KcPage kcContext={window.kcContext} /></StrictMode>
);
```

```tsx
// src/main-kc.dev.tsx — `pnpm dev` renders a mocked page
import { getKcContextMock } from "./login/mocks/getKcContextMock";
const kcContext = getKcContextMock({ pageId: "login.ftl", overrides: {} });
// render <KcPage kcContext={kcContext} /> as above
```

## Monorepos

The theme can be one package in a pnpm workspace, for example `packages/my-theme` next to `apps/web`.
- Keep the Keycloakify config, `src/.gitignore` and the `public/keycloak-theme` folder inside that package.
- Run the CLI from the package directory, or use `--project`.
