# Nuestro fork: sincronización y despliegue en Dokploy

## Remotos

- `origin` → `pabloluna3596afk/hermes-agent` (nuestro, `main` = lo que se despliega)
- `upstream` → `NousResearch/hermes-agent` (original)

Actualizar a mano:

```sh
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

Regla: los cambios propios van en archivos nuevos (`*.dokploy.yml`, `DEPLOY-DOKPLOY.md`,
plugins, skills), no editando archivos del original. Así los merges no generan conflictos.

## Sincronización automática

`.github/workflows/sync-upstream.yml` corre cada lunes (o a mano desde Actions → Sync upstream)
y abre un PR `sync/upstream → main`. Al hacer merge del PR, Dokploy redepliega.

Configuración (una sola vez):

1. GitHub → Settings → Developer settings → Fine-grained tokens → crear un token solo para
   `pabloluna3596afk/hermes-agent` con **Contents**, **Pull requests** y **Workflows** en Read & write.
2. Repo → Settings → Secrets and variables → Actions → nuevo secreto `SYNC_TOKEN` con ese token.
3. Repo → Actions → habilitar workflows en el fork.
4. Deshabilitar los workflows del original (CI, deploy-site, etc.) para que no corran en el fork:

   ```sh
   gh workflow list --repo pabloluna3596afk/hermes-agent --json name,path -q '.[] | select(.path != ".github/workflows/sync-upstream.yml") | .path' \
     | xargs -I{} gh workflow disable {} --repo pabloluna3596afk/hermes-agent
   ```

## Dokploy

El servidor (arm64) **no compila**: `.github/workflows/build-image.yml` construye la imagen en
un runner ARM de GitHub y la publica en `ghcr.io/pabloluna3596afk/hermes-agent:latest`.
Tras el primer build: GitHub → Packages → `hermes-agent` → Package settings → visibilidad **Public**
(si no, Dokploy necesita credenciales de registry).

1. Proyecto `main` → compose **raw** `hermes` con el contenido de `docker-compose.dokploy.yml`
   (igual que chatwoot).
2. Environment:

   ```
   HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
   HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=<contraseña fuerte>
   HERMES_DASHBOARD_BASIC_AUTH_SECRET=<openssl rand -base64 32>
   ```

3. Domains → servicio `hermes`, puerto `9119`, HTTPS con Let's Encrypt.
4. Deploy. Para redeploy automático tras cada build, copiar el Webhook URL del compose en Dokploy
   al secreto `DOKPLOY_WEBHOOK_URL` del repo.
5. Primera configuración (modelo, API keys, Telegram, etc.): desde el dashboard, o por terminal
   del contenedor en Dokploy con `hermes setup`.
6. Si el login del dashboard falla detrás de Traefik, editar `/opt/data/config.yaml` en el volumen:

   ```yaml
   dashboard:
     public_url: "https://hermes.tudominio.com"
     trusted_proxies:
       - "10.0.1.0/24"   # subred de dokploy-network: docker network inspect dokploy-network
   ```

Los datos (config, sesiones, memoria, `.env` con las keys) viven en el volumen `hermes-data`:
configurar un backup del volumen en Dokploy.
