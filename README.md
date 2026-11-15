# pet-form-webinar

Landing de registro para la clase gratuita en vivo de **Plan de Edición Total (PET)**: cuenta regresiva, formulario de inscripción (va a Google Sheets mediante un Apps Script) y redirección al canal o grupo de WhatsApp de la convocatoria. Cada nueva convocatoria reutiliza esta misma página cambiando fecha, textos y enlace.

## Dónde vive

| Entorno | URL | Notas |
|---|---|---|
| **Producción (canónico)** | https://form-webinar.planediciontotal.com | Hostinger, hosting compartido. Es la URL de `<link rel="canonical">`, `og:url`, `og:image` y del JSON-LD en `index.html`, y la homepage del repo en GitHub. |
| Vercel (antiguo entorno de pruebas) | https://pet-editagratis-form.vercel.app | Proyecto de Vercel `pet-editagratis-form`. **Desactualizado:** sigue sirviendo el primer webinar (`startDate` 2026-08-10); no se actualiza con los push a `main`. No hay `vercel.json` y Vercel **no lee `.htaccess`**, así que ahí no hay CSP ni cabeceras. |
| GitHub Pages | https://davidprietooficial-ux.github.io/pet-form-webinar/ | Solo se publica a mano (`workflow_dispatch` de `pages.yml`). **Desactualizado:** también muestra el webinar del 2026-08-10. |

## Aviso: producción va por delante de este repo

Revisado el 2026-11-14: lo que hay publicado en Hostinger **no coincide con `main` ni con la rama `hostinger`** (último build: commit `66bedee`, 2026-08-28).

| | Repo (`main` / `hostinger`) | Producción |
|---|---|---|
| Fecha | Lunes 2026-08-31, 6:00 p. m. | Lunes 2026-10-19, 7:00 p. m. |
| Nombre | "Masterclass de edición en vivo" | "Workshop de edición en vivo" |
| Destino tras registrarse | Canal de WhatsApp (`ENLACE_CANAL_WHATSAPP`) | Grupo de WhatsApp (enlace de invitación, solo en el código local) |
| Teléfono | Selector de país + número | Solo número (país fijo `+57` en campo oculto) |
| Aviso "Protégete de estafas" | No | Sí, debajo del formulario |

Esos cambios existen solo como modificaciones **sin commit** en una copia de trabajo local del repo (cambian `index.html`, las 3 legales distintas de cookies, `src/estilos/sitio.css` y `src/lib/formulario.ts`). **Antes de tocar nada, hay que hacer commit de esa versión en `main`**; si no, el próximo despliegue desde este repo devolvería producción al webinar del 31 de agosto. Si `ENLACE_CANAL_WHATSAPP` y `WEBINAR_ACTUAL` del Apps Script ya apuntan a la convocatoria del 19 de octubre: [por confirmar].

## Stack

- **Vite 6** + **TypeScript 5** (sin framework, HTML estático con mejora progresiva).
- **Tailwind CSS 4** (`@tailwindcss/vite`) + tokens propios en `src/estilos/tokens.css`.
- Animaciones con Web Animations API + IntersectionObserver, sin librerías.
- Backend del formulario: **Google Apps Script** (Sheets + correo de aviso + correo de bienvenida). Su código **no está en este repo** (si tiene copia en otro repo de GitHub: [por confirmar]): vive en la carpeta `apps-script/` (`Código.js` + `appsscript.json`) de la carpeta local del proyecto original (`plan-edicion-total/`, al lado de `sitio/`, que es la copia de trabajo de este repo). Es un clon de `clasp`: se edita `Código.js` → `clasp push` → `clasp redeploy <deploymentId>` sobre el deployment existente, para no cambiar la URL `/macros/s/…/exec` que usa el formulario.
- Prettier con `prettier-plugin-tailwindcss`.

## Estructura

