# Tareas — 001-libros-mvp

Desglose para la futura implementación. Las tareas están ordenadas por dependencia y no contienen
código. Una tarea solo se marca `[x]` cuando su condición "Hecho cuando" se cumple y las pruebas
indicadas pasan. Referencias: `RF-x` de `spec.md`, decisiones `D-x` de `plan.md`.

## Fase 0 — Preparación

- [ ] **T-001** Crear la estructura de paquetes (`src/libros/...`, `tests/...`) y la configuración
  de proyecto con Python 3.11+ sin dependencias de ejecución, `pytest` y `pytest-cov` como
  dependencias de desarrollo, y `pythonpath` apuntando a `src`.
  *Hecho cuando:* `pytest` colecta sin errores, `import libros` funciona y no hay dependencias de
  ejecución externas.

- [ ] **T-002** Implementar fixtures compartidas: reloj fijo, generador de IDs secuencial,
  repositorio en memoria y archivo de datos temporal por prueba.
  *Hecho cuando:* existe una prueba smoke que usa las cuatro fixtures y pasa de forma determinista.

## Fase 1 — Dominio (pruebas primero)

- [ ] **T-010** Definir los errores de negocio con `code`, `message` en español y `field`
  opcional (`validation`, `not_found`, `no_active_session`, `active_session_exists`,
  `invalid_page`, `session_not_completed`, `session_mismatch`, `invalid_rating`,
  `confirmation_required`).
  *Hecho cuando:* cada código tiene prueba de mensaje y campo, y no hay mensajes en inglés.

- [ ] **T-011** Modelar `Book`, `ReadingSession`, `Review` y `Note` como estructuras puras con
  sus campos y tipos (D-07), sin lógica de I/O.
  *Hecho cuando:* se construyen objetos válidos y se rechazan campos obligatorios ausentes.

- [ ] **T-012** Validar metadatos del libro: `trim`, título ≤200, autor ≤120, editorial ≤120,
  género ≤60, `total_pages` entero 1–20 000, año 1–actual u omitido, opcionales vacíos permitidos
  y duplicados título/autor permitidos (**RF-001, RF-002**).
  *Hecho cuando:* las pruebas de límites (200/201, 120/121, 60/61, 0/1/20 000/20 001, año futuro)
  y de cadenas de solo espacios pasan.

- [ ] **T-013** Implementar el porcentaje con truncamiento entero (`página × 100 // total`, 100
  solo al igualar el total) y la página mostrada según haya sesión activa, lecturas completadas
  o ninguna (**RF-007**).
  *Hecho cuando:* pasan los casos 0 %, 1 de 1, 90 de 300, 299 de 300, 300 de 300 y los tres
  escenarios sin sesión activa.

- [ ] **T-014** Derivar el estado (archivado > leyendo > leído > pendiente), contar lecturas
  completadas y calcular relecturas como completadas − 1, sin persistir estos valores
  (**RF-004, RF-009**).
  *Hecho cuando:* pasan todas las combinaciones de precedencia y los contadores con 0, 1 y 3
  sesiones completadas, incluyendo sesiones no completadas.

- [ ] **T-015** Implementar las reglas de sesión: iniciar con página 0, impedir una segunda
  sesión activa, validar página 0..total (rechazando decimales/negativos/excedidos), aceptar
  retroceso, cerrar automáticamente al igualar el total y prohibir cambios en sesiones
  completadas (**RF-005, RF-006, RF-008**).
  *Hecho cuando:* las pruebas cubren inicio en `pendiente` y en `leído`, doble sesión activa,
  página inválida, retroceso, página final y modificación de sesión completada.

- [ ] **T-016** Implementar la edición de metadatos, incluyendo el rechazo si `total_pages` queda
  por debajo de la página de la sesión activa y la conservación de páginas de sesiones
  completadas (**RF-003**).
  *Hecho cuando:* pasan las pruebas de edición válida, reducción conflictiva y reducción con
  sesiones ya completadas.

- [ ] **T-017** Implementar reseñas: una por sesión completada, calificación entera 1–5
  obligatoria, opinión ≤2 000 opcional, `created_at`/`updated_at` correctos, actualización sin
  duplicar y señal derivada de "pendiente de reseña" (**RF-010, RF-011**).
  *Hecho cuando:* pasan las pruebas de alta, edición, rating inválido, sesión activa, sesión de
  otro libro y detección de pendientes.

- [ ] **T-018** Implementar notas: texto 1–1 000 caracteres, página 0..total opcional, sesión
  opcional del mismo libro, fecha de creación automática, edición conservando `created_at` y
  orden de más reciente a más antigua (**RF-012**).
  *Hecho cuando:* pasan las pruebas de nota con y sin página/sesión, texto vacío, sesión ajena,
  página fuera de rango y edición con timestamps correctos.

