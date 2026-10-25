# Especificación — 001-libros-mvp

## 1. Contexto
Aplicación local, mono-usuario, para gestionar una biblioteca personal de **libros físicos**.
Cubre el alta manual del catálogo, el seguimiento de la lectura en curso con porcentaje
automático, el historial de relecturas, una reseña por cada lectura terminada y notas
fechadas. No requiere cuentas, red ni servicios externos.

## 2. Usuarios
- **Dueña/o de la biblioteca**: único usuario de la instalación. Quiere saber qué libros
  posee, cuánto lleva leído de cada uno, cuántas veces lo ha leído, qué opinó al terminarlo
  y qué ideas anotó durante o después de la lectura.
- Supuestos: uso offline, un solo proceso a la vez, cientos de libros, datos en el equipo.

## 3. Historias de usuario
- **US-01 (P1) — Registrar un libro.** Como dueña/o quiero dar de alta un libro físico con
  título, autor y total de páginas para saber que lo poseo.
  *Prueba independiente:* alta válida → aparece en la biblioteca en estado `pendiente` con 0 %.
- **US-02 (P1) — Consultar la biblioteca.** Como dueña/o quiero ver todos mis libros con
  estado, página actual, porcentaje y veces leído.
  *Prueba independiente:* con 3 libros en estados distintos, la lista muestra cada estado y avance.
- **US-03 (P1) — Actualizar el avance.** Como dueña/o quiero indicar la página en la que voy
  para que el porcentaje se calcule solo.
  *Prueba independiente:* página 90 de 300 → 30 %; página 300 → 100 % y libro `leído`.
- **US-04 (P2) — Reseñar al terminar.** Como dueña/o quiero calificar con 1–5 estrellas y
  escribir una opinión larga al terminar una lectura.
  *Prueba independiente:* lectura completada → reseña guardada con fecha y hora.
- **US-05 (P2) — Anotar ideas.** Como dueña/o quiero agregar notas pequeñas en cualquier
  momento (durante la sesión, fuera de ella o después de terminar) con fecha y hora.
  *Prueba independiente:* dos notas en momentos distintos → ambas visibles y ordenadas por fecha.
- **US-06 (P3) — Reencontrar libros.** Como dueña/o quiero buscar por título o autor, filtrar
  por estado y ordenar la lista.
  *Prueba independiente:* búsqueda "garcia marquez" encuentra "García Márquez" sin distinguir acentos.
- **US-07 (P3) — Conservar el historial.** Como dueña/o quiero archivar libros que ya no
  quiero ver sin perder sus lecturas, reseñas y notas.

## 4. Requisitos funcionales (EARS)

### 4.1 Catálogo
**RF-001 — Alta manual.** El sistema deberá permitir registrar un libro con título, autor y
total de páginas obligatorios, y editorial, año y género opcionales.
- **Cuando** se confirme un alta con los obligatorios no vacíos y válidos, **el sistema deberá**
  guardarlo en estado `pendiente` con fecha de alta.
- **Si** falta título, autor o total de páginas, **entonces** el sistema deberá rechazar el alta
  e indicar los campos faltantes.

**RF-002 — Validación de metadatos.** El sistema deberá validar longitudes y rangos.
- **Si** el título supera 200 caracteres, el autor 120, el género 60 o la editorial 120,
  **entonces** el sistema deberá rechazar el alta o la edición indicando el límite.
- **Si** el total de páginas no es un entero entre 1 y 20 000, **entonces** el sistema deberá
  rechazarlo.
- **Si** el año no es un entero entre 1 y el año actual, **entonces** el sistema deberá rechazarlo.
- Los libros con mismo título y autor deberán poder registrarse por separado.

**RF-003 — Edición.** El sistema deberá permitir editar los metadatos de un libro activo.
- **Cuando** se modifiquen campos válidos, **el sistema deberá** guardar los cambios y actualizar
  la fecha de última actividad.
- **Si** se reduce el total de páginas por debajo de una página ya registrada, **entonces** el
  sistema deberá aceptar el cambio, recalcular porcentajes y limitar a 100 los que excedan;
  las lecturas completadas conservan su estado `completada`.

