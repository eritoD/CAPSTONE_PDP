# SportMatch

**Entrena, Conecta, Comparte.**

SportMatch es una aplicación móvil y una plataforma web de matchmaking deportivo que conecta a deportistas amateur según deporte, nivel, horario y ubicación, e integra a clubes y comunidades deportivas locales.

---

##  Descripción

SportMatch conecta a deportistas amateur con compañeros de entrenamiento compatibles, permite crear y postular a actividades deportivas grupales, y da visibilidad digital a clubes y comunidades deportivas mediante un directorio, convocatorias y gestión de postulantes.

**¿A quién va dirigido?**

- **Deportistas amateur** que buscan compañeros de entrenamiento compatibles en su zona.
- **Clubes y comunidades deportivas locales** que buscan captar deportistas mediante convocatorias.
- **Administradores de la plataforma**, encargados de la operación y moderación general.

**¿Qué problema resuelve?**

El abandono de la actividad física es un problema recurrente entre deportistas amateur, generalmente asociado a la falta de acompañamiento y motivación al entrenar solo. Las redes sociales genéricas y las apps de seguimiento deportivo individual (Strava, Nike Run Club) no permiten filtrar por deporte, nivel, horario y ubicación al mismo tiempo, y los clubes locales carecen de una vitrina digital propia para publicar sus necesidades y revisar postulantes.

SportMatch resuelve ambos problemas en una sola plataforma: **matchmaking deportivo** + **convocatorias y gestión de clubes**, con chat para coordinar la participación una vez confirmada.

---

##  Tecnologías utilizadas

- **Aplicación móvil:** React Native + Expo (Android)
- **Aplicación web:** panel para representantes de club y administración
- **Backend:** FastAPI (Python), arquitectura de Gateway + microservicios por dominio
- **Base de datos:** PostgreSQL + PostGIS (consultas geoespaciales)
- **Autenticación:** JWT + RBAC (control de acceso por rol)
- **Pagos:** Webpay Plus (Transbank), ambiente de integración
- **Mapas y geolocalización:** Geoapify
- **Contenedores:** Docker + Docker Compose
- **Control de versiones:** Git / GitHub
- **Gestión ágil:** Jira

---

##  Instrucciones para ejecutar el proyecto localmente

### Requisitos previos

- Docker y Docker Compose
- Node.js (LTS) y npm
- Python 3.11+
- Expo CLI (`npm install -g expo-cli`)
- Cuenta de integración Webpay Plus (credenciales de prueba de Transbank)

### 1. Clonar los repositorios

```bash
git clone https://github.com/eritoD/CAPSTONE_PDP.git
cd CAPSTONE_PDP
```

> Backend, frontend móvil y la carpeta de infraestructura/documentación viven como carpetas hermanas dentro de este repositorio (o como submódulos, según la estructura que ya tengan). Ajusten esta sección a la organización real de sus carpetas.

### 2. Levantar el backend (Gateway + microservicios + base de datos)

```bash
cd sportmatch-backend
cp .env.example .env        # completar variables (DB, JWT_SECRET, credenciales Webpay)
docker compose up --build
```

Esto levanta:
- El **API Gateway** (puerto `8000`)
- El microservicio de usuarios (**Ms_Users**)
- La base de datos **PostgreSQL + PostGIS**

Verificar que el Gateway responde en `http://localhost:8000/api/v1/health` (o el endpoint de salud equivalente).

### 3. Configurar y ejecutar la aplicación móvil

```bash
cd Frontend-SportMatch-APP
npm install
```

Crear un archivo `.env.local` en la raíz del proyecto móvil con la URL del backend:

```
EXPO_PUBLIC_API_URL=http://TU_IP_LOCAL:8000/api/v1
```

> En un celular físico, `localhost` apunta al propio teléfono, no al computador. Usar la IP local de la máquina donde corre el backend (`ipconfig getifaddr en0` en Mac), y asegurarse de que ambos dispositivos estén en la misma red Wi-Fi, o exponer el backend con un túnel (ngrok) si están en redes distintas.

```bash
npx expo start -c
```

Escanear el código QR con la app Expo Go en un dispositivo Android.

### 4. (Opcional) Ejecutar el panel web

```bash
cd sportmatch-web
npm install
npm run dev
```

