# 04 — Arquitectura del Sistema

> Visión transversal del sistema Valorizador CEN: stack consolidado, estructura del repositorio, cómo correr el prototipo en local, dependencias entre módulos y decisiones que se postergan para etapas posteriores de escalamiento.
>
> Este documento es el punto de entrada para un desarrollador nuevo que se incorpora al proyecto. Apóyate en `02-backend.md` y `03-frontend.md` para los detalles de cada componente.

---

## 1. Resumen ejecutivo

El Valorizador CEN es una **aplicación web local** para el prototipo, con dos procesos independientes que corren en la misma máquina del desarrollador o usuario validador:

- **Backend**: API HTTP en Python (FastAPI) con SQLite local como persistencia.
- **Frontend**: SPA en React + TypeScript servida por Vite.

El sistema está diseñado para validar el dominio y la experiencia de modelado con usuarios reales del CEN, no para producción. Las decisiones operacionales (despliegue, auth, integraciones) están explícitamente postergadas (ver §6) y la arquitectura se construye para que esas decisiones puedan tomarse después sin reescribir el núcleo.

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│  Browser                    │         │  Filesystem local           │
│  (Chrome / Edge / Firefox)  │         │                             │
│                             │         │  data/                      │
│  React + Vite               │ ◀────── │   ├── valorizador.db        │
│  → http://localhost:5173    │         │   └── uploads/              │
└──────────────┬──────────────┘         │       └── catalogo_*.xlsx   │
               │                        └──────────▲──────────────────┘
               │ HTTP                              │
               │ /api/v1                           │
               ▼                                   │
┌─────────────────────────────┐                    │
│  FastAPI + Uvicorn          │ ──────────────────┘
│  → http://localhost:8000    │   SQLAlchemy / openpyxl
│                             │
│  - Motor de cálculo         │
│  - Ingesta Excel            │
│  - Generador Excel/PDF      │
└─────────────────────────────┘
```

Sin Docker, sin nginx, sin TLS, sin auth en el prototipo.

---

## 2. Stack consolidado

### 2.1 Backend (Python 3.12)

| Categoría | Tecnología | Razón |
|---|---|---|
| Framework HTTP | **FastAPI** 0.110+ | Performance, OpenAPI auto, validación con Pydantic |
| Modelado de dominio | **Pydantic v2** | Valida y serializa; mismos modelos del motor de cálculo |
| ORM | **SQLAlchemy 2** | Maduro, performante, soporta SQLite y PostgreSQL sin cambios |
| Migraciones | **Alembic** | Estándar con SQLAlchemy |
| BD | **SQLite** | Cero configuración, archivo único, suficiente para prototipo |
| Excel | **openpyxl** | Lectura y escritura `.xlsx` puras Python |
| PDF | **WeasyPrint** | Genera PDF desde HTML+CSS (más flexible que ReportLab) |
| Tests | **pytest** + **hypothesis** | Tests deterministas + property-based |
| Server | **uvicorn** | ASGI, integra con FastAPI |
| Dependencias | **uv** (preferido) o **poetry** | Resolución rápida, lockfile |

### 2.2 Frontend (Node 20+)

| Categoría | Tecnología | Razón |
|---|---|---|
| Framework | **React 18** | Estándar de facto |
| Lenguaje | **TypeScript 5** | Type-safety en un dominio complejo |
| Build | **Vite** | Dev server rápido, HMR, build optimizado |
| UI primitives | **shadcn/ui** + **Tailwind CSS** | Componentes accesibles, theming consistente |
| Canvas / grafo | **React Flow** v11+ | Drag-and-drop, nodos custom, edges, viewport |
| Estado servidor | **TanStack Query** v5 | Cache, refetch, mutations, optimistic updates |
| Estado UI | **Zustand** | Liviano, sin boilerplate, suficiente para el editor |
| Formularios | **React Hook Form** + **Zod** | Validación tipada en el cliente |
| Grillas | **TanStack Table** | Listado de proyectos, partidas, factores |
| Routing | **React Router** v6 | Navegación entre pantallas |
| HTTP | **axios** | Cliente HTTP con interceptors |
| Package manager | **pnpm** | Rápido, eficiente en disco |

### 2.3 Versionado y editor

- Git con **branch principal `main`**. Branches feature en formato `feat/<nombre>`.
- Convención de commits: **Conventional Commits** (`feat:`, `fix:`, `docs:`, `chore:`, …).
- Editor recomendado: VS Code con extensiones de Python, Pylance, ESLint, Prettier, Tailwind.

---

## 3. Estructura del repositorio

```
cen-valorizador/
├── README.md                  # cómo correr en local
├── .gitignore
├── .editorconfig
├── docs/
│   ├── 01-contexto-y-dominio.md
│   ├── 02-backend.md
│   ├── 03-frontend.md
│   ├── 04-arquitectura-sistema.md
│   └── 05-template-excel-catalogo.md
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── alembic/versions/
│   ├── api/
│   ├── valorizador/
│   │   ├── dominio/
│   │   ├── motor/
│   │   ├── catalogo/
│   │   ├── persistencia/
│   │   ├── servicios/
│   │   └── exports/
│   └── tests/
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── canvas/
│       ├── formularios/
│       ├── panel-estimacion/
│       ├── proyecto/
│       ├── catalogo/
│       ├── components/ui/
│       ├── hooks/
│       ├── stores/
│       └── types/
├── data/                      # generado en runtime; NO commitear
│   ├── valorizador.db
│   └── uploads/
└── scripts/
    ├── dev.sh                 # levanta backend + frontend en paralelo
    ├── bootstrap.sh           # crea BD + carga catálogo de ejemplo
    └── test.sh                # ejecuta toda la suite
