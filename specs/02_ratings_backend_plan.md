# Plan de Implementación: Sistema de Ratings — Backend

## Contexto

Se va a construir un sistema de calificaciones (ratings) de cursos para la plataforma Platziflix. Los usuarios podrán calificar un curso con una puntuación del 1 al 5 y opcionalmente dejar un comentario de texto. No existe autenticación en el sistema, por lo que el `user_id` es un entero libre que el cliente envía directamente.

El sistema garantiza una única calificación por usuario por curso mediante un constraint UNIQUE. Si el usuario ya calificó el curso y vuelve a enviar una calificación, se actualiza en lugar de crear un registro duplicado (patrón upsert).

Los endpoints de listado de cursos (`GET /courses` y `GET /courses/{slug}`) se actualizan para incluir el promedio de calificaciones y el total de votos, de modo que el frontend pueda mostrar esta información sin hacer peticiones adicionales.

---

## PASO 1: Crear modelo CourseRating

Crear el archivo `app/models/rating.py`.

El modelo `CourseRating` hereda de `BaseModel` (que ya provee `id`, `created_at`, `updated_at`, `deleted_at`).

Campos adicionales del modelo:

```
CourseRating
├── course_id   Integer, FK → courses.id, NOT NULL, indexed
├── user_id     Integer, NOT NULL, indexed  (sin FK — no hay tabla users)
├── rating      Integer, NOT NULL           (valor entre 1 y 5 inclusive)
└── comment     Text, nullable
```

Constraints a definir en el modelo:

- `CheckConstraint` sobre `rating`: `rating >= 1 AND rating <= 5`
- `UniqueConstraint` sobre `(course_id, user_id)` — una calificación por usuario por curso

Relación ORM:

- `course` → relación many-to-one hacia `Course` (back_populates a definir en PASO 3)

Nombre de tabla en la base de datos: `course_ratings`

Esquema lógico de la tabla resultante:

```
course_ratings
├── id           SERIAL PRIMARY KEY
├── course_id    INTEGER NOT NULL REFERENCES courses(id)
├── user_id      INTEGER NOT NULL
├── rating       INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5)
├── comment      TEXT
├── created_at   TIMESTAMP
├── updated_at   TIMESTAMP
└── deleted_at   TIMESTAMP  (soft delete heredado de BaseModel)

UNIQUE INDEX sobre (course_id, user_id)
INDEX sobre course_id
INDEX sobre user_id
```

---

## PASO 2: Actualizar `__init__.py` de modelos

Archivo: `app/models/__init__.py`

Agregar la importación del nuevo modelo para que SQLAlchemy lo registre en la metadata y Alembic pueda detectarlo en la autogeneración:

```
Importaciones actuales: Base, BaseModel, Teacher, Course, Lesson, course_teachers
Agregar: CourseRating
```

El `__init__.py` debe exportar `CourseRating` junto con los modelos existentes.

---

## PASO 3: Agregar relación en modelo Course

Archivo: `app/models/course.py`

Agregar el campo de relación ORM en la clase `Course`:

```
ratings → relación one-to-many hacia CourseRating
          back_populates="course"
          lazy="dynamic" o lazy="select" (según necesidad de queries)
          cascade="all, delete-orphan"
```

Esta relación permite que `CourseService` pueda calcular el promedio y conteo directamente desde el objeto `Course` si lo necesita, o que se puedan hacer queries más eficientes con joins.

---

## PASO 4: Crear Pydantic schemas en `app/schemas/rating.py`

Crear el directorio `app/schemas/` si no existe, y dentro el archivo `rating.py`.

Schemas a definir:

**RatingRequest** — body del POST, validado con Pydantic:
```
RatingRequest
├── user_id   int        (requerido, > 0)
├── rating    int        (requerido, entre 1 y 5 inclusive — usar Field con ge=1, le=5)
└── comment   str | None (opcional, longitud máxima recomendada: 1000 caracteres)
```

**RatingResponse** — respuesta al crear/actualizar un rating:
```
RatingResponse
├── id          int
├── course_id   int
├── user_id     int
├── rating      int
├── comment     str | None
├── created_at  datetime
└── updated_at  datetime
```

**RatingStatsResponse** — respuesta del endpoint de estadísticas:
```
RatingStatsResponse
├── avg_rating      float | None   (None si no hay calificaciones)
├── rating_count    int
└── distribution    dict[int, int] (clave: estrella 1-5, valor: cantidad de votos)

Ejemplo de distribution:
{
  "1": 2,
  "2": 0,
  "3": 5,
  "4": 12,
  "5": 8
}
```

