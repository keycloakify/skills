# Setup

## Starters

Starting from a starter is the recommended path:

| Framework | Repo |
|---|---|
| React (most complete, recommended) | `git clone https://github.com/keycloakify/keycloakify-starter` |
| Angular | `git clone https://github.com/keycloakify/keycloakify-starter-angular-vite keycloakify-starter` |
| Svelte | `git clone https://github.com/keycloakify/keycloakify-starter-svelte keycloakify-starter` |

```bash
cd keycloakify-starter
yarn install            # or npm/pnpm/bun; delete yarn.lock if you don't use yarn
npx keycloakify add-story
npm run storybook
```

Framework support:
- Admin themes are React-only.
- Angular and Svelte support login and account themes. Their account UI is based on the older multi-page account UI.
- Keycloakify can't be installed into an existing Angular app. Use a standalone project or a monorepo subproject.

## Prerequisites

- Node.js plus a package manager.
- **Maven** on PATH, plus a JDK, to build the jar.
  - macOS: `brew install maven`
  - Debian/Ubuntu: `sudo apt-get install maven`
  - Windows: `choco install openjdk` and `choco install maven`
  - Check with `mvn --version`.
- **Docker**, only for `npx keycloakify start-keycloak`.

## Project layout (React starter)

```
src/
  login/
    KcPage.tsx        # entry: switch on kcContext.pageId
    KcContext.ts      # KcContext type (extend it for custom pages/attributes)
    KcPageStory.tsx   # createKcPageStory / getKcContextMock for Storybook & dev
    i18n.ts           # i18nBuilder config
    main.css          # optional global CSS (import it in KcPage.tsx)
    pages/            # ejected pages, e.g. Login.tsx, *.stories.tsx
  account/            # after initialize-account-theme
  email/              # after initialize-email-theme (native)
  kc.gen.ts(x)        # GENERATED – do not edit (ThemeName, etc.)
  main.tsx            # renders KcPage when window.kcContext is defined
public/keycloakify-dev-resources/   # GENERATED for Storybook – do not edit
vite.config.ts
.storybook/
```

Exact files can vary between starter versions. Inspect the project and don't assume this layout.

## vite.config.ts

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { keycloakify } from "keycloakify/vite-plugin";

export default defineConfig({
  plugins: [
    react(),
    keycloakify({
      themeName: "my-theme",              // shown in the Admin Console dropdown; default: package.json name. Array = theme variants
      accountThemeImplementation: "none", // "none" | "Single-Page" | "Multi-Page" — set by initialize-account-theme, never edit by hand
      // keycloakVersionTargets: { ... }, // which jars to emit; see docs (key syntax depends on accountThemeImplementation)
      // environmentVariables: [ ... ],   // env vars readable at runtime from the theme; see docs
      // startKeycloakOptions: { ... },   // see testing.md
      // postBuild, kcContextExclusionsFtl, groupId, artifactId, keycloakifyBuildDirPath ...
    })
  ]
});
```

The starter defines a `build-keycloak-theme` script that runs the Vite build and then `keycloakify build`. Run it with `npm run build-keycloak-theme`.

The `keycloakVersionTargets` and `environmentVariables` shapes change between versions. Read the docs before setting them:
- https://docs.keycloakify.dev/features/compiler-options/keycloakversiontargets
- https://docs.keycloakify.dev/features/compiler-options/environmentvariables

## Adding Keycloakify to an existing Vite project

This is supported but harder than using a starter. Follow https://docs.keycloakify.dev/integration-keycloakify-in-your-codebase/vite. In short:
1. Add the plugin.
2. Put the theme under a folder such as `src/keycloak-theme/`.
3. In `main.tsx`, branch on `window.kcContext` so the app renders the theme only when Keycloak injects a context.