**RF-004 — Estado derivado.** El sistema deberá mostrar un único estado por libro, con esta
precedencia: `archivado`; si no, `leyendo` si existe una sesión activa; si no, `leído` si hay
al menos una lectura completada; si no, `pendiente`.

### 4.2 Lectura y progreso
**RF-005 — Iniciar lectura.** El sistema deberá permitir iniciar una sesión de lectura.
- **Cuando** se inicie sobre un libro activo sin sesión activa, **el sistema deberá** crear una
  sesión con página actual 0 y fecha de inicio.
- **Si** el libro ya tiene una sesión activa, **entonces** el sistema deberá impedirlo e indicar
  cuál es la sesión en curso.
- **Cuando** se inicie una sesión sobre un libro `leído`, **el sistema deberá** contarla como
  nueva lectura (relectura) manteniendo el historial previo.

**RF-006 — Actualizar página.** El sistema deberá permitir fijar la página actual de la sesión
activa.
- **Si** la página no es un entero entre 0 y el total de páginas, **entonces** el sistema deberá
  rechazarla sin modificar el progreso.
- **Cuando** la página sea menor que la actual, **el sistema deberá** aceptar el retroceso
  (corrección de error del usuario).

**RF-007 — Porcentaje automático.** El sistema deberá calcular el porcentaje de avance de la
sesión activa como `piso(página_actual × 100 ÷ total_páginas)`, y deberá mostrar 100 únicamente
cuando la página actual iguale el total. Sin sesión activa: 100 % si hay lecturas completadas,
0 % si no.

**RF-008 — Fin automático.** **Cuando** la página actual alcance el total de páginas, **el
sistema deberá** cerrar la sesión como `completada` con fecha de fin, marcar el libro como
`leído` y dejarla en estado `pendiente de reseña`.
- **Mientras** una sesión está `completada`, **el sistema deberá** impedir cambios de página
  en ella; para seguir leyendo deberá iniciarse una nueva sesión.

**RF-009 — Contador de lecturas.** El sistema deberá mostrar las lecturas completadas de cada
libro y, cuando existan dos o más, las relecturas (completadas − 1). Las sesiones no completadas
no cuentan.

### 4.3 Reseñas
**RF-010 — Reseña por lectura terminada.** El sistema deberá permitir registrar una reseña por
cada sesión completada, con calificación entera de 1 a 5 obligatoria y opinión larga opcional de
hasta 2 000 caracteres.
- **Si** la calificación falta o está fuera del rango, **entonces** el sistema deberá rechazar
  la reseña.
- **Cuando** se guarde o edite una reseña, **el sistema deberá** registrar fecha y hora de
  creación y de última edición.
- **Si** la sesión no está completada, **entonces** el sistema deberá impedir la reseña.

**RF-011 — Reseñas pendientes.** Mientras existan sesiones completadas sin reseña, el sistema
deberá señalarlo en el detalle del libro y permitir registrarlas en cualquier momento posterior.

### 4.4 Notas
**RF-012 — Notas fechadas.** El sistema deberá permitir agregar notas de texto de hasta 1 000
caracteres a un libro activo, con fecha y hora de creación automáticas, en cualquier momento:
durante una sesión activa, sin sesión, o después de completar una lectura.
- **Si** el texto está vacío, **entonces** el sistema deberá rechazar la nota.
- **Donde** el usuario indique una página, **el sistema deberá** validarla entre 0 y el total de
  páginas; la página y la sesión asociada son opcionales.
- **Si** se asocia una sesión de otro libro, **entonces** el sistema deberá rechazar la nota.
- **Cuando** se edite una nota, **el sistema deberá** conservar la fecha de creación y registrar
  la de edición; las notas no se eliminan en el MVP y se listan de la más reciente a la más antigua.

### 4.5 Consulta
**RF-013 — Listado.** El sistema deberá listar los libros activos con título, autor, estado,
página actual, total de páginas, porcentaje y lecturas completadas.