**RatingListResponse** — respuesta del endpoint de listado de ratings:
```
RatingListResponse
├── ratings       list[RatingResponse]
├── total         int
├── avg_rating    float | None
└── rating_count  int
```

Todos los schemas de respuesta deben configurar `model_config = ConfigDict(from_attributes=True)` para compatibilidad con objetos ORM.

---

## PASO 5: Crear RatingService en `app/services/rating_service.py`

El servicio recibe una sesión de base de datos como dependencia (inyectada por FastAPI con `get_db`). Encapsula toda la lógica de negocio relacionada con ratings.

Métodos a implementar:

**`create_or_update(db, course_id, user_id, rating, comment) → dict`**

Lógica:
1. Buscar si ya existe un registro en `course_ratings` con `(course_id=course_id, user_id=user_id)` y `deleted_at IS NULL`
2. Si existe: actualizar campos `rating`, `comment`, `updated_at`
3. Si no existe: crear nuevo registro `CourseRating`
4. Hacer commit y retornar el registro como diccionario

Campos del diccionario retornado: `id, course_id, user_id, rating, comment, created_at, updated_at`

**`get_by_course(db, course_id) → dict`**

Lógica:
1. Obtener todos los ratings del curso donde `deleted_at IS NULL`
2. Calcular `avg_rating` y `rating_count`
3. Retornar diccionario con `ratings (lista), total, avg_rating, rating_count`

**`get_stats(db, course_id) → dict`**

Lógica:
1. Obtener todos los ratings activos del curso
2. Calcular `avg_rating` (promedio, redondeado a 2 decimales; None si no hay datos)
3. Calcular `rating_count` (total de calificaciones)
4. Calcular `distribution`: diccionario `{1: 0, 2: 0, 3: 0, 4: 0, 5: 0}` con el conteo real por estrella
5. Retornar diccionario con los tres campos

Nota de eficiencia: usar `func.avg()` y `func.count()` de SQLAlchemy para el promedio y conteo en una sola query cuando sea posible, y la distribución en otra query agrupada por `rating`.

**`delete(db, course_id, user_id) → bool`**

Lógica:
1. Buscar el registro con `(course_id, user_id)` activo
2. Si no existe: retornar `False`
3. Si existe: aplicar soft delete (asignar `deleted_at = datetime.utcnow()`)
4. Hacer commit y retornar `True`

**`get_course_rating_summary(db, course_id) → dict`**

Método auxiliar usado por `CourseService`:

Retorna solo `{"avg_rating": float | None, "rating_count": int}` de forma eficiente con una sola query de agregación.

---

## PASO 6: Actualizar CourseService

Archivo: `app/services/course_service.py`

**Modificar `get_all_courses()`:**

Después de obtener la lista de cursos, para cada curso calcular `avg_rating` y `rating_count` usando `RatingService.get_course_rating_summary()` o directamente con una subquery/join.

Formato de respuesta actualizado por curso:
```
{
  "id": ...,
  "name": ...,
  "description": ...,
  "thumbnail": ...,
  "slug": ...,
  "avg_rating": 4.3,      ← NUEVO (float o None)
  "rating_count": 15      ← NUEVO (int)
}
```

**Modificar `get_course_by_slug(slug)`:**

Agregar los mismos campos `avg_rating` y `rating_count` al diccionario de respuesta del curso individual.

Formato de respuesta actualizado:
```
{
  "id": ...,
  "name": ...,
  "description": ...,
  "thumbnail": ...,
  "slug": ...,
  "teacher_id": [...],
  "classes": [...],
  "avg_rating": 4.3,      ← NUEVO
  "rating_count": 15      ← NUEVO
}
```

Consideración de rendimiento: si la lista de cursos es grande, calcular el avg_rating en un solo query con JOIN y GROUP BY en lugar de N queries individuales (una por curso).

---

## PASO 7: Agregar endpoints en `app/main.py`

Agregar los siguientes cuatro endpoints. Todos validan que el curso exista antes de operar (retornan 404 si no existe el slug).

**POST `/courses/{slug}/rate`**

- Propósito: crear o actualizar el rating de un usuario para un curso
- Request body: `RatingRequest` (user_id, rating, comment)
- Validación previa: verificar que el curso exista por slug; si no, retornar 404
- Delegar a: `RatingService.create_or_update()`
- Respuesta exitosa: `201 Created` con `RatingResponse`
- Errores posibles: 404 curso no encontrado, 422 validación de campos

```
POST /courses/{slug}/rate
Body: { "user_id": 42, "rating": 5, "comment": "Excelente curso" }
Response 201: { "id": 1, "course_id": 3, "user_id": 42, "rating": 5, ... }
```

**GET `/courses/{slug}/ratings`**

