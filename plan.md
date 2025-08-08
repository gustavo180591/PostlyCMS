# Plan de Implementación — PostlyCMS
> Stack: **SvelteKit 2 · Svelte 5 · TypeScript · Tailwind CSS 4 · Prisma · PostgreSQL · Zod · Docker · Vitest · Playwright**  
> Objetivo: CMS básico con **registro con foto**, **aprobación manual** y **acceso por roles (admin/user)**. Orientado a **apps privadas**, **portales educativos** y **membresías**.

---

## 0) Resultado esperado
Un proyecto que corre al clonar y seguir el README, con:
- Registro con foto → estado `PENDING` → aprobación manual en `/admin` → acceso a `/app` si `APPROVED`.
- CMS con **Entradas** y **Páginas** (SEO, draft/published/scheduled, categorías/tags, media library, revisiones).
- RBAC: **admin** (todo) y **user** (zona privada, lectura premium, editar perfil).
- Emails transaccionales (alta, aprobación/rechazo, reset; verificación opcional).
- DevX sólido (Docker, seeds, tests, lint/format) y guías de deploy (Vercel/Fly/Railway).

---

## 1) Preparación del entorno
- [x] **Node 20+** y **pnpm o npm** instalados.
- [x] **Docker** y **Docker Compose** instalados.
- [x] Crear carpeta del proyecto `postlycms/` o clonar el repo base.
- [x] Instalar dependencias: `npm i`.
- [x] Copiar variables: `cp .env.example .env` y completar claves.

> **Pro tip:** en desarrollo, usar MailHog/SMTP de pruebas y `STORAGE_DRIVER=local`.

---

## 2) Base de datos y Prisma
- [x] Configurar `DATABASE_URL` en `.env` (PostgreSQL local o Docker).
- [x] Revisar/ajustar **`prisma/schema.prisma`** (modelos: `User`, `Post`, `Page`, `Category`, `Tag`, `PostTag`, `Media`, `Revision`, `AuditLog`, `Session`, `Token`).
- [x] Generar cliente: `npx prisma generate`.
- [x] Crear/migrar esquema: `npx prisma migrate dev` (dev) o `npx prisma migrate deploy` (prod).
- [x] Cargar datos demo: `npm run seed`.

**Criterio de aceptación**
- `User.email`, `Post.slug`, `Page.slug` con índices únicos.
- Seed con: **1 admin**, **1 user APPROVED**, **1 user PENDING**, **posts demo**, **páginas demo**, **categorías/tags**.

---

## 3) Autenticación y sesiones
- [ ] Implementar `Session` con cookie **HttpOnly, Secure, SameSite=Lax** (`sid`).  
- [ ] `hooks.server.ts` debe hidratar `event.locals.user` cuando la sesión es válida.
- [ ] **Login**: valida credenciales (Zod), crea `Session`, setea cookie, redirige a `/app` (si `APPROVED`) o `/auth/pending` (si `PENDING`).  
- [ ] **Logout**: elimina `Session`, limpia cookie.  
- [ ] **Reset password**: genera `Token` tipo `RESET`, envía email, valida y actualiza hash.  
- [ ] **Email verify (opcional)**: flag `EMAIL_VERIFY=1`; `Token` tipo `VERIFY` y endpoint de verificación.

**Seguridad de passwords**
- [ ] Usar **Argon2** (o bcrypt como fallback) en producción.
- [ ] Política mínima: 8+ chars, blacklist básica, rate-limit.

---

## 4) Registro con foto y estados de cuenta
- [ ] Formulario **/auth/register**: `name`, `email`, `password`, **foto de perfil** (preview), **aceptación Términos/Privacidad**.  
- [ ] Guardar la foto en **Media Library** (local o S3-like); asociar `photoUrl` en `User`.  
- [ ] Estado inicial `PENDING`; mostrar vista `/auth/pending` tras registrarse.
- [ ] **Email “alta recibida”** se envía al completar el registro.

**Criterio de aceptación**
- Subida con preview antes de enviar (UI), validación de tamaño/tipo, sanitización de nombre de archivo.
- Si `EMAIL_VERIFY=1`, tras registro pedir verificación.

---

