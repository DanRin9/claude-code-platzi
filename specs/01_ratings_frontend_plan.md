# Plan de Implementación: Sistema de Ratings — Frontend

## Contexto

Se va a construir el sistema de ratings de cursos en el frontend de Platziflix. El objetivo es permitir que los usuarios califiquen cursos con una puntuación del 1 al 5, y que todos los usuarios puedan ver el rating promedio de cada curso.

El sistema se apoya en tres endpoints nuevos del backend y en la extensión de los endpoints existentes (`GET /courses` y `GET /courses/{slug}`) que ahora incluirán `avg_rating` y `rating_count` en su respuesta.

**Componentes a crear:**
- `StarRating` — visualización readonly de estrellas (muestra promedio)
- `RatingInput` — componente interactivo para que el usuario califique

**Decisiones clave:**
- El `user_id` se genera una vez y se persiste en `localStorage` como UUID
- No se usa ninguna librería externa; las estrellas se implementan con Unicode (★) o SVG custom
- La arquitectura sigue el patrón existente: servicio de API → hook → componente

---

## PASO 1: Extender tipos TypeScript en `src/types/index.ts`

### Modificar interfaces existentes

Agregar `avg_rating` y `rating_count` a `Course` y `CourseDetail`, ya que el backend ahora los incluye en la respuesta:

```typescript
// Modificar Course (campos adicionales opcionales para compatibilidad con datos sin rating)
interface Course {
  id: number
  title: string
  teacher: string
  duration: number
  thumbnail: string
  slug: string
  avg_rating?: number      // NUEVO — promedio de 1 a 5, puede ser null si sin ratings
  rating_count?: number    // NUEVO — cantidad total de ratings
}
```

`CourseDetail` hereda de `Course`, por lo que recibe los campos automáticamente.

### Agregar nuevas interfaces

```typescript
// Distribución de ratings (cuántas estrellas recibió cada puntuación)
interface RatingDistribution {
  1: number
  2: number
  3: number
  4: number
  5: number
}

// Respuesta del endpoint GET /courses/{slug}/ratings/stats
interface RatingStats {
  avg_rating: number
  rating_count: number
  distribution: RatingDistribution
}

// Body del endpoint POST /courses/{slug}/rate
interface RatingPayload {
  user_id: string     // UUID del localStorage
  score: number       // 1 a 5
  comment?: string    // opcional
}

// Respuesta de POST /courses/{slug}/rate (ajustar según lo que devuelva el backend)
interface RatingResponse {
  id: number
  course_id: number
  user_id: string
  score: number
  comment?: string
  created_at: string
}
```

---

## PASO 2: Crear servicio de API en `src/services/ratingsApi.ts`

Crear el archivo `src/services/ratingsApi.ts` con las funciones que encapsulan las llamadas HTTP a los endpoints de ratings. Este archivo sigue el mismo patrón que cualquier fetch directo que se use en las páginas, pero centralizado para reutilización.

### Constante base

```typescript
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:8000'
```

### Funciones a implementar

**`getRatingStats(slug: string): Promise<RatingStats>`**
- Llama a `GET /courses/{slug}/ratings/stats`
- Devuelve la interfaz `RatingStats`
- Lanza error si la respuesta no es `ok`

**`submitRating(slug: string, payload: RatingPayload): Promise<RatingResponse>`**
- Llama a `POST /courses/{slug}/rate`
- Headers: `Content-Type: application/json`
- Body: `JSON.stringify(payload)`
- Devuelve la respuesta del backend tipada como `RatingResponse`
- Lanza error con mensaje descriptivo si falla (para mostrarlo en el componente)

**`deleteRating(slug: string, userId: string): Promise<void>`**
- Llama a `DELETE /courses/{slug}/rate/{userId}`
- No devuelve body relevante
- Lanza error si la respuesta no es `ok`

### Esquema del archivo

```typescript
// src/services/ratingsApi.ts
import type { RatingStats, RatingPayload, RatingResponse } from '@/types'

const API_BASE = ...

export async function getRatingStats(slug: string): Promise<RatingStats> { ... }
export async function submitRating(slug: string, payload: RatingPayload): Promise<RatingResponse> { ... }
export async function deleteRating(slug: string, userId: string): Promise<void> { ... }
```

---

## PASO 3: Crear hook `useRating` en `src/hooks/useRating.ts`

