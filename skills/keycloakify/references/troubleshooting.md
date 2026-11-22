# Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| My change disappeared after `pnpm install` | The file wasn't owned, so `sync-extensions` overwrote it. Run `npx keycloakify own --path "<path>"`, redo the change and commit. |
| `own` or `init` complains about git | It needs a git repo with the work committed. Run `git init`, then commit first. |
| New page folder never renders | No `case "<pageId>":` in `login/pages/PageIndex.tsx`, or `index.ts` doesn't `export default`. |
| Type error accessing page fields | `kcContext` isn't narrowed. Add `assert(kcContext.pageId === "<id>.ftl")` in that component. |
| `kc.gen.tsx` out of date (missing env var or theme name) | It regenerates on dev or build. Run `npx keycloakify update-kc-gen`. Never edit it by hand. |
| `mvn: command not found` | Install Maven and a JDK, and put them on PATH. |
| `start-keycloak` fails | Docker isn't running, or the port is in use. Change it with `startKeycloakOptions.port`. |
| Theme missing in the Admin Console | Wrong jar for the Keycloak version, jar not in `providers/`, or `kc.sh build` not run on an optimized image. |
| Works in Storybook, broken in Keycloak | The real `kcContext` differs from the mock. Reproduce with `start-keycloak`, log `kcContext`, and make the mock match. |
| Form submit returns to the same page with an error, or "invalid request" | The markup lost a field `name`, a hidden input, the `action` URL or the submit `name`. Compare with the synced original (`own --revert` into a scratch branch). |
| Translation shows the raw key | The key is missing from `withCustomTranslations` for that language (or for `en`), or it's server-defined and needs `advancedMsg`. |
| Realm Localization override ignored | Add `<@addToXKeycloakifyMessagesIfMessageKey str="key" />` to `kcContextExclusionsFtl`. |
| Env var always shows the default | It isn't set on the Keycloak process. Use `dockerExtraArgs` with `start-keycloak`. Also check the name against `kc-env.ts`. |
| Boolean env var behaves wrongly | `kcContext.properties` values are strings. Parse them in `useKcEnv` / `parseKcEnv`. |
| Flash of light theme before dark | Add an early color-scheme script under `public/keycloak-theme/login/` that sets the `dark` class before React loads. |
| Layout broken in Arabic or Persian | Physical utilities (`ml-*`, `left-*`) were used. Switch to logical ones, and set `dir` on `<html>`. |
| Email shows `${...}` literally, or has no translations | The value was hard-coded instead of using `exp(...)`, or the locale is missing from the `buildEmailTheme({ locales })` list or from `src/email/locales/`. |
| Old theme still shown after redeploy | Keycloak caches themes. Restart it. In dev, use `--spi-theme-cache-themes=false --spi-theme-cache-templates=false`. |

For projects on the legacy `keycloakify/login` API and the v10 → v11 migration, see [legacy-api.md](legacy-api.md).

Docs and FAQ: https://docs.keycloakify.dev (index: https://docs.keycloakify.dev/llms.txt)
