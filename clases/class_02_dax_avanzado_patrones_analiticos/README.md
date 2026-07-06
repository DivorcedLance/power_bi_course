# Clase 2 · DAX avanzado y patrones analíticos (Superstore)

## Descripción del ejercicio

Partiendo del modelo en estrella construido en la **Clase 1**, ampliar el informe con una segunda página centrada en **análisis avanzado**, no en transformación. El objetivo es practicar patrones DAX que se usan en cualquier proyecto real, sobre datos que ya conoces, para centrarte en el razonamiento analítico.

El informe debe permitir responder a:

- ¿Qué productos/ca테고rías explican el 80% de las ventas (Pareto / ABC)?
- ¿Cómo se comporta la retención de clientes mes a mes desde su primera compra (cohorte)?
- ¿Cómo segmentar a los clientes por Recency / Frequency / Monetary (RFM)?
- ¿Cuáles son los Top N productos en cada métrica (Top N dinámico, no hardcodeado)?
- ¿Qué pasa cuando hago clic en un cliente o un producto (drill-through al detalle)?

## Recursos necesarios

| Recurso | Detalle |
|---|---|
| `.pbix` de la Clase 1 | El modelo `fct_sales` + `dim_*` ya creado. |
| Power BI Desktop | Última versión. |
| Dataset | El `superstore.csv` ya está en la carpeta de la Clase 1, no hay que traerlo. |

> Si partiste de cero, rehaz la Clase 1 primero. Toda esta clase asume que el modelo está limpio y que `dim_date` está marcada como tabla de fechas.

## Lo que vas a practicar

- DAX idiomático con variables (`VAR` / `RETURN`).
- `RANKX` y `TOPN` aplicados a ranking dinámico.
- Funciones de *time intelligence* con contexto de cliente: `CALCULATE`, `FILTER`, `MINX`.
- Patrón ABC / Pareto con `% acumulado`.
- Análisis de cohortes con `MINX` + `SUMMARIZE`.
- Segmentación RFM con `SWITCH` y percentiles.
- Drill-through entre páginas.
- Formato condicional con medidas DAX (no solo por reglas estáticas).

---

## Paso a paso

### 1. Página "ABC / Pareto"

Construir una matriz con `Product Name`, sus ventas, el % sobre el total y el % acumulado, y clasificar cada producto como A / B / C.

```dax
Product Sales = SUM ( fct_sales[sales] )

Product Sales % =
DIVIDE (
    [Product Sales],
    CALCULATE ( [Product Sales], ALL ( dim_product ) )
)

Product Cumulative % =
VAR vProductSales = [Product Sales]
VAR vTotalSales   = CALCULATE ( [Product Sales], ALL ( dim_product ) )
VAR vRunningTotal =
    CALCULATE (
        [Product Sales],
        TOPN (
            ...rank del producto actual...,
            dim_product,
            [Product Sales], DESC
        )
    )
RETURN
    DIVIDE ( vRunningTotal, vTotalSales )
```

Truco: para el acumulado, una alternativa más eficiente y legible es ordenar todos los productos por ventas y calcular el *running total* con `WINDOW` (DAX 2.x). Si trabajas con versiones anteriores, usa el patrón `FILTER ( ..., [Product Sales] >= ... )`.

Clasificación ABC:

```dax
Product ABC Class =
SWITCH (
    TRUE (),
    [Product Cumulative %] <= 0.80, "A",
    [Product Cumulative %] <= 0.95, "B",
    "C"
)
```

Visualización: gráfico combinado (columna + línea). Eje X: `Product Name` (Top 50). Columna: `Product Sales %`. Línea: `Product Cumulative %`. Línea de referencia horizontal en 80%.

### 2. Análisis de cohortes

Una cohorte es el conjunto de clientes cuya primera compra fue en el mismo mes. Lo que se mide es cuántos de esos clientes vuelven a comprar en los meses siguientes.

```dax
Customer First Purchase =
CALCULATE (
    MIN ( fct_sales[order_date] ),
    ALLEXCEPT ( fct_sales, dim_customer[customer_id] )
)
```

Crear `cohort_month = FORMAT ( [Customer First Purchase], "YYYY-MM" )` y `months_since_cohort = DATEDIFF ( [Customer First Purchase], MAX ( fct_sales[order_date] ), MONTH )`.

Matriz final:

- Filas: `cohort_month`.
- Columnas: `months_since_cohort` (0, 1, 2, ...).
- Valores: `DISTINCTCOUNT(dim_customer[customer_id])` o el % de retención sobre el tamaño inicial de la cohorte.