```
index.html                    Landing completa (hero, formulario, video, carrusel, pie). El comentario inicial guarda el historial de cada webinar.
terminos-y-condiciones/       Página legal (entrada propia de Vite)
politica-de-cookies/          Página legal
aviso-legal/                  Página legal (trae la URL del canal de WhatsApp escrita)
politica-de-privacidad/       Página legal
src/main.ts                   Punto de entrada: barra flotante, cupos simulados, toggle del video, videos del carrusel
src/lib/consentimiento.ts     Banner de cookies y cola de terceros (nada carga sin aceptar, salvo el Meta Pixel)
src/lib/tracking.ts           IDs de GA4 / GTM / Clarity / TikTok y eventos de conversión
src/lib/formulario.ts         Validación, envío por fetch al Apps Script, evento Lead y redirección
src/lib/motion.ts             Reveal y cuenta regresiva ([data-cuenta-hasta])
src/lib/reproductor.ts        Reproductor propio del video de instrucciones (retoma desde localStorage)
src/lib/sanitizar.ts          Utilidades de limpieza de texto
src/estilos/                  tokens.css (colores, espaciado) y sitio.css
public/.htaccess              HTTPS, CSP, cabeceras de seguridad, caché y bloqueos (Apache/LiteSpeed)
public/efectos/               Videos y pósters del carrusel de efectos (9:16)
public/fuentes/               Fuentes woff2 autoalojadas
public/og.jpg                 Miniatura para compartir (1200x630), se genera con `npm run og`
scripts/csp-hash.mjs          Calcula el sha256 de los scripts inline de dist/ para la CSP
scripts/og.sh                 Genera public/og.jpg desde public/video-hero.jpg (requiere ffmpeg)
.github/workflows/            hostinger.yml (build a la rama `hostinger`) y pages.yml (GitHub Pages, manual)
```

## Local

Requiere Node 20 (es la versión de los workflows).

```bash
npm install
npm run dev            # http://localhost:5173 (abre el navegador)
npm run build          # tsc --noEmit + vite build → dist/
npm run preview        # sirve dist/ en http://localhost:4173
npm run formato        # prettier --write .
npm run formato:check  # prettier --check .
npm run csp:hash       # hash de los scripts inline de dist/ (correr después de build)
npm run og             # regenera public/og.jpg
npm run empaquetar     # build + zip de dist/ en pet-hostinger.zip (sin .DS_Store ni ._*)
```

El build quita los comentarios HTML (plugin `quitarComentariosHtml` en `vite.config.ts`), no genera sourcemaps y crea un `index.html` por cada página legal. `base` es `/` salvo que se pase `VITE_BASE_PATH` (solo lo usa Pages).

## Despliegue a producción (Hostinger)

0. Leer antes el aviso de arriba: mientras `main` no tenga la versión de producción, **no desplegar desde este repo**.
1. Hacer los cambios en `main` y hacer push.
2. La Action **Publicar en Hostinger** (`.github/workflows/hostinger.yml`) compila y reescribe la rama **`hostinger`** con el contenido de `dist/` en la raíz (push `--force`, rama huérfana, no se edita a mano). Comprueba que `dist/.htaccess` exista.
3. El subdominio en Hostinger **no está conectado a Git** (el comentario del workflow habla de apuntar hPanel a la rama `hostinger`, pero hoy no se usa así). Hay que subir los archivos a mano:
   - Opción A: descargar el contenido de la rama `hostinger` (por ejemplo `git fetch origin && git archive --format=zip -o pet-hostinger.zip origin/hostinger`).
   - Opción B: `npm run empaquetar` y usar `pet-hostinger.zip`.
4. En hPanel → Administrador de archivos, subir todo a la carpeta del subdominio `form-webinar` (o usar la API/MCP de Hostinger), reemplazando lo anterior. **No olvidar `.htaccess`** (archivo oculto): sin él no hay CSP, HTTPS forzado ni caché.
5. Verificar: abrir el sitio, enviar un registro de prueba, comprobar que llega la cabecera `Content-Security-Policy` (`curl -sI https://form-webinar.planediciontotal.com/`) y que `https://form-webinar.planediciontotal.com/.env` devuelva 403/404 (hoy devuelve 403).

