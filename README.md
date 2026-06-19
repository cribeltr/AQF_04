# Gestión de equipos críticos HHHA

**Motor de trabajo** y **centro de control operacional** para la gestión del
Mantenimiento Preventivo (MP), correctivo, **pendientes** y reprogramaciones de
~1.000 equipos clínicos: no solo registra, también permite **gestionar** el día a día
(operatividad, pendientes, servicio técnico) y **exportar** una estructura compartible.

Reconstrucción desde cero de la app anterior: **misma lógica de negocio, interfaz
mucho más simple y directa**. Un único archivo `index.html` (HTML + CSS + JavaScript
*vanilla* + [SheetJS](https://sheetjs.com/) **embebido en el propio archivo**). Sin
paso de build y **sin dependencias de red**: funciona **100% offline** desde el primer
momento (la exportación e importación de Excel no requieren internet).

## Cómo usar — rutina diaria

La app **parte siempre sin datos**. Cada día, en la pantalla **«Comenzar el día»**:

1. Abre **`index.html`** en un navegador moderno (Chrome/Edge recomendado para el
   auto-guardado a carpeta). Funciona también como archivo local (`file://`).
2. **Paso 1 — Cargar programación (.xlsm):** selecciona `ProgramaciónMP2026.xlsm`
   (incluido en `sample-data/`). Es el maestro de equipos; la app **solo lee** la
   planilla, nunca la escribe.
3. **Paso 2 — Cargar respaldo del día (JSON):** selecciona el respaldo exportado el
   día anterior (trae mantenciones, pendientes y correctivos). El primer día aún no
   hay respaldo: empiezas a registrar y exportas al terminar.
4. Trabaja desde el **Tablero** (operatividad, KPIs, pendientes…). **Al terminar el
   día, exporta** con **«Respaldar ahora»** (JSON + Excel con ID único) para volver a
   cargarlo mañana. Si configuras una **carpeta de auto-guardado**, esto es automático.

> **Sin restauración automática:** los datos viven en los archivos (programación +
> respaldo), no en el navegador; por eso se cargan a diario. Solo se recuerdan las
> **preferencias** (tema, densidad, columnas). Si por error recargas la pestaña con
> trabajo sin exportar, la pantalla de inicio ofrece **«Recuperar esa sesión»** como
> red de seguridad. Para limpiar todo: **«Más» → «Reset total»** o `index.html#reset`.

> **Atajos:** `⌘K` / `Ctrl+K` abre la **paleta de comandos** (busca equipos y
> ejecuta acciones) · `/` enfoca la búsqueda · `j` / `k` mueven el foco por la
> tabla y **Enter** abre la fila enfocada · **Enter** en la búsqueda abre la ficha
> si hay un solo resultado · la búsqueda filtra in situ en Equipos, Pendientes,
> Correctivos y Preventivo (desde otras vistas salta a Equipos).

> **Apariencia:** sistema visual **«Consola clínica»** (base slate fría, acento
> cobalto, tipografía IBM Plex, LEDs de operatividad). Botones en la barra superior
> para **tema claro/oscuro** (`☾`) y **densidad cómoda/compacta** (`▦`); ambas
> preferencias se recuerdan (por defecto: claro y cómoda).

### Migrar datos de la app anterior

**«Importar respaldo anterior»** (en **«Más ▾»**; es una **migración de una sola vez**,
por eso no ocupa sitio en el panel lateral) lee un respaldo de la app antigua
(`tipo: "respaldo-gmp2026"`) y migra mantenciones, pendientes y correctivos a la
estructura nueva. Equipos y registros **no** se importan: se re-leen del `.xlsm`
oficial. Archivo de prueba: `sample-data/ejemplo_respaldo_anterior.json`.

> El **panel lateral** solo lleva las vistas del día y la acción diaria
> **«Cargar respaldo (JSON)»**. Las importaciones ocasionales (respaldo anterior,
> planilla integrada, consolidado) están en **«Más ▾»**.

**«Importar planilla integrada»** lee `Sistema_Gestion_MP2026_Integrado.xlsx`: los
expedientes **correctivos** detallados (OT + envíos + visitas + línea de compra +
reparación en terreno) se **fusionan por Folio SIGEM** con lo ya importado, y los
**pendientes** se añaden marcados con fuente «integrado» (filtrable en la vista
Pendientes). Los respaldos reales y esta planilla se conservan en `data/`.

**«Importar consolidado»** lee los `.xlsx` tipo «Consolidado de mantenciones /
pendientes registradas» (detecta el tipo por los encabezados; admite varios a la
vez). Registra cada fila como mantención o pendiente, enlazando por N° Serie /
N° Inventario, normalizando estados/resultados/fechas, mapeando los tipos de
pendiente y aplicando Borrador→Oficial. Las filas cuyo equipo no está en la
planilla se conservan igualmente.

## Funcionalidades

- **Lectura del `.xlsm`** (`PMP_2026` + `Registro_MP-2026`): encabezados en la fila 7,
  datos desde la fila 8, identificación de columnas por encabezado, exclusión de
  `Q`/`S`, columnas auxiliares tras `AR` ignoradas, N° Inventario y Serie como
  **texto** (conservan ceros a la izquierda). Datos sucios en celdas de mes → ignorados.
- **Arranque sin datos (rutina diaria)**: la app no auto-restaura la sesión; parte
  vacía y exige cargar la programación (.xlsm) y el respaldo (JSON) cada día. Solo se
  persisten las **preferencias** (tema, densidad, columnas). Como red de seguridad, si
  hay trabajo sin exportar en el navegador, el inicio ofrece **«Recuperar esa sesión»**.
  **Reset total** limpia todo.
- **Exportación compartible con ID único**: «Respaldar ahora» genera el **JSON**
  reimportable *y* un libro **Excel** (`.xlsx`) **autoexplicativo y usable sin la app**,
  con hojas **Resumen** (ID único de exportación + indicadores de operatividad),
  **Equipos** (con estado), **Bitácora** (registro cronológico de actividad),
  **Pendientes** (con estado, compromiso y seguimiento), **Servicio Técnico**,
  **No Operativos**, y el detalle (Mantenciones, Correctivos, Avances, Resumen mes×código).
  **Cada exportación lleva un ID distinto** (`GEC-AAAAMMDD-HHMMSS-XXXX`) y un número de
  versión incremental, para identificar inequívocamente cada versión compartida.
- **Auto-guardado a carpeta** (File System Access API, Chrome/Edge): eliges una carpeta
  una vez y la app escribe ahí **JSON + Excel** de forma automática (cada cierto número
  de cambios y **al ocultar/cerrar la pestaña**), sin diálogos. Es la vía **100% fiable**.
- **Protección al cerrar el navegador**: si hay cambios sin respaldar, al cerrar/recargar
  se muestra una **advertencia** del navegador y, si no hay carpeta de auto-guardado, se
  **descarga un respaldo de seguridad** `GEC-HHHA-seguridad-<ID>.json` (mejor esfuerzo:
  algunos navegadores limitan las descargas al cerrar; para garantía total usa la carpeta
  de auto-guardado). El respaldo de seguridad es reimportable como cualquier otro.
- **Tablero / centro de control**: banda de KPIs reactivos, tablero de operatividad
  y panel «Por volcar al `.xlsm`» (celda exacta hoja/fila/columna, copiar/exportar).
- **Equipos**: tabla densa (~1.000 filas, sin virtualización), búsqueda global con
  *debounce* (atajo `/` para enfocar; **Enter** abre la ficha si queda un único
  resultado; desde otras vistas salta a Equipos), **filtro por encabezado tipo Excel**
  (multi-selección), selector de columnas recordado, orden por columna, filtros rápidos
  combinables, exportación a Excel.
- **Ficha de equipo**: navegación **anterior/siguiente** dentro de la lista filtrada;
  **pendientes del equipo en línea** (completar/editar sin salir de la ficha); historial
  cronológico unificado (MP + pendientes + correctivos) **filtrable** (por tipo y texto),
  con **columnas configurables** (Fecha, Tipo, Qué pasó, Detalle, Ejecutor, Estado equipo,
  Resultado — recordadas) y **edición por fila** (cada evento abre su editor y vuelve a la
  ficha al cerrar; los resultados de planilla ofrecen «Registrar» para capturar el día).
- **Pendientes**: además de abrir cada uno, botón **✓** para completar sin abrir el panel.
- **Mantenimiento Preventivo**: registro del detalle, ciclo Borrador→Oficial,
  pendiente automático con causal o gestión pendiente.
- **Mantenimiento Correctivo**: expediente flexible por **Folio SIGEM** (sub-eventos
  en cualquier orden, OT por completar).
- **Pendientes**: 5 tipos en 3 orígenes, con **bitácora `seguimiento[]`** (agregar en
  un clic) y `enEsperaDe`.
- **Reprogramación**: ciclo C1/C5/C6/C7/C8 (generar → imprimir → 2 firmas → oficializar).
- **Resumen mes × código** con conteos y *drill-down* a la vista de Equipos.
- **Indicadores de operatividad** en el Tablero: panel «Operatividad de la flota» con
  conteo de **Operativos**, **En servicio técnico** y **No operativos** (con LED de
  estado), cada uno clicable para filtrar la vista de Equipos por ese estado.
- **Gestión de pendientes** como eje del trabajo: panel de indicadores (abiertos,
  vencidos, en espera de terceros, completados), creación/edición, **completar** y
  **reabrir**, bitácora `seguimiento[]`, `enEsperaDe` y filtros por estado/tipo/origen.

> La antigua **«grabación de uso»** fue **eliminada**: la app ya no registra ni
> exporta telemetría de navegación.
- **Diseño responsivo**: la ficha de equipo y las tablas se adaptan al ancho de la
  pantalla, sin scroll horizontal.
- **Sistema visual «Consola clínica»**: consola de operaciones biomédicas con base
  slate fría, acento cobalto, tipografía IBM Plex, paleta de estado estricta y LEDs
  de operatividad. Incluye **tema claro/oscuro** y **densidad cómoda/compacta**
  conmutables (se recuerdan), **paleta de comandos `⌘K`** (equipos + acciones) y
  **navegación por teclado `j`/`k`** en las tablas. Es solo apariencia y atajos: la
  lógica de negocio y los datos no cambian.

## Lógica de negocio

Documentada en `docs/` (fuente de verdad del comportamiento):

- `docs/Especificacion_MP2026.md` — especificación de implementación.
- `docs/ArbolDecision_MP2026.md` — modelo de datos y reglas de decisión (Parte A/B).
- `docs/Diagramas_MP2026_ajustado.md` — proceso de mantenimiento (auditado).
- `docs/arbol-pendientes.html` — modelo del módulo de pendientes (5 tipos / 3 orígenes).

Puntos clave: clave del equipo = serie si existe, si no inventario · la planilla
manda sobre la app · Borrador nace y se promueve a Oficial al recargar · cumplimiento
solo cuenta `Si`/`Si-RA` · `C1/C5/C6/C7/C8` → reprogramación ≤30 días, `C2/C3/C4` sin fecha.

## Estructura del repositorio

```
index.html                 La aplicación (HTML + JS vanilla + SheetJS embebido, offline)
docs/                       Especificación y diagramas (lógica de negocio)
sample-data/
  ProgramaciónMP2026.xlsm   Planilla oficial de ejemplo (fuente de datos)
  ejemplo_respaldo_anterior.json  Respaldo viejo de prueba para la migración
```
