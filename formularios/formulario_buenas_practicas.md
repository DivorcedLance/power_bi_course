# Formulario de Buenas Prácticas · Power BI

> Checklist razonado de lo que **sí** y lo que **no** se debe hacer en un proyecto Power BI. Estructurado por área. Cada sección tiene un principio general, las prácticas concretas, y los anti-patrones más comunes.

## Índice

1. [Modelado de datos](#1-modelado-de-datos)
2. [Power Query (ETL)](#2-power-query-etl)
3. [DAX](#3-dax)
4. [Visualización](#4-visualización)
5. [Rendimiento](#5-rendimiento)
6. [Documentación y nomenclatura](#6-documentación-y-nomenclatura)
7. [Despliegue y ciclo de vida](#7-despliegue-y-ciclo-de-vida)
8. [Seguridad](#8-seguridad)
9. [Checklist final pre-publicación](#9-checklist-final-pre-publicación)

---

## 1. Modelado de datos

### Principio general

> El modelo debe responder a **una sola pregunta por cada nivel de análisis**, no a todas a la vez. Un buen modelo en estrella es predecible, performante y fácil de extender.

### Prácticas recomendadas

- **Esquema en estrella**: una tabla de hechos rodeada de dimensiones. Nunca雪花 (copo de nieve) a menos que sea estrictamente necesario.
- **Una sola tabla de fechas** marcada como tal. Dimensiones de rol con `USERELATIONSHIP`.
- **Relaciones 1:N** con dirección única desde la dimensión al hecho. Bidireccional solo si la necesitas de verdad y entiendes sus consecuencias.
- **Oculta las claves foráneas** en la tabla de hechos (FKs) si no se usan en visuales. Reduce el ruido en el panel de campos.
- **Tabla de medidas** (`_measures`) para alojar todas las medidas. No las disperses entre las tablas de negocio.
- **No mezcles niveles de granularidad** en la misma tabla de hechos. Si una fila es una línea de pedido, no le añadas columnas agregadas.

### Anti-patrones comunes

- ❌ Una sola tabla "plana" con todo. Crece rápido, filtra mal, es difícil de mantener.
- ❌ Tablas calendario duplicadas (`dim_date_order`, `dim_date_due`, ...).
- ❌ Relaciones bidireccionales "por si acaso". Generan ambigüedad.
- ❌ Mezcla de hechos en la misma tabla (e.g. ventas y objetivos). Cada hecho a su tabla.

---

## 2. Power Query (ETL)

### Prácticas recomendadas

- **Tipar todo** correctamente desde el primer paso. Un tipo mal asignado contamina el modelo.
- **Renombrar columnas** a `snake_case` sin espacios ni acentos. Ahorra comillas y errores en DAX.
- **Reference > Duplicate.** Si necesitas una variante de una consulta, usa Reference y transforma a partir de ahí.
- **Activa el query folding** cuando sea posible: usa conectores nativos (SQL, OData) en lugar de CSV/Excel cuando puedas.
- **Quita columnas y filas que no necesites** antes de cargar. Menos datos → modelo más pequeño.
- **Un paso = una transformación discreta.** Nombra los pasos con sentido (`"Quitados duplicados"`, no `"Quitados duplicados1"`).
- **Parametriza lo variable:** rutas de ficheros, años, regiones, etc. Convierte lo que se repite en parámetro.

### Anti-patrones comunes

- ❌ Dejar la columna `Row ID` o columnas de auditoría en el modelo. No aportan nada y ocupan espacio.
- ❌ Hacer `Merge` para crear columnas que podrían resolverse en DAX con `LOOKUPVALUE` (más eficiente).
- ❌ Calcular cosas en Power Query que requieren el filter context (totales, %). Eso es DAX.
- ❌ Queries de Power Query con 30+ pasos porque nadie se atreve a tocar nada. Refactoriza.

---

## 3. DAX

### Prácticas recomendadas

- **Variables (`VAR`) siempre.** Para cualquier sub-cálculo que se repita o que mejore la legibilidad.
- **`DIVIDE` en lugar de `/`.** Maneja el BLANK correctamente.
- **Medidas para todo lo agregado.** Columnas calculadas solo para atributos estáticos de la fila.
- **Nombres de medida descriptivos:** `Sales YoY %`, no `medida_17`.
- **Documenta cada medida:** descripción, formato, carpeta. Sin excepciones.
- **Usa `REMOVEFILTERS` en lugar de `ALL`** cuando solo quieras quitar filtros. Es más explícito.
- **Evita `FILTER` dentro de `CALCULATE`** cuando puedas usar un predicado directo. `CALCULATE([Sales], dim_product[category] = "Tech")` es más rápido que `CALCULATE([Sales], FILTER(ALL(dim_product), dim_product[category] = "Tech"))`.
- **Time intelligence solo con tabla de fechas marcada.** Verifica el icono de "table" en la vista de modelo.

### Anti-patrones comunes

- ❌ Columnas calculadas con `FILTER` y funciones de agregado. Es trabajo de medida, no de columna.
- ❌ `IF` anidados de 5 niveles. Usa `SWITCH(TRUE(), ...)` o una tabla de búsqueda.
- ❌ `CALCULATE([Sales])` sin filtros, "para asegurarme". Si no añades filtro, no necesitas `CALCULATE`.
- ❌ Medidas que devuelven texto hardcodeado. Si la medida es "tipo texto", probablemente debería ser una columna.
- ❌ Tests con `IF(HASONEVALUE(...), VALUES(...), "Todos")` en lugar de `SELECTEDVALUE(..., "Todos")`. Lo segundo es idiomático.

---

## 4. Visualización

### Prácticas recomendadas

- **Elige el visual por la pregunta, no por la estética.** Si la pregunta es "evolución", línea. Si es "comparar", barra. Si es "composición", treemap o stacked bar.
- **Ordena los ejes con sentido.** Categorías por valor descendente, no alfabético. Tiempo cronológico.
- **Reduce el ruido.** Quita gridlines, leyendas redundantes, títulos obvios. El color debe codificar información, no decorar.
- **Máximo 5-7 categorías en pie/donut.** Más allá, usa treemap o bar.
- **KPI cards**: número grande, comparación debajo (`vs 1.234 PY`), icono de tendencia.
- **Tablas**: usarlas para detalle, no para análisis. Para análisis, scatter, matrix o bar.
- **Tooltip pages** para detalle al pasar el ratón. Más limpio que mostrar 20 columnas en una tabla.
- **Tooltips en medidas y títulos** con `CONCATENATEX` o `FORMAT` para enriquecer contexto.
- **Tema corporativo** desde el inicio. No cambies colores manualmente en cada visual.

### Anti-patrones comunes

- ❌ Gráficos 3D. Distorsionan la percepción y no aportan nada.
- ❌ Pie charts con 12+ categorías. Ininteligible.
- ❌ Duplicar la misma información en varios visuales sin propósito.
- ❌ Ejes que no empiezan en 0 en gráficos de barra (falsea la comparación). En líneas, sí puede ser útil.
- ❌ Títulos de visual en blanco porque "ya se ve". Un buen título dice la conclusión, no el contenido.

---

## 5. Rendimiento

### Prácticas recomendadas

- **Reduce cardinalidad.** Convierte columnas de texto a enteros cuando puedas (lookup a otra tabla).
- **Tipos de datos óptimos:**
  - `Entero` en vez de `Decimal` cuando no se necesita precisión.
  - `Fecha` en vez de `Fecha y hora` cuando la hora no importa.
  - `Texto` con `No resumir` en columnas de alta cardinalidad que no se agregan.
- **Mide antes de optimizar.** Performance Analyzer, DAX Studio, VertiPaq Analyzer. Sin medida, es intuición.
- **Aggregations** para tablas de hechos >5M filas.
- **Elimina columnas y tablas no usadas** antes de publicar. Cada MB cuenta en Service.
- **Variables en medidas pesadas.** Permiten que el motor almacene el resultado intermedio.

### Anti-patrones comunes

- ❌ "Lo voy a optimizar luego" sin haber medido primero.
- ❌ Crear columnas calculadas en DAX que se podían hacer en Power Query.
- ❌ `FILTER(ALL(table), ...)` sobre tablas de hechos grandes. O(n) en la tabla completa.
- ❌ Relaciones bidireccionales o many-to-many "porque rinde bien en local". No escala.

---

## 6. Documentación y nomenclatura

### Convención de nombres (sugerida)

| Elemento | Patrón | Ejemplo |
|---|---|---|
| Tabla de hechos | `fct_` + singular | `fct_sales`, `fct_orders` |
| Tabla dimensión | `dim_` + singular | `dim_customer`, `dim_product` |
| Tabla calendario | `dim_date` | `dim_date` |
| Tabla auxiliar | prefijo descriptivo | `_measures`, `TopN Params`, `WhatIf Discount` |
| Medida | Nombre de negocio | `Total Sales`, `Profit Margin %` |
| Columna | `snake_case`, sin acentos | `order_date`, `customer_id` |
| Relación | no se nombra | — |

### Qué documentar

- **En cada medida:** nombre, descripción de 1 línea, formato, carpeta temática. Visible en el panel de modelo.
- **En el dataset:** descripción general, origen de datos, frecuencia de actualización, contacto de soporte.
- **Cambios relevantes:** un `CHANGELOG.md` o una página oculta en el informe. "2026-06-15: añadido RLS por región."
- **Decisiones de modelo no obvias:** "La tabla de hechos incluye líneas de pedido canceladas (flag `is_cancelled`). Decisión de negocio de 2026-04."

---

## 7. Despliegue y ciclo de vida

### Prácticas recomendadas

- **Workspaces dedicados.** Nunca publiques a "Mi área de trabajo personal" si otros consumen el informe.
- **Pipeline Dev → Test → Prod.** Aunque sea manual con `Save as` y un checklist, formalízalo.
- **Refresh programado y monitorizado.** Configura alertas de fallo. Un informe que no actualiza es un informe muerto.
- **Versiona el `.pbix`** (o mejor, el `.pbip` que es texto plano) en Git. Sí, los `.pbix` se pueden ignorar y trabajar con `.pbip`.
- **Pruebas de humo después de cada despliegue:** abrir el informe, aplicar un slicer, ver un KPI. 5 minutos que ahorran horas.

### Anti-patrones comunes

- ❌ Publicar la versión de pruebas sobre la de producción porque "es la misma".
- ❌ Cambios en el modelo en Prod sin pasar por Test.
- ❌ Sin gateway configurado y refresh fallando silenciosamente.
- ❌ Permisos manuales a usuarios sueltos en lugar de grupos de seguridad + App.

---

## 8. Seguridad

### Prácticas recomendadas

- **Row-Level Security (RLS)** para todo dato que sea sensible (ventas por región, RRHH por departamento, etc.).
- **RLS dinámica con `USERPRINCIPALNAME()`** y una tabla de mapping. Más mantenible que roles estáticos.
- **Testear RLS en Service** con un usuario real, no solo "View as Role" en Desktop.
- **No publiques datasets en Service sin la fuente de datos configurada de forma segura.** Gateway, OAuth, lo que toque.
- **Auditoría periódica:** cada trimestre, revisa quién tiene acceso y qué ve. Altas y bajas.

### Anti-patrones comunes

- ❌ Confiar en "el filtro del slicer" para seguridad. Los slicers no son seguridad: el usuario puede quitarlos.
- ❌ RLS con `USERPRINCIPALNAME() = "[email protected]"` hardcodeado.
- ❌ Mezcla de datos en el mismo informe con niveles de sensibilidad distintos (datos personales y KPIs anónimos). Mejor dos informes.

---

## 9. Checklist final pre-publicación

Antes de dar un informe por bueno, verifica:

### Datos
- [ ] El modelo se actualiza correctamente (manual y programado).
- [ ] No hay errores en Power Query al refrescar.
- [ ] El modelo está tipado y no contiene columnas obvias no usadas.
- [ ] La tabla de fechas está marcada y todas las relaciones temporales funcionan.

### DAX
- [ ] Todas las medidas tienen `DIVIDE` para divisiones.
- [ ] Todas las medidas con sub-cálculos usan `VAR`.
- [ ] Cada medida tiene descripción, formato y carpeta.
- [ ] Las medidas de time intelligence devuelven BLANK en el primer año (no error).

### Visualización
- [ ] Todos los títulos de visual dicen la conclusión, no el contenido.
- [ ] Los slicers afectan a todas las visuales de la página.
- [ ] El tooltip aporta información útil, no repite el visual.
- [ ] No hay visuales con un solo punto de datos o sin sentido.
- [ ] El informe se entiende en 30 segundos por un tercero.

### Rendimiento
- [ ] Performance Analyzer: ningún DAX query > 500 ms en visual crítica.
- [ ] Tamaño del modelo < 50 MB (ajustar según el caso).
- [ ] Sin columnas de fecha como `DateTime` si no se necesita la hora.

### Documentación
- [ ] Descripción del dataset rellenada.
- [ ] CHANGELOG actualizado.
- [ ] Contacto de soporte identificado.

### Despliegue
- [ ] Workspace dedicado (no "Mi área personal").
- [ ] Permisos vía grupo de seguridad, no usuarios sueltos.
- [ ] RLS testada con un usuario real.
- [ ] Refresh programado y validado.
- [ ] App publicada con permisos correctos.
