---
description: Verifica deploy post-push Strev — espera Render (API) + Cloudflare Pages (web), pollea /health, corre suite Playwright contra prod (https://strev.app). Reporta pass/fail por spec. Uso: tras `git push origin main`.
---

# /post-deploy-check

Pipeline de smoke post-push contra producción Strev.

**Idioma**: respuestas al usuario en castellano.

## Cuándo invocar

Después de `git push origin main` que dispara deploy auto:
- API: Render auto-deploy from `main` push en `StrevFit/strev-api`
- Web: Cloudflare Pages auto-deploy from `main` push en `StrevFit/strev-web`

Para PRs sin merge: skip (no hay deploy auto).

## Targets actuales

| Target | URL | Plataforma | Cold start |
|---|---|---|---|
| API health | `https://api.strev.app/health` | Render free | 30-75s |
| Web | `https://strev.app` | Cloudflare Pages | <5s (CDN) |

## Estado actual de servicios (pre-flight)

Antes de cualquier check, conocer el estado actual:

- **Stripe**: en TEST mode (sin cuenta real, sin Price IDs prod). Tests Stripe checkout solo validan redirect a `checkout.stripe.com` — NO completar pago real ni asumir webhook flow funcional end-to-end.
- **Sentry**: trial expirado. Si `VITE_SENTRY_DISABLED=true` está activo en CF Pages: errores frontend NO se reportan a Sentry. Para diagnosticar fallos: Render dashboard logs (backend) + browser DevTools (frontend) + ErrorBoundary del propio Strev.
- **Beta accounts** vivas: las 3 documentadas en `projects/strev/context/beta-test-accounts.md` (Pwd `<YOUR_BETA_PASSWORD>`). Suite Playwright las consume vía env vars `E2E_TRAINER_EMAIL/PASSWORD` + `E2E_ATHLETE_EMAIL/PASSWORD`.

## Flujo

### Fase A — Pre-flight

1. Confirma cwd = `~/Desktop/VanguardIA`.
2. `git log -1 --format="%h %s"` → muestra commit a verificar.
3. `git rev-parse HEAD` → guarda hash `EXPECTED`.
4. Pregunta: "¿Acabas de pushear a main?" Si no, abort.
5. Exporta env vars Playwright (para Fase D):
   ```bash
   export E2E_TRAINER_EMAIL="<TRAINER_EMAIL>"
   export E2E_TRAINER_PASSWORD="<YOUR_BETA_PASSWORD>"
   export E2E_ATHLETE_EMAIL="<ATHLETE_EMAIL>"
   export E2E_ATHLETE_PASSWORD="<YOUR_BETA_PASSWORD>"
   ```

### Fase B — Esperar deploys

#### B.1 Render API (cold start tolerante)

```bash
START=$(date +%s); MAX=300
while :; do
  HTTP=$(curl -s -o /tmp/health.json -w "%{http_code}" https://api.strev.app/health)
  if [ "$HTTP" = "200" ]; then
    echo "✓ API up after $(($(date +%s)-START))s"
    cat /tmp/health.json | jq .
    break
  fi
  ELAPSED=$(($(date +%s)-START))
  if [ "$ELAPSED" -gt "$MAX" ]; then
    echo "✗ API timeout — Render dashboard: https://dashboard.render.com"
    exit 1
  fi
  printf "  esperando %ds (HTTP %s)\r" "$ELAPSED" "$HTTP"
  sleep 5
done
```

Si `/health` devuelve campo `commit` o `version`: comparar contra `EXPECTED` (saber si Render desplegó NUESTRO commit, no uno antiguo). Si distinto: deploy en cola, esperar +60s y reintentar.

#### B.2 Cloudflare Pages

```bash
for i in $(seq 1 30); do
  HTTP=$(curl -sI -o /dev/null -w "%{http_code}" https://strev.app)
  [ "$HTTP" = "200" ] && { echo "✓ Web up"; break; }
  sleep 2
done
```

Para verificar versión nueva (no cacheada CF):
```bash
curl -s https://strev.app/index.html | grep -oE 'index-[A-Za-z0-9_-]+\.js' | head -1
```
Comparar con build local en `src/strev-web/dist/index.html` si está actualizado. Si CF sirve hash viejo después de 2 retries: posible cache, esperar 30-60s y reintentar.

### Fase C — Smoke endpoints API

```bash
# Health detallado
curl -s https://api.strev.app/health | jq .

# Waitlist (público, no requiere auth)
curl -s -X POST https://api.strev.app/api/leads/waitlist \
  -H "Content-Type: application/json" \
  -d '{"email":"smoke-deploy-check@strev.app"}' \
  -w "\nHTTP: %{http_code}\n"
# Esperado: 200 (idempotente) o 409 si ya existe

# Login con cuenta beta (auth flow)
curl -s -X POST https://api.strev.app/api/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$E2E_TRAINER_EMAIL\",\"password\":\"$E2E_TRAINER_PASSWORD\"}" \
  -w "\nHTTP: %{http_code}\n" -o /tmp/login.json
# Esperado: 200 + Set-Cookie strev_token
```

Si alguno falla → reportar al user, NO continuar a Fase D.

### Fase D — Playwright suite contra prod

```bash
cd ~/Desktop/VanguardIA/src/strev-web
npm run test:e2e
```

(default `playwright.config.js` apunta a `https://strev.app`, 5min timeout, 1 worker)

Specs en orden de criticidad. Si los primeros fallan, abortar para no malgastar tiempo:

1. `01-seo-pages.spec.js` — landing + privacy + terms + alternativas
2. `03-core-loop.spec.js` — login → dashboard → core flow
3. `02-registration.spec.js` — registro trainer + atleta
4. `10-reset-password.spec.js`
5. `06-plan-gating.spec.js`
6. `07-notifications.spec.js`
7. `09-stripe-checkout.spec.js` — **Stripe test mode**: solo verifica redirect a checkout.stripe.com, NO completa pago.
8. `08-visual-audit.spec.js`
9. `04-core-loop-full.spec.js`
10. `05-mobile.spec.js`

Subset rápido (~3-5 min): solo specs 1-3.

### Fase E — Reporte al user

```
═══════════════════════════════════════
POST-DEPLOY CHECK — commit <hash>
═══════════════════════════════════════
✓ Render API up    (Xs cold start)
✓ Cloudflare Pages up    (build hash <new>)
✓ /health: { commit: ..., uptime: ... }
✓ Waitlist endpoint: 200
✓ Beta login: 200 + Set-Cookie strev_token

Playwright suite (Y/Z passing):
  ✓ 01-seo-pages           (12s)
  ✓ 03-core-loop           (45s)
  ✗ 09-stripe-checkout     (120s — line 47, redirect timeout)
  ⏳ skipped resto por fail anterior

→ Acción sugerida:
  1. Investigar 09-stripe-checkout en Render dashboard logs (Sentry desactivado)
  2. Si flake transient: re-run `/post-deploy-check`
  3. Si bug real: rollback con `git revert HEAD && git push origin main`
```

## Reglas

1. **NUNCA** crear cuentas reales en producción. Si registration test crea user → ephemeral con prefix `smoke-deploy-` para limpieza fácil.
2. **NUNCA** completar payment real en Stripe. La cuenta está en test mode, pero verificar solo redirect.
3. Si `/health` responde lento (>30s) pero <5min: NORMAL (cold start Render free). No reportar como fallo.
4. CF Pages cache stale persistente (>2 retries con hash viejo): reportar al user, no auto-purgar (cache TTL típico 1-5min).
5. Si Playwright spec timeout >5min: marcar inconcluso, continuar resto.
6. **Sentry desactivado** (trial expirado). NO referenciar como fuente de diagnóstico hasta que Alex termine downgrade a Developer free + reactive `VITE_SENTRY_DISABLED=false` en CF Pages.

## Recovery

- Render duerme y no responde: típico free tier después de 15min idle. Reintentar 5min total.
- CF Pages cache stale: hard reload no es opción desde CLI. Reportar y esperar TTL.
- E2E spec falla por flake: re-run individual con `npx playwright test e2e/<spec>.spec.js`.
- Beta password rotada (improbable, está en runbook): pedir al user actualizar `projects/strev/context/beta-test-accounts.md` antes de continuar.
