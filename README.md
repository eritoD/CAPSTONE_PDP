Elemento requerido en readme: 

Nombre del proyecto: #SportMatch# > Entrena, Conecta, Supera.
SportMatch es una aplicación móvil y plataforma web de matchmaking deportivo que conecta a deportistas amateur según deporte, nivel, horario y ubicación, e integra a clubes y comunidades deportivas locales.

Descripción: qué hace, a quién va dirigido, qué problema resuelve
SportMatch conecta a deportista amateur con compañeros de entrenamiento compatibles, permite crear y unirse a actividades deportivas grupales, y da visibilidad digital a clubes y comunidades deportivas mediante un directorio, convenio y gestión de membresías.
Va dirigido a:
- Deportistas amateur que buscan compañeros de entrenamientos compatibles en su zona.
- Clubes y comunidades deportivas locales que buscan captar y fidelizar socios.
- Adminitradores de la plataforma y app, encargados de la operación y moderación general.

Buscamos resolver el abandono deportista. Luego de estudiar el mercado pudimos verificar que 1 de cada 2 personas abandona una rutina deportiva antes de los 3 meses por falta de acompañamiento y motivación. Las redes sociables genéricas y las apps de seguimientos deportivo individual (Strava, Nike Run Club) no permiten filtrar por deporte, nivel, horario y ubicación al mismo tiempo, y los clubes locales careces de una vitrina digital propia. SportMatch resuelve ambos problemas en una sola plataforma: matchmaking deportivo + gestión de comunidades/clubes.
  
Tecnologías utilizadas (lenguajes, frameworks, base de datos, cloud)
- Aplicación móvil: React Native + Expo (Android).
- Backend: FastAPI(Python), arquitectura de microservicios por dominio.
- Base de Datos: PstgreSQL + PostGIS
- Autenticación: JWT + RBAC (control de acceso por rol)
- Pagos: Webpay Plus (Transbank), ambiente de integración.
- Contenedores: Docker + Docker Compose.
- Control de versiones: Git / GitHub.

Instrucciones para ejecutar el proyecto localmente:
Requisitos Previos:
- Docker y Docker Compose
- Node.js (LTS) y npm
- Python 3.11 +
- Expo CLI (`npm install -g expo-cli`)
- Cuenta de integración Webpay Plus (credenciales de pruebas).

Backend (microservicios + base de datos:


Integrantes del equipo con sus roles:

- Erick Díaz: Project Maneger, base de datos, QA e integraciones externas.
- Jocelyn Pérez: Desarrollo Frontend / Mobile - Aplicación React Native + Expo, UX/UI.
- Benjamín Pezoa: Desarrollo Backend - microservicios (FastAPI) y algoritmo de matching.

Metodología de trabajo del equipo (Scrum, Kanban, DevOps, etc.)

El proyecto lo desarrollamos bajo Scrumban (marco ágil híbrido Scrum + Kanban) organizado en 6 ciclos de trabajo distribuidos en 18 semanas, con un tablero Kanban para el seguimiento diario de tareas y revisiones/retrospectivas al cierre de cada ciclo.
El alcance funcional está organizado en épicas de producto (EP-01 a EP-09) y habilitadores técnicos (EN-01 a EN-03), documentadps en el catálogo de épicas y en el Product Backlog priorizado del proyecto.

Arquitectura de la solución (descripción o diagrama)