Aplicar formato condicional de color (escala de un color, máximo = tamaño de cohorte inicial).

### 3. Segmentación RFM

```dax
Customer Recency = DATEDIFF ( [Customer Last Purchase], TODAY (), DAY )
Customer Frequency = DISTINCTCOUNT ( fct_sales[order_id] )
Customer Monetary = [Total Sales]
```

`Customer Last Purchase` se calcula igual que `Customer First Purchase` pero con `MAX` en lugar de `MIN`.

Segmentación (ejemplo simple, ajustable a negocio):

```dax
Customer Segment =
VAR r = [Customer Recency]
VAR f = [Customer Frequency]
VAR m = [Customer Monetary]
VAR mp =
    CALCULATE (
        PERCENTILEX.INC ( ..., [Customer Monetary], 0.75 ),
        ALLEXCEPT ( dim_customer )
    )
RETURN
    SWITCH (
        TRUE (),
        r <= 30  && f >= 3 && m >= mp, "Champions",
        r <= 60  && f >= 2,              "Loyal",
        r <= 90,                          "Potential",
        r <= 180,                         "At Risk",
        "Hibernating"
    )
```

Visualización: scatter chart con eje X = `Recency`, eje Y = `Frequency`, tamaño = `Monetary`, leyenda = `Customer Segment`. Encima, tarjetas con `Count of Customer Segment`.

### 4. Top N dinámico

Crear una **tabla auxiliar** (paramétrica) con `Top N Value = GENERATESERIES(5, 50, 5)` y una columna `Top N Value` con valores 5, 10, 15, ..., 50. Marcarla como "no relacionar con nada" (es una tabla desconectada).

```dax
Top N Sales =
VAR vN = SELECTEDVALUE ( 'Top N Params'[Top N Value], 10 )
VAR vTbl =
    TOPN (
        vN,
        SUMMARIZE ( fct_sales, dim_product[product_name], "S", [Total Sales] ),
        [S], DESC
    )
RETURN
    SUMX ( vTbl, [S] )
```

Más limpio con una **medida filtro**:

```dax
Is Top N Product =
VAR vN = SELECTEDVALUE ( 'Top N Params'[Top N Value], 10 )
VAR vRank =
    RANKX (
        ALL ( dim_product[product_name] ),
        [Total Sales],, DESC
    )
RETURN
    IF ( vRank <= vN, 1, 0 )
```

Aplicar `Is Top N Product = 1` como filtro a nivel de visual. Slicer con la columna de la tabla paramétrica controla el "N".

### 5. Drill-through al detalle

1. Duplicar la página de resumen y renombrarla `Detalle Cliente`.
2. Quitar todos los slicers y filtros visuales.
3. Añadir el campo `Customer ID` al área **Drill-through** de la página.
4. Crear visuales: una tabla con las líneas de pedido de ese cliente, KPIs (total gastado, margen medio, días desde última compra), mini-chart de evolución.
5. En la página de resumen, añadir un botón con acción **Drill through** que apunte a `Detalle Cliente` y filtro de página. Esto permite a un usuario ir de la lista de clientes al detalle con un clic.

> **Buena práctica**: el drill-through debe respetar la granularidad. En la página de detalle, todas las visuales deberían tener el mismo nivel (línea de pedido del cliente), no mezclar "ventas por categoría" (que es una agregación) con la tabla de líneas.

### 6. Formato condicional con DAX

- **Iconos** en KPI cards: `IF ( [Sales YoY %] > 0, "▲", "▼" )`.
- **Color de fondo** en una tabla de productos: medida que devuelva un color hex según `Profit`:

```dax
Product Profit Color =
VAR p = [Product Sales] * [Product Margin %]
RETURN
    SWITCH (
        TRUE (),
        p < 0, "#F4C7C3",
        p < 100, "#FFE699",
        "#C6EFCE"
    )
```

Aplicar en **Formato condicional → Color de fondo → Valor de campo** y seleccionar la medida.

---

## Explicación y buenas prácticas

### Por qué `VAR` siempre

Una medida DAX sin variables es:

```dax
Bad : Margin = DIVIDE ( SUM ( fct_sales[profit] ), SUM ( fct_sales[sales] ) )
```

Con variables:

```dax
Good: Margin =
VAR profit = SUM ( fct_sales[profit] )
VAR sales  = SUM ( fct_sales[sales] )
RETURN DIVIDE ( profit, sales )
```

Razones concretas:

