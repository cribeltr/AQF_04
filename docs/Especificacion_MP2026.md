# Especificación de implementación — Sistema de Gestión MP 2026

Complemento técnico de `Diagramas_MP2026_ajustado.md`, `arbol-pendientes.html` y `ArbolDecision_MP2026.md`. **Fuente de datos: `ProgramaciónMP2026.xlsm`** (documento oficial). Entregar todo junto a Claude Code.

> **Objetivo: reconstruir la app desde cero.** La versión anterior (`SistemaGestionMP2026.html`) cumple la lógica pero **se volvió compleja de usar y genera más trabajo del que ahorra**. La meta es **facilitar el trabajo diario**: misma lógica de negocio (de los documentos), con una interfaz mucho más simple y directa (§8). Ante la duda entre potencia y simplicidad de uso, **gana la simplicidad**.

---

## 1. Stack

- Un solo archivo `.html` (HTML + CSS + JS *vanilla*, sin paso de build).
- **+ SheetJS (`xlsx`) vía CDN** para leer el `.xlsm`.
- Funciona offline tras la primera carga.

---

## 2. Carga de datos — importar `ProgramaciónMP2026.xlsm`

> El `.xlsm` es el **registro oficial validado por resolución**: la programación anual y el cumplimiento formal (MP hecha / no hecha / causal). La app **lo lee, nunca lo escribe**. El usuario lo actualiza a mano; al **re-importarlo**, la app refresca el estado Oficial (ver ciclo Borrador→Oficial). La app aporta lo que el `.xlsm` no guarda: detalle de ejecución, lo correctivo, pendientes y bitácora.

Leer dos hojas:

| Hoja | Contenido | Meses |
|---|---|---|
| `PMP_2026` | Programación anual | **1 columna por mes** (solo `P`) |
| `Registro_MP-2026` | Registro de cumplimiento | **par P/R por mes** |

**Reglas de lectura (ambas hojas):** encabezados en la **fila 7**, datos desde la **fila 8**; identificar columnas por encabezado; **excluir** `Q` (Observación) y `S` (Responsable MP); en `Registro_MP-2026` **ignorar** las columnas auxiliares a la derecha de `AR`.

**Campos por equipo — columnas A–S:**

| Col | Campo | Notas |
|---|---|---|
| A | Fam | Familia/categoría (útil para agrupar) |
| B | ID | Orden en la planilla. **No** es identificador único |
| C | N° Carpeta | |
| D | N° Inventario | **TEXTO** (ej. `2-115361`) |
| E | Equipo | |
| F | Servicio | |
| G | Unidad | |
| H | Ubicación | |
| I | Procedencia | |
| J | Marca | |
| K | Modelo | |
| L | Serie | **TEXTO**, conserva ceros a la izquierda (`0024` ≠ `24`) |
| M | Año Instalación | |
| N | Vida Útil Residual | |
| O | Clasificación | |
| P | ENU / Baja | |
| ~~Q~~ | ~~Observación~~ | **EXCLUIR** |
| R | Frecuencia MP | Trimestral, Semestral, etc. |
| ~~S~~ | ~~Responsable MP~~ | **EXCLUIR** |

> **Identificador único:** N° de Serie o N° de Inventario (no el ID). Leer ambos como *string*.

**Programación — `PMP_2026` (columnas T–AE = Ene…Dic):** un valor por mes en la subcolumna `P`: `X` programada · `R` reprogramada · `RA` reprogramada de año anterior · `PM` puesta en marcha · *(vacío)* sin MP.

**Registro — `Registro_MP-2026` (pares P/R):**

| Mes | P/R | Mes | P/R |
|---|---|---|---|
| Ene | T / U | Jul | AF / AG |
| Feb | V / W | Ago | AH / AI |
| Mar | X / Y | Sep | AJ / AK |
| Abr | Z / AA | Oct | AL / AM |
| May | AB / AC | Nov | AN / AO |
| Jun | AD / AE | Dic | AP / AQ |

