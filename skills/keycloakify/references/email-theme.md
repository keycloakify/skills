# Email theme

There are two options.

## A. Native FreeMarker templates

```bash
npx keycloakify initialize-email-theme   # choose "native"
```

- This creates `src/email/` with Keycloak's FreeMarker email templates and message bundles. Edit them directly.
- Put images and other assets in `src/email/resources/`. Reference them with `${url.resourcesUrl}/<file>`.
- Many email clients block images by default. Test in real clients.

## B. JSX emails (Vite only)

- React (jsx-email): https://github.com/timofei-iatsenko/keycloakify-emails
- Svelte: https://github.com/keycloakify/svelte-email
- Angular: https://github.com/keycloakify/angular-email

Write the templates as components. They are rendered to FreeMarker at build time. Follow each repo's README for the setup.

## Testing and enabling

- Build the jar, then select it under Realm Settings → Themes → Email theme.
- To trigger emails:
  - Configure SMTP under Realm Settings → Email (a local catcher like Mailpit works well).
  - Run "forgot password" or "verify email" flows from `npx keycloakify start-keycloak`.

Docs: https://docs.keycloakify.dev/theme-types/email-theme