- [ ] **T-019** Implementar la semántica de archivado: fijar `archived_at`, ocultar de consultas,
  conservar el agregado completo y bloquear operaciones posteriores (**RF-017**).
  *Hecho cuando:* pasan las pruebas de archivado con y sin sesión activa, conservación de hijos y
  rechazo de cualquier mutación posterior.

- [ ] **T-020** Calcular la última actividad como máximo de fechas del libro, sesiones, reseñas y
  notas, sin persistirla (**RF-016**).
  *Hecho cuando:* la prueba con actividad solo en una nota, solo en una sesión y solo en el libro
  devuelve el máximo correcto.

## Fase 2 — Persistencia

- [ ] **T-021** Implementar la escritura atómica: temporal en el mismo directorio, `flush` +
  `fsync`, `os.replace` y limpieza del temporal ante error (**RF-018**, D-04).
  *Hecho cuando:* una falla simulada antes de `replace` deja intacto el archivo anterior y no
  quedan temporales huérfanos.

- [ ] **T-022** Implementar serialización/deserialización del esquema v1 con round-trip exacto de
  libros, sesiones, reseñas y notas (**RF-018**, D-03).
  *Hecho cuando:* guardar y volver a cargar un agregado completo produce estructuras equivalentes
  y fechas/IDs idénticos.

- [ ] **T-023** Validar el archivo al cargar: estructura, tipos, `schema_version: 1`, IDs únicos,
  una sola sesión activa por libro y referencias de reseña/nota coherentes (**RF-019**).
  *Hecho cuando:* pasan las pruebas con JSON corrupto, versión desconocida y cada violación
  semántica, todas reportadas sin reescribir el archivo.

- [ ] **T-024** Cubrir arranque sin archivo (biblioteca vacía) y error de lectura que no sobrescribe
  el archivo original (**RF-019**).
  *Hecho cuando:* la primera ejecución crea datos vacíos en memoria y persiste solo al mutar, y el
  archivo ilegible permanece byte a byte igual tras el error.

- [ ] **T-025** Implementar `JsonBookRepository` según el puerto, guardando todo el archivo una vez
  por mutación exitosa y manteniendo archivados en disco.
  *Hecho cuando:* las pruebas verifican una sola escritura por caso de uso exitoso, ninguna en
  error y presencia de libros archivados tras recargar.

## Fase 3 — Aplicación

- [ ] **T-030** Declarar los puertos `BookRepository`, `Clock` e `IdGenerator` y los DTO de
  entrada/salida de los casos de uso (D-02, D-05).
  *Hecho cuando:* el dominio y los casos de uso no importan `infrastructure` ni `interfaces`, y
  hay una prueba de arquitectura que lo verifica.

- [ ] **T-031** Implementar `add_book` y `edit_book` con validación de dominio y persistencia única
  (**RF-001–RF-003**).
  *Hecho cuando:* pasan las pruebas de alta válida/inválida, edición válida y conflicto de páginas.

- [ ] **T-032** Implementar `start_reading` y `update_progress`, incluyendo cierre automático y
  sesión pendiente de reseña (**RF-005–RF-009**).
  *Hecho cuando:* pasan las pruebas de inicio, conflicto de sesión activa, avance, rechazo sin
  sesión activa y finalización en la última página.

- [ ] **T-033** Implementar `add_or_edit_review` con selección por `--sesion` o por última sesión
  completada sin reseña, y rechazo si no hay candidata (**RF-010, RF-011**).
  *Hecho cuando:* pasan las pruebas de alta, actualización, selección por defecto y ausencia de
  sesión pendiente.

- [ ] **T-034** Implementar `add_note` y `edit_note` con validación de pertenencia y timestamps
  (**RF-012**).
  *Hecho cuando:* pasan las pruebas de creación fuera/dentro de sesión, edición y errores de
  validación.

- [ ] **T-035** Implementar `list_books` con búsqueda parcial en título/autor sin mayúsculas ni
  acentos, filtro por estado y orden por título, autor o última actividad (**RF-013–RF-016**).
  *Hecho cuando:* pasan los casos `garcia`/`García`, filtros combinados con búsqueda, los tres
  órdenes y la lista vacía.

- [ ] **T-036** Implementar `get_book_detail` con sesiones, reseñas y notas en el orden definido y
  exclusión de archivados (**RF-020**).
  *Hecho cuando:* el detalle devuelve todos los hijos ordenados y responde `NOT_FOUND` si el libro
  está archivado o no existe.

- [ ] **T-037** Implementar `archive_book` con confirmación explícita y bloqueo posterior
  (**RF-017**).
  *Hecho cuando:* sin confirmación devuelve `CONFIRMATION_REQUIRED`, con confirmación archiva y
  toda operación posterior falla con `NOT_FOUND`.