El hook centraliza toda la lógica de estado relacionada con el rating de un curso: gestión del `user_id`, carga de stats, envío de rating y eliminación.

### Responsabilidades del hook

1. **Obtener o generar `user_id`**: al montarse, leer `localStorage.getItem('platziflix_user_id')`. Si no existe, generar un UUID con `crypto.randomUUID()` y guardarlo.
2. **Estado de las estadísticas**: mantener `stats: RatingStats | null`, `loadingStats: boolean`, `errorStats: string | null`.
3. **Estado del envío**: mantener `submitting: boolean`, `submitError: string | null`, `submitSuccess: boolean`.
4. **Métodos expuestos**: `submitRating(score, comment?)` y `deleteRating()`.

### Firma del hook

```typescript
// src/hooks/useRating.ts
interface UseRatingReturn {
  userId: string | null
  stats: RatingStats | null
  loadingStats: boolean
  errorStats: string | null
  submitting: boolean
  submitError: string | null
  submitSuccess: boolean
  handleSubmitRating: (score: number, comment?: string) => Promise<void>
  handleDeleteRating: () => Promise<void>
  refreshStats: () => Promise<void>
}

export function useRating(courseSlug: string): UseRatingReturn { ... }
```

### Flujo interno

```
montaje del hook
  → leer/generar user_id en localStorage
  → llamar getRatingStats(courseSlug) para cargar stats iniciales

handleSubmitRating(score, comment?)
  → setSubmitting(true)
  → llamar submitRating(courseSlug, { user_id, score, comment })
  → en éxito: setSubmitSuccess(true), llamar refreshStats()
  → en error: setSubmitError(mensaje)
  → setSubmitting(false)

handleDeleteRating()
  → llamar deleteRating(courseSlug, userId)
  → en éxito: llamar refreshStats()
```

**Nota:** El hook solo se usa en componentes cliente (`'use client'`). En server components (`page.tsx`) el fetch de stats se hace directamente.

---

## PASO 4: Crear componente `StarRating` (readonly)

Archivo: `src/components/StarRating/StarRating.tsx` + `StarRating.module.scss`

Este componente es puramente de visualización. Muestra estrellas rellenas, parcialmente rellenas y vacías según el valor de `rating`.

### Props

```typescript
interface StarRatingProps {
  rating: number        // valor de 0 a 5 (puede ser decimal: 3.7)
  maxStars?: number     // default: 5
  size?: 'sm' | 'md' | 'lg'   // para controlar tamaño en diferentes contextos
  showValue?: boolean   // si mostrar "3.7" en texto junto a las estrellas
}
```

### Lógica de rendering

```
Para cada estrella de 1 a maxStars:
  - si rating >= i → estrella llena (★)
  - si rating >= i - 0.5 y rating < i → media estrella (con CSS clip-path o char alternativo)
  - si rating < i - 0.5 → estrella vacía (☆)
```

Alternativa más simple: solo estrellas llenas y vacías (sin medias). Usar `Math.round(rating)` para determinar cuántas llenas mostrar. Decidir en implementación.

### Estilos sugeridos

```scss
// StarRating.module.scss
.container { display: flex; align-items: center; gap: 2px; }
.star { color: var(--color-primary); /* #ff2d2d o gold */ font-size: 1rem; }
.star--empty { color: var(--color-light-gray); }
.value { font-size: 0.85rem; color: var(--text-secondary); margin-left: 4px; }
```

El color de las estrellas rellenas puede ser gold (`#ffd700`) en lugar del rojo primario de Platzi, para mantener la convención visual de ratings. Decidir en implementación.

### JSX esquemático

```tsx
<div className={styles.container} aria-label={`Rating: ${rating} de ${maxStars}`}>
  {stars.map((type, i) => (
    <span key={i} className={type === 'full' ? styles.star : styles.starEmpty}>
      {type === 'full' ? '★' : '☆'}
    </span>
  ))}
  {showValue && <span className={styles.value}>{rating.toFixed(1)}</span>}
</div>
```

**Accesibilidad:** El `div` contenedor debe tener `role="img"` y `aria-label` descriptivo.

---

## PASO 5: Crear componente `RatingInput` (interactivo)

Archivo: `src/components/RatingInput/RatingInput.tsx` + `RatingInput.module.scss`

Componente cliente (`'use client'`) que permite al usuario seleccionar una puntuación y enviarla.