Nunca subir la raíz del repo (serviría `package.json` y `src/`).

## Tracking e integraciones

| Herramienta | ID público | Dónde | Cómo carga |
|---|---|---|---|
| Meta Pixel | `884471167722492` | `index.html` líneas 158–177 (snippet inline) y 246–252 (`<noscript>`) | Sin esperar consentimiento (decisión del cliente, 2026-08-06). PageView al cargar, `Lead` al enviar el formulario (`src/lib/formulario.ts` línea 168). |
| Google Analytics 4 | `G-QKYH7ZEGHQ` | `src/lib/tracking.ts` línea 47 | Tras aceptar "analítica" en el banner. **Ver problemas conocidos.** |
| Microsoft Clarity | `y1tp72wjj8` | `src/lib/tracking.ts` línea 51 | Tras aceptar "analítica". **Ver problemas conocidos.** Confirmar enmascaramiento de campos en su panel. |
| GTM / Google Ads / TikTok | vacíos | `src/lib/tracking.ts` líneas 48–50 | No cargan. No llenar GA4 y GTM a la vez (se duplican las visitas). |
| Formulario | Apps Script | `action` del `<form>` en `index.html` línea 392 | `fetch` POST; el éxito se decide por el JSON `{ ok, mensaje }`. |

Eventos propios (`registrarConversiones` en `tracking.ts`): `conversion`, `clic_whatsapp` y `clic_telefono`, enviados a `gtag` y `fbq('trackCustom')`.

El formulario lleva un honeypot (`web`) y un campo oculto `token` que el Apps Script comprueba. No es un secreto real (está a la vista en el HTML), solo frena bots que llamen directo a la URL. Si se cambia en el Apps Script (constante `TOKEN_SECRETO`), hay que cambiarlo también en el `<input name="token">` de `index.html` (línea 408).

Hotmart: no hay integración en este repo.

## Qué se edita más a menudo

| Dato | Dónde |
|---|---|
| **Fecha y hora del webinar** | Los 3 `data-cuenta-hasta` de `index.html` (líneas 263, 312 y 610) **y** el `startDate` del JSON-LD (línea 229). Hay que cambiar los cuatro o los contadores se desincronizan. Formato `2026-08-31T18:00:00-05:00`. Además, el texto visible "Lunes · En vivo · 6:00 p. m." (línea 342). |
| Nombre del evento ("Masterclass…") | `<title>` (183), `og:title` (191), etiqueta de la barra flotante (285), antetítulo del hero (347) y `name` del JSON-LD (227). |
| **Enlace de WhatsApp** | `data-exito-redirige` del `<form>` (`index.html` línea 391) + `aviso-legal/index.html` + la constante `ENLACE_CANAL_WHATSAPP` en `apps-script/Código.js` (fuera de este repo). |
| Pestaña de Google Sheets del webinar | Constante `WEBINAR_ACTUAL` en el Apps Script (fuera de este repo). Cambiarla **antes** de abrir inscripciones. |
| Título, descripción y tarjeta para compartir | `<title>`, `meta description`, `og:*` (líneas 183–212) y `name`/`description` del JSON-LD. |
| Antetítulo y `<h1>` del hero | `index.html` alrededor de la línea 350. |
| Cupos simulados | `data-cupos-inicial` y `data-cupos-piso` (líneas 365–366). Es urgencia simulada, no inventario real. |
| Mensaje de éxito | `data-mensaje-exito` (línea 479). |
| Video del hero / instrucciones | `public/video-hero.*`, `public/video-instrucciones.*`. |
| Efectos del carrusel | `public/efectos/` + los bloques `data-video-efecto` en `index.html` (cada efecto va dos veces por el loop infinito). |
| Colores y espaciado | `src/estilos/tokens.css`. |
| Textos legales | Las 4 carpetas legales. Son borrador, sin revisión legal. |

