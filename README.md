# RAG Estimation Platform

Plataforma web para estimar presupuestos de proyectos tecnológicos usando **RAG** (Retrieval-Augmented Generation). Permite ingestar presupuestos históricos, indexarlos con embeddings vectoriales, y generar estimaciones de horas y costes para nuevos proyectos basándose en datos reales.

El sistema consta de dos repositorios independientes:

- **rag-estimation-platform** (este repo) — Aplicación web Rails para la gestión de estimaciones, exportación PDF/JSON y UI.
- **[rag-estimation-service](https://github.com/LIDR-ai/lidr-rag-estimation-service-demo)** — Servicio RAG con FastAPI, pgvector y OpenAI para indexación y estimación con IA.

## Arquitectura del sistema

```
┌─────────────────┐        HTTP (Faraday)        ┌──────────────────────┐
│                 │  ───────────────────────────► │                      │
│   Rails App     │  POST /api/v1/ingest         │   RAG Service        │
│   :3000         │  POST /api/v1/estimate       │   :8000              │
│                 │  POST /api/v1/search         │                      │
│  ┌───────────┐  │  GET  /api/v1/ingest/:id/... │  ┌────────────────┐  │
│  │ SolidQueue│  │  ◄─────────────────────────  │  │ FastAPI        │  │
│  │ Jobs      │  │        JSON responses        │  │ + pgvector     │  │
│  └───────────┘  │                              │  │ + OpenAI API   │  │
│                 │                              │  └────────────────┘  │
└────────┬────────┘                              └──────────┬───────────┘
         │                                                  │
         ▼                                                  ▼
┌─────────────────┐                              ┌──────────────────────┐
│  PostgreSQL 16  │                              │  PostgreSQL 16       │
│  (Rails data)   │                              │  + pgvector          │
│  :5433          │                              │  (embeddings)        │
└─────────────────┘                              │  :5432               │
                                                 └──────────────────────┘
```

| Componente | Stack | Responsabilidad |
|---|---|---|
| **rag-estimation-platform** | Rails 8, PostgreSQL 16, Hotwire (Turbo + Stimulus), Tailwind CSS, SolidQueue | UI web, autenticación, gestión de estimaciones, exportación PDF/JSON |
| **rag-estimation-service** | FastAPI, PostgreSQL 16 + pgvector, OpenAI API | Indexación de presupuestos, búsqueda vectorial, estimación de horas con LLM |

## Requisitos previos

- **Ruby 3.4.4**
- **PostgreSQL 16**
- **Node.js** (para Tailwind CLI — lo instala `tailwindcss-rails`)
- **Python 3.11+** (para el servicio RAG)
- **Docker y Docker Compose** (alternativa para BD y/o servicio RAG)
- **API key de OpenAI** (para el servicio RAG)

## Setup del servicio RAG (rag-estimation-service)

> Se explica primero porque la plataforma Rails depende de él.

### Opcion A — Con Docker (recomendado)

```bash
cd ../rag-estimation-service
cp .env.example .env
# Editar .env → añadir OPENAI_API_KEY
docker compose up --build
```

Esto levanta:
- PostgreSQL con pgvector en `:5432`
- Servicio FastAPI en `:8000`

Verificar que funciona:

```bash
curl http://localhost:8000/api/v1/health
```

Docs interactivos de la API: `http://localhost:8000/docs`

### Opcion B — Local (sin Docker)

1. Instalar PostgreSQL 16 con la extensión [pgvector](https://github.com/pgvector/pgvector)
2. Crear el entorno e instalar dependencias:

```bash
cd ../rag-estimation-service
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

3. Configurar variables de entorno:

```bash
cp .env.example .env
# Editar .env → añadir OPENAI_API_KEY y ajustar DATABASE_URL si es necesario
```

4. Ejecutar migraciones y arrancar:

```bash
alembic upgrade head
uvicorn app.main:app --reload --port 8000
```

## Setup de la plataforma (rag-estimation-platform)

### Opcion A — Local (recomendado para desarrollo)

1. **Base de datos:** necesitas PostgreSQL corriendo. Puedes usar Docker solo para la BD:

```bash
docker compose -f docker-compose.dev.yml up db
```

Esto levanta PostgreSQL en `:5433` (para no chocar con el `:5432` del servicio RAG).

2. **Variables de entorno:**

```bash
# Crear .env.development (o verificar que existe)
cat .env.development
```

Debe contener:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5433/rag_estimation_platform_development
RAG_SERVICE_BASE_URL=http://localhost:8000
```

3. **Instalar dependencias y preparar la BD:**

```bash
bin/setup
```

4. **Cargar datos de prueba:**

```bash
bin/rails db:seed
```

5. **Arrancar la aplicación:**

```bash
bin/dev
```

Esto levanta Rails en `:3000` + el watcher de Tailwind CSS.

### Opcion B — Todo en Docker

```bash
docker compose -f docker-compose.dev.yml up --build
```

> **Nota:** dentro del contenedor, `RAG_SERVICE_BASE_URL` apunta a `http://host.docker.internal:8000` para conectar con el servicio RAG que corre en el host.

## Datos de prueba

Los seeds crean datos de ejemplo para probar toda la plataforma:

| Recurso | Datos |
|---|---|
| **Usuario demo** | `demo@lidr.co` / `password123` |
| **Documentos** | 7 presupuestos de ejemplo (SaaS, e-commerce, fintech, etc.) |
| **Estimaciones** | 7 estimaciones con bloques funcionales, tareas y referencias |

```bash
bin/rails db:seed
```

## Como interactuan ambos servicios

### Ingesta de presupuestos

1. El usuario sube un JSON en `/ingest`
2. Rails envia `POST /api/v1/ingest` al servicio RAG
3. El servicio RAG chunkeriza el presupuesto, genera embeddings (OpenAI), y lo almacena en pgvector
4. `IngestionStatusJob` en Rails sondea `GET /api/v1/ingest/:id/status` hasta que completa

### Estimacion de proyectos

1. El usuario crea una estimacion y la estructura en bloques/tareas
2. `EstimationJob` envia `POST /api/v1/estimate` con la descripcion del proyecto
3. El servicio RAG busca presupuestos similares por vector y genera estimacion de horas con LLM
4. `TaskValidationJob` envia `POST /api/v1/estimate/validate` para refinar horas tarea a tarea

### Busqueda

1. El usuario busca en `/search`
2. Rails envia `POST /api/v1/search` al servicio RAG
3. Devuelve chunks relevantes con score de similaridad

## Tests

```bash
bin/rails test          # Todos los tests
bin/rails test:system   # Tests de sistema (requiere Chrome/Chromium)
```

Los tests usan **WebMock** — no requieren el servicio RAG corriendo. Fixtures en `test/fixtures/`.

## Comandos utiles

| Comando | Descripcion |
|---|---|
| `bin/dev` | Arranca Rails + Tailwind watcher |
| `bin/rails db:seed` | Carga datos de prueba |
| `bin/rails db:migrate` | Ejecuta migraciones pendientes |
| `bin/rails routes` | Lista todas las rutas |
| `bin/rails test` | Ejecuta tests |

## Variables de entorno

### rag-estimation-platform

| Variable | Descripcion | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | — |
| `RAG_SERVICE_BASE_URL` | URL del servicio RAG | `http://localhost:8000` |

### rag-estimation-service (referencia rapida)

| Variable | Descripcion | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL+asyncpg connection string | — |
| `OPENAI_API_KEY` | API key de OpenAI | — |
| `EMBEDDING_MODEL` | Modelo de embeddings | `text-embedding-3-small` |
| `LLM_MODEL` | Modelo LLM para estimaciones | `gpt-4o` |
| `API_KEY` | API key del servicio | `dev-api-key` |
