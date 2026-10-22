# SportMatch — Informe de integración (15-oct-2026)

Objetivo de la sesión: actualizar todos los repos desde GitHub, analizar el proyecto, dejarlo
ejecutable y conectar los dos repos nuevos (**Ms-club** y **sportmatch-web**) para que funcionen
**a través del gateway**.

Se trabajó con un agente por repo (7 en total). Cada uno solo tocó su propia carpeta. La
integración final y las pruebas de punta a punta se hicieron al final, con el stack completo levantado.

**Ningún cambio está commiteado ni subido.** Revisar los diffs (sección 4) y commitear cada
repo por separado cuando el equipo lo apruebe.

---

## 1. Actualización desde GitHub

Todos los repos tenían el árbol limpio. Se hizo `git fetch` + `git merge --ff-only`, sin conflictos.

| Repo | Rama | Antes → Después | Novedades principales |
|---|---|---|---|
| Frontend-SportMatch-APP | main | a504498 → 22b0bc1 (+17) | mapa, planes/suscripción (visual), vista club (visual), recuperación, reportes, calificaciones, cancelar postulación, web |
| Ms_gateway | main | 168acca → f2b6699 (+6) | rutas de clubes, pagos y chats; calificaciones; editar/eliminar actividades |
| Ms_Users | main | 5bfaa9f → e9d7819 (+2) | reportes, bloqueos, silenciar chat, ubicación actual (migraciones 006–008) |
| Ms_Activities | main | 5d7c62a → 22059c9 (+6) | editar/cancelar actividades, calificaciones y reputación, historial, outbox de avisos (migraciones 004–006) |
| Ms_Matching | main | fab8472 → dbb7fae (+1) | lista de chats con no leídos, adjuntos, ubicación, WebSocket (migraciones 004–005) |
| Ms-club | master | 40923d1 (recién clonado) | clubes, convocatorias, postulaciones, comunicados, pagos Webpay (HU-30 a HU-40) |
| sportmatch-web | respaldo-sprint5 | b840ea8 (recién clonado) | panel web de clubes (antes 100 % con datos simulados) |

Los `stash` viejos de Ms_Users y Ms_gateway (imports rotos `CAPSTONE_PDP.proyecto…` del IDE)
**no se aplicaron**: no son trabajo real.

---

## 2. Arquitectura resultante

```
App móvil (Expo)  ─┐
                   ├──►  Ms_gateway :8000  ──►  ms_users      :8001  → BD sportmach_users
Web Next.js :3000 ─┘     (JWT, CORS,       ──►  ms_matching   :8002  → BD sportmach_matching
                          proxy, avisos)   ──►  ms_activities :8003  → BD sportmach_activities
                                           ──►  ms_club       :8004  → BD sportmach_clubs   (perfil "clubs")
```

- Los frontends **solo** hablan con el gateway.
- El JWT lo emite Ms_Users. Ms-club lo valida con el mismo secreto, issuer y audience: el `sub` es el `usuario.id` y es igual a `club.usuario_id`. El rol viene en el claim `roles`.
- El gateway reúne las notificaciones leyendo el `notification_outbox` de Activities, Matching y Clubs.
- Postgres 18 corre nativo en el Mac (`~/sportmach-db`, :5432). Cada servicio entra con su propio rol, nunca con `postgres`.
- Ojo con los nombres: en esta máquina las bases y roles se llaman `sportmach_*`, sin "t". En el código de Ms-club aparece `sportmatch_*`.

---

## 3. Qué hizo cada agente

### Ms-club (repo nuevo)
- Se creó `Ms-club/.env` (ignorado por git):
  - BD `sportmach_clubs` con el rol `sportmach_clubs`.
  - JWT igual al del gateway.
  - `WEBPAY_MOCK_MODE=true`, `WEBPAY_PUBLIC_URL=http://127.0.0.1:8000` y `WEBPAY_FRONTEND_RETURN_URL=http://localhost:3000/pago`.