- **P** (Programa): `X` / `R` / `RA` / `PM`.
- **R** (Resultado): `Sí` realizada · `Si-RA` año anterior realizada · `C1`–`C8` reprogramada (causal) · `FS` fuera de servicio · `No` no realizada · `NU` no ubicable · `Baja` dada de baja.

> Datos sucios en celdas de mes → ignorar.

---

## 3. Causales de reprogramación

| Código | Descripción | Comportamiento |
|---|---|---|
| C1 | No se puede desocupar el equipo del paciente (indicación clínica) | reprogramar ≤30 d |
| C2 | Equipo en servicio técnico | sin fecha |
| C3 | No operativo, espera repuestos/accesorios | sin fecha |
| C4 | En préstamo a otro hospital | sin fecha |
| C5 | Sin HH del funcionario SEC (carga) | reprogramar ≤30 d |
| C6 | Sin HH del servicio técnico externo | reprogramar ≤30 d |
| C7 | Ausencia justificada del funcionario SEC > 15 días | reprogramar ≤30 d |
| C8 | Contingencia hospitalaria | reprogramar ≤30 d |

- **C2, C3, C4** → `reprog30 = FALSE`: sin nueva fecha; se registra en el mes real al reintegrar el equipo.
- **C1, C5, C6, C7, C8** → `reprog30 = TRUE`: reprogramar dentro de 30 días.

---

## 4. Ejecutores (lista cerrada)

Carlos Bahamondes Seguel · Cristián Beltrán Oviedo · Cristina Rozas Urrutia · Daniel Díaz Neira · Ignacio Berner Bergara · Macarena Toledo · Marco Ulloa · Matías Soazo Garrido · Ricardo Matus Aroca · Tito Millapán Riquelme · Personal Externo.

---

## 5. Almacenamiento (persistencia)

> Esquema tomado de la app actual (ya probado), en `ArbolDecision_MP2026.md` §A.1.

- **IndexedDB** = almacén principal; **localStorage** = respaldo y migración (si IndexedDB está vacío, migra desde localStorage). Doble respaldo para que funcione incluso abierta como archivo local (`file://`).
- Tres claves lógicas: **preferencias** (últimos valores usados), **planilla cruda** (lo leído del Excel: equipos + registros) y **datos de usuario** (lo creado en la app: mantenciones, pendientes, correctivos). Siempre priorizar los datos de usuario; la planilla se recupera recargando el Excel.
- El `.xlsm` es solo lectura: la app **nunca lo escribe**.
- **Exportar / Importar JSON** para respaldo manual (estructura ordenada: cada colección con sus campos, fecha y versión, legible y reimportable), y **Reset total** que borra las tres claves.

**Respaldo automático.** La app persiste sola en IndexedDB + localStorage, así que **cerrar el navegador normalmente no pierde datos**. Para el respaldo externo:
- **Al cerrar/recargar** (`beforeunload`): si hay cambios desde el último respaldo, **avisar** que conviene exportar. (Los navegadores no permiten descargar un archivo automáticamente al cerrar de forma fiable; por eso es aviso, no descarga forzada.)
- **Auto-guardado a carpeta** (recomendado, donde el navegador lo soporte — *File System Access API*): el usuario elige **una vez** una carpeta de respaldo y la app escribe ahí el JSON automáticamente (al cerrar y/o cada cierto número de cambios), sin diálogo cada vez. Si no hay soporte, un botón **"Respaldar ahora"** bien visible + el aviso al salir.

---

## 5b. Importar respaldo de la app anterior (botón de migración)

Botón **"Importar respaldo anterior"** que lee un respaldo de la app antigua y carga los datos reales sin pérdida.

**Formato de entrada:** JSON con `tipo: "respaldo-gmp2026"`, `version: 2` (esquema en `ArbolDecision_MP2026.md` §A.3; ejemplo de prueba: `ejemplo_respaldo_anterior.json`). Colecciones: `equipos, registros, manuales, pendientes, correctivos, empresas, ingenieros, meta`.