- Propósito: listar todos los ratings de un curso
- Sin body ni query params requeridos (opcionalmente paginación futura)
- Validación previa: verificar que el curso exista por slug
- Delegar a: `RatingService.get_by_course()`
- Respuesta: `200 OK` con `RatingListResponse`

```
GET /courses/curso-de-react/ratings
Response 200: {
  "ratings": [...],
  "total": 15,
  "avg_rating": 4.3,
  "rating_count": 15
}
```

**GET `/courses/{slug}/ratings/stats`**

- Propósito: estadísticas agregadas del curso (promedio, conteo, distribución)
- Validación previa: verificar que el curso exista por slug
- Delegar a: `RatingService.get_stats()`
- Respuesta: `200 OK` con `RatingStatsResponse`

```
GET /courses/curso-de-react/ratings/stats
Response 200: {
  "avg_rating": 4.3,
  "rating_count": 15,
  "distribution": { "1": 0, "2": 1, "3": 2, "4": 5, "5": 7 }
}
```

**DELETE `/courses/{slug}/rate/{user_id}`**

- Propósito: eliminar (soft delete) el rating de un usuario específico
- Validación previa: verificar que el curso exista por slug
- Delegar a: `RatingService.delete()`
- Respuesta exitosa: `204 No Content`
- Si no existe el rating: `404 Not Found` con mensaje descriptivo

```
DELETE /courses/curso-de-react/rate/42
Response 204: (sin body)
Response 404: { "detail": "Rating no encontrado para user_id=42 en este curso" }
```

Inyección de dependencias: los endpoints reciben `db: Session = Depends(get_db)` e instancian `RatingService` dentro del handler, igual que el patrón actual con `CourseService`.

---

## PASO 8: Crear migración Alembic

La migración crea la tabla `course_ratings` con todos sus campos, índices y constraints.

Comando a ejecutar:

```bash
make create-migration
# Ingresar mensaje: "add_course_ratings_table"
# Luego aplicar:
make migrate
```

O directamente:

```bash
docker-compose exec api bash -c "cd /app && alembic revision --autogenerate -m 'add_course_ratings_table'"
docker-compose exec api bash -c "cd /app && alembic upgrade head"
```

La migración autogenerada debe incluir:

```
Tabla creada: course_ratings
Columnas: id, course_id, user_id, rating, comment, created_at, updated_at, deleted_at
FK: course_ratings.course_id → courses.id
CheckConstraint: rating >= 1 AND rating <= 5
UniqueConstraint: (course_id, user_id)
Índices: ix_course_ratings_course_id, ix_course_ratings_user_id
```

Verificar visualmente el archivo de migración generado antes de aplicarlo para confirmar que Alembic detectó correctamente el nuevo modelo y la relación.

---

## PASO 9: Actualizar seed con ratings de ejemplo

Archivo: `app/db/seed.py`

Agregar en la función `create_sample_data()`, después de crear los cursos y lecciones, ratings de ejemplo para que la demo funcione con datos realistas.

Distribución sugerida de ratings de ejemplo:

```
Curso de React (slug: curso-de-react):
  user_id=1, rating=5, comment="Muy buen curso, aprendí mucho"
  user_id=2, rating=4, comment="Buen contenido pero le falta profundidad"
  user_id=3, rating=5, comment=None
  user_id=4, rating=3, comment="Regular, esperaba más ejemplos"
  → avg_rating esperado: 4.25, rating_count: 4

Curso de Python (slug: curso-de-python):
  user_id=1, rating=5, comment="El mejor curso de Python que he tomado"
  user_id=2, rating=5, comment=None
  user_id=5, rating=4, comment="Muy completo"
  → avg_rating esperado: 4.67, rating_count: 3

Curso de JavaScript (slug: curso-de-javascript):
  user_id=3, rating=2, comment="Me esperaba más contenido avanzado"
  user_id=4, rating=4, comment=None
  → avg_rating esperado: 3.0, rating_count: 2
```

También actualizar `clear_all_data()` para que elimine los registros de `course_ratings` antes de los cursos (respetar el orden de FK).

---

## PASO 10: Agregar tests

Archivo: `app/test_main.py`

Agregar nuevas clases de test siguiendo el patrón existente: usar `Mock(spec=RatingService)` junto con `app.dependency_overrides` para aislar los tests de la base de datos.

**Clase `TestRatingEndpoints`**

Casos de test a cubrir:

