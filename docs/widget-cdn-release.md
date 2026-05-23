# Quelora Widget — CDN Release System

> [↑ Docs index](../README.md) · Source: `quelora-widget-community/.ai/cdn-release.md`
> Related: [packages/quelora-widget-community](../packages/quelora-widget-community.md)

## 1. Arquitectura

```
Developer machine
  └─ npm run release:minor
       ├── 1. Bump version en package.json  (0.1.0 → 0.2.0)
       ├── 2. rollup build  →  js/dist/
       └── 3. Upload a R2   →  quelora-cdn/v0.2.0/...

Cloudflare R2 (quelora-cdn)
  └─ v0.2.0/
       ├── quelora.js        ← público
       ├── chunks/           ← público
       ├── worker/           ← público
       ├── css/              ← público
       ├── locales/          ← público
       ├── vendors/          ← público
       ├── sw.js             ← público
       ├── enterprise/       ← PROTEGIDO (Worker valida Origin)
       └── plugins/          ← público

Cloudflare Worker (cdn-guard.js)
  └─ cdn.quelora.com/*
       ├── /v*/enterprise/* → valida Origin en KV ALLOWED_DOMAINS → 403 si no está
       └── resto             → sirve directo desde R2
```

**Regla de acceso enterprise:** El Worker intercepta cualquier request a
`/v*/enterprise/*` y verifica que el hostname del `Origin` del request esté
como clave en el KV `ALLOWED_DOMAINS`. Si no → 403 Forbidden.
Esto protege los módulos enterprise (gamification, survey, chat, resilience, SSE,
p2p, live, banana) sin afectar al widget base.

---

## 2. Archivos del sistema

| Archivo | Función |
|---|---|
| `package.json` | Versión semver + npm scripts de release |
| `scripts/release.mjs` | Script principal: bump → build → upload a R2 |
| `.cloudflare/wrangler.toml` | Config del Worker (bucket, KV, dominio) |
| `.cloudflare/cdn-guard.js` | Cloudflare Worker — access control + serve desde R2 |
| `.env.example` | Template de variables de entorno (no commitear `.env`) |

---

## 3. Setup inicial (una sola vez)

### 3.1 Instalar Wrangler CLI
```bash
npm install -g wrangler
wrangler login     # abre el browser para autenticar con Cloudflare
```

### 3.2 Crear el bucket R2
```bash
wrangler r2 bucket create quelora-cdn
```

### 3.3 Crear el KV namespace para el allowlist de dominios
```bash
wrangler kv namespace create ALLOWED_DOMAINS
# Guarda el ID que imprime — va en .cloudflare/wrangler.toml → id = "..."
```

### 3.4 Configurar el dominio CDN
En el dashboard de Cloudflare, en el DNS de `quelora.com`:
- Crear registro CNAME: `cdn` → `quelora-cdn.r2.cloudflarestorage.com`
- Asegurarse de que esté en modo "Proxied" (nube naranja)

### 3.5 Completar wrangler.toml
Abrir `.cloudflare/wrangler.toml` y reemplazar `PLACEHOLDER_KV_NAMESPACE_ID`
con el ID obtenido en 3.3.

### 3.6 Crear el token R2 para uploads
En Cloudflare Dashboard → R2 → Manage R2 API Tokens:
- Crear token con permiso `Object Read & Write` en bucket `quelora-cdn`
- Guardar `Access Key ID` y `Secret Access Key`

### 3.7 Configurar variables de entorno locales
```bash
cp .env.example .env
# Editar .env con los valores reales:
# CF_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET_NAME
```

### 3.8 Instalar dependencias del release script
```bash
# Desde la raíz del monorepo (apps/)
npm install
```

### 3.9 Desplegar el Worker
```bash
cd .cloudflare
wrangler deploy
```

---

## 4. Workflow de release

```bash
# Bug fix (1.0.0 → 1.0.1)
npm run release:patch

# Feature nueva, compatible (1.0.0 → 1.1.0)
npm run release:minor

# Breaking change (1.0.0 → 2.0.0)
npm run release:major

# Solo build, sin subir (para verificar antes de publicar)
npm run release:dry
```

Cada comando:
1. Incrementa la versión en `package.json`
2. Corre el build de Rollup
3. Sube todos los archivos de `js/dist/` a R2 bajo `v{X.Y.Z}/`

El script lee las credenciales desde las variables de entorno (`.env` o shell).
Recomendado usar `dotenv` o cargar el `.env` antes de correr:
```bash
# Opción simple (bash)
export $(cat .env | xargs) && npm run release:patch
```

---

## 5. Gestión de acceso enterprise (clientes)

Los dominios autorizados viven en el KV `ALLOWED_DOMAINS`.
Clave = hostname del cliente (sin protocolo ni path). Valor = cualquier string.

```bash
# Autorizar un dominio
wrangler kv key put --namespace-id=<KV_ID> "cliente.com" "1"

# Revocar un dominio
wrangler kv key delete --namespace-id=<KV_ID> "cliente.com"

# Listar dominios autorizados
wrangler kv key list --namespace-id=<KV_ID>
```

Los cambios en KV toman efecto de forma inmediata sin re-deploy del Worker.

---

## 6. Cómo integra el widget el cliente

El cliente coloca en su HTML:
```html
<script>
  window.QUELORA_CONFIG = {
    cid: 'QU-XXXXXXXX-XXXXX',
    apiUrl: 'https://api.quelora.org',
    // ...
  };
</script>
<script type="module" src="https://cdn.quelora.com/v1.0.0/quelora.js"></script>
```

La versión específica (`v1.0.0`) garantiza estabilidad — el cliente no recibe
cambios inesperados hasta que actualice deliberadamente la URL.

---

## 7. Rollback

Para volver a una versión anterior, el cliente simplemente cambia la URL:
```html
<script type="module" src="https://cdn.quelora.com/v0.9.0/quelora.js"></script>
```

Las versiones anteriores permanecen en R2 indefinidamente a menos que se borren
manualmente:
```bash
# Listar archivos de una versión
wrangler r2 object list quelora-cdn --prefix="v0.9.0/"

# Eliminar una versión completa (con cuidado)
# No hay comando bulk delete en wrangler — usar la API o el dashboard
```

---

## 8. Verificación post-deploy

```bash
# Verificar que el archivo principal es accesible
curl -I https://cdn.quelora.com/v{VERSION}/quelora.js

# Verificar que enterprise está protegido (debe devolver 401/403)
curl -I https://cdn.quelora.com/v{VERSION}/enterprise/survey/survey.js

# Verificar con Origin autorizado (debe devolver 200)
curl -I -H "Origin: https://cliente.com" \
  https://cdn.quelora.com/v{VERSION}/enterprise/survey/survey.js
```