**Qué se importa:**
- `manuales` → **`detalleMP`** (mantenciones de la app).
- `pendientes` → **`pendientes`** (traduciendo el tipo, ver tabla).
- `correctivos` → **`correctivos`** (con sus `avances`).
- `empresas`, `ingenieros` → a sus catálogos.
- `equipos` y `registros` → **NO** se importan: se re-leen del `.xlsm` oficial (la fuente).

**Mapeo de tipos de pendiente (viejo → modelo de 5 tipos del §6):**

| Tipo viejo | origen | tipo |
|---|---|---|
| Reprogramación Mantención Preventiva | preventiva | reprogramacion |
| Protocolo Interno de Mantenimiento | preventiva | protocolo_interno |
| Protocolo Externo de Mantenimiento | preventiva | protocolo_externo |
| Otro | administrativo | otro |

(El formato viejo no trae `documental`; ese nace del flujo correctivo.)

**Campos que cambian de nombre:** `respEjec → respEjecutivo`, `respAdmin → respAdministrativo`, `gestiones → seguimiento` (bitácora del pendiente); en equipos `id → idPlanilla`, `clasificacion → clasif`, `enu → enuBaja`, `regObs → reg`. Conservar `equipoKey` (serie o inventario) para revincular cada registro con su equipo.

**Preservar:** el estado **Oficial/Borrador** de cada mantención y el avance de pendientes/correctivos. Tras importar, aplicar la promoción Borrador→Oficial (§9) al recargar el `.xlsm`.

---

## 6. Modelo de Pendientes

> Según `arbol-pendientes.html`. **Cinco tipos en tres orígenes**, sobre un núcleo común.

**Núcleo común:** `equipo · serie · inventario · servicio · tipo · fechaCompromiso · prioridad · situación · estado · respEjecutivo · respAdministrativo · enEsperaDe · descripcion · tareas[] · seguimiento[] · fechaCompletado · creado`.
El **N° Inventario autocompleta** equipo, serie y servicio; `creado` y `díasDeAtraso` se calculan solos.

**Seguimiento (bitácora de gestiones).** Muchos pendientes dependen de terceros y se trabajan a lo largo del tiempo. Cada pendiente lleva una bitácora `seguimiento[]` de entradas `{ fecha, texto, autor? }` que el usuario va sumando — ej. *"12-06 solicité cotización a don Ricardo"* · *"15-06 reiterado; dice que aún no la envían"*. El campo **`enEsperaDe`** indica de quién/qué depende (proveedor, repuesto, firma…). En la interfaz, agregar una entrada debe ser de **un clic** (campo + "Agregar"); el historial se muestra fechado, lo más reciente arriba. (Equivale a los `avances[]` de los correctivos.)

- **Origen `correctivo`** (hay OT en SIGEM):
  - **`documental`** — el equipo ya opera pero falta el Reporte de Servicio. Campos extra: `folioSIGEM · empresa · origen (visita | servicio_técnico)`. Tareas: gestionar → recibir/verificar → archivar. Resp. ejecutivo = quien consigue el reporte; administrativo por defecto = Cristián.
- **Origen `preventiva`** (sin Folio SIGEM):
  - **`protocolo_interno`** — falta el reporte interno de la MP.
  - **`protocolo_externo`** — lo completa el proveedor. Campo extra: `empresa`.
  - **`reprogramacion`** — la MP no se ejecutó. Tareas: generar e imprimir + firma del supervisor y del jefe de equipos.
- **Origen `administrativo`**:
  - **`otro`** — solo el núcleo común, sin campos extra.

> Solo `documental` añade `folioSIGEM`, `empresa` y `origen`. Una MP **vacía** o con resultado **`No`** **no** crea pendiente formal: queda como *MP sin resultado por gestionar*, visible en el Tablero (no reprograma).

---

## 7. Interfaz — vistas y navegación