- Se aplicaron como `postgres` las migraciones **001 a 006** y el seed `demo_planes.sql` (2 planes de prueba).
  - Los archivos del repo dan permisos (GRANT) al rol `sportmatch_clubs`, que aquí no existe. Se aplicaron desde una copia con el nombre cambiado a `sportmach_clubs`, **sin editar el repo**.
  - Antes se sacó un respaldo de la base: `/private/tmp/claude-501/msclub-migrations/sportmach_clubs_pre_migracion.dump`.
- Permisos mínimos para el rol de la app sobre las tablas nuevas y el esquema `clubs_api`.
- Tests: **389 passed**.
- Prueba local con uvicorn: health, clubes, deportes y planes respondieron 200.

### Ms_gateway
- **No faltaba ninguna ruta.** Todas las de Ms-club ya estaban publicadas y probadas en `tests/test_clubs_proxy.py`, incluidas las de Webpay simulado.
- Cambios en archivos versionados:
  - `compose.yaml`: el contexto de `ms_club` pasa de `../ms-club` a `../Ms-club`. En Linux, que distingue mayúsculas, fallaba. Se agregaron comentarios sobre el perfil.
  - `scripts/dev-lan.sh`: levanta con `--profile clubs`, fija `CLUBS_SERVICE_URL=http://ms_club:8004` y agrega a CORS los orígenes `:3000` de la web.
  - `.env.example`:
    - `CLUBS_SERVICE_URL` queda vacío, con una explicación: `127.0.0.1:8004` no sirve dentro de Docker.
    - Se agregan las variables `CLUBS_DB_*`, `WEBPAY_*` y los orígenes CORS `:3000`.
    - Se agrega `CLUBS_DB_PASSWORD=CHANGE_ME`, porque sin ella Compose fallaba incluso sin el perfil.
  - `README.md`: nueva sección "Levantar ms-club y el sitio web".
- `.env` local (no versionado): `COMPOSE_PROFILES=clubs`, `CLUBS_SERVICE_URL`, `CLUBS_DB_*`, `CLUBS_DATABASE_URL`, `WEBPAY_*` y CORS con `localhost:3000` y `127.0.0.1:3000`.
- Tests: **150 passed, 10 skipped**. Los saltados necesitan una base de pruebas aparte.

### sportmatch-web (repo nuevo)
Ahora usa el backend real a través del gateway, configurable con `NEXT_PUBLIC_API_URL` (por defecto `http://localhost:8000`).

- **Archivos nuevos:**
  - `lib/api.ts`: cliente HTTP con Bearer, maneja el 401 cerrando la sesión y muestra el `detail` del backend.
  - `lib/auth.ts`: login real.
  - `lib/clubes.ts`: tipos y llamadas a la API.
  - `components/MiClub.tsx`.
  - Páginas nuevas `/plan` y `/pago`.
  - `.env.example`.
- **Archivo eliminado:** `lib/auth.mock.ts`.
- **Modificados:** todas las páginas del panel, el login, `AuthGuard`, `SesionContext`, `ui.tsx` y `.gitignore`. El `.gitignore` tenía `.env*`, que también ignoraba `.env.example`; se agregó `!.env.example`.
- **Roles:** `club_admin` entra al panel de club y `admin` al de administración. `player` y `usuario` quedan fuera con un mensaje claro.
- **No hay refresh token:** al vencer el token (30 min) se vuelve al login con el aviso "Tu sesión expiró".
- **Mapa de vistas y endpoints:**

  | Vista | Endpoints |
  |---|---|
  | login | `POST /users/auth/login` |
  | inicio | `/clubs/me`, convocatorias, conteo de candidatos, avisos |
  | clubes y clubes/[id] | `/clubs`, `/clubs/deportes`, `/clubs/{id}` |
  | perfil | `GET /clubs/me` (si responde 404, muestra el formulario para crear), `POST` y `PATCH /clubs` |
  | convocatorias | crear y cerrar |
  | postulantes | listar y cambiar estado |
  | avisos | publicar y listar comunicados |
  | plan | planes, suscripción e inicio del pago |
  | pago | confirmar el pago (`commit`) y mostrar el comprobante |

