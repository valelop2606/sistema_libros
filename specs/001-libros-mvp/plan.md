# Plan técnico — 001-libros-mvp

## 1. Resumen
MVP como **aplicación de línea de comandos local en Python 3.11+**, usando solo la biblioteca
estándar en ejecución, con arquitectura en capas y persistencia en un único archivo JSON
versionado. No hay servidor, base de datos, cuentas ni dependencias de red. El diseño prioriza
la testabilidad del dominio y el cumplimiento literal de `spec.md`.

## 2. Decisiones técnicas

| ID | Decisión | Justificación | Alternativa descartada |
|---|---|---|---|
| D-01 | CLI de Python 3.11+ con `argparse`, `json`, `pathlib`, `datetime`, `uuid`, `unicodedata` | Entorno ya disponible (3.14), cero instalación, offline, fácil de probar y de ejecutar en Windows | Web local (FastAPI + React): añade servidor, build y HTTP sin aportar valor al MVP |
| D-02 | Cuatro capas: `domain`, `application`, `infrastructure`, `interfaces/cli` | Cumple el principio 3; el dominio no conoce UI ni disco y se prueba sin I/O | Monolito con lógica en los comandos: acopla reglas a la presentación |
| D-03 | Un único archivo JSON con `schema_version: 1` y agregados anidados por libro | Cientos de libros caben en memoria; escritura de archivo completo = atomicidad simple; portable y legible | SQLite: robusto pero añade SQL y migraciones innecesarias; varios JSON: rompe atomicidad global |
| D-04 | Escritura atómica: temporal en el mismo directorio + `os.replace`, con `fsync` previo | Una interrupción no corrompe el archivo anterior (RF-018) | Escritura directa: riesgo de archivo truncado |
| D-05 | IDs `uuid4`; fechas ISO 8601 en UTC con sufijo `Z` | Sin colisiones, sin coordinación, ordenables como texto | Autoincrementales: requieren contador persistente y complican merges |
| D-06 | No persistir estado ni porcentaje: se derivan de sesiones y `archived_at` | Elimina inconsistencias; una sola fuente de verdad | Campos `status`/`percentage` persistidos: se desincronizan |
| D-07 | `Book` es la raíz de agregado; sesiones, reseñas y notas se anidan en él | Carga/guardado y archivado en cascada simples; el archivo se escribe completo | Entidades separadas con referencias: más joins y validaciones cruzadas |
| D-08 | Un solo proceso y usuario; sin bloqueo de archivo | Supuesto de la spec; evita complejidad distribuida | Lock de sistema de archivos: no justificado aún |
| D-09 | Mensajes y comandos CLI en español; identificadores y módulos en inglés | Principio 6 de la constitución | Comandos en inglés: rompe la convención de idioma |
| D-10 | Pruebas con `pytest` (solo desarrollo); sin red | Estándar simple y legible; permite fakes de reloj/IDs | `unittest` puro: más verboso, sin fixtures tan claras |

## 3. Arquitectura

```
interfaces/cli   →  application  →  domain
       ↓                 ↓
        └──── infrastructure (implementa puertos)
```

- **domain**: entidades, invariantes, cálculo de porcentaje/estado, validaciones y errores de
  negocio. Sin importaciones de I/O.
- **application**: casos de uso; orquesta dominio, repositorio, reloj y generador de IDs;
  define los DTO de entrada/salida.
- **infrastructure**: repositorio JSON, escritura atómica, reloj del sistema, generador UUID.
- **interfaces/cli**: parseo de argumentos, formato de tablas, mapeo de errores a mensajes y
  códigos de salida. La única capa que imprime.

Estructura prevista (se creará en la fase de implementación):

```
src/libros/
  domain/        models.py, validation.py, progress.py, states.py, errors.py
  application/   use_cases.py, dto.py, ports.py
  infrastructure/ json_repository.py, atomic_write.py, clock.py, ids.py
  interfaces/cli/ main.py, commands.py, render.py
tests/
  unit/ integration/ contract/ fixtures/
```

Dependencias permitidas: `cli → application → domain`; `infrastructure` implementa los puertos
declarados en `application`; `domain` no importa ninguna otra capa.

