# Árbol de Decisión — Sistema de Gestión MP 2026

Documento escrito (no diagrama) que describe **la lógica de decisión** y **los datos que la aplicación guarda**. Reconstruido directamente del código de `SistemaGestionMP2026.html`.

---

## PARTE A — DATOS QUE GUARDA

### A.1 Dónde y cómo persiste
La app guarda todo en el navegador, con **doble respaldo** para que funcione incluso abierta como archivo local (`file://`):

- **IndexedDB** → almacén principal.
- **localStorage** → respaldo y migración. Si IndexedDB está vacío, migra desde localStorage.
- Tres claves lógicas:
  - `LS_PREF` → preferencias de uso (últimos valores usados).
  - `LS_DATOS` → planilla cruda leída del Excel (equipos + registros).
  - `LS_USUARIO` → datos creados en la app (mantenciones, pendientes, correctivos, etc.).
- Prioriza siempre los datos de usuario; si la planilla no cabe en `file://`, se recupera recargando el Excel.
- **Reset total**: borra `LS_DATOS`, `LS_USUARIO` y `LS_PREF`.

### A.2 Objeto de estado global (`state`)
```
state = {
  equipos:     [],   // catálogo leído de la planilla
  registros:   [],   // resultados mensuales leídos de la planilla
  manuales:    [],   // mantenciones preventivas registradas en la app
  pendientes:  [],   // tareas/gestiones por resolver
  correctivos: [],   // eventos de mantenimiento correctivo (expedientes)
  empresas:    [],   // empresas para compras por Trato Directo
  ingenieros:  [],   // ingenieros externos reutilizables
  meta:        null  // metadatos de la última carga
}
```

### A.3 Esquema de cada colección

**equipos[]** (una fila por equipo)
`key`, `fila`, `id`, `carpeta`, `inventario`, `equipo`, `servicio`, `unidad`, `ubicacion`, `procedencia`, `marca`, `modelo`, `serie`, `anio`, `vidaUtil`, `clasificacion`, `enu`, `prog:{mes→código}`, `regObs`.
→ `serie` e `inventario` se tratan como **texto** (preservan ceros a la izquierda).

**registros[]** (planilla, resultado de un equipo en un mes)
`equipoKey`, `mes` (0–11), `programa` (X/R/RA/PM), `resultado`, `obs`.

**manuales[]** (mantención registrada en la app)
`id`, `creado`, `equipoKey`, `fecha`, `programacion`, `ejecutor`, `resultado`, `observaciones`, `estadoEquipo`, `gestionPendiente`, `tipoMantenimiento` (Interno/Externo), `ingenieroExterno`, `empresaExterna`, `tipoRegistro` (**Oficial/Borrador**), `oficializado?`, `actualizado?`.

**pendientes[]**
`id`, `creado`, `origenMP?`, `equipoKey`, `tipo`, `causal?`, `fechaReporte?`, `fechaCompromiso`, `tipoMantenimiento`, `descripcion`, `tareas[]` (cada una con `texto`/`hecha` o `opciones`/`estado`/`subtareas`), `gestiones[]`, `respAdmin`, `respEjec`, `estado`, `fechaCompletado`.

**correctivos[]** (expediente de un evento)
`id`, `creado`, `equipoKey`, `tipoEvento`, `requerimiento`, `fechaDocumento`, `folioSigem`, `tecnico`, `numeroEnvio`, `estadoEquipo`, `tipoCompra`, `informeTecnico*`, `empresa`, `ordenCompra*`, `ingenieroExterno`, `numeroReporteServicio`, `tipoServicio`, `estadoEvento` (Abierto/Cerrado), `fechaCierre`, `avances[]`, `gestionPendiente`, `tipoRegistro` (Oficial/Borrador).

**meta** → `archivo`, `fechaCarga`, `hojaPMP`, `hojaReg`, `avisos`, `oficializados`, `ignoradosDetalle`.