- **Cambios de formulario que exigen los schemas reales:**
  - Logo e Instagram ahora son URL https, porque Ms-club no recibe archivos.
  - Las convocatorias piden dirección y requisitos (uno por línea).
  - Los avisos usan la audiencia `TODOS`, `POSTULANTES` o `MIEMBROS`.
- **Sigue simulado:** solo el número "Postulaciones" del inicio del admin, porque no hay endpoint. La etiqueta lo indica.
- `npm run lint` sin errores y `npm run build` compila las 14 rutas.

### Ms_Users (solo análisis, sin cambios)
- Tests: 197 passed, 17 skipped.
- **Migraciones 006–008:** solo crean tablas. Se aplicaron al levantar el stack (`users_migrate`). No hay tabla de control: todas son idempotentes y corren en cada arranque.
- **Roles:** cada cuenta tiene un solo rol (`usuario.rol_id`). El registro siempre asigna `player` y no existe endpoint para cambiar el rol. Para pruebas se cambia por SQL (sección 6).
- **Endpoints:** no existe refresh ni `GET /me`. El login devuelve `user.role` y el JWT trae `roles` y `sub`.

### Ms_Matching (solo análisis, sin cambios)
- Tests: 166 passed. Se corrieron contra un Postgres temporal, sin tocar la base real.
- Las migraciones 004–005 se ensayaron en una transacción con rollback. Se aplicaron al levantar el stack.
- El gateway ya expone chats, mensajes, lectura y el WebSocket `/api/v1/chats/ws`.
- **Falta la migración `003_notification_outbox.sql`** en el repo (ver pendientes).

### Ms_Activities (solo análisis, sin cambios)
- Tests: 74 passed.
- Las migraciones 004–006 se aplicaron sin error. El gateway lee su outbox y el rol tiene los permisos necesarios.
- El gateway expone las 16 rutas, incluido `/api/v1/ratings`.

### Frontend-SportMatch-APP (solo análisis, sin cambios de código)
- `services/auth.ts` y `matchStore.ts` usan el backend real.
- Todas las llamadas reales coinciden con el gateway en rutas, métodos y campos.
- `npx tsc --noEmit` sin errores, lint con 0 errores, y pasan los tests propios (27 + 15 + 11 + 4).
- **Clubes, planes y pago en la app siguen siendo solo visuales** (ver pendientes).

---

## 4. Archivos modificados (por commitear)

| Repo | Archivos |
|---|---|
| Ms_gateway | `compose.yaml`, `scripts/dev-lan.sh`, `.env.example`, `README.md` |
| sportmatch-web | ver la sección 3 (14 archivos modificados, 1 eliminado y 7 nuevos o carpetas nuevas) |
| Resto | sin cambios versionados |

No versionados (locales): `Ms_gateway/.env`, `Ms-club/.env`, `sportmatch-web/.env.local` y los `.venv/` de cada servicio Python.

---

## 5. Verificación de punta a punta (todo a través del gateway :8000)

Stack levantado con `docker compose up --build -d` (perfil `clubs` vía `.env`): **5 contenedores healthy**.