## 4. Modelo de datos

### 4.1 Archivo `biblioteca.json` (ejemplo v1)

```json
{
  "schema_version": 1,
  "books": [
    {
      "id": "2f1c9a6e-7b3d-4c58-9f0a-1d2e3f4a5b6c",
      "title": "Cien años de soledad",
      "author": "Gabriel García Márquez",
      "total_pages": 471,
      "publisher": "Sudamericana",
      "year": 1967,
      "genre": "Novela",
      "created_at": "2026-10-20T14:03:00Z",
      "updated_at": "2026-10-24T21:15:00Z",
      "archived_at": null,
      "sessions": [
        {
          "id": "9a0b1c2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d",
          "started_at": "2026-10-21T08:00:00Z",
          "finished_at": "2026-10-24T21:10:00Z",
          "current_page": 471
        },
        {
          "id": "5e6f7a8b-9c0d-4e1f-8a2b-3c4d5e6f7a8b",
          "started_at": "2026-10-25T09:30:00Z",
          "finished_at": null,
          "current_page": 120
        }
      ],
      "reviews": [
        {
          "id": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
          "session_id": "9a0b1c2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d",
          "rating": 5,
          "opinion": "La estructura circular de Macondo…",
          "created_at": "2026-10-24T21:15:00Z",
          "updated_at": null
        }
      ],
      "notes": [
        {
          "id": "6b7c8d9e-0f1a-4b2c-8d3e-4f5a6b7c8d9e",
          "session_id": "5e6f7a8b-9c0d-4e1f-8a2b-3c4d5e6f7a8b",
          "page": 120,
          "text": "La repetición de nombres como forma de memoria.",
          "created_at": "2026-10-25T10:05:00Z",
          "updated_at": null
        }
      ]
    }
  ]
}
```

### 4.2 Campos e invariantes

| Entidad | Campos | Reglas |
|---|---|---|
| Libro | `id`, `title`, `author`, `total_pages`, `publisher`, `year`, `genre`, `created_at`, `updated_at`, `archived_at`, hijos | Título/autor no vacíos tras `trim`; límites de RF-002; `total_pages` 1–20 000; año 1–actual u omitido; IDs únicos en todo el archivo |
| Sesión | `id`, `started_at`, `finished_at`, `current_page` | Máximo una sesión con `finished_at = null` por libro; página 0..`total_pages` mientras está activa; una sesión completada no se modifica |
| Reseña | `id`, `session_id`, `rating`, `opinion`, `created_at`, `updated_at` | `session_id` debe ser una sesión completada del mismo libro; una reseña por sesión; `rating` entero 1–5; `opinion` ≤2 000 caracteres o `null` |
| Nota | `id`, `session_id`, `page`, `text`, `created_at`, `updated_at` | `text` no vacío ≤1 000; `session_id` opcional del mismo libro; `page` opcional 0..`total_pages`; creada con fecha/hora; editable sin perder `created_at` |
| Archivo | `schema_version`, `books` | Versión exacta `1`; cualquier otra o JSON inválido ⇒ error de carga sin sobrescritura |

Reglas derivadas (no almacenadas):
- **Estado**: `archived_at != null` ⇒ archivado; si no, sesión activa ⇒ `leyendo`; si no, alguna
  sesión completada ⇒ `leído`; si no, `pendiente`.
- **Porcentaje**: con sesión activa, `página_actual × 100 // total_páginas`, y 100 solo si
  `página_actual == total_páginas`; sin sesión activa, 100 si hay alguna completada, si no 0.
- **Página mostrada**: la de la sesión activa; si no, `total_páginas` si hay alguna completada; si no 0.
- **Lecturas completadas**: número de sesiones con `finished_at != null`; relecturas = máx(0, −1).
- **Última actividad**: máximo entre `created_at`/`updated_at` del libro, `started_at`/`finished_at`
  de sesiones y `created_at`/`updated_at` de reseñas y notas. Se calcula al consultar; no se persiste.

### 4.3 Integridad y compatibilidad
- Al cargar se valida el esquema estructural y semántico; datos inválidos se reportan como archivo
  ilegible y **no** se reescriben (RF-019).
