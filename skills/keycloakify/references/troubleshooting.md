# Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `mvn: command not found` during build | Install Maven and a JDK, and put `mvn` on PATH (`mvn --version`). |
| `start-keycloak` fails | Docker isn't running, or the port is taken. Change it with `startKeycloakOptions.port`. |
| Theme missing from the Admin Console dropdown | Wrong jar for the Keycloak version, jar not in `/opt/keycloak/providers/`, or `kc.sh build` not run on an optimized image. Check the startup logs. |
| Works in Storybook, broken in Keycloak | The real `kcContext` differs from the mock. Reproduce with `start-keycloak`, log `kcContext`, and adjust your story overrides to match. |
| Ejected page has no effect | No matching `case "<pageId>":` in `KcPage.tsx`. |
| Custom CSS on `kc*` classes stopped applying | `kcClsx(...)` was removed from the ejected component, or `classes` maps that key to something else. |
| Translation key type error | The key is missing from `withCustomTranslations` for `en`, or it is server-defined. Use `advancedMsg` for server-defined keys. |
| Translations imported from JSON ignored | They must be inline literals, because the build analyzes them statically. |
| Realm Localization overrides ignored | See the docs page "My Realm Overrides Translations do not apply" (uses `kcContextExclusionsFtl`). |
| Form submit fails | An ejected form changed `action`, an input `name`, or hidden inputs. Restore them from the default page. |
| Errors in an ejected `Template.tsx` after an upgrade | It's outdated. Re-eject it and port your changes (see migration below). |
| Old theme still shown after redeploy | Keycloak caches themes. Restart it. In dev, disable the theme cache (`--spi-theme-cache-themes=false --spi-theme-cache-templates=false`). |

## Migration v10 → v11 (key points)

- **i18n**: `createUseI18n(...)` → `i18nBuilder.withThemeName<ThemeName>().withCustomTranslations({...}).build()`, exporting `useI18n` and `type I18n`.
- **Ejected `Template.tsx`**:
  - `currentLanguageTag` → `currentLanguage` and `enabledLanguages`.
  - The `html.lang` effect was removed.
  - Use `useInitialize({ kcContext, doUseDefaultCss })` from `keycloakify/login/Template.useInitialize`.
- **Sanitizing**: import `kcSanitize` from `keycloakify/lib/kcSanitize` and wrap `dangerouslySetInnerHTML.__html` values.
- **Webpack only**: `keycloakify-resources` was renamed to `keycloakify-dev-resources`.
- **`keycloakVersionTargets`**: the key names changed if you came from ≤ 10.1.4.

For the full and current notes, see https://docs.keycloakify.dev/release-notes-and-upgrade-instructions