| # | Prueba | Resultado |
|---|---|---|
| 1 | `GET /health/ready` | `ready` |
| 2 | Rutas públicas `/clubs`, `/clubs/deportes`, `/clubs/announcements`, `/payments/plans` | 200 |
| 3 | CORS preflight desde `http://localhost:3000` | 200 + `access-control-allow-origin` |
| 4 | Registro → verificación → cuenta pasada a `club_admin` → login | rol `club_admin` en el token |
| 5 | Club crea su club (`POST /clubs`) | 201 |
| 6 | Club publica convocatoria | 201, `ABIERTA`, 5 cupos |
| 7 | Deportista (demo.camila) postula | `PENDIENTE`, con el perfil deportivo traído de Ms_Users |
| 8 | Club lista candidatos y **acepta** | 200 |
| 9 | Club publica comunicado | 201 |
| 10 | Deportista ve sus postulaciones y su bandeja de comunicados | 200 |
| 11 | **Notificaciones**: la deportista recibe "¡Te seleccionaron!" y "Bienvenida" | OK (llegan desde el outbox de clubes) |
| 12 | Pago Webpay simulado: initiate → mock → aprobar → return → **redirige a `localhost:3000/pago`** → commit | `APPROVED`; suscripción `ACTIVE` 30 días |
| 13 | Web Next.js en :3000: las 10 rutas (`/login`, `/inicio`, `/clubes`, `/perfil`, `/convocatorias`, `/postulantes`, `/avisos`, `/plan`, `/pago`…) | 200 |

No se hizo la prueba visual con clics en el navegador: la extensión de Chrome no estaba conectada.
La web usa exactamente los mismos endpoints validados arriba.

---

## 6. Cómo ejecutarlo

```bash
# 1. Docker Desktop abierto y Postgres local corriendo (pg_isready -h localhost)
cd ~/Desktop/soportmach/CAPSTONE_PDP/proyecto/Ms_gateway
docker compose up --build -d          # incluye ms_club por COMPOSE_PROFILES=clubs
#   o, para probar desde el teléfono:  sh scripts/dev-lan.sh
curl http://127.0.0.1:8000/health/ready

# 2. Web de clubes
cd ../sportmatch-web
cp .env.example .env.local            # ya existe en esta máquina
npm install && npm run dev            # http://localhost:3000

# 3. App móvil
cd ../Frontend-SportMatch-APP && npx expo start --lan
```

**Cuentas de prueba:**

| Uso | Correo | Contraseña |
|---|---|---|
| Club (panel web) | `demo.club@sportmach.example.com` | `DemoSport2026!` |
| Deportistas | `demo.camila@`, `demo.matias@`, `demo.valentina@sportmach.example.com` | `DemoSport2026!` |

- La cuenta de club es `club_admin`, ya tiene el club "Club Demo SportMatch", una convocatoria y el plan pagado.
- Los deportistas tienen el correo verificado y perfil completo.

Para convertir otra cuenta en club (y luego volver a iniciar sesión):
```sql
-- psql -h localhost -U postgres -d sportmach_users
UPDATE usuario SET rol_id = (SELECT id FROM rol WHERE nombre = 'club_admin'), fecha_actualizacion = now()
WHERE lower(email) = lower('correo@ejemplo.cl') AND is_active;
```

---

## 7. Normas respetadas

- Sin commits ni push. Cada agente tocó solo su repo.
- Ningún secreto en archivos versionados. Los `.env` y `.env.local` están ignorados y los `.env.example` no tienen valores reales.
- Se mantuvo la arquitectura `api → services → repositories`, con psycopg 3 sin ORM.
- Cada servicio usa su propio rol de BD. `postgres` solo se usó para migraciones y para el cambio de rol de prueba.
- Los frontends solo hablan con el gateway.
- No se rediseñó la UI. En la web solo se cambió lo que exigían los schemas reales.

---

## 8. Pendientes y riesgos (para el equipo)

1. **Ms_Matching: falta `db/migrations/003_notification_outbox.sql`.**
   - El gateway intenta leer `matching_api.notification_outbox` cada 2 s y registra `Notification ingestion failed for matching_api`.
   - No se cae, pero **no hay avisos de solicitudes, matches ni mensajes**, y `test_notifications.py` del gateway fallará con una base de pruebas.
   - Hay que pedírsela al dueño de ms-machine: está documentada en `Ms_gateway/docs/notificaciones.md` pero nunca se subió.
2. **App móvil: clubes, planes y pago no están conectados.**
   - Los planes están escritos en el código, la lista de clubes es fija y "postular" solo muestra un aviso.
   - No hay pantalla `app/pago.tsx` y el rol club se elige en un selector del login en vez de leer `club_admin` del token.
   - Es trabajo nuevo, no se tocó.
