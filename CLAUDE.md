# CLAUDE.md — Platziflix

Proyecto educativo del Curso de Claude Code de Platzi (profesor: Eduardo Alvarez).
Plataforma de streaming de cursos online tipo Netflix: listado de cursos, detalle de curso con clases, y reproductor de video.

---

## Estructura general del monorepo

```
claude-code-platzi/
├── Backend/      # API REST en Python/FastAPI
├── Frontend/     # Web en Next.js 15 + React 19
├── Mobile/       # Android en Kotlin/Jetpack Compose (uso mínimo)
└── README.md
```

---

## Backend

### Stack
- Python 3.11+, FastAPI, SQLAlchemy 2.0, PostgreSQL 15
- Alembic para migraciones
- `uv` como gestor de paquetes (reemplaza pip/poetry)
- Docker + docker-compose para el entorno completo

### Cómo correr

```bash
cd Backend
make start        # levanta db + api (docker-compose up -d)
make migrate      # aplica migraciones Alembic
make seed         # carga datos de prueba
make logs         # ver logs
make stop         # detener
make seed-fresh   # limpiar y re-sembrar datos
```

La API queda en `http://localhost:8000`. La DB PostgreSQL en `localhost:5432`.

Variables de entorno (docker-compose las setea automáticamente):
- `DATABASE_URL=postgresql://platziflix_user:platziflix_password@db:5432/platziflix_db`

### Estructura de archivos

```
Backend/
├── app/
│   ├── main.py                    # Entrypoint FastAPI, definición de endpoints
│   ├── core/config.py             # Settings con pydantic-settings (lee .env)
│   ├── db/
│   │   ├── base.py                # Engine SQLAlchemy + get_db dependency
│   │   └── seed.py                # Script de seed: create_sample_data() / clear_all_data()
│   ├── models/
│   │   ├── base.py                # BaseModel: id, created_at, updated_at, deleted_at (soft delete)
│   │   ├── course.py              # Course: name, description, thumbnail, slug
│   │   ├── lesson.py              # Lesson: course_id, name, description, slug, video_url  ← MODELO ACTIVO
│   │   ├── class.py               # Class: mismo schema que lesson, tabla 'classes'  ← ARCHIVO HUÉRFANO (no exportado)
│   │   ├── teacher.py             # Teacher: name, email
│   │   ├── course_teacher.py      # Tabla asociativa course_teachers (many-to-many)
│   │   └── __init__.py            # Exporta: Base, BaseModel, Teacher, Course, Lesson, course_teachers
│   ├── services/
│   │   └── course_service.py      # CourseService: get_all_courses(), get_course_by_slug()
│   ├── alembic/
│   │   └── versions/d18a08253457_*.py  # Migración inicial: crea teachers, courses, lessons, course_teachers
│   └── test_main.py               # Tests con pytest + FastAPI TestClient (mocks de CourseService)
├── Dockerfile
├── docker-compose.yml
├── Makefile                       # Comandos de gestión del entorno
├── pyproject.toml                 # Dependencias del proyecto
└── specs/
    └── 00_contracts.md            # Contratos de API y entidades
```

### Modelos de base de datos

**teachers** — `Teacher`
| campo | tipo | notas |
|-------|------|-------|
| id | Integer PK | |
| name | String(255) | |
| email | String(255) | unique, indexed |
| created_at / updated_at / deleted_at | DateTime | BaseModel |

**courses** — `Course`
| campo | tipo | notas |
|-------|------|-------|
| id | Integer PK | |
| name | String(255) | |
| description | Text | |
| thumbnail | String(500) | URL |
| slug | String(255) | unique, indexed |
| created_at / updated_at / deleted_at | DateTime | |

Relaciones: `teachers` (M2M via `course_teachers`), `lessons` (1-to-many)

**lessons** — `Lesson` (tabla usada en producción; las clases del curso)
| campo | tipo | notas |
|-------|------|-------|
| id | Integer PK | |
| course_id | Integer FK → courses.id | indexed |
| name | String(255) | |
| description | Text | |
| slug | String(255) | indexed |
| video_url | String(500) | URL del video |
| created_at / updated_at / deleted_at | DateTime | |