### Props

```typescript
interface RatingInputProps {
  courseSlug: string
  onRated?: (newStats: RatingStats) => void   // callback tras envío exitoso
  currentUserRating?: number                  // si el usuario ya tiene un rating previo
}
```

### Estado interno del componente

```typescript
const [hoverRating, setHoverRating] = useState<number>(0)   // estrella sobre la que está el mouse
const [selectedRating, setSelectedRating] = useState<number>(currentUserRating ?? 0)
const [comment, setComment] = useState<string>('')
const { userId, submitting, submitError, submitSuccess, handleSubmitRating, handleDeleteRating } = useRating(courseSlug)
```

### Lógica de interacción

```
hover sobre estrella i → setHoverRating(i)
mouse sale del área → setHoverRating(0)
click en estrella i → setSelectedRating(i)
click "Enviar" → handleSubmitRating(selectedRating, comment)
click "Eliminar mi rating" → handleDeleteRating()
```

### JSX esquemático

```tsx
<div className={styles.container}>
  <p className={styles.label}>Califica este curso:</p>
  <div className={styles.stars} onMouseLeave={() => setHoverRating(0)}>
    {[1, 2, 3, 4, 5].map(star => (
      <button
        key={star}
        className={star <= (hoverRating || selectedRating) ? styles.starActive : styles.star}
        onMouseEnter={() => setHoverRating(star)}
        onClick={() => setSelectedRating(star)}
        aria-label={`Calificar con ${star} estrella${star > 1 ? 's' : ''}`}
      >
        ★
      </button>
    ))}
  </div>
  <textarea
    placeholder="Comentario opcional..."
    value={comment}
    onChange={e => setComment(e.target.value)}
  />
  <button onClick={handleSubmit} disabled={selectedRating === 0 || submitting}>
    {submitting ? 'Enviando...' : 'Enviar calificación'}
  </button>
  {submitError && <p className={styles.error}>{submitError}</p>}
  {submitSuccess && <p className={styles.success}>¡Gracias por tu calificación!</p>}
</div>
```

**Accesibilidad:** Los botones de estrella deben ser elementos `<button>` reales (no `<span>`), para navegación por teclado.

---

## PASO 6: Integrar `StarRating` en el componente `Course` (tarjeta)

Archivo a modificar: `src/components/Course/Course.tsx`

### Cambio en props

```typescript
// Antes
interface CourseProps {
  id: number
  title: string
  teacher: string
  duration: number
  thumbnail: string
  slug: string
}

// Después (agregar campos opcionales)
interface CourseProps {
  // ...campos existentes...
  avg_rating?: number
  rating_count?: number
}
```

### Cambio en JSX

Agregar `StarRating` debajo del título o encima de la duración, dentro de la tarjeta:

```tsx
// Dentro del JSX de Course.tsx, en la sección de metadatos
{avg_rating !== undefined && avg_rating > 0 && (
  <StarRating
    rating={avg_rating}
    showValue
    size="sm"
  />
)}
{rating_count !== undefined && rating_count > 0 && (
  <span className={styles.ratingCount}>({rating_count} calificaciones)</span>
)}
```

### Test a actualizar

En `src/components/Course/__test__/Course.test.tsx` agregar casos de prueba:
- Que `StarRating` se renderiza cuando se pasa `avg_rating`
- Que NO se renderiza cuando `avg_rating` es 0 o undefined

---

## PASO 7: Integrar ratings en `CourseDetail`

### 7a. Fetch server-side en `src/app/course/[slug]/page.tsx`

La página ya hace fetch del detalle del curso (que ahora incluye `avg_rating` y `rating_count`). Adicionalmente, hacer fetch de las stats completas para mostrar la distribución:

```typescript
// En page.tsx (server component)
const [courseData, ratingStats] = await Promise.all([
  fetch(`${API_BASE}/courses/${slug}`).then(r => r.json()),
  fetch(`${API_BASE}/courses/${slug}/ratings/stats`).then(r => r.json()).catch(() => null)
])
```

Pasar `ratingStats` como prop al componente `CourseDetail`.

### 7b. Modificar `CourseDetail.tsx`

Agregar una sección de "Estadísticas y calificación" entre el header del curso y la lista de clases:

