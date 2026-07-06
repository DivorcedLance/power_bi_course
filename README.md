# Curso de Power BI · Nivel intermedio

Material del curso de Power BI a nivel intermedio. Estructura:

- **`clases/`** — Ejercicios prácticos paso a paso, en orden de dificultad creciente.
- **`formularios/`** — Hojas de referencia rápida (DAX, buenas prácticas).
- **`referencias/`** — Catálogos y guías de consulta (elementos visuales).

## Clases

| # | Tema | Dificultad | Datos |
|---|---|---|---|
| 1 | [Dashboard de análisis de ventas con el dataset *Superstore*](clases/class_01_analisis_ventas_superstore/README.md) | Intermedio (fundamentos) | `superstore.csv` (en la carpeta) |
| 2 | [DAX avanzado y patrones analíticos](clases/class_02_dax_avanzado_patrones_analiticos/README.md) | Intermedio (DAX) | Mismo `superstore.csv` de la Clase 1 |
| 3 | [Modelado dimensional avanzado](clases/class_03_modelado_dimensional_avanzado/README.md) | Intermedio-avanzado (modelo) | 6 CSVs en `clases/class_03.../data/` (Sales and Marketing Sample) |
| 4 | [Producción, optimización y despliegue](clases/class_04_produccion_y_despliegue/README.md) | Avanzado (producción) | Continúa Clase 3 |

> **Convención del curso**: todos los datos están descargados en la carpeta de cada clase. No necesitas ir a buscar nada a internet salvo que quieras la fuente original (los enlaces están en cada README).

## Formularios (referencia rápida)

- [Formulario DAX](formularios/formulario_dax.md) — Funciones, patrones y reglas de oro de DAX.
- [Formulario de buenas prácticas](formularios/formulario_buenas_practicas.md) — Modelado, DAX, visualización, rendimiento, seguridad, despliegue.

## Referencias

- [Elementos visuales de Power BI](referencias/elementos_visuales.md) — Catálogo de visuales nativos y personalizados con guía de uso.
- [Importación de datos y Power Query](referencias/importacion_y_power_query.md) — CSV, Excel, Web, Web.Contents, JSON, autenticación, paginación, query folding.
- [Modelo en estrella](referencias/modelo_estrella.md) — Dimensiones, hechos, relaciones, SCD, variantes y anti-patrones.

## Requisitos generales

- **Power BI Desktop** (última versión): [powerbi.microsoft.com](https://powerbi.microsoft.com/).
- Para Clase 4: cuenta de **Power BI Service** Pro o Premium Per User (PPU), DAX Studio opcional.
- Lectura recomendada de los formularios antes de empezar las clases (al menos el de DAX).

## Orden recomendado

1. Leer `formularios/formulario_dax.md` y `formularios/formulario_buenas_practicas.md` (secciones 1-3: Modelado, Power Query, DAX).
2. Si vas a usar web/APIs, repasa `referencias/importacion_y_power_query.md` antes de la Clase 3.
3. Seguir `clases/` en orden. Cada clase asume la anterior como base.
4. Volver a los formularios y referencias cuando una clase toque un tema concreto.