### A.4 Catálogos fijos (constantes)
- **Programación (PROG_CODES):** `X` Programada · `R` Reprogramada · `RA` Reprogramada de año anterior · `PM` Puesta en Marcha.
- **Resultados (RESULTADOS):** `Si` Realizada · `Si-RA` Año anterior realizada · `C1…C8` Reprogramada por causal · `FS` Fuera de servicio · `No` No realizada · `NU` No ubicable · `Baja` Dado de baja.
- **Causales (CAUSALES):**
  - `C1` Imposibilidad de desocupar el equipo (indicación clínica).
  - `C2` Equipo en servicio técnico.
  - `C3` Equipo no operativo, esperando repuestos/accesorios.
  - `C4` Equipo en préstamo a otro hospital/institución.
  - `C5` No hay horas-hombre del funcionario SEC (alta carga).
  - `C6` No hay horas-hombre del servicio técnico externo.
  - `C7` Ausencia justificada del funcionario SEC > 15 días.
  - `C8` Contingencia hospitalaria.
- **Clasificación causales:** `REPRO_30DIAS = {C1,C5,C6,C7,C8}` · `REPRO_SIN_FECHA = {C2,C3,C4}`.
- **Ejecutores:** Carlos Bahamondes Seguel, Cristián Beltrán Oviedo *(Resp. Administrativo por defecto)*, Cristina Rozas Urrutia, Daniel Díaz Neira, Ignacio Berner Bergara, Macarena Toledo, Marco Ulloa, Matías Soazo Garrido, Ricardo Matus Aroca, Tito Millapán Riquelme, Personal Externo.

---

## PARTE B — ÁRBOL DE DECISIÓN (LA LÓGICA)

### B.1 Identidad del equipo (clave `key`)
```
SI el equipo tiene serie válida → key = "S:" + serie (mayúsculas)
SINO                            → key = "I:" + inventario (mayúsculas)
```
Esta clave une planilla, mantenciones, pendientes y correctivos del mismo equipo.

### B.2 Carga de la planilla (origen del dato)
```
AL cargar el Excel:
  ├─ Lee equipos desde la hoja PMP_2026 (encabezados fila 7, datos desde fila 8)
  ├─ Lee resultados mensuales desde Registro_MP-2026 (columnas pareadas Programa/Resultado)
  ├─ Datos sucios en celdas de mes → se IGNORAN (quedan en meta.ignoradosDetalle)
  └─ Aplica la Regla Oficial/Borrador (ver B.3)
```

### B.3 Regla Oficial / Borrador (promoción de registros)
```
Todo resultado leído de la planilla        → es OFICIAL (verde).
Registro creado en la app                  → nace BORRADOR (ámbar) por defecto.

AL recargar la planilla, para cada manual BORRADOR de 2026:
  SI ese equipo+mes YA aparece con resultado en la planilla
     → el manual se promueve a OFICIAL (marca fecha "oficializado")
  SINO
     → sigue BORRADOR
```
*Idea: el Borrador es lo que el técnico adelanta en terreno; se vuelve Oficial cuando la planilla lo confirma.*

### B.4 ¿Qué resultado cuenta por equipo+mes? (unificación)
```
Para cada par equipo+mes:
  SI la planilla tiene resultado        → MANDA la planilla (origen: planilla)
  SINO, SI la app tiene mantención      → cuenta la app (origen: app)
        └─ entre varias MP del mismo equipo+mes → gana la de FECHA más reciente
  SINO                                  → sin resultado
```

### B.5 Registro de una mantención preventiva
```
Validar obligatorios: fecha, programación, ejecutor, resultado.
  SI falta alguno → no guarda, avisa.

Guardar el registro (Oficial o Borrador según se marque).

LUEGO, según el resultado elegido:
  SI resultado ∈ {C1…C8}  → es una REPROGRAMACIÓN (ver B.6 y B.7)
  SINO                     → mantención normal (Si, Si-RA, FS, No, NU, Baja)

SI se marcó "gestión pendiente" Y no existe ya un pendiente de esta MP:
  → se genera automáticamente un pendiente vinculado (ver B.8)
```

### B.6 Reglas de reprogramación por causal
```
SI causal ∈ {C1, C5, C6, C7, C8}  (REPRO_30DIAS)
  → DEBE reprogramarse dentro de los 30 días siguientes.
    · El mes original conserva la programación X y registra la causal.
    · El mes de destino se marca con R (Reprogramada).
    · Se vigila la ventana de 30 días (Vigilancia Paso 6) con alertas de color.

SI causal ∈ {C2, C3, C4}  (REPRO_SIN_FECHA)
  → reprogramación SIN nueva fecha definida (depende de un tercero:
     servicio técnico, repuestos, o devolución del préstamo).
```

