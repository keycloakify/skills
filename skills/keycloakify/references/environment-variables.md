# Theme environment variables

Theme env vars make one built jar configurable per deployment: logo URL, colors, layout, feature toggles, and so on, with no rebuild.
- You declare them in `vite.config.ts` → `keycloakify({ environmentVariables })`.
- Keycloak resolves them at runtime from the server's environment variables, or from the default.
- They reach the theme as `kcContext.properties` (always **strings**).
- In email templates, they are passed to `keycloakify-emails` through `buildContext.environmentVariables`.

Docs: https://docs.keycloakify.dev/features/compiler-options/environmentvariables

## Recommended pattern: one typed source of truth

Don't duplicate names and defaults across `vite.config.ts`, mocks and components. Define them once in `src/kc-env.ts`:

```ts
// src/kc-env.ts
export type KcEnv = {
  MYTHEME_LOGO_URL: string;
  MYTHEME_LAYOUT: "two-column" | "centered-card";
  MYTHEME_SHOW_REALM_NAME: boolean;
};

export const KC_ENV_DEFAULTS = {
  MYTHEME_LOGO_URL: "",
  MYTHEME_LAYOUT: "two-column",
  MYTHEME_SHOW_REALM_NAME: true
} as const satisfies KcEnv;

export type KcEnvName = keyof KcEnv;

export function serializeKcEnvValue(value: KcEnv[KcEnvName]): string {
  return typeof value === "boolean" ? String(value) : value;
}

function parseBoolean(raw: string, fallback: boolean) {
  return raw === "true" ? true : raw === "false" ? false : fallback;
}

/** Raw Keycloak strings → typed values. The only place parsing happens. */
export function parseKcEnv(properties: Record<KcEnvName, string>): KcEnv {
  return {
    ...properties,
    MYTHEME_SHOW_REALM_NAME: parseBoolean(properties.MYTHEME_SHOW_REALM_NAME, KC_ENV_DEFAULTS.MYTHEME_SHOW_REALM_NAME)
  } as KcEnv;
}

/** Shape expected by keycloakify({ environmentVariables }). */
export const kcEnvironmentVariables = (Object.keys(KC_ENV_DEFAULTS) as KcEnvName[]).map(name => ({
  name,
  default: serializeKcEnvValue(KC_ENV_DEFAULTS[name])
}));
```

```ts
// vite.config.ts
import { kcEnvironmentVariables } from "./src/kc-env";
keycloakify({ environmentVariables: kcEnvironmentVariables /* … */ });
```

`update-kc-gen` then emits `KcEnvName` and `kcEnvDefaults` (string values) in `src/kc.gen.tsx`.

## Typing kcContext and the mocks

```ts
// src/login/KcContext.ts
import type { KcEnvName, ThemeName } from "../kc.gen";
export type KcContextExtension = {
  themeName: ThemeName;
  properties: Record<KcEnvName, string> & {};
};
```

```ts
// src/login/mocks/getKcContextMock.ts
import { kcEnvDefaults, themeNames } from "../../kc.gen";
const kcContextExtension: KcContextExtension = {
  themeName: themeNames[0],
  properties: { ...kcEnvDefaults }
};
```

## Reading values in components

Go through a hook. Never read `kcContext.properties` directly:

```ts
// src/login/useKcEnv.ts
import { type KcEnv, parseKcEnv } from "#/kc-env.ts";
import { useKcContext } from "./KcContext";

export function useKcEnv(): KcEnv {
  const { kcContext } = useKcContext();
  return parseKcEnv(kcContext.properties);
}

// usage
const { MYTHEME_LAYOUT, MYTHEME_SHOW_REALM_NAME } = useKcEnv();
```

- Validate enum-like values in one place (for example a `ThemeUtils.ts` that falls back to the default on unknown values). Admins make typos.
- To let admins override a translated default, use an "empty means use i18n" convention: `const welcome = WELCOME_MESSAGE.trim() || msgStr("welcomeMessage")`.

## Setting them in Keycloak

- **Docker / Kubernetes:** set real environment variables on the Keycloak container, for example `-e MYTHEME_LAYOUT=centered-card`.
- **`start-keycloak`:** pass `dockerExtraArgs: ["-e", "MYTHEME_LAYOUT=centered-card"]` in `startKeycloakOptions`.
- **Storybook:** expose them as toolbar globals or story args that override `kcContext.properties`, to try combinations without rebuilding.

Document every variable in the README: name, default, allowed values and what it does.
