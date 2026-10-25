# Constitución — sistema_libros

1. **Simplicidad**: el MVP se limita a libros físicos; toda funcionalidad no exigida por la spec queda fuera. Se prefiere la solución más simple que cumpla los criterios de aceptación.
2. **La spec manda**: ningún requisito vive solo en el código; todo cambio de comportamiento exige actualizar `spec.md` antes o junto con la implementación, y cada RF es trazable a tareas y escenarios de prueba.
3. **Separación lógica/interfaz**: el dominio (libros, progreso, reseñas) no depende de la UI ni del almacenamiento; la interfaz solo orquesta casos de uso. Los contratos entre capas se documentan en `plan.md`.
4. **Tests como contrato**: cada RF tiene al menos una prueba automatizada derivada de su criterio EARS; sin prueba no se considera terminado. Los cálculos de progreso y relecturas requieren casos borde explícitos.
5. **Persistencia local y portable**: el estado se guarda en JSON local, sin servicios externos ni credenciales; escrituras atómicas y datos recuperables ante cierre inesperado. El esquema JSON es versionado.
6. **Idioma y claridad**: documentación, requisitos, nombres de dominio y mensajes al usuario en español; código e identificadores en inglés. Sin comentarios que repitan el código.
