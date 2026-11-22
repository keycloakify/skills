# Build and deploy

## Build

```bash
pnpm build-keycloak-theme   # tsc -b && vite build && keycloakify build
```

The jars are written to `dist_keycloak/`.

## Choosing which jars to emit

By default you get one jar per Keycloak version range:
- `keycloak-theme-for-kc-22-to-25.jar` for Keycloak 22–25
- `keycloak-theme-for-kc-all-other-versions.jar` for Keycloak 11–21 and 26+

If you target one Keycloak line, emit a single jar with a clear name:

```ts
keycloakify({
  keycloakVersionTargets: {
    "22-to-25": false,
    "all-other-versions": "my-theme.jar"
  }
})
```

The keys you can use depend on `accountThemeImplementation`. See https://docs.keycloakify.dev/features/compiler-options/keycloakversiontargets.

## postBuild hooks

`postBuild(buildContext)` runs after the theme files are generated and before the jar is packaged. Use it to:
- **Build the email theme** with `buildEmailTheme(...)` from `keycloakify-emails` (see [email-theme.md](email-theme.md)).
- **Prune unused default resources.** If `doUseDefaultCss` is false, the Patternfly CSS, images and JS that came with the base theme are dead weight:
  ```ts
  postBuild: async buildContext => {
    for (const themeName of buildContext.themeNames) {
      const dir = path.join(buildContext.keycloakifyBuildDirPath, "resources", "theme", themeName, "login", "resources");
      for (const sub of ["css", "img", "js", "resources-common"]) {
        fs.rmSync(path.join(dir, sub), { recursive: true, force: true });
      }
    }
  }
  ```
  Only do this after checking that nothing references those files: Template scripts, `kcContext.scripts`, and so on. Then check the theme with `start-keycloak`.

Useful `buildContext` fields:
- `themeNames`
- `themeSrcDirPath`
- `keycloakifyBuildDirPath`
- `environmentVariables`

## CI

- The runner needs Node, pnpm or npm, Maven and a JDK. Run `lint`, `tsc -b`, `build-storybook` (it catches broken stories) and `build-keycloak-theme`.
- Publish the jar as a GitHub release asset. The starter's workflow does this whenever the `package.json` version changes. It needs Settings → Actions → Workflow permissions set to "Read and write".

## Deploy

Jars go in `/opt/keycloak/providers/`.

**Quick test with Docker**

```bash
docker run -p 8080:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -e MYTHEME_LAYOUT=centered-card \
  -v "./dist_keycloak/my-theme.jar":/opt/keycloak/providers/my-theme.jar \
  quay.io/keycloak/keycloak:26.0.7 start-dev
```

**Production image**

```dockerfile
FROM quay.io/keycloak/keycloak:26.0.7 AS builder
COPY dist_keycloak/my-theme.jar /opt/keycloak/providers/
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:26.0.7
COPY --from=builder /opt/keycloak/ /opt/keycloak/
ENTRYPOINT ["/opt/keycloak/bin/kc.sh", "start", "--optimized"]
```

**Other platforms**
- Bare metal: copy the jar into `providers/`, run `kc.sh build`, then restart.
- Helm: use an init container that downloads the release jar into the providers volume.
- Phase Two: Config → Resources → Theme.

Theme env vars are ordinary container environment variables. See [environment-variables.md](environment-variables.md).

## Enable the theme

Admin Console → **your realm** (not `master`) → Realm Settings → Themes:
- Choose the Login, Account and Email themes. The name is `themeName`.
- The login theme can also be set per client.

If the theme isn't listed, check these:
- The jar is in `providers/`.
- `kc.sh build` ran (needed for optimized images).
- The jar matches the Keycloak version.
- The startup logs.
