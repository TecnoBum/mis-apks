# Mis Apks

Panel de control de las apps de Apklis de Johan (TecnoBum): versiones,
estados, alertas y lanzamiento de rondas de actualización. Primer paso
para automatizar el flujo que hoy se hace a mano con el agente:

**AppCreator24 → descargar APK → firmar con el keystore → subir a Apklis
como nueva versión → verificar en consola.**

## Estado (v1.3.0 — 20 nov 2026, construida; publicación pendiente junto al catálogo completo de 53 apps y el área de Ingresos)

- **Orden por defecto: última actualización, de más antigua a más
  reciente** (usa el campo real `actualizada_en` de `data/apps.json`;
  las apps sin fecha conocida van primero, que son las que más tiempo
  llevan sin actualizar). Botón para invertir la dirección («Más
  reciente primero»). Se mantienen los órdenes por nombre, precio y
  nº de versiones.
- **Actualizaciones programadas**: botón «Programar» en cada app y
  «Programar ronda» junto a la ronda completa, con selector de fecha
  (mínimo hoy, validado). La programación se guarda en el dispositivo
  (localStorage, clave `misapks_programadas`) y se muestra un
  distintivo con la fecha en la tarjeta de la app («Programada: 25 nov
  2026»; «Vencida: …» si pasó la fecha). Sección «Programadas» con la
  lista (app, fecha, tipo app/ronda) y botón para cancelar cada una.
  **El panel es estático y no ejecuta nada**: al programar o cancelar
  se copia al portapapeles el mensaje para pegarlo en el chat —
  «Programa la actualización de \<Nombre\> para el \<fecha\>»,
  «Programa la ronda completa (17 apps) para el \<fecha\>» o
  «Cancela la actualización programada de \<Nombre\> para el
  \<fecha\>» — y el asistente crea la programación; el día señalado
  construye la versión nueva y pide la contraseña del keystore para
  firmar y subir.
- **Uso de Muse**: sección con el % usado (barra de progreso), el %
  restante en grande, el plan, la fecha de restablecimiento del límite
  y cuándo se actualizó el dato (datos reales de `data/uso.json`,
  leídos en el build; tipo `UsoMuse` en `lib/types.ts`, validado en
  `lib/seed-data.ts`). Si se llega al 90% (`limite_90`), la sección
  avisa de que las tareas grandes están en pausa hasta el
  restablecimiento. Sin datos (o con datos inválidos), muestra «sin
  datos» sin romper el panel.
- Icono real de cada app junto a su nombre: extraídos de las APK firmadas
  locales con `scripts/extraer-iconos.py` (las 11 de AppCreator24 se leen
  del manifiesto con `aapt2 dump badging`; Mis Tarjetas se dibuja fiel
  desde su vector `ic_launcher.xml` porque no tiene PNG). Van embebidos
  en base64 en `lib/icons.ts` (generados desde `public/icons/`): el
  conector de GitHub rechaza archivos binarios en el push, así que los
  PNG no se suben como archivos al repo.
- Diseño móvil: la tabla se vuelve tarjetas (una por app) en pantallas
  estrechas, cabecera fija, botones de 44 px, aviso cuando no hay
  conexión y botón **Compartir** por app (usa el menú de compartir del
  teléfono y si no, copia el pedido).
- Dashboard con las 25 apps (11 de pago + 14 excepciones): nombre,
  paquete, versión en Apklis (con nº de versiones y fecha de última
  actualización), origen (AppCreator24 o nativa/WebView), estado, ojo
  abierto/cerrado, precio y notas.
- **Generabilidad por app**: insignia «Sí» / «Sí, subida bloqueada» /
  «No se puede generar», con la vía (AppCreator24, Nativa aapt2 o
  WebView aapt2) y el motivo del bloqueo. 21 generables, 4 con la
  subida bloqueada (Mis Tarjetas, Antivirus Actualización, Mi Línea y
  MultiHerramientas CU). El resto de fichas de la cuenta (53 en total)
  aún no está catalogado.
- Buscador por nombre o paquete; filtros por grupo, generable y
  estado; orden por nombre, precio, nº de versiones o última
  actualización. Enlaces directos a la ficha en tienda y en consola.
- Resumen con la cuenta Apklis (53 fichas, 166.3k descargas, 573
  versiones, 32 aceptadas, 0 en revisión — revisión 20 nov 11:00).
- Botón **Actualizar** por app (o **Crear ficha** si no tiene ficha) y
  **Ronda completa (17 apps)**: 11 de pago + 6 de la lista ampliada
  del 14 nov (Medicina Clínica, Tasa Cuba, TV Online, Gestor - Mi
  Negocio, Klick, Mi Licencia). Al pulsar, el trabajo se crea en la
  cola en estado `pendiente`. **El panel NO ejecuta la subida**: la
  ejecución la hace el agente (hoy no existe API pública de
  AppCreator24 ni de Apklis).
- **CORREGIDO (20 nov): en GitHub Pages los botones estaban
  deshabilitados** (pedían Turso). Ahora en modo estático copian el
  pedido completo —con versión y vía— para pegarlo en el chat.
- Historial de rondas y trabajos con estado por trabajo
  (pendiente / en curso / publicada / fallida / parcial).
- Alertas visibles: ojo cerrado, ficha sin publicar, registros
  huérfanos, subida bloqueada y versión de AppCreator24 distinta a
  la de Apklis.
- Interruptor de modo claro/oscuro, mobile-first, iconos SVG, sin
  emojis en la interfaz. Textos en español.

## Arquitectura