- Las sesiones completadas conservan su página aunque luego se reduzca `total_pages`; el porcentaje
  de una sesión completada siempre se muestra como 100.
- No se migra nada en el MVP: solo se acepta `schema_version: 1`.

## 5. Contratos

### 5.1 Puertos (interfaces de aplicación)
| Puerto | Operaciones | Notas |
|---|---|---|
| `BookRepository` | `load_all`, `save_all` | Implementación JSON; `save_all` atómico; también en memoria para pruebas |
| `Clock` | `now_utc` | Reloj fijo en pruebas para timestamps deterministas |
| `IdGenerator` | `new_id` | UUID4; secuencial determinista en pruebas |

### 5.2 Casos de uso
| Caso de uso | Entrada | Salida | Errores esperados | RF |
|---|---|---|---|---|
| `add_book` | metadatos | libro creado | `VALIDATION_ERROR` | 001, 002 |
| `edit_book` | id + campos | libro actualizado | `NOT_FOUND`, `VALIDATION_ERROR`, `TOTAL_PAGES_BELOW_ACTIVE_PAGE` | 003 |
| `start_reading` | id | sesión activa | `NOT_FOUND`, `ACTIVE_SESSION_EXISTS` | 005 |
| `update_progress` | id + página | sesión + libro | `NOT_FOUND`, `NO_ACTIVE_SESSION`, `INVALID_PAGE` | 006–008 |
| `add_or_edit_review` | id, sesión opcional, rating, opinión | reseña | `NOT_FOUND`, `SESSION_NOT_COMPLETED`, `SESSION_MISMATCH`, `INVALID_RATING` | 010, 011 |
| `add_note` / `edit_note` | id, texto, página/sesión opcionales | nota | `NOT_FOUND`, `VALIDATION_ERROR`, `SESSION_MISMATCH` | 012 |
| `list_books` | búsqueda, estado, orden | lista derivada | — | 013–016 |
| `get_book_detail` | id | libro + historial derivado | `NOT_FOUND` | 020 |
| `archive_book` | id + confirmación | libro archivado | `NOT_FOUND`, `CONFIRMATION_REQUIRED` | 017 |

`NOT_FOUND` cubre IDs inexistentes y libros archivados, ya que estos no admiten más operaciones.
Cada caso de uso mutador persiste una sola vez, al finalizar con éxito.

### 5.3 Contrato CLI
Ejecutable: `python -m libros <comando> [opciones]`.

| Comando | Argumentos principales | Efecto |
|---|---|---|
| `agregar` | `--titulo --autor --paginas [--editorial --anio --genero]` | Alta manual y salida con el ID |
| `listar` | `[--buscar --estado pendiente|leyendo|leido --orden titulo|autor|actividad]` | Tabla de libros activos |
| `ver` | `<id>` | Detalle con sesiones, reseñas y notas |
| `editar` | `<id> [--titulo --autor --paginas --editorial --anio --genero]` | Edición de metadatos |
| `iniciar` | `<id>` | Nueva sesión de lectura |
| `avanzar` | `<id> --pagina N` | Actualiza progreso y puede cerrar la lectura |
| `resenar` | `<id> [--sesion ID] --estrellas 1..5 [--opinion TEXTO]` | Crea o actualiza la reseña de la sesión indicada o de la última completada sin reseña |
| `nota` | `<id> --texto TEXTO [--pagina N] [--sesion ID]` | Crea nota fechada |
| `editar-nota` | `<id-libro> <id-nota> --texto TEXTO` | Edita nota conservando su creación |
| `archivar` | `<id> --confirmar` | Archiva de forma irreversible en el MVP |

Convenciones:
- Ruta de datos: `--datos RUTA` > variable `LIBROS_ARCHIVO` > `./biblioteca.json`.
- Salida humana en español por `stdout`; errores por `stderr` con prefijo `Error:`.
- Códigos de salida: `0` éxito, `2` error de validación/conflicto/confirmación, `1` error
  inesperado (incluye archivo ilegible). No se usa salida JSON en el MVP.
- `--confirmar` es obligatorio para archivar; sin él se devuelve `CONFIRMATION_REQUIRED`.

