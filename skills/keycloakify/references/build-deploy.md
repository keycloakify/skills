# Build and deploy

## Build

```bash
npm run build-keycloak-theme   # vite build + keycloakify build
```

Output goes to `dist_keycloak/`. By default you get one jar per Keycloak version range, for example:

| Jar | Keycloak versions |
|---|---|
| `keycloak-theme-for-kc-22-to-25.jar` | 22–25 |
| `keycloak-theme-for-kc-all-other-versions.jar` | 11–21 and 26+ |

The set of jars depends on `keycloakVersionTargets` and `accountThemeImplementation`. Always check the actual file names in `dist_keycloak/`.

In CI, the runner needs Node, Maven and a JDK. The starter ships a GitHub Actions workflow that publishes the jars as release assets whenever the `package.json` version is bumped. It needs Settings → Actions → Workflow permissions set to "Read and write".

## Deploy

Theme jars go in `/opt/keycloak/providers/`.

**docker run (quick test)**

```bash
docker run -p 8080:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -v "./dist_keycloak/keycloak-theme-for-kc-all-other-versions.jar":/opt/keycloak/providers/keycloak-theme.jar \
  quay.io/keycloak/keycloak:26.0.7 start-dev
```

**Custom image (production)**

```dockerfile
FROM quay.io/keycloak/keycloak:26.0.7 AS builder
COPY dist_keycloak/keycloak-theme-for-kc-all-other-versions.jar /opt/keycloak/providers/
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:26.0.7
COPY --from=builder /opt/keycloak/ /opt/keycloak/
ENTRYPOINT ["/opt/keycloak/bin/kc.sh", "start", "--optimized"]
```

**Bare metal**

```bash
cp keycloak-theme-*.jar /opt/keycloak/providers/
/opt/keycloak/bin/kc.sh build
```

Then restart Keycloak.

**Other platforms**
- Docker Compose: mount a folder of jars, for example `./themes:/opt/keycloak/providers/`.
- Helm (for example the Bitnami chart): use an init container that downloads the jar from your GitHub release into the providers volume.
- Phase Two: Config → Resources → Add new resource → Theme, then upload the jar.

## Enable the theme

Admin Console → **your realm** (never `master` for apps) → Realm Settings → Themes:
- Choose the login, account and email themes. The name is `themeName` from `vite.config.ts`.
- The login theme can also be set per client, under the client's Login settings.
- The account theme is realm-level only.

If the theme isn't listed, check these:
- The jar is in `providers/`.
- `kc.sh build` ran (needed for optimized images).
- The jar matches the Keycloak version.
- The Keycloak startup logs.