1. **Tablero "¿qué hago hoy?" / Centro de control** — KPIs + accesos: MP del mes sin resultado, MP vencidas, pendientes, equipos detenidos, reprogramaciones por firmar.
2. **Equipos** — tabla densa filtrable + ficha de detalle.
3. **Mantenimiento Preventivo** — registro del **detalle** de la MP del mes (fecha, ejecutor, tipo interno/externo, estado resultante, observaciones).
4. **Mantenimiento Correctivo** — apertura OT + ruta A/B/C/D + línea de compra + los dos cierres.
5. **Pendientes** — cinco tipos en tres orígenes (§6).
6. **Reprogramación** — ciclo C1/C5/C6/C7/C8 (generar → imprimir → 2 firmas → escribir el código en el `.xlsm` → recargar → Oficial).

**Comportamientos de la vista Equipos** (JS *vanilla*, sin frameworks; el usuario viene de Excel, así que la tabla debe sentirse como una planilla):
- **Búsqueda global** con *debounce* ~300 ms sobre varias columnas (inventario, equipo, serie, servicio, marca, modelo).
- **Filtro por encabezado, tipo Excel:** cada columna tiene un menú (icono de embudo) con sus valores únicos y *checkboxes* para elegir **uno o más**; se combinan en AND entre columnas y con la búsqueda.
- **Selector de columnas:** mostrar/ocultar las columnas que el usuario quiera; la selección se **recuerda** (en preferencias).
- **Filtros rápidos** combinables (AND): familia · servicio · estado · mes (atajos sobre lo mismo).
- **Tabla densa** (padding mínimo); ~1.000 filas **sin virtualización** (scroll/paginación simple).
- **Ordenamiento por columna** (asc → desc → sin orden).
- **Estado y tareas pendientes con *badge* de color** por fila.
- **Exportar la vista** filtrada a CSV/XLSX (además del respaldo JSON del §5).

**Ficha de Equipo — historial unificado ("qué pasó con este equipo").** Al abrir un equipo (clic en su fila), además de sus datos (inventario, serie, servicio, ubicación, estado actual) se muestra una **línea de tiempo / tabla cronológica simple** que junta **todo lo que le ha pasado**, de todas las fuentes, ordenado por fecha:
- **Mantenciones** (preventivas): fecha · resultado · ejecutor · estado · observaciones.
- **Pendientes**: su creación y cada entrada de la bitácora de seguimiento (fecha · texto).
- **Correctivos**: la apertura de la OT y cada avance (envío, retorno, informe técnico, OC, visita, cierre) con su fecha.

Formato: columnas **Fecha · Qué pasó · Detalle**, con un color o icono sutil por tipo (MP / pendiente / correctivo) y lo más reciente arriba. El objetivo es entender la historia del equipo de un vistazo, sin tecnicismos.

**KPIs reactivos**: las tarjetas del Tablero se recalculan al aplicar filtros.

**Resumen mes × código (tabla dinámica con *drill-down*).** Para quien viene de Excel: una tabla con **filas = meses** y **columnas = códigos** (`X` programada · `R` reprogramada · `RA` año anterior · **sin registro**, y opcionalmente `Sí`/causales). Cada celda muestra el **conteo de equipos**; al **hacer clic en una celda** se abre la vista de Equipos **filtrada por ese mes + ese código**. Es el puente entre el panorama y el detalle.

**Ayuda "Por volcar al `.xlsm`"** (clave para el flujo diario). Como la app no escribe el `.xlsm`, debe facilitar el paso manual de oficializar. Un panel en el Tablero lista las mantenciones en **Borrador** listas para escribirse en la planilla:
- Por cada una: **equipo** (inventario/serie), **mes** y **código resultado** (Sí, C5, Baja…), con **dónde escribirlo**: hoja `Registro_MP-2026`, fila del equipo, columna del mes (subcolumna `R` del par P/R).
- **Copiar al portapapeles** o exportar la lista para transcribir rápido; agrupable por servicio o por mes.
- Las de **resultado directo** (Sí, FS, No, NU, Baja, Si-RA) están listas de inmediato; las de **reprogramación (C1–C8)** aparecen aquí solo cuando su reporte ya tiene las **2 firmas** (recién ahí se escribe el código en el Excel).
- Al reimportar el `.xlsm`, lo que ya quedó en la planilla pasa a Oficial y **sale de este panel**.