### B.7 Ciclo del reporte de reprogramación (etapas y firmas)
La etapa se deduce de las tareas hechas del pendiente:
```
SIN checklist                          → "Por generar"      (rojo)
Generado, falta imprimir               → "Por imprimir"     (ámbar)
Impreso, faltan ambas firmas           → "Faltan 2 firmas"  (ámbar)
Falta solo la del supervisor clínico   → "Falta firma del supervisor"
Falta solo la del jefe de equipos méd. → "Falta firma del jefe"
Ambas firmas listas:
   SI la planilla ya trae el código    → "Oficializado"     (verde) → cerrar
   SINO                                 → "Firmado"          (por oficializar en Excel)
```
*Dos firmas requeridas: supervisor del servicio clínico + jefe de equipos médicos.*

### B.8 Generación automática de pendientes (desde una MP)
```
SI la MP tiene causal (C1…C8):
  → pendiente tipo "Reprogramación Mantención Preventiva"
    con tareas: generar → imprimir → firma supervisor → firma jefe
    (lleva causal + fecha del reporte)
SINO:
  → pendiente tipo "Protocolo Interno/Externo de Mantenimiento"
    con la lista de tareas de protocolo según tipo (Interno = 1 tarea; Externo = 2)

Fecha de compromiso = fecha del evento que lo originó.
Responsable administrativo = Cristián Beltrán Oviedo (por defecto).
```

### B.9 Resolución de una tarea de protocolo
```
SI la tarea tiene opciones (Sí/No/Imprimir/Gestión):
   · estado "Sí" o "No"        → RESUELTA
   · estado "Gestión" con subtareas → resuelta solo si TODAS las subtareas están hechas
   · "Imprimir" o "Gestión" sin completar → sigue PENDIENTE
SINO (tarea simple):
   → resuelta cuando está marcada como "hecha"
```

### B.10 Evento correctivo (expediente)
```
Registrar evento (Abierto/Cerrado, Oficial/Borrador).

SI hay compra asociada:
   · captura N° orden de compra, fecha, descripción.
   SI la compra es "Trato Directo":
      → exige N° informe técnico, fecha, responsable y empresa
        (si falta alguno, no guarda).

SI es Reporte de Servicio → captura ingeniero externo, N° reporte, tipo (Reparación/Diagnóstico).

SI el evento se marca "Cerrado" → exige fecha de cierre.

SI se marcó "gestión pendiente" → genera pendiente vinculado (igual que B.8).
```

### B.11 Cálculo de cumplimiento
```
Realizadas = resultados con "Si" o "Si-RA".

Por cada mes:
  SI el mes es futuro → % = sin dato (no penaliza)
  SINO               → % cumplimiento = round(100 × realizadas / programadas)

Total = round(100 × total_realizadas / total_programadas)
```
*Solo "Si"/"Si-RA" cuentan como cumplido; las causales C1–C8 cuentan como reprogramadas, no como realizadas.*

### B.12 Tablero de Operatividad (qué equipo necesita atención)
Un equipo entra al tablero si cumple **al menos una** condición:
```
· estado distinto de "Operativo"            (no operativo / en servicio técnico)
· tiene un expediente correctivo abierto
· quedó en FS o NU sin resolver
· tiene pendientes vencidos
· tiene protocolos sin resolver
```
La criticidad se eleva según la racha de días detenido y la etapa del expediente.

---

### Resumen de los puntos de decisión clave
1. **Clave del equipo:** serie si existe, si no inventario.
2. **Origen del resultado:** planilla manda; la app completa los vacíos; gana la MP más reciente.
3. **Oficial vs Borrador:** la planilla oficializa; el Borrador se promueve al recargar.
4. **Causal → reprogramación:** {C1,C5,C6,C7,C8} a 30 días; {C2,C3,C4} sin fecha.
5. **Reporte de reprogramación:** generar → imprimir → 2 firmas → oficializar.
6. **Gestión pendiente:** dispara un pendiente automático (reprogramación o protocolo).
7. **Cumplimiento:** solo "Si"/"Si-RA", excluyendo meses futuros.
