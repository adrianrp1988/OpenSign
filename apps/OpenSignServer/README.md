# OpenSign Server — Resumen para desarrolladores

Este documento describe a alto nivel la estructura, funcionamiento y los pasos necesarios para que un desarrollador se incorpore al equipo de desarrollo del servicio OpenSignServer (carpeta apps/OpenSignServer). Está pensado como guía de onboarding: arquitectura, dependencias, variables de entorno, cómo ejecutar localmente, pruebas y puntos clave del código.

---

## 1. Visión general del proyecto
OpenSignServer es el backend del proyecto OpenSign, implementado sobre Parse Server y Express. Provee:
- API REST y real-time (Parse Server).
- Gestión de archivos (S3 / DigitalOcean Spaces o almacenamiento local).
- Envío de correos (SMTP o Mailgun).
- Funciones cloud personalizadas (carpeta `cloud`).
- Migraciones de base de datos (`migrationdb` + parse-dbtool).
- Conversión y firma de PDF (utilidades en `Utils.js` / libs de pdf).

El punto de entrada es `index.js` que configura ParseServer, middlewares de Express y arranque del servidor.

---

## 2. Estructura de carpetas y archivos clave
Rama: `staging` — carpeta objetivo: `apps/OpenSignServer`.

Archivos y carpetas principales que debes conocer:
- `index.js` — punto de entrada. Inicializa dotenv, Express, ParseServer, adaptadores de archivos y correo, monta rutas y lanza migraciones.
- `package.json` — scripts, dependencias y engines Node (18 || 20 || 22).
- `Utils.js` — utilidades comunes: manejo de PDFs, URLs firmadas, validaciones, helpers de mail y formateo de fechas.
- `cloud/` — funciones y lógica cloud (ejecutadas por Parse Server). `index.js` de `cloud` se importa en runtime.
- `auth/` — adaptadores de autenticación (ej. SSO).
- `migrationdb/` — scripts/recursos para migraciones.
- `public/` — activos estáticos servidos por Express.
- `files/` — (si se usa local) carpeta para almacenamiento de archivos.
- `spec/` — tests y especificaciones.
- `utils/` — utilidades compartidas (ej. `fileUtils.js`).
- Archivos de despliegue y config: `Dockerhubfile`, `app.yaml`, `openshift.json`, `scalingo.json`, `.ebextensions`, etc.
- `CODE_OF_CONDUCT.md` — conducta del proyecto.

Puedes ver toda la carpeta en:
https://github.com/adrianrp1988/OpenSign/tree/staging/apps/OpenSignServer

---

## 3. Arquitectura y flujo principal
- Express es el framework HTTP; Parse Server se monta (por defecto) en el mount path `process.env.PARSE_MOUNT || '/app'`.
- `index.js`:
  - Configura el adaptador de archivos: `@parse/s3-files-adapter` para almacenamiento en S3/Spaces o `@parse/fs-files-adapter` para local según `USE_LOCAL`.
  - Configura el adaptador de correo: SMTP (nodemailer) o Mailgun (mailgun.js) si están habilitados.
  - Protege accesos a archivos locales comprobando URLs firmadas (`validateSignedLocalUrl`).
  - Lanza migraciones: `parse-dbtool migrate` (se construye comando y se ejecuta con `exec`).
  - Monta rutas personalizadas (`./cloud/customRoute/customApp.js`).
- Parse Server maneja clases/objetos de la app (usuarios, documentos, plantillas, eventos, etc.). Muchas operaciones del dominio están implementadas en las Cloud Functions dentro de `cloud/`.

---