**course_teachers** — tabla asociativa (course_id PK + teacher_id PK)

### Endpoints implementados

```
GET /          → { "message": "Bienvenido a Platziflix API" }
GET /health    → { status, service, version, database: bool, courses_count? }

GET /courses
  → [ { id, name, description, thumbnail, slug }, ... ]
  Soft-delete filtrado (deleted_at IS NULL)

GET /courses/{slug}
  → { id, name, description, thumbnail, slug, teacher_id: [int], classes: [{id, name, description, slug}] }
  classes viene de la tabla lessons (filtrando deleted_at IS NULL)
  404 si no existe
```

**Endpoint PENDIENTE** (definido en contratos pero no implementado):
```
GET /courses/{slug}/classes/{id}  → clase individual con video_url
```

### Tests (Backend)

```bash
# Dentro del contenedor:
docker-compose exec api bash -c "cd /app && uv run pytest app/test_main.py -v"
```

Tests en `app/test_main.py`: usan `Mock(spec=CourseService)` y `app.dependency_overrides`.
Clases de tests: `TestRootEndpoint`, `TestHealthEndpoint`, `TestCoursesEndpoints`, `TestContractCompliance`.

---

## Frontend

### Stack
- Next.js 15.3.3 (App Router), React 19, TypeScript
- SCSS Modules para estilos
- Vitest + React Testing Library + jsdom para tests
- Yarn como gestor de paquetes

### Cómo correr

```bash
cd Frontend
yarn install
yarn dev      # http://localhost:3000
yarn test     # tests unitarios con vitest
yarn build    # build de producción
```

### Estructura de archivos

```
Frontend/
├── src/
│   ├── app/
│   │   ├── layout.tsx                          # Layout raíz
│   │   ├── page.tsx                            # Página home: listado de cursos
│   │   ├── page.module.scss
│   │   ├── course/[slug]/
│   │   │   ├── page.tsx                        # Detalle del curso
│   │   │   ├── error.tsx / loading.tsx / not-found.tsx
│   │   │   └── *.module.scss
│   │   └── classes/[class_id]/
│   │       ├── page.tsx                        # Reproductor de clase
│   │       └── page.module.scss
│   ├── components/
│   │   ├── Course/
│   │   │   ├── Course.tsx                      # Tarjeta de curso (thumbnail, title, teacher, duration)
│   │   │   ├── Course.module.scss
│   │   │   └── __test__/Course.test.tsx
│   │   ├── CourseDetail/
│   │   │   ├── CourseDetail.tsx                # Vista detalle: header + lista de clases
│   │   │   └── CourseDetail.module.scss
│   │   └── VideoPlayer/
│   │       ├── VideoPlayer.tsx                 # <video> nativo con controls
│   │       ├── VideoPlayer.module.scss
│   │       └── VideoPlayer.test.tsx
│   ├── types/index.ts                          # Interfaces TypeScript del dominio
│   ├── styles/
│   │   ├── reset.scss
│   │   └── vars.scss
│   └── test/setup.ts                           # Setup de Vitest
├── docs/
│   └── curso-react-reproductor-video.feature   # Spec BDD (Gherkin)
├── .cursor/rules/
│   ├── project-stack.mdc                       # Guía Next.js best practices
│   ├── components-guide.mdc                    # Convenciones de componentes
│   ├── scss-conventions.mdc
│   └── unit-testing-guidelines.mdc
├── next.config.ts
├── vitest.config.ts
└── package.json
```

### Tipos (src/types/index.ts)

```typescript
Course         { id, title, teacher, duration, thumbnail, slug }
Class          { id, title, description, video, duration, slug }
CourseDetail   extends Course { description, classes: Class[] }
Progress       { progress: number, user_id: number }
QuizOption     { id, answer, correct }
Quiz           { id, question, options: QuizOption[] }
FavoriteToggle { course_id: number }
```

### Páginas y flujo de navegación

```
/                         → Home: fetch GET /courses → grid de CourseComponent
/course/[slug]            → Detalle: fetch GET /courses/{slug} → CourseDetailComponent
/classes/[class_id]       → Clase: fetch GET /classes/{class_id} → VideoPlayer
```