## Seguridad (`public/.htaccess`)

- HTTPS obligatorio, quita `www`, HSTS de un año con subdominios, URLs limpias (`/x` sirve `x.html` si existe).
- `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`.
- **CSP estricta** (línea 92). Incluye el hash sha256 del único script inline (el snippet del Meta Pixel). **Si se toca ese snippet, aunque sea un espacio**, correr `npm run build && npm run csp:hash` y pegar el nuevo hash en `script-src`; si no, el pixel se bloquea sin aviso.
- Caché de un año para assets con hash; HTML sin caché.
- Bloquea archivos ocultos, `.env`, `.map`, `.json`, `.md`, `.yml`, etc., y `/.git` y `/node_modules`.
- Rechaza con 403 los métodos `TRACE`, `TRACK`, `DEBUG`, `CONNECT`, `PATCH`, `PUT` y `DELETE`, y limita el cuerpo a 2 MB.
- Solo aplica en Hostinger (Apache/LiteSpeed); comprobado que producción envía la CSP. En Vercel y Pages no hay CSP.

## Problemas conocidos

1. **GA4 no recibe datos.** `src/lib/tracking.ts` líneas 84–86: el `gtag` propio hace `window.dataLayer.push(args)` (un array). gtag.js solo procesa objetos `arguments`, así que ignora los `js`, `config` y eventos. Corrección: `window.dataLayer.push(arguments)` dentro de una `function` normal (no flecha).
2. **La CSP bloquea Microsoft Clarity.** `public/.htaccess` línea 92: `script-src` solo permite `https://www.clarity.ms`, pero el tag carga después su script desde otro subdominio de `clarity.ms`; falta `https://*.clarity.ms` en `script-src` (y revisar `img-src`). Además, `src/lib/tracking.ts` líneas 106–110 carga el tag sin crear antes el stub `window.clarity` que trae el snippet oficial.
3. **La CSP no permite el Conversions API Gateway de Meta.** `connect-src` (línea 92) solo trae `https://www.facebook.com`; falta el host del gateway [por confirmar: dominio exacto configurado en el Administrador de eventos].
4. **`ErrorDocument 404 /404.html`** (última línea de `.htaccess`), pero no existe ningún `404.html` en el build.
5. Restos de la plantilla original: `.htaccess` menciona `docs/seguridad.md` y la ruta `sitio/public/.htaccess`; `consentimiento.ts`, `sanitizar.ts`, `tokens.css` y `vite.config.ts` hablan de `npm run gate`. Ninguno de los dos existe en este repo.
6. `package.json` conserva el nombre antiguo `plan-de-edicion-total`, y `src/main.ts` dice "5 efectos × 2" aunque el carrusel tiene 18 videos (9 × 2).
7. **El repo está por detrás de producción** (ver el aviso del principio). Vercel y GitHub Pages siguen con el primer webinar.
8. Pendientes del cliente anotados en `index.html`: efecto de avance/desaceleración del video (sin confirmar), más videos para el carrusel y revisión legal de los textos.

## Repos relacionados

Todos en `github.com/davidprietooficial-ux`, con la convención `pet-<sitio>`.

| Repo | Sitio |
|---|---|
| **pet-form-webinar** (este) | form-webinar.planediciontotal.com (Hostinger) |
| pet-clase | clase.planediciontotal.com (Hostinger) |
| pet-plandedicion-venta | pet-plandedicion-venta.vercel.app (Vercel) |
| pet-capcutclub | Landing de CapCut Club |
| pet-capcutclub-preview | Boceto de revisión en GitHub Pages |
| pet-workshopexclusivo | workshopexclusivo.planediciontotal.com (Hostinger, privado) |

Subdominios de `planediciontotal.com` en Hostinger: `clase`, `form-webinar`, `venta-envivo`, `capcutclub` (y `workshopexclusivo` en creación). Ninguno está conectado a Git: los archivos se suben a mano o con la API/MCP de Hostinger.