- **Next.js 14** con App Router y TypeScript (`npx tsc --noEmit`).
- **Datos en Turso** (base compartida de Klick / CubaOnline),
  tablas `misapks_apps` y `misapks_jobs` (ver `schema.sql`).
  Sin Turso configurado, el panel muestra los datos semilla
  (`data/apps.json`) y la cola de trabajos queda deshabilitada.
- Las credenciales de Turso se leen **solo** de las variables de
  entorno `TURSO_DATABASE_URL` y `TURSO_AUTH_TOKEN`. Nunca se
  escriben en archivos ni se imprimen.

```
mis-apks/
├── app/
│   ├── page.tsx            # Dashboard (componente Panel)
│   ├── layout.tsx          # Tema claro/oscuro sin parpadeo
│   ├── globals.css         # Estilos (variables claro/oscuro)
│   └── api/
│       ├── apps/route.ts   # GET /api/apps
│       └── jobs/
│           ├── route.ts    # GET /api/jobs · POST /api/jobs
│           └── [id]/route.ts  # GET /api/jobs/[id]
├── components/
│   ├── Panel.tsx           # Dashboard interactivo
│   └── InterruptorTema.tsx # Toggle claro/oscuro
├── lib/
│   ├── types.ts            # App, Job, Alerta
│   ├── icons.ts            # Iconos de las 25 apps en base64 (ver README)
│   ├── seed-data.ts        # Datos semilla + cálculo de alertas
│   └── turso.ts            # Cliente Turso y consultas
├── data/apps.json          # Semilla: 25 apps con su estado (20 nov 2026)
├── schema.sql              # Tablas misapks_apps / misapks_jobs
└── scripts/seed.mjs        # Crea tablas y carga la semilla en Turso
```

## API

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| GET | `/api/apps` | Lista de apps + alertas + fuente (`turso` o `semilla`) |
| GET | `/api/jobs` | Historial de trabajos (más recientes primero) |
| POST | `/api/jobs` | Crear trabajo: `{"tipo":"app","app":"<paquete>"}` o `{"tipo":"ronda"}`. Devuelve el trabajo en estado `pendiente` (201). Sin Turso configurado responde 503. |
| GET | `/api/jobs/[id]` | Un trabajo por su id |

## Puesta en marcha

```bash
npm install
npm run typecheck   # npx tsc --noEmit
npm run dev         # http://localhost:3000
```

Para activar la base de datos (una sola vez):

```bash
# En las variables de entorno de Vercel (o en local, sin guardarlas):
#   TURSO_DATABASE_URL=libsql://...
#   TURSO_AUTH_TOKEN=...
npm run db:seed     # crea las tablas y carga las 25 apps
```

No hay CLI de Turso autenticado en la VM: el seed usa
`@libsql/client` por HTTP con las credenciales del entorno.

## Despliegue

**EN VIVO en GitHub Pages: https://tecnobum.github.io/mis-apks/** — repo
público `TecnoBum/mis-apks`, rama `main`. **v1.2.0 (20 nov 2026, commit
12050095, VERIFICADO en vivo)**: iconos de las 25 apps, diseño móvil con
tarjetas, cabecera fija y Compartir. **v1.3.0 (20 nov 2026, construida,
SIN publicar)**: orden por defecto por fecha de actualización y
actualizaciones programadas; la salida está en `/tmp/mis-apks-pages/out`
para fusionarla con el catálogo completo de 53 apps y el área de
Ingresos y publicar una sola vez.
**APK (20 nov 2026)**: proyecto `~/workspace/mis-apks-apk/` (WebView del panel, paquete `com.tecnobum.misapks`, v1.0.0 vc1, COMPILADA y VERIFICADA, SIN FIRMAR — falta la contraseña del keystore de Johan). Para republicar tras un cambio,
reconstruir la versión estática y subirla:

```bash
./scripts/build-pages.sh   # deja el sitio en /tmp/mis-apks-pages/out
# copiar out/ a ~/workspace/mis-apks/pages-out/ y subir con el conector
# de GitHub (push-files) a la rama main del repo TecnoBum/mis-apks
```

Lecciones de Pages (13 nov 2026): la exportación estática necesita
`basePath: "/mis-apks"` en `next.config.mjs` (si no, los assets salen
como `/_next/...` y dan 404) y un archivo `.nojekyll` vacío en la raíz
del sitio (Jekyll ignora las carpetas con `_`, como `_next`).
**v1.1.0 (20 nov 2026)**: en modo estático los botones de actualizar
copian el pedido al portapapeles (antes estaban deshabilitados sin
Turso).

**Bloqueado:** el despliegue en Vercel (403, equipo tecnobum) sigue
pendiente de que Johan reautorice el acceso al equipo en la conexión
de Vercel. No se creó proyecto Vercel.

## Qué falta para la ejecución automática total

1. **AppCreator24 y Apklis no tienen API pública.** Hoy la ejecución
   es vía agente con el navegador (sesión de Johan).
2. **La contraseña del keystore se pide por ronda y nunca se
   almacena.** El agente la usa de forma transitoria (stdin → archivo
   temporal → borrado) para firmar con `jarsigner` (v1) + `apksigner`
   (v2/v3 con `--append-signature`).
3. **El agente actualiza el estado de cada trabajo** (`en_curso` →
   `publicada` / `fallida`) y las versiones en `misapks_apps` tras
   verificar en la consola de Apklis. Esos endpoints de actualización
   todavía no existen en esta v1.
4. **Alertas de Apklis** (ojo cerrado, versión desactualizada) hoy se
   calculan desde los datos guardados; una comprobación automática
   contra la consola requiere la sesión del navegador.

## Seguridad

- La contraseña del keystore **nunca** se guarda ni se escribe en
  archivos, logs o la base de datos.
- Las credenciales de Turso viven solo en variables de entorno.
- El panel no sube nada por sí mismo: solo lanza trabajos a la cola.