3. **Una sola URL de retorno de Webpay.** `WEBPAY_FRONTEND_RETURN_URL` apunta hoy a la web (`localhost:3000/pago`). Si la app también paga (`sportmatchapp://pago`), Ms-club tiene que aceptar un retorno por cliente; es una decisión de diseño.
4. **Nombre del rol en las migraciones de Ms-club.** Usan `sportmatch_clubs` y aquí el rol es `sportmach_clubs`. Cada migración nueva habrá que aplicarla cambiando el nombre. Lo ideal es que el repo reciba el rol como parámetro, pero eso le toca al dueño.
5. **Permisos de más en `sportmach_clubs`.** Es dueño de las tablas originales (`club`, `deporte`…), así que tiene todos los permisos sobre ellas, incluido DELETE. Para mínimo privilegio: `ALTER TABLE … OWNER TO postgres` y volver a correr la 001.
6. **Adjuntos de chat:** `CHAT_ATTACHMENT_HOSTS` está vacía, así que enviar imágenes o archivos responde 503. El texto y la ubicación funcionan.
7. **Web: avisos de Next 16.4 en desarrollo.** Aparece `Could not validate that a segment in your UI has instant navigation` en las páginas del panel. Viene de `cacheComponents`/`partialPrefetching` en `next.config.ts`, que ya traía el repo. No bloquea: todas las rutas responden 200 y el build pasa. No se pudo confirmar si ya aparecía antes de los cambios.
8. **Restos en la base `sportmach_activities`.** Hay un esquema `matching_api` vacío y tablas antiguas en `public`; parecen de un arranque antiguo con la base equivocada. No afectan, no se tocaron.
9. **App móvil:** la clave `GEOAPIFY_API_KEY` está escrita en `services/location.ts` y versionada. Conviene restringirla o moverla a `EXPO_PUBLIC_*`. Además, el texto "Modo simulado" de `forgot-password.tsx` quedó obsoleto.
10. **Limpieza:** borrar el respaldo `/private/tmp/claude-501/msclub-migrations/` cuando ya no haga falta, porque contiene datos de la base.
11. `compose.windows.yaml` del gateway no incluye `ms_club` y usa otros nombres de carpeta. Quedó fuera del alcance.

---

## 9. Segunda parte: app móvil conectada a clubes (15-oct-2026, tarde)

Se pidió levantar la versión móvil, poder crear un club propio y que el deportista vea los clubes y pueda postular desde la app.

### Cambios en Frontend-SportMatch-APP (sin commit)
| Archivo | Cambio |
|---|---|
| `services/clubes.ts` (nuevo) | Cliente de Ms-club vía gateway: `listarClubes` (`GET /clubs`), `getClub` (`GET /clubs/{id}`), `misPostulaciones` (`GET /clubs/announcements/applications/me`) y `postular` (`POST /clubs/announcements/{id}/applications`). |
| `app/(deportista)/clubs.tsx` | El directorio deja de ser un arreglo fijo. Lista los clubes reales, con buscador por nombre, deportes, comuna, insignia de verificado y estados de carga y error. Se recarga al volver a la pantalla. |
| `app/(deportista)/club/[id].tsx` | El perfil se carga por id desde el backend: descripción, historia, contacto y **convocatorias abiertas reales**. Cada convocatoria muestra deporte, categoría, fecha, lugar, cupos y requisitos. Se puede postular con un mensaje opcional. Si ya postulaste, muestra el estado (enviada, preseleccionado, aceptado o no seleccionado). Si no hay cupos, lo indica. Los errores del backend se muestran tal cual, por ejemplo "Ya postulaste a esta convocatoria." |

Se mantuvo el mismo diseño, con los colores del tema y los mismos estilos. `npx tsc --noEmit` y `expo lint` pasan con 0 errores, y el bundle de iOS compila.