```
test_create_rating_success
  Arrange: mock RatingService.create_or_update retorna dict con datos válidos
  Act:     POST /courses/curso-de-react/rate con body {user_id:1, rating:5}
  Assert:  status_code == 201, response contiene id, course_id, user_id, rating

test_create_rating_course_not_found
  Arrange: mock CourseService.get_course_by_slug retorna None
  Act:     POST /courses/no-existe/rate con body válido
  Assert:  status_code == 404

test_create_rating_invalid_rating_value
  Arrange: (sin mock necesario — validación Pydantic)
  Act:     POST /courses/curso-de-react/rate con body {user_id:1, rating:6}
  Assert:  status_code == 422

test_create_rating_invalid_rating_zero
  Arrange: (sin mock necesario)
  Act:     POST /courses/curso-de-react/rate con body {user_id:1, rating:0}
  Assert:  status_code == 422

test_get_ratings_success
  Arrange: mock RatingService.get_by_course retorna dict con lista de ratings
  Act:     GET /courses/curso-de-react/ratings
  Assert:  status_code == 200, response contiene keys: ratings, total, avg_rating, rating_count

test_get_ratings_course_not_found
  Arrange: mock CourseService retorna None
  Act:     GET /courses/no-existe/ratings
  Assert:  status_code == 404

test_get_ratings_stats_success
  Arrange: mock RatingService.get_stats retorna {avg_rating:4.3, rating_count:5, distribution:{...}}
  Act:     GET /courses/curso-de-react/ratings/stats
  Assert:  status_code == 200, response contiene avg_rating, rating_count, distribution
           distribution tiene keys "1" a "5"

test_delete_rating_success
  Arrange: mock RatingService.delete retorna True
  Act:     DELETE /courses/curso-de-react/rate/1
  Assert:  status_code == 204

test_delete_rating_not_found
  Arrange: mock RatingService.delete retorna False
  Act:     DELETE /courses/curso-de-react/rate/999
  Assert:  status_code == 404

test_delete_rating_course_not_found
  Arrange: mock CourseService retorna None
  Act:     DELETE /courses/no-existe/rate/1
  Assert:  status_code == 404
```

**Clase `TestCourseRatingFields`**

Verificar que los endpoints existentes de cursos ahora incluyan los campos de rating:

```
test_courses_list_includes_rating_fields
  Arrange: mock CourseService.get_all_courses retorna lista con avg_rating y rating_count
  Act:     GET /courses
  Assert:  cada curso en la respuesta tiene "avg_rating" y "rating_count"

test_course_detail_includes_rating_fields
  Arrange: mock CourseService.get_course_by_slug retorna dict con avg_rating y rating_count
  Act:     GET /courses/curso-de-react
  Assert:  response contiene "avg_rating" y "rating_count"
```

**Clase `TestRatingContractCompliance`**

Verificar que la estructura de respuesta cumple con los contratos definidos:

```
test_rating_response_schema
  Verificar que RatingResponse contiene exactamente: id, course_id, user_id, rating, comment, created_at, updated_at

test_stats_response_schema
  Verificar que RatingStatsResponse contiene: avg_rating, rating_count, distribution
  Verificar que distribution tiene las 5 claves (1 a 5)
```

---

## Verificacion

Comandos para verificar que la implementacion funciona correctamente:

**1. Levantar el entorno:**
```bash
make start
make migrate
make seed
```

**2. Ejecutar tests:**
```bash
make test
```

**3. Verificar endpoints manualmente con curl:**

```bash
# Crear rating
curl -X POST http://localhost:8000/courses/curso-de-react/rate \
  -H "Content-Type: application/json" \
  -d '{"user_id": 10, "rating": 5, "comment": "Excelente"}'

# Ver ratings del curso
curl http://localhost:8000/courses/curso-de-react/ratings

# Ver estadisticas
curl http://localhost:8000/courses/curso-de-react/ratings/stats

# Eliminar rating
curl -X DELETE http://localhost:8000/courses/curso-de-react/rate/10

# Verificar que avg_rating aparece en listado de cursos
curl http://localhost:8000/courses

# Verificar que avg_rating aparece en detalle de curso
curl http://localhost:8000/courses/curso-de-react
```

**4. Verificar documentacion automatica:**

Acceder a `http://localhost:8000/docs` y confirmar que los 4 nuevos endpoints aparecen correctamente documentados con sus schemas de request y response.

**5. Verificar constraint unico (upsert):**
```bash
# Primer rating
curl -X POST http://localhost:8000/courses/curso-de-react/rate \
  -d '{"user_id": 99, "rating": 3}'

# Actualizar el mismo rating (debe hacer update, no crear duplicado)
curl -X POST http://localhost:8000/courses/curso-de-react/rate \
  -d '{"user_id": 99, "rating": 5, "comment": "Cambie de opinion"}'

# El stats debe mostrar solo 1 rating de user_id=99
curl http://localhost:8000/courses/curso-de-react/ratings/stats
```
