# 001-libros-mvp — índice de planificación

Especificación del MVP de `sistema_libros`: biblioteca personal de libros físicos con control de
lectura, relecturas, reseñas y notas fechadas.

> **Estado: planificación aprobada. Implementación no iniciada (0 de 42 tareas completadas).**

## Documentos

| Documento | Rol | Estado |
|---|---|---|
| [spec.md](spec.md) | Contexto, usuarios, 7 historias, 20 RF en EARS, 15 casos límite, exclusiones, dudas abiertas y criterios de finalización (CF-01–CF-08) | Aprobada (revisión de QA aplicada) |
| [plan.md](plan.md) | Arquitectura en capas, modelo JSON v1, contratos de puertos/CLI/errores, 10 decisiones justificadas, estrategia de pruebas y riesgos | Aprobado |
| [tasks.md](tasks.md) | 6 fases y 42 tareas con checkbox y condición "Hecho cuando" | Pendiente de implementación |

Los principios que gobiernan estas decisiones están en la
[constitución](../../docs/constitution.md).

## Decisiones clave

- **Interfaz:** CLI local en Python 3.11+ con solo biblioteca estándar (D-01).
- **Arquitectura:** `interfaces/cli → application → domain`, con infraestructura implementando
  puertos y dominio sin I/O (D-02).
- **Persistencia:** un `biblioteca.json` con `schema_version: 1`, escritura atómica y agregados
  anidados por libro (D-03, D-04, D-07).
- **Datos derivados:** estado, porcentaje, página mostrada, relecturas y última actividad se
  calculan; nunca se persisten (D-06).
- **Reseñas:** una por sesión completada, calificación 1–5 obligatoria y opinión ≤2 000 opcional;
  se pueden editar sin duplicar (RF-010).
- **Archivado:** oculta el libro y bloquea operaciones, conservando todo el historial; sin
  restauración en el MVP (RF-017).

## Trazabilidad

- Cada RF tiene al menos una prueba planificada (matriz en `plan.md` §7).
- Cada tarea referencia sus RF y su condición de cierre.
- Los criterios globales de terminado están en `spec.md` §8.

## Orden de ejecución

Fase 0 (preparación) → Fase 1 (dominio) → Fase 2 (persistencia) → Fase 3 (aplicación) →
Fase 4 (CLI) → Fase 5 (aceptación y cierre). No se debe avanzar de fase sin cumplir las
condiciones "Hecho cuando" de la anterior.