---

##  Integrantes del equipo y roles

| Integrante | Rol en Scrum | Área principal | Qué hace |
|---|---|---|---|
| **Erick Díaz** | Scrum Master y líder de proyecto | Gateway y coordinación | Ordena el backlog, dirige las ceremonias Scrum, desarrolla el Gateway, coordina integraciones y revisa pull requests. |
| **Benjamín Pezoa** | Equipo de desarrollo | Backend y lógica | Desarrolla los microservicios en FastAPI, el modelo de datos, el algoritmo de matching, y las integraciones con Webpay Plus y Geoapify. |
| **Jocelyn Pérez** | Equipo de desarrollo | Frontend y documentación | Desarrolla la app móvil en React Native/Expo y el panel web, diseña UX/UI, registra evidencias de prueba y mantiene la documentación del proyecto. |


---

##  Metodología de trabajo

El proyecto se desarrolla bajo **Scrum**, con un equipo de 3 integrantes y **sprints de 2 semanas**, gestionados en **Jira**.

El alcance funcional está organizado en:
- **10 épicas de producto** (EP-01 a EP-10): cuentas y acceso, perfil deportivo, búsqueda por ubicación, matching, actividades, convocatorias de clubes, notificaciones, administración, chat, y planes/suscripciones.
- **3 épicas técnicas / habilitadores** (EN-01 a EN-03): infraestructura de backend, infraestructura de clientes, y calidad/seguridad/automatización.
- **57 historias de usuario**, documentadas en el `User Story Mapping` y priorizadas con **MoSCoW** en el `Product Backlog`.
- **7 sprints** planificados en el `Sprint Backlog`, con Sprint Planning, Daily Scrum, Sprint Review y Retrospectiva al cierre de cada ciclo.

La documentación completa de alcance y planificación vive en la carpeta `/docs` de este repositorio: Acta de Constitución, Análisis del Caso, Épicas, Historias de Usuario, Product Backlog, Sprint Backlog y las retrospectivas de cada sprint.

---

##  Arquitectura de la solución

SportMatch sigue una arquitectura de **API Gateway + microservicios**, con un dominio de datos independiente por servicio:

```
┌─────────────────┐     ┌──────────────────┐
│  App móvil        │     │  Panel web         │
│  (React Native)   │     │  (clubes / admin)  │
└────────┬──────────┘     └────────┬───────────┘
         │ HTTPS / JSON            │
         └───────────┬─────────────┘
                      ▼
              ┌───────────────┐
              │  API Gateway   │   ← punto único de entrada, valida JWT y rol
              │   (FastAPI)    │
              └───────┬────────┘
                      │ HTTP interno
      ┌───────┬───────┼───────┬────────┬─────────┐
      ▼       ▼       ▼       ▼        ▼         ▼
  Usuarios Matching Activ.  Clubes   Chat    Planes/Pagos
  (Ms_Users) (plan.) (plan.) (plan.) (plan.)  (Webpay Plus)
      │       │       │       │        │         │
      └───────┴───────┴───────┴────────┴─────────┘
                      ▼
            PostgreSQL + PostGIS
         (separación lógica por dominio)
```

- **Autenticación y autorización:** JWT emitido por Ms_Users, validado en el Gateway; control de acceso por rol (deportista, representante de club, administrador).
- **Geolocalización:** Geoapify para geocodificación y cálculo de distancia aproximada; nunca se expone la coordenada exacta de otro usuario.
- **Pagos:** Webpay Plus (Transbank) en ambiente de integración; SportMatch nunca almacena datos de tarjeta, solo el resultado de la transacción.
- **Estado actual:** el microservicio de usuarios (Ms_Users) ya está implementado e integrado; Matching, Actividades, Clubes, Chat y Planes/Pagos se incorporan progresivamente entre el Sprint 3 y el Sprint 7 (ver `Sprint Backlog` en `/docs`).


---

##  Estructura del repositorio

```
CAPSTONE_PDP/
├── sportmatch-backend/        # Gateway, microservicios, Docker Compose
├── Frontend-SportMatch-APP/   # Aplicación móvil (React Native + Expo)
├── sportmatch-web/            # Panel web (clubes y administración)
└── docs/                      # Acta, épicas, historias, backlog, retrospectivas
```