## 4. Principales dependencias
(En `package.json` — lista resumida)
- parse-server, parse, parse-dbtool
- express, cors
- @parse/s3-files-adapter, @parse/fs-files-adapter
- aws-sdk / @aws-sdk/* (para S3), multer, multer-s3
- nodemailer, mailgun.js
- pdf-lib, @pdf-lib/fontkit, signpdf libs (@signpdf/*)
- sharp, libreoffice-convert (para conversiones/preview)
- mongodb (driver), mongoose no está incluido; se usa Parse + mongodb
- herramientas dev: eslint, prettier, jasmine, mongodb-runner, nyc

---

## 5. Variables de entorno importantes
(define un `.env` local con estas variables según tu entorno)

Almacenamiento:
- USE_LOCAL — si `'true'` usa FSFilesAdapter (local), si `'false'` usa S3 adapter.
- DO_ENDPOINT — endpoint (ej. spaces) o URL completa.
- DO_SPACE — bucket/space.
- DO_BASEURL — base URL pública si aplica.
- DO_REGION, DO_ACCESS_KEY_ID, DO_SECRET_ACCESS_KEY

Base de datos y servidor:
- DATABASE_URI o MONGODB_URI — URI MongoDB (ej. `mongodb://localhost:27017/dev`)
- MASTER_KEY — master key de Parse (secreto)
- APP_ID o APP_ID en env (`process.env.APP_ID`) — identificador de la app
- SERVER_URL — URL pública del servidor (ej. `https://mi-dominio.com`)

Correo:
- SMTP_ENABLE — 'true'/'false'
- SMTP_HOST, SMTP_PORT, SMTP_USERNAME, SMTP_PASS, SMTP_USER_EMAIL
- MAILGUN_API_KEY, MAILGUN_DOMAIN, MAILGUN_SENDER

Otros:
- PARSE_MOUNT — ruta donde montar Parse (default `/app`)
- GOOGLE_CLIENT_ID — para login Google
- Any other env referenced: (ver `index.js` y `Utils.js` para más variables)

Seguridad: Mantén MASTER_KEY y credenciales S3/SMTP fuera del repositorio y en almacenamiento seguro.

---

## 6. Ejecutar localmente (onboarding rápido)
1. Clona el repo y sitúate en la carpeta:
   - git clone ... && cd OpenSign/apps/OpenSignServer
2. Instala dependencias:
   - npm ci
3. Prepara `.env` con las variables mínimas (MONGODB_URI o usar mongodb local, MASTER_KEY, APP_ID, SERVER_URL). Para pruebas locales simplifica `USE_LOCAL=true`.
4. Ejecuta MongoDB (o usa la integración `mongodb-runner` para tests).
5. Ejecuta:
   - npm start    # arranca server (index.js)
   - npm run watch  # con nodemon para desarrollo
6. Para ejecutar tests:
   - npm test  (usa `mongodb-runner` y `jasmine`)
7. Lint y formateo:
   - npm run lint
   - npm run prettier
8. Observa logs en consola. En arranque `index.js` ejecuta migraciones automáticas (parse-dbtool).

Notas:
- Si usas `USE_LOCAL=false`, asegúrate de tener credenciales S3/Spaces válidas.
- El servidor publica Parse en `SERVER_URL` y monta la API en `PARSE_MOUNT` (por defecto `/app`).

---

## 7. Migraciones de base de datos
- En el arranque `index.js` construye y ejecuta `npx parse-dbtool migrate` usando `MASTER_KEY`, `APPLICATION_ID` (APP_ID) y `SERVER_URL`.
- Revisa `migrationdb/` para scripts específicos.
- Antes de ejecutar migraciones en producción, haz backup de la base de datos.

---

## 8. Almacenamiento y URLs firmadas
- Si `USE_LOCAL === 'true'` usa `@parse/fs-files-adapter` (almacenamiento en `files/`).
- Si se usa remoto, `@parse/s3-files-adapter` con opciones para DigitalOcean Spaces / S3 (presigned URLs habilitadas).
- `index.js` y `Utils.js` contienen lógica para validar y generar URLs firmadas (`getSignedLocalUrl`, `getPresignedUrl`, `validateSignedLocalUrl`).

---

## 9. Correo electrónico
- El proyecto puede enviar correos por SMTP (nodemailer) o Mailgun (mailgun.js).
- Los templates de correo se configuran en `files/` y el adaptador `parse-server-api-mail-adapter`.
- `isMailAdapter` se determina en `index.js` según las credenciales disponibles.

---

## 10. PDF, firma y utilidades
- `Utils.js` implementa:
  - flattenPdf: elimina widgets de formulario y aplana PDFs (pdf-lib).
  - funciones de sanitización, generación de IDs, formateo de fechas, generación de plantillas email.
  - integración con librerías `@signpdf/*` y `pdf-lib` para firma.
- Revisa `cloud/` y `utils/fileUtils.js` para el flujo completo de subida, presigned URLs y firma.

---

## 11. Tests y cobertura
- Tests con Jasmine. `npm test` arranca `mongodb-runner` y ejecuta pruebas.
- Cobertura: `npm run coverage` usa nyc.
- Mantén pruebas ejecutables en local para validar PRs.

---

## 12. Estilo de código y linters
- ESLint y Prettier configurados. Usa:
  - npm run lint
  - npm run prettier
- `.eslintrc.json` y `.prettierrc` están en la carpeta.

---

## 13. Despliegue
- Hay múltiples artefactos para despliegue: `Dockerhubfile`, `openshift.json`, `app.yaml`, `scalingo.json`, `.ebextensions`.
- Revisa cada archivo según la plataforma objetivo (K8s/OpenShift/Heroku/Elastic Beanstalk/Scalingo).

---

## 14. Seguridad y buenas prácticas
- Nunca subir MASTER_KEY, credenciales S3 o SMTP al repo.
- Validar entradas en cloud functions y rutas custom.
- Revisar y limitar permisos de archivos/URLs firmadas.
- Mantener dependencias actualizadas y auditar vulnerabilidades.

---

## 15. Onboarding sugerido (primeras tareas)
1. Clona el repo y levanta la app con `USE_LOCAL=true`.
2. Corre la suite de tests y arregla cualquier test que falle en tu entorno.
3. Lee `cloud/` para entender las Cloud Functions y revisa `customRoute` para endpoints custom.
4. Revisa `migrationdb/` y prueba una migración en entorno local.
5. Haz un cambio pequeño (ej.: mejora docstring, test o linter) y abre un PR para familiarizarte con el flujo.

---

## 16. Contribuciones y flujo de trabajo
- Sigue el `CODE_OF_CONDUCT.md`.
- Flujo recomendado:
  - Fork -> rama feature/topic (`feature/<descripcion>` o `fix/<issue-id>`) -> push -> PR hacia `staging` (o la rama indicada por el equipo).
  - Ejecuta tests y linters antes de solicitar revisión.
  - Mensajes de commit claros y atómicos.
- Añade reviewers y referencias a issues relacionados.

---

## 17. Dónde mirar primero (archivos a revisar al empezar)
- `index.js` — arranque y configuración global
- `Utils.js` — utilidades y lógica de PDF/correo
- `cloud/` — funciones del negocio
- `auth/` — adaptadores SSO/google
- `migrationdb/` — migraciones
- `files/`, `public/`, `spec/`

---

## 18. Recursos y contacto
- Repo (carpeta): https://github.com/adrianrp1988/OpenSign/tree/staging/apps/OpenSignServer
- Si quieres, puedo:
  - Añadir este README al repo y abrir un PR.
  - Generar un checklist de tareas de onboarding para nuevos devs.
  - Extraer y listar todas las variables de entorno usadas automáticamente.

---

## 19. Notas finales
He revisado el contenido visible en `apps/OpenSignServer` (package.json, index.js, Utils.js y la estructura de carpetas). Si necesitas, genero un README más corto o más técnico (diagramas de flujo, secuencia de llamadas, o un checklist de seguridad). También puedo crear/commitear el README en una rama y abrir PR hacia `staging`.