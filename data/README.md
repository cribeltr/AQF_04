# Respaldos de datos — MP 2026

Copia de seguridad versionada de los registros reales, para que **no se pierdan**
aunque se borre el navegador. Estado al **18-06-2026**.

## Archivos

| Archivo | Qué es | Uso |
|---|---|---|
| **`Respaldo_GMP2026_20260618.json`** | **Respaldo real de la app anterior** (`tipo: respaldo-gmp2026`, v2): **79 mantenciones, 63 pendientes, 14 correctivos, 5 empresas, 1 ingeniero**. | **Fuente reimportable.** En la app: «Importar respaldo anterior». |
| `Registros_GMP2026_20260618.xlsx` | Exportación legible de esos mismos datos (hojas Equipos, Programación, Mantenciones, Pendientes, Correctivos). | Referencia / lectura. Redundante con el JSON. |
| `Sistema_Gestion_MP2026_Integrado.xlsx` | Planilla integrada **más reciente**: 15 OT correctivas con expediente detallado (envíos, visitas, línea de compra, reparación en terreno) y un módulo de pendientes con taxonomía propia (82 registros). | **Importable** en la app: «Importar planilla integrada». |
| `Sistema_Gestion_MP2026_Integrado_VF.xlsx` | Planilla integrada (variante VF). | Archivo histórico / consulta. |

## Cómo recuperar los registros en la app

### A) Respaldo de la app anterior (mantenciones + pendientes + correctivos base)

1. Abre `../index.html`.
2. Carga primero `../sample-data/ProgramaciónMP2026.xlsm` (o tu `.xlsm` oficial actual).
3. Menú lateral → **«Importar respaldo anterior»** → elige
   `Respaldo_GMP2026_20260618.json`.
4. Verás las 79 mantenciones, 63 pendientes y 14 correctivos.

### B) Planilla integrada (expedientes correctivos detallados + pendientes extra)

5. Menú lateral → **«Importar planilla integrada»** → elige
   `Sistema_Gestion_MP2026_Integrado.xlsx`.
6. Los **correctivos** se **fusionan por Folio SIGEM** (los 14 existentes se
   enriquecen con sus envíos/visitas/compras y se añade 1 OT nueva → 15 en total).
   Los **pendientes** (82) se añaden marcados con **fuente «integrado»** para que
   los distingas de los 63 del respaldo y depures duplicados a mano (filtro
   por fuente en la vista Pendientes).
7. Usa **«Respaldar ahora»** para generar tu copia consolidada JSON + Excel.

> Importación verificada — Respaldo JSON: 79 / 63 / 14 (+23 avances).
> Planilla integrada: +1 correctivo nuevo, 14 enriquecidos (+15 avances),
> 82 pendientes (otro 30 · documental 16 · reprogramación 36). Sin tipos sin
> mapear. Los `equipos` y `registros` **no** se importan: se re-leen del `.xlsm`
> oficial (la fuente de verdad).
