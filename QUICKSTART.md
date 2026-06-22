# Guía rápida — Gestión de equipos críticos HHHA (v1.0)

App de un solo archivo (`index.html`), **100% offline**, sin instalación.

## Empezar (2 minutos)

1. Abre **`index.html`** en **Chrome o Edge** (doble clic; funciona como archivo local).
2. En **«Comenzar el día»**:
   - **Paso 1 — Cargar programación (.xlsm):** elige `ProgramaciónMP2026.xlsm`
     (hay una de ejemplo en `sample-data/`). Es el maestro de equipos; solo se lee.
   - **Paso 2 — Cargar respaldo del día (.json):** el respaldo que exportaste ayer.
     El primer día no hay: empiezas a registrar y exportas al terminar.

> La app **parte siempre sin datos**: los datos viven en los archivos (planilla +
> respaldo), no en el navegador. Por eso se cargan a diario.

## El día a día

- **Tablero** = qué hacer hoy: % operativos, MP del mes, pendientes, **operatividad**
  (Operativos / Servicio Técnico / No operativos) y **«Sin seguimiento»** (lo detenido
  sin actualizar hace ≥ N días).
- **Cambiar el estado real de un equipo:** ábrelo (ficha) o toca su badge de estado en
  **Equipos** → «Cambiar estado» (estado, motivo, responsable). Queda en la bitácora con
  fecha/hora. **El estado real lo manda la app**, no el `.xlsm`.
- **Pendientes:** créalos, complétalos/reábrelos, añade seguimiento. El panel marca los
  **vencidos** y los **sin seguimiento**.
- **Registrar MP / Correctivo:** desde la ficha del equipo.

## Al terminar el día (¡importante!)

- Pulsa **«Respaldar ahora»** → descarga **JSON** (para recargar mañana) **+ Excel**
  compartible. Cada exportación lleva un **ID único**.
- ¿No quieres pulsar nada? **«Más» → «Elegir carpeta de auto-guardado»** (Chrome/Edge):
  guarda JSON + Excel solo, también al cerrar. **Es la forma 100% segura de no perder datos.**
- Si cierras con cambios sin guardar y no usas carpeta, la app **avisa** y descarga un
  **respaldo de seguridad** (mejor esfuerzo).

## El Excel para compartir

Hojas autoexplicativas (se puede trabajar sin la app): **Resumen** (con ID e
indicadores), **Equipos** (con estado y última actualización), **Bitácora**
(cronología), **Pendientes**, **Servicio Técnico**, **No Operativos**, **Sin
seguimiento**, y el detalle.

## Atajos

`⌘K` / `Ctrl+K` paleta de comandos · `/` buscar · `j`/`k` moverse por la tabla ·
**Enter** abre · botones de **tema** (☾) y **densidad** (▦) en la barra superior.

## Reinicio

**«Más» → «Reset total»** (o `index.html#reset`) borra todo y vuelve a empezar.