**Exportar a Excel (en toda vista).** Cada vista con tabla —Equipos, Pendientes, Correctivos, Mantenciones, Resumen mes×código, Ficha de Equipo— tiene siempre un botón **"Exportar a Excel"** que descarga lo visible **respetando filtros, orden y columnas** (XLSX; CSV opcional).

**Correctivos — expediente flexible alrededor del folio SIGEM.** El **folio SIGEM** es la llave del expediente; todo cuelga de él. Los sub-registros (envío, retorno, visita técnica, repuesto, reparación en terreno, línea de compra, baja) se pueden ingresar **de forma independiente y en cualquier orden**, sin tener que crear antes la Orden de Trabajo:
- Al registrar un sub-evento (p. ej. un **envío**), el sistema **pide el folio SIGEM** (obligatorio: identifica el expediente).
- Si **ya existe** un expediente con ese folio → el sub-evento se **adjunta** a él.
- Si **no existe** → se crea el expediente con ese folio y los **datos de apertura de la OT quedan por completar**; el sub-evento queda adjunto.
- Cuando luego se registre la **OT** con ese folio → se **completan** sus datos de apertura (técnico, falla, fecha, estado inicial) **sin duplicar** el expediente; el envío y lo ya adjuntado quedan vinculados.
- La app **marca los expedientes con la OT incompleta** ("OT por completar") para que nada quede suelto.

**Navegación:** el botón **Volver** regresa a la vista de origen (la navegación conserva el contexto de origen).

---

## 8. Dirección visual / UX — "centro de control operacional"

Herramienta de trabajo diario para gestionar 800–1.000 equipos. **Eficiencia y claridad por sobre lo decorativo.** Esta es la corrección principal frente a la app anterior: **menos pasos, menos campos por pantalla, menos clics**. Si un dato no es imprescindible para la acción del momento, no se pide en ese paso. (Claude Code puede apoyarse en el skill `frontend-design` para los tokens.)

- **Pantalla principal = centro de control:** el estado global de los equipos se ve de inmediato, sin saltar entre pantallas.
- **Búsqueda siempre visible** arriba: encontrar cualquier equipo en segundos.
- **Banda de KPIs** bajo la búsqueda: % operativos · fuera de servicio · tareas pendientes · MP vencidas · equipos en servicio técnico (con tiempo detenido).
- **Tabla grande y densa** como zona principal: una fila por equipo (inventario, nombre, ubicación, servicio, estado, tareas pendientes); ver muchos a la vez.
- **Filtros in-situ** (botones, desplegables, búsqueda), **sin modales ni ventanas**: pasar de 1.000 equipos a los relevantes en segundos.
- **Color con moderación:** sobre todo para alertas y estados; el resto neutro y discreto, para trabajar horas **sin fatiga visual**.
- **Mínimos clics** en las acciones frecuentes; sensación de panel de operaciones, no de sitio web.

---

## 9. Reglas de negocio y lógica de decisión

> Reglas de proceso (de los diagramas) + lógica rescatada del código actual (`ArbolDecision_MP2026.md` §B). **Se conserva la lógica; lo que NO se replica es la complejidad de uso.**