### Convenciones de componentes
- Cada componente en su propia carpeta: `ComponentName/ComponentName.tsx + .module.scss + .test.tsx`
- PascalCase para componentes, camelCase para clases CSS
- Arrow functions con FC<Props>
- Tipos importados desde `@/types` cuando existen; crear nuevos solo si son específicos del componente

---

## Mobile (superficial)

Android — `Mobile/PlatziFlixAndroid/`

### Stack
- Kotlin, Jetpack Compose, Hilt (DI), Retrofit (red)
- Arquitectura CLEAN: data / domain / presentation
- MVVM con StateFlow para UI state

### Estructura principal

```
data/
  entities/CourseDTO.kt          # DTO de red
  mappers/CourseMapper.kt        # DTO → domain model
  network/ApiService.kt          # Retrofit interface
  repositories/RemoteCourseRepository.kt
domain/
  models/Course.kt               # Modelo de dominio
  repositories/CourseRepository.kt  # Interfaz
presentation/
  courses/
    screen/CourseListScreen.kt
    viewmodel/CourseListViewModel.kt
    state/CourseListUiState.kt
    components/{CourseCard, ErrorMessage, LoadingIndicator}.kt
```

Solo implementa listado de cursos. Consume `GET /courses`.

---

## Discrepancias conocidas (Backend ↔ Frontend)

Estos son bugs/deudas técnicas activas al 2026-06-14:

### 1. Nombres de campos: `name` vs `title`
- Backend devuelve `name` (courses y lessons)
- Frontend espera `title` en su type `Course` y `Class`
- **Impacto**: la home page muestra `undefined` en títulos si la API está conectada

### 2. Formato de respuesta: objeto wrapper vs array directo
- `page.tsx` home hace `return data.data` pero la API devuelve un array directo (sin wrapper `.data`)
- **Impacto**: `courses` es `undefined` → crash en runtime

### 3. Campos faltantes en la API
- Frontend `Course` espera `teacher` (string) y `duration` (number)
- Backend no devuelve ninguno de esos campos
- `teacher_id` en el backend es un array de IDs, no el nombre del profesor

### 4. Endpoint `/classes/{class_id}` no existe en el backend
- `Frontend/src/app/classes/[class_id]/page.tsx` hace fetch a `GET /classes/{class_id}`
- El backend no tiene ese endpoint (solo `/courses/{slug}` que incluye clases como lista)
- **Impacto**: la página de clase crashea con 404

### 5. `class.py` modelo huérfano
- `Backend/app/models/class.py` define `Class` con tabla `classes`, pero no está exportado en `__init__.py`
- La tabla `classes` no existe en la migración; la migración real crea `lessons`
- `course_service.py` usa `Lesson` pero devuelve el resultado como clave `"classes"` en el JSON

### 6. `CourseDetail` asume `duration` en clases
- `CourseDetail.tsx` hace `course.classes.reduce((acc, cls) => acc + cls.duration, 0)`
- `Class.duration` es `number` en el type pero la API no lo devuelve → suma `NaN`

---

## API Contracts (referencia rápida)

Definidos en `Backend/specs/00_contracts.md`.

### GET /courses
```json
[{ "id": 1, "name": "...", "description": "...", "thumbnail": "...", "slug": "..." }]
```

### GET /courses/:slug
```json
{
  "id": 1, "name": "...", "description": "...", "thumbnail": "...", "slug": "...",
  "teacher_id": [1, 2],
  "classes": [{ "id": 1, "name": "...", "description": "...", "slug": "..." }]
}
```

### GET /courses/:slug/classes/:id (pendiente de implementar)
```json
{
  "id": 1, "name": "...", "description": "...", "slug": "...",
  "video_url": "https://..."
}
```

---

## Seed data (datos de prueba)

Definido en `Backend/app/db/seed.py`:
- 3 teachers: Juan Pérez, María García, Carlos Rodríguez
- 3 courses: Curso de React (slug: `curso-de-react`), Curso de Python, Curso de JavaScript
- 6 lessons distribuidas entre los cursos
- Relaciones M2M teacher↔course ya asignadas