1. **Performance**: la expresión a la derecha del `VAR` se evalúa una sola vez por celda. Sin variables, si reutilizas el cálculo, el motor puede recomputarlo.
2. **Legibilidad**: una medida larga se lee como una receta. `vRank`, `vN`, `vTotal` explican la intención.
3. **Debug**: puedes aislar cada variable en una medida aparte o comentarla para encontrar bugs.

### `RANKX` vs `TOPN`

- `RANKX` devuelve un número (la posición). Útil para filtros, formato condicional y para construir rankings virtuales sin perder el contexto de la visual.
- `TOPN` devuelve una **tabla**. Útil cuando quieres materializar un subconjunto y operar sobre él (sumar, promediar, etc.).

Regla práctica: si necesitas "los 10 primeros X" como *resultado*, usa `TOPN`. Si necesitas saber "el rango de X" como *atributo*, usa `RANKX`.

### `% acumulado` y orden

El Pareto depende crucialmente del orden. Si ordenas alfabéticamente, el acumulado es inútil. La forma correcta:

1. Definir el ranking con `RANKX` ordenado por la métrica (descendente).
2. Calcular el acumulado como suma de la métrica para todos los productos con ranking ≤ el actual.

Esto se puede hacer con `WINDOW` (DAX 2.x, en modo *preview*) o, en DAX clásico, con un patrón `FILTER` + `CALCULATE`.

### Cohortes: el detalle de implementación

`ALLEXCEPT(fct_sales, dim_customer[customer_id])` es la forma idiomática de "para este cliente, dame todas sus líneas de pedido". A partir de ahí:

- `MIN ( fct_sales[order_date] )` → fecha de primera compra.
- `MAX ( fct_sales[order_date] )` → fecha de última compra.
- `COUNTROWS ( VALUES ( fct_sales[order_id] ) )` → número de pedidos.

Si usas `ALL(dim_customer)` o nada, el contexto de filtro de la visual filtrará la primera/última compra, lo cual es un bug muy común.

### Por qué una tabla paramétrica y no un parámetro What-if

- El **parámetro What-if** crea automáticamente una tabla desconectada con valores numéricos, pero solo acepta números. Está pensado para simulaciones ("¿qué pasa si subo el descuento un 5%?").
- Una **tabla paramétrica** hecha a mano con `GENERATESERIES` o `DATATABLE` puede tener texto, jerarquías, etc. Útil para segmentadores cualitativos ("Top 5 / 10 / 20").

### Drill-through: lo que la gente se equivoca

- Confundir **drill-through** con **drill-down**. El drill-down navega *dentro* de una jerarquía (Año → Mes). El drill-through salta a *otra página* con un contexto ya aplicado.
- Poner todos los visuales en la página de drill-through sin relación entre sí. La página debe contar **una sola historia** con el mismo nivel de granularidad.
- Olvidar el botón "Atrás" en la página de detalle. Power BI añade uno automático, pero personalizarlo con bookmark mejora la experiencia.

### Formato condicional con DAX: cuándo merece la pena

- Para reglas simples (mayor que X, en verde), basta con el formato condicional por reglas.
- Para reglas que **dependen del contexto de la fila** (margen > 10% Y ventas > media), necesitas una medida.
- Para iconos/estados compuestos (semaforos, flechas con tooltip), también necesitas DAX.

---

## Checklist de autoevaluación

- [ ] La página de Pareto muestra línea de referencia al 80% y los productos A, B y C distinguibles visualmente.
- [ ] La matriz de cohortes tiene `cohort_month` en filas, `months_since_cohort` en columnas y un heatmap con color.
- [ ] La segmentación RFM tiene al menos 4 categorías y el scatter muestra la leyenda.
- [ ] El slicer "Top N" cambia el contenido de la visual asociada.
- [ ] El drill-through desde la lista de clientes lleva a una página de detalle coherente.
- [ ] Todas las medidas usan `VAR` para sub-cálculos.
- [ ] El formato condicional en la tabla de productos se actualiza con los slicers.

## Extensiones posibles

- Sustituir la segmentación RFM por una basada en **K-Means** ejecutado en Python (Python visual) para clientes con compras > 1.
- Añadir una página de **market basket**: "los clientes que compran X también compran Y" usando un join self-join sobre `fct_sales` por `order_id`.
- Convertir el Top N dinámico en un **ranking móvil** que se actualice al cambiar el slicer de tiempo.
- Usar **field parameters** (DAX 2.x) para que el usuario elija la métrica del ranking.