```

`data/` está en `.gitignore`. Los archivos Excel del consultor se almacenan ahí en runtime.

---

## 4. Dependencias entre módulos

```
┌─────────────────────────────────────────────────────────┐
│                  Frontend (React)                       │
│  - canvas, formularios, panel-estimacion                │
│  - depende de Backend vía HTTP /api/v1                  │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP/JSON
┌────────────────────────▼────────────────────────────────┐
│                  Backend / api/                         │
│  - routers (FastAPI)                                    │
│  - schemas HTTP                                         │
└─────┬──────────────────────────────────────────┬────────┘
      │                                          │
      ▼                                          ▼
┌──────────────┐                          ┌────────────────┐
│ servicios/   │ ◀────── orquesta ──────▶ │ persistencia/  │
│              │                          │  (SQLAlchemy)  │
└─────┬────────┘                          └───────┬────────┘
      │                                           │
      ▼                                           ▼
┌──────────────────┐                       ┌────────────┐
│ dominio/, motor/ │                       │ SQLite     │
│  (puros)         │                       │ data/*.db  │
└─────────┬────────┘                       └────────────┘
          │
          ▼
┌──────────────────┐
│ catalogo/        │
│  (ingesta Excel) │
└──────────────────┘
```

**Reglas de dependencia (importadas con flecha hacia abajo, nunca al revés):**

- `dominio/` y `motor/` **no importan** de `persistencia/`, `api/`, ni `servicios/`. Solo dependen de Pydantic y la stdlib.
- `catalogo/` (ingesta) depende de `dominio/` y `openpyxl`, no de `persistencia/`.
- `servicios/` orquesta `motor/`, `catalogo/`, `persistencia/`, `exports/`.
- `api/` depende de `servicios/` y `dominio/` (para schemas), no llama directo a `persistencia/`.
- `exports/` depende de `dominio/` y `motor/` (toma resultados ya calculados).

Esta separación es la garantía de que el motor de cálculo es testeable aislado, reutilizable desde notebooks/CLI, y sobrevive cambios futuros de stack web/persistencia.

---

## 5. Cómo correr el sistema en local

### 5.1 Pre-requisitos

- **Python 3.12+**
- **Node 20+** y **pnpm 9+**
- **uv** (Python deps) o Poetry, según preferencia del equipo
- Sistema operativo: Windows 10+, macOS 13+ o Linux. Validado principalmente en Windows (entorno del usuario).

### 5.2 Primera vez (bootstrap)

```bash
# 1. Clonar el repo
git clone <repo-url>
cd cen-valorizador

# 2. Backend: instalar deps y crear BD
cd backend
uv sync                                    # o: poetry install
uv run alembic upgrade head                # crea schema en data/valorizador.db
uv run python -m valorizador bootstrap     # carga catálogo de ejemplo

# 3. Frontend: instalar deps
cd ../frontend
pnpm install
```

### 5.3 Día a día

Dos terminales o un script que lance ambos en paralelo (`scripts/dev.sh`):

```bash
# Terminal 1: backend
cd backend
uv run uvicorn api.main:app --reload --port 8000

# Terminal 2: frontend
cd frontend
pnpm dev                                   # vite, puerto 5173
```

Abrir `http://localhost:5173` en el navegador. El frontend habla con `http://localhost:8000/api/v1` (configurable en `frontend/.env.local`).

### 5.4 Tests

```bash
# Backend
cd backend
uv run pytest                              # toda la suite
uv run pytest tests/motor/                 # solo motor de cálculo
uv run pytest -k "intereses_intercalarios" # filtrado

# Frontend (cuando exista la suite)
cd frontend
pnpm test
pnpm typecheck                             # tsc --noEmit
```

### 5.5 Linters y formato

```bash
# Backend
uv run ruff check backend/
uv run ruff format backend/

# Frontend
pnpm lint                                  # eslint
pnpm format                                # prettier
```

### 5.6 Reset de BD

Cuando se necesita partir de cero:

```bash
rm backend/data/valorizador.db
cd backend
uv run alembic upgrade head
uv run python -m valorizador bootstrap
```

### 5.7 Importar catálogo del consultor

Vía UI (rol curador en `/catalogo`) o CLI:

```bash
cd backend
uv run python -m valorizador ingest-catalogo /ruta/al/catalogo_cen_2026.06.1.xlsx
```

---

## 6. Decisiones postergadas para etapas posteriores

> Estas decisiones **no son problemas no resueltos**: son explícitamente diferidas porque el prototipo prioriza validar dominio y UX. La arquitectura está pensada para tomarlas sin reescritura del núcleo.

### 6.1 Despliegue

- **Hoy (prototipo)**: corre en localhost del desarrollador/usuario validador.
- **Etapa siguiente**: empaquetar en Docker Compose (backend + frontend estático + Postgres) y desplegar en infraestructura on-premise CEN. La data del Plan de Expansión es sensible regulatoriamente; descartado cloud público por ahora.
- **Eventualmente**: Kubernetes o similar si crece la base de usuarios.

### 6.2 Persistencia

- **Hoy**: SQLite local, archivo único, sin réplica.
- **Etapa siguiente**: migrar a **PostgreSQL** (cambio menor con SQLAlchemy). El esquema Alembic ya es portable.
- **Decisiones a tomar**: backup, retención de snapshots históricos, performance bajo concurrencia.

### 6.3 Autenticación y autorización

- **Hoy**: sin auth. Header `X-Rol-Simulado` para alternar roles.
- **Etapa siguiente**: SSO con **Azure AD** o el directorio corporativo del CEN, vía OIDC. Roles mapeados desde grupos del directorio (`curador_catalogo`, `modelador`).
- **Cambios necesarios**: middleware en `backend/api/deps.py` que reemplace el header simulado; flujo de login en frontend (redirect OIDC + manejo de tokens).

### 6.4 Integraciones con sistemas CEN

- **InfoTécnica**: cuando se modela un seccionamiento, los antecedentes de la línea existente (tipo conductor, n° conductores por fase, cable guardia) deberían provenir de InfoTécnica vía API.
- **Sistema de Información Pública / repositorios de obras**: para auto-llenar antecedentes del decreto (nombre, número, plazo).
- **Estructura prevista**: módulo `backend/valorizador/integraciones/` con un adapter por sistema externo.

### 6.5 Funcionalidad postergada

| Funcionalidad | Etapa |
|---|---|
| Vista **DEE Planta** (planta topológica) | v2 |
| Trazado geográfico con **Google Earth** | v2 |
| **Comparación lado a lado** entre proyectos | v2 |
| **Export JSON** estructurado para integración con otros sistemas | v2 |
| **Internacionalización** | si se requiere |
| **Modo oscuro** completo | si se solicita |
| **Acceso desde tablet/mobile** | no priorizado (editor canvas es desktop-first) |
| **Comparación de snapshots** de un mismo proyecto en distintas versiones de catálogo | v2 |

### 6.6 Operación

- **Logging**: hoy stdout simple. Etapa siguiente: estructurado (JSON) con `structlog`, agregado a un sistema central.
- **Métricas**: hoy ninguna. Etapa siguiente: Prometheus + Grafana on-premise.
- **Auditoría**: hoy registro mínimo en BD (`creado_por`, `modificado_en`). Etapa siguiente: log de auditoría inmutable con `usuario + acción + recurso + diff`.
- **Backups**: hoy ninguno (es un prototipo). Etapa siguiente: backup nocturno de Postgres + retención.

---

## 7. Roadmap de implementación post-documentos

Una vez aprobados los 5 documentos, el orden sugerido de implementación del prototipo es:

| Fase | Entregable | Dependencias |
|---|---|---|
| **F1** | Bootstrap del repo: estructura, deps, alembic init, `Hello world` FastAPI + React | — |
| **F2** | Ingesta del catálogo Excel + persistencia + endpoints `GET /catalogo/...` | F1 |
| **F3** | Motor de cálculo aislado con tests sobre fixtures conocidos | F1 |
| **F4** | API de proyectos (CRUD + transiciones) + `POST /valorizacion` (preview) | F2, F3 |
| **F5** | Canvas básico (React Flow) con paleta y conexiones, sin formularios completos | F1 |
| **F6** | Formularios por componente y herencia de atributos | F4, F5 |
| **F7** | Panel de estimación en vivo conectado al motor | F4, F6 |
| **F8** | Vista del curador: subir borrador, ver diff, publicar | F2 |
| **F9** | Generación de outputs (memoria Excel y reporte PDF) | F3 |
| **F10** | Snapshot al pasar a "Valorizado" + clonación de proyectos | F4 |

Cada fase debe cerrarse con una **demo funcional al usuario** antes de avanzar a la siguiente.

---

## 8. Convenciones del proyecto

### 8.1 Nombres

- **Identificadores en el código**: español snake_case para dominio (`tipo_obra`, `costo_directo_base`); inglés cuando es estándar técnico (`router`, `query`, `mutation`).
- **Texto visible al usuario**: español chileno técnico, sin traducir términos del dominio (paño, vano, AIS, BBCC, etc.).
- **Archivos de doc**: `NN-titulo-kebab-case.md` dentro de `docs/`.

### 8.2 Versionado

- Versionado de catálogo: ver `05-template-excel-catalogo.md` §5.
- Versionado de la spec del template: en el mismo doc §8.
- Versionado de la aplicación: semver (`MAJOR.MINOR.PATCH`) cuando salgamos de prototipo. Hoy, sin versión publicada.

### 8.3 Decisiones arquitectónicas

Decisiones no triviales se registran como **ADRs** (Architecture Decision Records) cortos en `docs/adr/NNN-titulo.md` cuando aparezcan. Aún sin ADRs.

---

## 9. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| El consultor entrega el Excel en un formato distinto al spec | Spec §7 incluye checklist de entrega; reuniones de revisión antes de la primera entrega formal |
| El motor de cálculo produce números distintos a Excel manual del CEN | Tests de regresión sobre fixtures de obras conocidas; comparación lado a lado durante la fase F3 |
| React Flow no escala con 100+ nodos en una obra grande | El prototipo se valida con obras típicas (~20-40 nodos). Si una obra real excede, optimizar nodos virtualizados o paginar instalaciones |
| Decisiones del decreto no encajan en los componentes tipificados | Mecanismo de partidas ad-hoc cubre el caso excepcional; revisar con el CEN si se vuelve frecuente |
| La integración futura con InfoTécnica cambia el contrato de antecedentes de línea | El módulo de antecedentes está aislado en el formulario de línea; no acopla al resto del modelo |
| El catálogo crece y la ingesta se vuelve lenta | Validación se hace en memoria; al pasar de prototipo a producción, considerar batching o ingesta asíncrona |

---

## 10. Glosario técnico abreviado

| Término | Significado |
|---|---|
| **SPA** | Single Page Application (frontend) |
| **OIDC** | OpenID Connect (auth federada) |
| **ORM** | Object-Relational Mapper |
| **ADR** | Architecture Decision Record |
| **SQLite** | BD embebida en archivo único |
| **HMR** | Hot Module Replacement (Vite) |
| **Snapshot** | Estado inmutable de un proyecto valorizado |

Para el glosario del dominio (CEN), ver `01-contexto-y-dominio.md` §2.

---

## 11. Cómo seguir

- Para entender qué hace el sistema y qué problema resuelve: empieza por **`01-contexto-y-dominio.md`**.
- Para implementar el backend: **`02-backend.md`**.
- Para implementar el frontend: **`03-frontend.md`**.
- Para entregar el catálogo (consultor externo): **`05-template-excel-catalogo.md`**.
- Para correr el sistema o decidir cómo escalarlo: este documento.