**RF-014 — Búsqueda.** El sistema deberá buscar por coincidencia parcial en título o autor, sin
distinguir mayúsculas ni acentos. **Si** no hay coincidencias, **entonces** deberá mostrar un
mensaje de lista vacía.

**RF-015 — Filtro.** El sistema deberá filtrar por estado `pendiente`, `leyendo` o `leído`,
combinable con la búsqueda.

**RF-016 — Orden.** El sistema deberá ordenar por título (A–Z), autor (A–Z) o última actividad
(más reciente primero). La última actividad es la fecha más reciente del libro, sus sesiones,
reseñas o notas.

### 4.6 Archivado
**RF-017 — Archivar.** El sistema deberá permitir archivar un libro solo tras confirmación explícita.
- **Cuando** se archive, **el sistema deberá** ocultarlo de listados, búsquedas, filtros y
  ordenaciones, conservar intactos sus sesiones, reseñas y notas, y bloquear toda operación
  futura sobre él.
- **Si** el libro tiene una sesión activa, **entonces** el sistema deberá archivarlo dejando esa
  sesión registrada como no completada.
- El sistema no deberá ofrecer restauración de libros archivados en el MVP.

### 4.7 Persistencia
**RF-018 — Persistencia local.** El sistema deberá conservar todos los datos entre ejecuciones
en un archivo JSON local versionado, y guardar los cambios de forma atómica tras cada operación
exitosa.

**RF-019 — Archivo ilegible.** **Si** el archivo de datos no puede leerse o no cumple el esquema,
**entonces** el sistema deberá informar el problema, no sobrescribir el archivo y permitir
inspeccionarlo manualmente. **Si** el archivo no existe, **el sistema deberá** iniciar una
biblioteca vacía.

## 5. Casos límite
| Caso | Comportamiento esperado |
|---|---|
| Libro de 1 página actualizado a la página 1 | 100 %, sesión completada, libro `leído` |
| Página 299 de 300 | 99 % (truncamiento, nunca 100 antes de tiempo) |
| Página negativa, decimal o mayor al total | Rechazo, progreso intacto |
| Retroceso de página | Aceptado |
| Dos sesiones activas en el mismo libro | Rechazo con aviso |
| Libro sin sesiones | 0 %, estado `pendiente` |
| Sesión completada sin reseña | Libro `leído` + indicador "pendiente de reseña" |
| Nota sin página ni sesión, tras terminar | Aceptada, solo con fecha de creación |
| Nota con página fuera de rango | Rechazo |
| Búsqueda `garcia` sobre `García` | Coincidencia |
| Mismo título y autor registrado dos veces | Dos libros independientes |
| Reducir total de páginas bajo la página actual | Porcentajes recalculados, máx. 100 |
| Archivar libro con lectura activa | Oculto, historial conservado, sin más operaciones |
| Archivo JSON corrupto | Error legible, archivo no modificado |
| Cadena de solo espacios en campos obligatorios | Rechazo |

## 6. Fuera de alcance (MVP)
- Alta por ISBN o cualquier API externa; portadas e imágenes.
- Libros electrónicos, audiolibros, PDF o préstamos.
- Cuentas, multiusuario, sincronización en la nube o dispositivos.
- Importación/exportación distinta del archivo JSON interno y copias de seguridad.
- Etiquetas, listas personalizadas, recomendaciones y estadísticas agregadas.
- Búsqueda dentro del texto de notas u opiniones.
- Eliminación permanente de datos y restauración de archivados.
- Notificaciones, recordatorios y objetivos de lectura.

## 7. Dudas abiertas
1. **Plataforma e interfaz** (CLI, escritorio o web local): se decide en `plan.md`; no altera los RF.
2. **Restaurar archivados**: provisionalmente fuera del MVP; si se pide, añade RF y estado reversible.
3. **Varias copias físicas del mismo título**: provisionalmente se registran como libros separados,
   sin campo de ejemplares.
4. **Opinión larga obligatoria u opcional**: provisionalmente opcional, para no bloquear el cierre
   de la lectura con una calificación válida.
5. **Reabrir una sesión completada**: provisionalmente prohibido; se corrige iniciando otra sesión.
