# Entrega — Sistema de Gestión MP 2026

## Objetivo

**Reconstruir la aplicación desde cero.** La versión anterior (`SistemaGestionMP2026.html`) tiene la lógica correcta, pero **se volvió compleja de usar y genera más trabajo del que ahorra**. La nueva app debe **facilitar el trabajo diario**: misma lógica de negocio, interfaz tipo **centro de control operacional** mucho más simple. Ante la duda entre potencia y simplicidad de uso, **gana la simplicidad**.

## Archivos

1. **Especificacion_MP2026.md** — Cómo construirla: stack, carga del `.xlsm`, persistencia, **importar respaldo anterior**, pendientes, lógica de decisión, interfaz y dirección visual/UX. **Empezar por aquí.**
2. **ArbolDecision_MP2026.md** — Reglas de decisión y modelo de datos **del código actual** (fuente de verdad del *comportamiento* y del formato del respaldo viejo). Se conserva su lógica y datos; **no** su complejidad de uso.
3. **Diagramas_MP2026_ajustado.md** — La lógica del proceso de mantenimiento. Auditada.
4. **arbol-pendientes.html** — Modelo del módulo de pendientes (5 tipos en 3 orígenes). **Modelo válido de pendientes** (prevalece sobre §B.8 del árbol de decisión).
5. **ProgramaciónMP2026.xlsm** — **Fuente de datos y documento oficial** (validado por resolución). La app lo **lee**, nunca lo escribe.
6. **ejemplo_respaldo_anterior.json** — Caso reducido del **formato del respaldo viejo**, para probar el botón "Importar respaldo anterior" (§5b de la especificación).

## Migración de datos

La app nueva debe incluir un botón **"Importar respaldo anterior"** que lea el respaldo de la app vieja y migre las mantenciones, pendientes y correctivos a la estructura nueva (mapeo detallado en §5b). Así no se pierde el trabajo ya registrado (79 mantenciones, 63 pendientes, 14 correctivos).

## Resultado esperado

Un único archivo `.html` (HTML + JS *vanilla* + SheetJS vía CDN) con interfaz tipo centro de control: búsqueda fija, KPIs reactivos, tabla densa de ~1.000 equipos con filtros in-situ, y gestión de MP, correctivos, pendientes y reprogramaciones. Persistencia IndexedDB + localStorage, respaldo JSON, e importación del respaldo anterior.
