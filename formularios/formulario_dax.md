# Formulario DAX · Referencia rápida

> Hoja de consulta rápida para escribir DAX en Power BI. Organizada por categoría. Cada función incluye firma, descripción breve y un ejemplo típico. Para profundizar, ver las clases y la documentación oficial de Microsoft.

## Índice

1. [Agregaciones básicas](#1-agregaciones-básicas)
2. [Iteradores (funciones X)](#2-iteradores-funciones-x)
3. [Lógicas](#3-lógicas)
4. [Información y contexto](#4-información-y-contexto)
5. [Filter context (lo más importante)](#5-filter-context-lo-más-importante)
6. [Time intelligence](#6-time-intelligence)
7. [Texto](#7-texto)
8. [Fecha y hora](#8-fecha-y-hora)
9. [Funciones de tabla](#9-funciones-de-tabla)
10. [Ranking](#10-ranking)
11. [Parent-child](#11-parent-child)
12. [Estadísticas](#12-estadísticas)
13. [Patrones recurrentes](#13-patrones-recurrentes)
14. [Reglas de oro](#14-reglas-de-oro)

---

## 1. Agregaciones básicas

| Función | Firma | Descripción |
|---|---|---|
| `SUM` | `SUM(<column>)` | Suma de valores numéricos. |
| `AVERAGE` | `AVERAGE(<column>)` | Media aritmética. |
| `COUNT` | `COUNT(<column>)` | Cuenta valores numéricos no vacíos. |
| `COUNTA` | `COUNTA(<column>)` | Cuenta valores no vacíos de cualquier tipo. |
| `COUNTROWS` | `COUNTROWS(<table>)` | Cuenta filas de una tabla. |
| `MIN` / `MAX` | `MIN(<column>)` | Mínimo / máximo. |
| `DISTINCTCOUNT` | `DISTINCTCOUNT(<column>)` | Cuenta valores únicos. |

```dax
Total Sales   = SUM ( fct_sales[sales_amount] )
Unique Customers = DISTINCTCOUNT ( fct_sales[customer_key] )
```

## 2. Iteradores (funciones X)

Iteran fila a fila sobre una tabla. Usan el **row context** para evaluar una expresión.

| Función | Descripción |
|---|---|
| `SUMX(<tabla>, <expr>)` | Suma una expresión evaluada por fila. |
| `AVERAGEX` | Media por fila. |
| `MINX` / `MAXX` | Mínimo / máximo por fila. |
| `COUNTX` | Cuenta filas donde la expresión da un número. |
| `FILTER(<tabla>, <cond>)` | Devuelve la subtabla que cumple la condición. |

```dax
Sales with Margin =
SUMX (
    fct_sales,
    fct_sales[sales_amount] - fct_sales[total_cost]
)
```

> **Regla**: usa funciones X solo cuando necesites operar por fila (ratios, multiplicaciones entre columnas, etc.). Para sumar una columna directamente, `SUM` es más rápido.

## 3. Lógicas

| Función | Descripción |
|---|---|
| `IF(cond, a, b)` | Condicional. |
| `SWITCH(TRUE(), cond1, v1, cond2, v2, ..., else)` | Equivalente a `IF` anidados, más legible. |
| `AND` / `OR` / `NOT` | Operadores lógicos. |
| `COALESCE(a, b, ...)` | Primer valor no BLANK. |
| `IFERROR(expr, alt)` | Si hay error, devuelve `alt`. |
| `TRUE()` / `FALSE()` | Constantes booleanas. |

```dax
Customer Segment =
SWITCH (
    TRUE (),
    [recency] <= 30, "Activo",
    [recency] <= 90, "Tibio",
    "Frío"
)
```

## 4. Información y contexto

| Función | Descripción |
|---|---|
| `ISBLANK(<expr>)` | ¿Es BLANK? |
| `ISFILTERED(<col>)` | ¿La columna tiene un filtro directo? |
| `ISCROSSFILTERED(<col>)` | ¿La columna está filtrada por otra relación? |
| `HASONEVALUE(<col>)` | ¿Hay un único valor visible? Útil en tarjetas. |
| `SELECTEDVALUE(<col>, [<alt>])` | Devuelve el valor único o `alt`. |
| `ISINSCOPE(<col>)` | ¿La columna está en el nivel actual de la jerarquía? |
| `ISSELECTEDMEASURE(<measure>)` | (Field parameters) ¿Está seleccionada esta medida? |
| `USERNAME()` | Usuario actual (en Service, devuelve el UPN). |
| `USERPRINCIPALNAME()` | Igual que `USERNAME()` pero más fiable en Service. |

```dax
Selected Region = SELECTEDVALUE ( dim_region[region], "Todos" )
```

## 5. Filter context (lo más importante)

`CALCULATE` es la función más importante de DAX. Cambia el **filter context**.

| Función | Descripción |
|---|---|
| `CALCULATE(<expr>, <filtro1>, <filtro2>, ...)` | Evalúa la expresión con un filter context modificado. |
| `CALCULATETABLE(<tabla>, ...)` | Igual, pero devuelve una tabla. |
| `ALL(<tabla_o_col>)` | Quita todos los filtros de la tabla/columna. |
| `ALLEXCEPT(<tabla>, <col1>, ...)` | Quita todos los filtros **excepto** los de las columnas indicadas. |
| `ALLSELECTED(<tabla_o_col>)` | Quita filtros pero mantiene los del contexto externo (slicers). |
| `KEEPFILTERS(<expr>)` | Añade un filtro sin reemplazar los existentes. |
| `REMOVEFILTERS(<tabla_o_col>)` | Igual que `ALL` (alias recomendado). |

```dax
% del total = DIVIDE ( [Sales], CALCULATE ( [Sales], ALL ( dim_product ) ) )

Sales YTD = CALCULATE ( [Sales], DATESYTD ( dim_date[Date] ) )
```

> **Regla**: cuando filtres por una sola columna de una tabla, usa `KEEPFILTERS` o una referencia directa a la columna. Cuando necesites "quitar todos los filtros", usa `ALL` o `REMOVEFILTERS`.

## 6. Time intelligence

Todas asumen una tabla de fechas marcada como tal.

| Función | Descripción |
|---|---|
| `TOTALYTD(<expr>, <fecha>, [<fin>])` | Acumulado desde inicio de año. |
| `TOTALQTD` / `TOTALMTD` | Acumulado desde inicio de trimestre / mes. |
| `SAMEPERIODLASTYEAR(<fecha>)` | Mismo periodo, año anterior. |
| `PARALLELPERIOD(<fecha>, <n>, <intervalo>)` | Periodo paralelo `n` unidades antes. |
| `DATEADD(<fecha>, <n>, <intervalo>)` | Fecha desplazada. |
| `DATESINPERIOD(<fecha>, <inicio>, <n>, <intervalo>)` | Conjunto de fechas en un periodo. |
| `DATESBETWEEN(<fecha>, <inicio>, <fin>)` | Conjunto de fechas entre dos. |
| `DATESYTD` / `DATESQTD` / `DATESMTD` | Devuelven el conjunto de fechas acumulado. |
| `PREVIOUSYEAR` / `PREVIOUSMONTH` / `PREVIOUSDAY` | Periodo anterior. |
| `NEXTYEAR` / `NEXTMONTH` / `NEXTDAY` | Periodo siguiente. |

```dax
Sales YoY % =
VAR vCurrent = [Total Sales]
VAR vPrior   = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( dim_date[Date] ) )
RETURN
    IF ( ISBLANK ( vPrior ), BLANK (), DIVIDE ( vCurrent - vPrior, vPrior ) )
```

> **Cuidado**: en el primer año, `SAMEPERIODLASTYEAR` devuelve BLANK. Usa `IF(ISBLANK(...), BLANK(), ...)` para evitar divisiones inválidas.

## 7. Texto

| Función | Descripción |
|---|---|
| `CONCATENATE(a, b)` | Concatena dos textos. Equivale a `a & b`. |
| `CONCATENATEX(<tabla>, <expr>, [<sep>])` | Concatena filas de una tabla. |
| `FORMAT(<valor>, <formato>)` | Convierte a texto con formato. |
| `LEFT` / `RIGHT` / `MID` | Subcadenas. |
| `LEN` | Longitud. |
| `FIND` / `SEARCH` | Posición (case sensitive / insensitive). |
| `SUBSTITUTE` / `REPLACE` | Reemplazo. |
| `TRIM` / `UPPER` / `LOWER` / `PROPER` | Limpieza y case. |
| `UNICHAR` / `UNICODE` | Unicode. |

```dax
FullName  = dim_customer[first_name] & " " & dim_customer[last_name]
YearMonth = FORMAT ( dim_date[Date], "YYYY-MM" )
```

## 8. Fecha y hora

| Función | Descripción |
|---|---|
| `DATE(año, mes, día)` | Construye una fecha. |
| `YEAR` / `MONTH` / `DAY` | Extrae componentes. |
| `WEEKDAY` / `WEEKNUM` | Día de la semana / número de semana. |
| `TODAY()` / `NOW()` | Hoy / ahora. |
| `DATEDIFF(<d1>, <d2>, <intervalo>)` | Diferencia entre fechas. |
| `EOMONTH(<fecha>, <n>)` | Último día del mes. |
| `EDATE(<fecha>, <n>)` | Fecha desplazada `n` meses. |
| `QUARTER` | Trimestre. |

```dax
Fiscal Year =
IF ( MONTH ( TODAY () ) >= 7, YEAR ( TODAY () ) + 1, YEAR ( TODAY () ) )
```

## 9. Funciones de tabla

Devuelven tablas, no valores. Se usan dentro de `CALCULATE`, `FILTER`, `SUMMARIZE`, etc.

| Función | Descripción |
|---|---|
| `FILTER(<tabla>, <cond>)` | Subtabla filtrada. |
| `ALL` / `ALLEXCEPT` / `ALLSELECTED` | Quitar filtros. |
| `VALUES(<col>)` | Valores únicos visibles (sin nulos). |
| `DISTINCT(<col>)` | Valores únicos (con nulos). |
| `ADDCOLUMNS(<tabla>, <nombre>, <expr>, ...)` | Añade columnas calculadas en una tabla. |
| `SELECTCOLUMNS(<tabla>, <nombre>, <expr>, ...)` | Proyecta columnas de una tabla. |
| `SUMMARIZE(<tabla>, <grupo1>, ..., [medidas])` | Agrupa (versión clásica). |
| `SUMMARIZECOLUMNS(<grupo>, ..., [medidas])` | Agrupa (versión optimizada). |
| `GENERATESERIES(<inicio>, <fin>, [<paso>])` | Secuencia numérica. |
| `DATATABLE(...)` | Tabla hardcodeada. |
| `CALENDAR(<inicio>, <fin>)` | Tabla calendario entre dos fechas. |
| `CALENDARAUTO([<último_mes>])` | Tabla calendario auto-detectada. |
| `TOPN(<n>, <tabla>, <orden>, [ASC/DESC])` | Primeros N por orden. |
| `UNION` / `INTERSECT` / `EXCEPT` | Set operations. |
| `NATURALINNERJOIN` / `NATURALLEFTOUTERJOIN` | Joins por columnas con mismo nombre. |
| `ROW(<nombre>, <expr>, ...)` | Una sola fila. |
| `GENERATE(<t1>, <t2_expr>)` | Producto cartesiano + filtro. |
| `CROSSJOIN(<t1>, <t2>)` | Producto cartesiano. |

```dax
dim_date = CALENDARAUTO ()
```

> **Regla**: prefiere `SUMMARIZECOLUMNS` sobre `SUMMARIZE`. Es más rápido y el optimizador lo trata mejor.

## 10. Ranking

| Función | Descripción |
|---|---|
| `RANKX(<tabla>, <expr>, [<valor>, <orden>, <saltar>])` | Rank de un valor en una tabla. |
| `RANK.EQ` / `RANK.AVG` | SQL-like (poco usadas en DAX). |

```dax
Product Rank =
RANKX (
    ALL ( dim_product[product_name] ),
    [Total Sales]
)
```

## 11. Parent-child

| Función | Descripción |
|---|---|
| `PATH(<id>, <parent_id>)` | Cadena de IDs desde el nodo hasta la raíz. |
| `PATHITEM(<path>, <posición>, [<tipo>])` | Devuelve el nodo en una posición. |
| `PATHLENGTH(<path>)` | Profundidad del nodo. |
| `PATHCONTAINS(<path>, <item>)` | ¿La ruta contiene el item? |
| `LOOKUPVALUE(<resultado>, <col1>, <val1>, ...)` | Busca un valor en una tabla. |

```dax
Employee Path     = PATH ( dim_employee[id], dim_employee[manager_id] )
Level 1 Manager   =
LOOKUPVALUE (
    dim_employee[name],
    dim_employee[id],
    PATHITEM ( dim_employee[Employee Path], 1 )
)
```

## 12. Estadísticas

| Función | Descripción |
|---|---|
| `MEDIAN` | Mediana. |
| `PERCENTILE.INC` / `PERCENTILE.EXC` | Percentiles (inclusivo / exclusivo). |
| `STDEV.S` / `STDEV.P` | Desviación estándar (muestra / población). |
| `VAR.S` / `VAR.P` | Varianza. |

```dax
P75 Sales =
PERCENTILEX.INC (
    dim_customer,
    [Total Sales],
    0.75
)
```

## 13. Patrones recurrentes

### Margen / Margen %

```dax
Profit     = SUM ( fct_sales[profit] )
Margin Pct = DIVIDE ( [Profit], [Total Sales] )
```

### YoY % (con manejo de BLANK)

```dax
Sales YoY % =
VAR vCurrent = [Total Sales]
VAR vPrior   = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( dim_date[Date] ) )
RETURN
    IF ( ISBLANK ( vPrior ), BLANK (), DIVIDE ( vCurrent - vPrior, vPrior ) )
```

### Running total (acumulado)

```dax
Sales Running Total =
CALCULATE (
    [Total Sales],
    FILTER (
        ALL ( dim_date[Date] ),
        dim_date[Date] <= MAX ( dim_date[Date] )
    )
)
```

### Moving average (3 meses)

```dax
Sales MA 3 =
DIVIDE (
    CALCULATE (
        [Total Sales],
        DATESINPERIOD ( dim_date[Date], MAX ( dim_date[Date] ), -3, MONTH )
    ),
    3
)
```

### Top N dinámico

```dax
Is Top N =
VAR vN    = SELECTEDVALUE ( 'TopN Params'[Top N], 10 )
VAR vRank = RANKX ( ALL ( dim_product[product_name] ), [Total Sales] )
RETURN vRank <= vN
```

### ABC / Pareto

```dax
ABC Class =
VAR vPct = [Cumulative Sales %]
RETURN
    SWITCH (
        TRUE (),
        vPct <= 0.80, "A",
        vPct <= 0.95, "B",
        "C"
    )
```

### Clientes activos (rolling 30 días)

```dax
Active Customers =
CALCULATE (
    DISTINCTCOUNT ( fct_sales[customer_key] ),
    DATESINPERIOD ( dim_date[Date], MAX ( dim_date[Date] ), -30, DAY )
)
```

### Medida con seguridad dinámica (RLS)

```dax
Sales by Region =
VAR vUser = USERPRINCIPALNAME ()
VAR vRegion =
    LOOKUPVALUE (
        dim_employee[region],
        dim_employee[email], vUser
    )
RETURN
    CALCULATE (
        [Total Sales],
        FILTER ( ALL ( dim_region ), dim_region[region] = vRegion || ISBLANK ( vRegion ) )
    )
```

### Manejo seguro de BLANK en ratios

```dax
Safe Ratio =
VAR vA = [Numerator]
VAR vB = [Denominator]
RETURN
    IF (
        ISBLANK ( vA ) || ISBLANK ( vB ) || vB = 0,
        BLANK (),
        DIVIDE ( vA, vB )
    )
```

## 14. Reglas de oro

1. **`DIVIDE(a, b)`** en vez de `a / b`. Devuelve `BLANK()` en vez de error.
2. **Variables (`VAR`)** para sub-cálculos que reutilices. Más rápido y legible.
3. **`BLANK()`** es el valor ausente natural. Mejor que `0` o `""`.
4. **Medidas > columnas calculadas** para todo lo que sea agregado. Las columnas se materializan y crecen el modelo.
5. **`CALCULATE` cambia el filter context.** Es la única función que lo hace.
6. **Time intelligence solo funciona** con una tabla de fechas marcada.
7. **`FILTER` dentro de `CALCULATE` es O(n)**. Si puedes evitarlo, evítalo. Mejor `KEEPFILTERS` o `REMOVEFILTERS`.
8. **No confundas filter context con row context.** Las funciones X crean row context; `CALCULATE` lo convierte en filter context.
9. **`ALL` quita filtros, `ALLEXCEPT` quita todos menos algunos, `ALLSELECTED` mantiene los del contexto externo (slicers).**
10. **Documenta cada medida**: descripción, formato, carpeta. Una medida sin descripción es deuda técnica.