## 5) RBAC y guards
- [ ] Implementar helpers `requireUser`, `requireApproved`, `requireAdmin`.  
- [ ] Proteger **/app/** con `requireApproved` y **/admin/** con `requireAdmin`.  
- [ ] En `load` server-side de layouts, cortar acceso si no cumple (redirect/403).

**Pruebas unitarias (Vitest)**
- [ ] Reglas de acceso por rol/estado.
- [ ] Transiciones válidas `PENDING → APPROVED/REJECTED`.

---

## 6) Panel /admin — Gestión de usuarios
- [ ] Listado con filtros por `status` (foco en `PENDING`).  
- [ ] Vista detalle: datos, foto, historial básico (AuditLog), acciones: **Aprobar**, **Rechazar** (con motivo), **Promover/Degradar** rol.  
- [ ] Al **Aprobar**: set `status=APPROVED`, **rol por defecto `USER`**, enviar email de aprobación.  
- [ ] Al **Rechazar**: set `status=REJECTED`, guardar motivo (en `AuditLog.meta`) y enviar email.

**Export CSV**
- [ ] Endpoints `/admin/export/users` y `/admin/export/posts` (streaming `text/csv`, paginado).

---

## 7) CMS — Entradas y Páginas
- [ ] ABM de **Posts** y **Pages** con estados `draft/published/scheduled`.  
- [ ] Editor **Markdown** con preview en vivo o **Rich-text tipado** (schema JSON) — configurable.  
- [ ] Campos: `title`, `slug` único, `summary`, `content`, `publishedAt`, `isPremium` (en Posts), `categories`, `tags`, **portada destacada** (Media).  
- [ ] **Revisiones**: crear `Revision` en cada guardado (diff/meta).  
- [ ] **Scheduled**: CRON/Job que publique cuando `publishedAt <= now()`.

**Listado público (/blog)**
- [ ] Paginación, búsqueda (título/summary), filtros por categoría/tag.  
- [ ] Detalle `/blog/[slug]` respeta `isPremium` (requiere sesión aprobada para ver contenido completo si premium).

**Páginas públicas (/p/[slug])**
- [ ] SEO completo y contenido desde DB.

---

## 8) Media Library
- [ ] Driver **local** (carpeta pública) o **S3-like** (R2/MinIO/S3) mediante variables `STORAGE_DRIVER` y `S3_*`.  
- [ ] Subida con barra de progreso, previsualización, y metadatos (`alt`, `type`, `size`).  
- [ ] (Nice-to-have) **Thumbnails** y **re-dimensionado** server-side.

**Criterios**
- URLs estables; si local, servir desde `/static/uploads/`; si S3, usar endpoint y bucket configurables.
- Validación de tipos y límites de tamaño.

---

## 9) SEO y metadatos
- [ ] Helper para `title/description/OG/Twitter`.  
- [ ] `<svelte:head>` en Home, Blog, Post, Page.  
- [ ] `robots.txt` y `sitemap.xml` simples (opcional).

---

## 10) Emails transaccionales
- [ ] Configurar `SMTP_*` y `EMAIL_FROM` en `.env`.  
- [ ] Plantillas HTML/TXT editables: **alta recibida**, **aprobación**, **rechazo**, **reset password**, **verify email**.  
- [ ] Enviar en los eventos correspondientes (registro, cambio de estado, reset, verify).

**Test rápido (dev)**
- [ ] MailHog / Mailpit y verificación de contenido/links.

---

## 11) Seguridad y hardening
- [ ] **CSRF** (double submit cookie + campo oculto) en `actions`.  
- [ ] **Rate-limit** en login/registro: `RATE_LIMIT_*` (por IP/email).  
- [ ] Sanitización de **Markdown** (whitelist) o escape server-side.  
- [ ] Encabezados de seguridad (CSP, no-sniff, frame-ancestors).  
- [ ] Cookies seguras siempre; expirar sesiones inactivas.

---

## 12) Frontend y UI (Tailwind 4)
- [ ] Componentes reutilizables: `Card`, `Table`, `Modal`, `Form`, `Toast`, `Badge`.  
- [ ] Formularios con estados `loading/success/error`, `aria-*`, focus visible.  
- [ ] **Modo oscuro** por clase (`dark`).  
- [ ] Accesibilidad: labels, roles/aria, contraste.

---

## 13) Dashboard /admin
- [ ] Widgets: conteo de usuarios por **estado/rol**, posts **publicados/pendientes**, **últimas altas**.  
- [ ] Tabla “últimos registros” con acción rápida de aprobación.

---

## 14) Endpoints operativos
- [ ] `/health` → `ok`.  
- [ ] `/version` → JSON `{ version }` leído desde `package.json` o constante.  
- [ ] Logs en consola/archivo (sensible en prod).

---

## 15) Tests (Vitest + Playwright)
**Unitarios (Vitest)**
- [ ] RBAC guards y helpers de estado/rol.  
- [ ] Validaciones Zod (login, registro, posts/pages).  
- [ ] Transiciones de estados y generación de `Revision`.

**E2E (Playwright)**
- [ ] Flujo: **registrar** → **admin aprueba** → **usuario accede `/app`** → **visualiza post premium**.  
- [ ] Flujo de **reset password**.  
- [ ] Flujo de **scheduled publish** con mock de tiempo o CRON manual.

---

## 16) Docker y DevX
- [ ] `docker-compose` con `db` (PostgreSQL) y `app`.  
- [ ] Script único: `npm run dev:docker` levanta todo.  
- [ ] Scripts package.json: `dev`, `build`, `preview`, `lint`, `format`, `test`, `e2e`, `prisma:*`, `seed`, `dev:docker`.
- [ ] `.env.example` completo (DB, AUTH, RATE_LIMIT, SMTP, STORAGE, PUBLIC_BASE_URL, TZ).

**Criterios**
- El proyecto debe levantar con **un solo comando** y generar datos demo.

---

## 17) Deploy (Vercel / Fly.io / Railway)
- [ ] Crear base de datos gestionada y ajustar `DATABASE_URL`.  
- [ ] Configurar variables de entorno (AUTH_SECRET, SMTP, STORAGE, etc.).  
- [ ] Pipeline: `prisma migrate deploy` en el build o post-deploy.  
- [ ] Programar **CRON** (provider o GitHub Actions) para `scheduled publish`.  
- [ ] Verificar `/health` y `/version` post-deploy.

**Notas**
- En Vercel, usar Adaptador de SvelteKit y “Edge Functions” según necesidad (sesiones en DB).  
- En Fly/Railway, mantener `Dockerfile` y procesos de arranque (`node build/index.js`).

---

## 18) Checklists rápidas por rol

### Admin
- [ ] Ingresar a `/admin`, ver usuarios `PENDING`.
- [ ] Abrir detalle, aprobar/rechazar con motivo.
- [ ] Exportar CSV de usuarios y posts.
- [ ] Crear post **scheduled** con `publishedAt` futuro.

### Usuario
- [ ] Registrarse con foto y aceptar términos.
- [ ] Recibir email “alta recibida”.
- [ ] Tras aprobación, ingresar a `/app` y ver contenido premium.

---

## 19) Flujo E2E de aceptación (manual)
1. **Registro** con foto → vista **pendiente**.  
2. **Email** de alta recibida.  
3. **Admin** aprueba en `/admin/users/[id]` → email de aprobación.  
4. Usuario hace **login** → accede a `/app`.  
5. Abre `/blog/[slug premium]` → contenido completo visible.  
6. Admin crea post `scheduled` → CRON lo publica a la hora definida.  
7. Exporta **CSV** de usuarios y posts sin errores.

---

## 20) Roadmap corto (después del MVP)
- [ ] Argon2 + rotación opcional de hash.  
- [ ] Thumbnails y resizing en Media Library.  
- [ ] Editor RichText tipado con Zod.  
- [ ] Roles ampliables (`EDITOR`) + permisos granulares.  
- [ ] Vista de `AuditLog` con filtros.

---

## 21) Comandos útiles (recopilación)
```bash
# Local
cp .env.example .env
npm i
npx prisma generate
npx prisma migrate dev
npm run seed
npm run dev

# Docker (dev)
npm run dev:docker

# Producción / Deploy
npx prisma migrate deploy

# Tests
npm run test
npm run e2e
```

---

## 22) Matriz de riesgos y mitigación (resumen)
- **Pérdida de sesiones** → sesiones en DB con expiración + regeneración de cookie.  
- **SPAM en registro** → rate-limit, captcha simple opcional.  
- **XSS por Markdown** → sanitización whitelist server-side.  
- **Errores SMTP** → reintentos y fallback a cola; monitor con Mailpit/Mailhog en dev.  
- **Archivos maliciosos** → validación MIME y tamaño; antivirus opcional en servidor.

---

## 23) Criterio de “hecho” (DoD) del MVP
- Levanta en limpio con `npm run dev` o `npm run dev:docker`.
- Se puede **registrar**, **aprobar** y **acceder** a `/app`.
- ABM de **Posts/Pages** funcional con revisiones y media.
- **Emails** de alta/aprobación/rechazo/reset operativos en dev.
- **RBAC** y **guards** cubiertos por tests unitarios básicos.
- **E2E** del flujo principal verde en CI.