### Flujo para probar
1. **Web** (http://localhost:3000): entra con `club.nuevo@sportmach.example.com` / `DemoSport2026!`, que es una cuenta `club_admin` **sin club**. En **Mi club** creas el club y en **Convocatorias** publicas una.
2. **App** (Expo Go): entra como deportista (`demo.matias@sportmach.example.com` / `DemoSport2026!`). Desde Inicio, abre "Clubes" y verás tu club. Al entrar ves la convocatoria y puedes postular.
3. **Web:** en **Postulantes** aparece el deportista; lo aceptas y en la app su estado cambia a "Aceptado". Además le llega una notificación.

### Cómo se levantó
- Gateway en modo red local: `sh Ms_gateway/scripts/dev-lan.sh`, que queda en `http://<IP-LAN>:8000`.
- App: `npx expo start --lan` en `Frontend-SportMatch-APP`.
- Web: `npm run dev` en `sportmatch-web`.

### Sigue pendiente en la app
- Planes y pago con Webpay en la app: requieren una pantalla de retorno y decidir la URL de retorno (punto 3 de la sección 8).
- El selector "club" del login de la app no lee el rol `club_admin` del token. Los clubes se administran desde la web.

---

## 10. Datos de demostración (`Ms_gateway/scripts/seed_demo.py`, sin commit)

Es un script nuevo que crea, **solo por la API del gateway**, datos ficticios para mostrar la app. Se puede volver a correr sin duplicar nada (lo comprobé corriéndolo dos veces).

La única excepción a "solo por la API" es el rol `club_admin` de las cuentas de club: se asigna por SQL con `psql` como `postgres`, porque Ms_Users no tiene endpoint para cambiar roles.

```bash
cd Ms_gateway && python3 scripts/seed_demo.py      # requiere el stack arriba con el perfil clubs
```

Contraseña de todas las cuentas: `DemoSport2026!`

**14 deportistas.** Cada uno tiene perfil, deportes con nivel, objetivos, disponibilidad y zona con coordenadas, así que las sugerencias calculan distancias reales:

| Cuenta (`@sportmach.example.com`) | Comuna | Deportes |
|---|---|---|
| demo.sofia | Maipú | running, ciclismo |
| demo.diego | Maipú | fútbol |
| demo.javiera | Lampa | ciclismo, trekking |
| demo.benjamin | Lampa | fútbol, running |
| demo.antonia | San Joaquín | tenis |
| demo.tomas | San Joaquín | fútbol, natación |
| demo.fernanda | Puente Alto | trekking, running |
| demo.vicente | La Florida | ciclismo, running |
| demo.isidora | Ñuñoa | yoga, running |
| demo.martin | Quilicura | fútbol, ciclismo |
| demo.catalina | Pudahuel | natación |
| demo.agustin | Peñalolén | trekking, ciclismo |
| demo.florencia | San Bernardo | running |
| demo.lucas | Colina | tenis, fútbol |

**12 actividades** en los próximos 1 a 6 días, en Maipú, Lampa, San Joaquín, Puente Alto, La Florida, Ñuñoa, Quilicura, Peñalolén y San Bernardo, con cupos. Varias ya tienen postulantes.

**5 clubes con 6 convocatorias.** Las cuentas de club entran al panel web y algunas convocatorias ya tienen postulantes:

| Cuenta | Club | Comuna | Convocatorias |
|---|---|---|---|
| demo.club.maipu | Club Deportivo Maipú Unido | Maipú | Senior de fútbol (2 postulantes) |
| demo.club.lampa | Lampa Bike Club | Lampa | Equipo de ruta 2027 (1) |
| demo.club.sanjoaquin | Tenis San Joaquín | San Joaquín | Interclubes juvenil y pádel mixto (1) |
| demo.club.florida | Running Club La Florida | La Florida | Grupo de competición 10K/21K (2) |
| demo.club.puentealto | Club Natación Puente Alto | Puente Alto | Equipo máster (1) |