### 5.4 Contrato de errores
Todo error de negocio expone `code`, `message` (español) y `field` opcional. El mapeo a la CLI es
1:1; los tests verifican código y mensaje. `NOT_FOUND` no distingue inexistente de archivado, para
no exponer datos archivados.

## 6. Flujos clave

**Avance y cierre (RF-006–009).** `avanzar` carga el libro activo, exige sesión activa, valida la
página contra el total, actualiza la sesión y, si la página iguala el total, fija `finished_at`,
deja la sesión pendiente de reseña y persiste. Cualquier fallo ocurre antes de persistir.

**Reseña (RF-010–011).** Sin `--sesion`, se elige la sesión completada más reciente sin reseña; si
no existe, error. Se valida pertenencia, estado completado y, si ya había reseña, se actualiza
conservando `created_at` y fijando `updated_at`; si no, se crea. La operación persiste una vez.

**Relectura (RF-005, RF-009).** Sobre un libro `leído`, `iniciar` crea otra sesión con página 0;
las anteriores permanecen visibles en el detalle y el contador se deriva.

**Archivado (RF-017).** Con confirmación, se fija `archived_at`; no se borran hijos, se excluye de
todas las consultas y cualquier operación posterior responde `NOT_FOUND`.

## 7. Estrategia de pruebas

| Nivel | Alcance | Herramientas |
|---|---|---|
| Unitario de dominio | Reglas puras: validaciones, porcentaje (0, 1, 124/471, 470/471, 471/471), precedencia de estado, relecturas, ventanas de reseña, notas, archivado | `pytest`, sin I/O |
| Unitario de aplicación | Casos de uso con repositorio en memoria, reloj fijo e IDs secuenciales; errores y persistencia única | `pytest` |
| Integración de persistencia | Round-trip exacto, archivo inexistente, JSON corrupto, versión desconocida, datos semánticamente inválidos, fallo de escritura que preserva el archivo anterior | `pytest`, `tmp_path` |
| Contrato CLI | Comandos, argumentos obligatorios, códigos de salida, mensajes en español, búsqueda sin acentos, archivados ocultos | `pytest` invocando el ejecutable con archivo temporal |

Criterios:
- Todo RF tiene al menos una prueba automatizada trazable por nombre o marcador.
- Cobertura mínima de líneas en `domain` y `application`: 90 %; ramas del cálculo de porcentaje y
  de la derivación de estado: 100 %.
- Escenarios de aceptación de US-01 a US-07 ejecutados como pruebas de contrato o integración.
- Sin red, sin dependencias de servicios y con fechas/IDs deterministas.

Trazabilidad RF → pruebas: `RF-001..004 → tests/unit/test_catalog.py`, `RF-005..009 →
tests/unit/test_reading.py`, `RF-010..011 → tests/unit/test_reviews.py`, `RF-012 →
tests/unit/test_notes.py`, `RF-013..016 → tests/contract/test_queries.py`, `RF-017 →
tests/unit/test_archive.py`, `RF-018..019 → tests/integration/test_persistence.py`, `RF-020 →
tests/contract/test_detail.py`.

## 8. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Crecimiento del archivo único | Carga completa en memoria; aceptable para cientos de libros. Revisar SQLite si supera ~5 000 libros |
| Datos dentro de una carpeta sincronizada (OneDrive) | Documentar que el archivo puede entrar en conflicto de sincronización; permitir `--datos` fuera de la carpeta |
| Archivado irreversible por error | Confirmación obligatoria y mensaje explícito; posible restauración en v2 |
| Edición de `total_pages` sobre sesiones completadas | Se conservan páginas y el 100 %; se documenta la excepción de invariante |
| Sin bloqueo entre procesos | Supuesto mono-proceso; si se abre dos veces, la última escritura prevalece (se documenta) |
| Zona horaria | Todo en UTC; se muestra el valor ISO almacenado |

## 9. Supuestos abiertos
- Se adopta CLI; una interfaz web local queda como evolución futura sin cambiar dominio ni contratos.
- No hay migraciones ni compatibilidad con versiones futuras del esquema.
- No se implementa restauración de archivados, exportación adicional ni estadísticas (spec §6).