- [ ] **T-038** Garantizar que ningún caso de uso mutador persiste ante error y que todos mapean a
  los códigos de error del contrato (**RF-018**).
  *Hecho cuando:* una prueba por caso de uso verifica cero escrituras ante fallo y el código
  esperado.

## Fase 4 — Interfaz CLI

- [ ] **T-040** Crear el punto de entrada `python -m libros`, el parser de comandos y la resolución
  de ruta (`--datos` > `LIBROS_ARCHIVO` > `./biblioteca.json`) con códigos de salida 0/2/1
  (D-01, D-09).
  *Hecho cuando:* `python -m libros --help` lista los diez comandos y las pruebas verifican la
  precedencia de ruta y los códigos de ayuda/error.

- [ ] **T-041** Implementar los comandos `agregar`, `listar`, `ver` y `editar` con sus opciones
  (**RF-001–RF-003, RF-013–RF-016, RF-020**).
  *Hecho cuando:* cada comando produce la salida esperada y sus errores de validación son legibles.

- [ ] **T-042** Implementar `iniciar` y `avanzar`, mostrando página, porcentaje y fin de lectura
  (**RF-005–RF-009**).
  *Hecho cuando:* el flujo completo por CLI desde inicio hasta 100 % deja la sesión completada y
  el libro `leído`.

- [ ] **T-043** Implementar `resenar`, `nota` y `editar-nota` con validaciones y timestamps
  (**RF-010–RF-012**).
  *Hecho cuando:* las pruebas de contrato cubren alta/edición de reseña, nota con y sin página y
  edición de nota.

- [ ] **T-044** Implementar `archivar` con `--confirmar` obligatorio y mensaje explícito de
  irreversibilidad (**RF-017**).
  *Hecho cuando:* sin la bandera no hay cambios; con ella desaparece de `listar` y `ver` falla.

- [ ] **T-045** Implementar la presentación: tabla de libros, detalle con historial, mensajes de
  lista vacía y formato de fechas/porcentajes en español (D-09).
  *Hecho cuando:* las pruebas de contrato validan encabezados, columnas, mensajes y que no se
  expone ningún libro archivado.

- [ ] **T-046** Mapear errores a `stderr` con prefijo `Error:` y código de salida 2 para
  validación/conflicto/confirmación, y 1 para fallo inesperado o archivo ilegible.
  *Hecho cuando:* una prueba por código confirma canal, prefijo y exit code.

- [ ] **T-047** Escribir las pruebas de contrato CLI: búsqueda sin acentos, filtros, órdenes,
  campos obligatorios, IDs inexistentes, doble sesión activa y archivo ilegible.
  *Hecho cuando:* toda la suite de contrato pasa contra un archivo temporal, sin tocar datos del
  usuario.

## Fase 5 — Validación y cierre

- [ ] **T-050** Ejecutar los escenarios de aceptación de **US-01–US-03** de extremo a extremo
  (alta, listado, avance y porcentaje).
  *Hecho cuando:* los tres escenarios pasan automatizados y sus resultados coinciden con la spec.

- [ ] **T-051** Ejecutar los escenarios de aceptación de **US-04–US-05** (reseña al terminar y
  notas fechadas).
  *Hecho cuando:* ambos escenarios pasan y las fechas/horas son deterministas con el reloj fijo.

- [ ] **T-052** Ejecutar los escenarios de aceptación de **US-06–US-07** (búsqueda/filtro/orden y
  archivado con conservación de historial).
  *Hecho cuando:* ambos escenarios pasan y el archivo conserva los datos archivados verificables.

- [ ] **T-053** Verificar cobertura: ≥90 % de líneas en `domain` y `application`, 100 % de ramas en
  porcentaje y derivación de estado.
  *Hecho cuando:* el reporte de cobertura cumple los umbrales y la configuración falla si bajan.

- [ ] **T-054** Auditar la trazabilidad RF → prueba y confirmar que los 20 RF tienen al menos una
  prueba automatizada según la matriz del plan.
  *Hecho cuando:* la matriz está completa y no hay RF sin prueba ni pruebas sin RF.

- [ ] **T-055** Probar manualmente en Windows una ruta con espacios y acentos, un archivo en
  carpeta sincronizada y el arranque sin archivo previo.
  *Hecho cuando:* el flujo básico funciona en los tres casos y las limitaciones quedan documentadas.

- [ ] **T-056** Revisar la consistencia final entre `constitution.md`, `spec.md`, `plan.md` y este
  archivo; registrar cualquier desviación como cambio de spec antes de cerrar.
  *Hecho cuando:* no existen requisitos sin decisión ni decisiones que contradigan la constitución.