```tsx
// Sección nueva en CourseDetail.tsx
<section className={styles.ratingsSection}>
  <h3>Calificación del curso</h3>
  <div className={styles.statsRow}>
    <StarRating rating={course.avg_rating ?? 0} size="lg" showValue />
    <span className={styles.ratingCount}>
      {course.rating_count ?? 0} calificaciones
    </span>
  </div>
  {/* Distribución opcional: barra por cada estrella */}
  <RatingInput courseSlug={course.slug} />
</section>
```

**Nota:** `RatingInput` es un componente cliente. Si `CourseDetail` es server component, se debe importar con `dynamic` o extraerlo a un subcomponente cliente propio.

### Props adicionales para `CourseDetail`

```typescript
interface CourseDetailProps {
  course: CourseDetail   // ya existente
  ratingStats?: RatingStats   // NUEVO — puede ser null si el fetch falla
}
```

---

## PASO 8: Tests

### `StarRating.test.tsx` — `src/components/StarRating/StarRating.test.tsx`

Casos a cubrir:

1. **Renderiza correctamente con rating 0** — todas las estrellas vacías
2. **Renderiza correctamente con rating 5** — todas las estrellas llenas
3. **Renderiza correctamente con rating 3** — 3 llenas, 2 vacías
4. **Muestra el valor numérico cuando `showValue={true}`** — texto "3.0" visible
5. **No muestra el valor numérico cuando `showValue={false}` (default)**
6. **`aria-label` descriptivo presente** — para accesibilidad
7. **Respeta `maxStars` personalizado** — si `maxStars={3}`, solo 3 estrellas

### `RatingInput.test.tsx` — `src/components/RatingInput/RatingInput.test.tsx`

Casos a cubrir:

1. **Renderiza 5 botones de estrella**
2. **Click en estrella selecciona rating** — el botón queda "activo"
3. **Botón "Enviar" deshabilitado si no hay rating seleccionado**
4. **Botón "Enviar" habilitado tras seleccionar rating**
5. **Mockear `useRating` y verificar que `handleSubmitRating` se llama con args correctos al hacer click en "Enviar"**
6. **Muestra mensaje de error cuando `submitError` está presente** (mock del hook)
7. **Muestra mensaje de éxito cuando `submitSuccess` es true** (mock del hook)
8. **Hover sobre estrella cambia estilo visual** (verificar clase CSS activa)

### `useRating.test.ts` — `src/hooks/useRating.test.ts`

Casos a cubrir:

1. **Genera y persiste `user_id` en localStorage si no existe**
2. **Reutiliza `user_id` existente de localStorage**
3. **Llama a `getRatingStats` al montarse**
4. **`handleSubmitRating` llama a `submitRating` con args correctos**
5. **`handleSubmitRating` actualiza `submitSuccess` en éxito**
6. **`handleSubmitRating` actualiza `submitError` en falla**

**Patrón de mock para tests con fetch:**

```typescript
vi.mock('@/services/ratingsApi', () => ({
  getRatingStats: vi.fn().mockResolvedValue({ avg_rating: 4.2, rating_count: 10, distribution: {...} }),
  submitRating: vi.fn().mockResolvedValue({ id: 1, score: 5 }),
  deleteRating: vi.fn().mockResolvedValue(undefined),
}))
```

---

## Verificación

Una vez implementado todo, verificar en el browser:

1. **Home (`/`)**: las tarjetas de cursos muestran estrellas si el backend devuelve `avg_rating > 0`
2. **Detalle (`/course/[slug]`)**: la sección de ratings muestra el promedio actual con `StarRating` y el formulario de `RatingInput`
3. **Flujo completo de rating**: seleccionar estrellas → clic en "Enviar" → mensaje de éxito → las stats se actualizan (nuevo promedio visible sin recargar página)
4. **Eliminar rating**: el botón "Eliminar mi rating" elimina la calificación y actualiza el promedio
5. **Persistencia de user_id**: abrir DevTools → Application → Local Storage → verificar que `platziflix_user_id` existe y no cambia al recargar
6. **Responsive**: probar en mobile (375px) y desktop (1280px) — `RatingInput` y `StarRating` deben ser usables en ambos

### Comandos para verificar

```bash
cd Frontend
yarn dev          # levantar en http://localhost:3000
yarn test         # correr toda la suite de tests
yarn type-check   # verificar que no hay errores de TypeScript
yarn build        # verificar que el build de producción pasa sin errores
```
