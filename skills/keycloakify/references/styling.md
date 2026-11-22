# Styling

## styleLevelCustomization.tsx

This is the single place for theme-wide style decisions. It is owned by default after `init`.

```tsx
import type { ClassKey } from "@keycloakify/login-ui/useKcClsx";
import type { ReactNode } from "react";
import { ThemeProvider } from "#/components/ThemeProvider";
import { getTheme } from "#/lib/getColorScheme";
import "./index.css";
import { useKcContext } from "./KcContext";

type StyleLevelCustomization = {
  doUseDefaultCss: boolean;                 // true = keep Keycloak/Patternfly CSS
  classes?: { [key in ClassKey]?: string }; // remap kc* classes
  loadCustomStylesheet?: () => void;        // load a stylesheet once (e.g. conditional CSS)
  Provider?: (props: { children: ReactNode }) => ReactNode; // global providers (theme, toasts…)
};

function Provider(props: { children: ReactNode }) {
  const { kcContext } = useKcContext();
  return <ThemeProvider defaultTheme={getTheme(kcContext.darkMode)}>{props.children}</ThemeProvider>;
}

export function useStyleLevelCustomization(): StyleLevelCustomization {
  return { doUseDefaultCss: false, Provider };
}
```

There are two approaches:
- **Keep Keycloak's look and tweak it:** `doUseDefaultCss: true`. Then restyle `kc*` classes in CSS, or remap them in `classes` (for example `kcButtonPrimaryClass: "btn btn-primary"`).
- **Your own design system:** `doUseDefaultCss: false`. Style the owned components directly. `kcClsx("kcXxxClass")` calls can stay as hooks for CSS overrides.

## Choosing a styling stack

Keycloakify doesn't impose one. A theme is a normal Vite + React app, so use whatever the user's project already uses, and don't introduce Tailwind or shadcn unless asked:

| Stack | Notes |
|---|---|
| Plain CSS or CSS Modules | Import the stylesheets from `styleLevelCustomization.tsx` or the components. Zero extra setup. |
| Sass/Less | Install the preprocessor and Vite handles it. |
| Tailwind CSS | Use the Vite plugin and a CSS entry imported from `styleLevelCustomization.tsx` (example below). |
| Component libraries (shadcn/ui, MUI, Mantine, Chakra, Ant Design, PrimeReact…) | Use the components inside the owned pages and Template. Put the library's theme or context provider in the `Provider` returned by `styleLevelCustomization.tsx`. |
| CSS-in-JS (Emotion, styled-components) | Works as usual. Put any cache or theme provider in `Provider`. |
| Bootstrap or similar class frameworks | Can be layered onto the default markup through `classes` (`kcLabelClass: "form-label"`, `kcButtonPrimaryClass: "btn btn-primary"`) with `doUseDefaultCss: false`. |

Whatever the stack, the rest of this file (dark mode, RTL, assets, layouts, form contract) still applies.

## Example: Tailwind v4 + shadcn/ui (the reference project's setup)

- **Vite:** add `tailwindcss()` from `@tailwindcss/vite` to `plugins`, next to `react()` and `keycloakify(...)`.
- **CSS entry:** `src/login/index.css`, imported from `styleLevelCustomization.tsx`.
  ```css
  @import "tailwindcss";
  @import "tw-animate-css";
  @import "shadcn/tailwind.css";
  @source "../**/*.{ts,tsx}";
  @custom-variant dark (&:is(.dark *));
  @theme { --color-primary: var(--primary); /* … map tokens … */ }
  ```
- **`components.json`:** set `"tailwind": { "css": "src/login/index.css", "cssVariables": true }` and `"rtl": true`. Point the aliases at `#/components`, `#/lib/utils`, `#/components/ui` and `#/hooks`.
- **Aliases:** Node subpath imports in `package.json` work in Vite, TypeScript and Storybook with no extra config:
  ```json
  "imports": {
    "#/components/*": "./src/components/*.tsx",
    "#/lib/*": "./src/lib/*.ts",
    "#/hooks/*": "./src/hooks/*.ts",
    "#/*": "./src/*"
  }
  ```
  An `@/*` alias also works (`resolve.alias` plus tsconfig `paths`).
- **Shared components:** put them in `src/components/ui/` and use them from both the login and account themes. Add new ones with `npx shadcn add <component>`.
- **Fonts:** ship them with the theme (for example `@fontsource-variable/*` imports in CSS), not from external CDNs. Keycloak deployments often have strict CSP and no outbound access.

## Dark mode

Avoid a flash of the wrong theme, because Keycloak pages are full page loads:
1. Put a tiny script at `public/keycloak-theme/login/early-color-scheme.js`. It runs before React and decides dark or light from:
   - the `?dark=true|false` query param
   - stored preference
   - `prefers-color-scheme`

   It then adds the `dark` class to `<html>` and sets `color-scheme` and the background color.
2. In React, a `ThemeProvider` exposes the light/dark/system toggle and persists the choice.
3. Let the admin force light mode with a context flag (for example `kcContext.darkMode === false`, declared in `KcContextExtension`).

## RTL

- Use logical utilities everywhere (`ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`, `text-start`). Don't use `ml-*` or `left-*`.
- Set `dir` on `<html>` from the current language (`ar`, `fa`, `he`, `ur` → `rtl`). Check it with an Arabic story.
- In shadcn, set `"rtl": true` in `components.json`.

## Assets (logos, images)

- Import images from `src/` (`import logo from "./assets/img/logo.svg"`) so Vite fingerprints them.
- To make images configurable at runtime, use theme env vars that hold a URL. Support a `%BASE_URL%` token for self-hosted files in `public/`:
  ```ts
  export function resolveAssetUrl(url?: string) {
    if (!url) return undefined;
    return url.replace(/%BASE_URL%/g, () => {
      const base = import.meta.env.BASE_URL ?? "/";
      return base.endsWith("/") ? base.slice(0, -1) : base;
    });
  }
  ```
- Provide a dark variant with a fallback: `logoDarkUrl = resolveAssetUrl(LOGO_DARK_URL) || logoUrl`.
- Use `realm.displayName || realm.name` as alt text and as the brand-name fallback. Render `realm.displayNameHtml` only through `kcSanitize`.

## Layout variants

A theme can offer several layouts and select one at runtime with an env var, for example `two-column`, `centered-card` or `image-aside`:
- Implement each layout as a component in `login/components/layouts/` that receives the same `content` node.
- Have the Template switch on the env value.

This keeps pages layout-agnostic.