- **Clave del equipo:** serie si existe, si no inventario. Une planilla, mantenciones, pendientes y correctivos del mismo equipo.
- **Origen del resultado (equipo+mes):** manda la planilla; si está vacía, cuenta la mantención de la app; entre varias del mismo mes, gana la de fecha más reciente.
- **Borrador → Oficial:** lo leído de la planilla es Oficial; lo creado en la app nace Borrador y se **promueve a Oficial** cuando, al recargar el `.xlsm`, ese equipo+mes ya trae resultado.
- **C1/C5/C6/C7/C8** → reprogramación ≤30 días (mes origen conserva `X` + causal; mes destino `R`); se vigila la ventana de 30 días con alertas de color. **C2/C3/C4** → sin fecha (depende de un tercero).
- **C2/C3/C4 · FS · NU** → equipo no disponible. Estados: C2→Servicio Técnico · C3→No Operativo · C4→Préstamo · FS→No Operativo · NU→No Ubicable (estado propio). **Baja** → prioridad máxima.
- **Pendiente automático desde una MP:** con causal → pendiente *Reprogramación*; con "gestión pendiente" sin causal → pendiente *Protocolo* (interno/externo). Los tipos *documental* y *otro* nacen del flujo correctivo o se crean a mano (modelo completo de 5 tipos en §6 / `arbol-pendientes.html`).
- **Cumplimiento:** solo `Sí` y `Si-RA` cuentan como realizado; las causales C1–C8 son reprogramadas, no realizadas. Los meses futuros no penalizan.
- **Tablero de operatividad:** un equipo aparece si no está Operativo, tiene correctivo abierto, quedó en FS/NU sin resolver, o tiene pendientes vencidos / protocolos sin resolver. La criticidad sube con los días detenido.

---

## 10. Checklist para Claude Code

- [ ] Importar `ProgramaciónMP2026.xlsm` (solo lectura); leer `PMP_2026` y `Registro_MP-2026`.
- [ ] Encabezados fila 7, datos fila 8; excluir Q y S; ignorar auxiliares tras `AR`.
- [ ] N° Inventario y Serie como **string** (ceros a la izquierda) en todo el flujo.
- [ ] Meses: 1 col/mes en `PMP_2026`; pares P/R (T/U…AP/AQ) en `Registro_MP-2026`.
- [ ] Resultados: `Sí`, `Si-RA`, `C1–C8`, `FS`, `No`, `NU`, `Baja`.
- [ ] Pendientes con los 5 tipos / 3 orígenes (§6), con **bitácora `seguimiento[]`** fechada (agregar entrada en un clic) y `enEsperaDe`.
- [ ] Persistencia **IndexedDB + localStorage** (§5); Exportar/Importar JSON con estructura ordenada; la app **no** escribe el `.xlsm`.
- [ ] **Respaldo automático** (§5): aviso al salir si hay cambios sin respaldar + auto-guardado a carpeta donde haya soporte.
- [ ] **Exportar a Excel** en toda vista con tabla (respeta filtros/orden/columnas).
- [ ] Botón **"Importar respaldo anterior"** (§5b): mapea `manuales→detalleMP`, traduce los tipos de pendiente y trae los correctivos con sus avances.
- [ ] UI tipo centro de control (§8): búsqueda fija, KPIs reactivos, tabla densa, filtros in-situ, color solo para alertas; sin virtualización.
- [ ] Tabla con **filtro por encabezado tipo Excel** (multi-selección), **selector de columnas** (recordado) y orden por columna.
- [ ] **Resumen mes × código** con conteos y *drill-down*: clic en celda → Equipos filtrado por mes + código.
- [ ] **Ficha de Equipo**: historial cronológico unificado (MP + pendientes + correctivos) en tabla/línea de tiempo simple, lo más reciente arriba.
- [ ] Panel **"Por volcar al `.xlsm`"**: lista los Borradores listos con equipo, mes, código y ubicación (hoja/fila/columna), con copiar/exportar; los de reprogramación solo tras las 2 firmas.
- [ ] **Correctivos flexibles por folio SIGEM**: registrar cualquier sub-evento (envío, visita…) sin crear antes la OT; pedir el folio; agrupar por folio; completar la OT después sin duplicar.
- [ ] Las 6 vistas con navegación que conserva el origen + motor de estado del §1 del `.md`.
