# sistema_libros

Aplicación para gestionar una **biblioteca personal de libros físicos**: registrar lo que
posees, seguir el avance de lectura con porcentaje automático, contar relecturas, calificar y
opinar al terminar cada lectura, y anotar ideas con fecha y hora.

> **Estado actual: planificación completa (SDD), sin código de implementación.**
> Este repositorio contiene únicamente documentos de especificación, plan técnico y tareas.

## Alcance del MVP

- Alta **manual** de libros (sin ISBN ni APIs externas).
- Control de lectura: página actual, porcentaje calculado, cierre automático al llegar al final.
- Historial de relecturas por libro.
- Una reseña por lectura terminada: 1–5 estrellas + opinión larga opcional.
- Notas pequeñas fechadas, dentro o fuera de una sesión de lectura.
- Búsqueda por título/autor sin distinguir acentos, filtros por estado y ordenamientos.
- Archivado lógico con conservación del historial.
- Persistencia local en un archivo JSON versionado, sin cuentas ni red.

Fuera de alcance: ebooks, préstamos, multiusuario, sincronización, importación/exportación y
estadísticas (detalle en la [spec](specs/001-libros-mvp/spec.md#6-fuera-de-alcance-mvp)).

## Documentación (orden de lectura)

| Paso | Documento | Contenido |
|---|---|---|
| 1 | [Constitución](docs/constitution.md) | 6 principios innegociables del proyecto |
| 2 | [Especificación](specs/001-libros-mvp/spec.md) | Contexto, usuarios, US, 20 RF en EARS, casos límite, fuera de alcance, dudas y criterios de finalización |
| 3 | [Plan técnico](specs/001-libros-mvp/plan.md) | Arquitectura, modelo JSON, contratos, decisiones justificadas y pruebas |
| 4 | [Tareas](specs/001-libros-mvp/tasks.md) | 42 tareas con checkbox y condición "Hecho cuando" |
| — | [Índice de la spec](specs/001-libros-mvp/README.md) | Resumen y estado de la planificación |

## Estructura del repositorio

```
docs/
  constitution.md
specs/
  001-libros-mvp/
    README.md
    spec.md
    plan.md
    tasks.md
```

## Metodología

El proyecto sigue **Specification-Driven Development (SDD)**: primero los principios, luego la
especificación funcional, una revisión de QA, el plan técnico y finalmente el desglose de tareas.
Cada cambio de comportamiento debe actualizar la spec y mantener la trazabilidad RF → tarea →
prueba. No se considera terminada una funcionalidad sin prueba automatizada.

## Próximos pasos

1. Completar las tareas de la [Fase 0](specs/001-libros-mvp/tasks.md#fase-0--preparación).
2. Implementar el dominio y sus pruebas antes de la CLI.
3. Verificar los criterios globales de finalización (CF-01–CF-08) de la spec.

Requisitos previstos para la implementación: Python 3.11+ (probado con 3.14), sin dependencias de
ejecución y `pytest` como dependencia de desarrollo. La interfaz prevista es una CLI local; el
detalle está en el [plan técnico](specs/001-libros-mvp/plan.md).
