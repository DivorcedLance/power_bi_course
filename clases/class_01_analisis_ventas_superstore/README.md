# Clase 1 · Dashboard de análisis de ventas con el dataset *Superstore*

## Descripción del ejercicio

Partiendo de un único CSV plano con datos de ventas retail a nivel de línea de pedido, construir un **dashboard ejecutivo de análisis de ventas** en Power BI Desktop. El objetivo no es "conectar y pintar", sino recorrer el flujo completo que se espera en un proyecto real:

1. Conexión a la fuente.
2. Limpieza y tipado en Power Query.
3. Modelado dimensional (estrella) con tabla de hechos y dimensiones.
4. Tabla calendario.
5. Medidas DAX reutilizables (no se calcula nada a nivel de columna visual).
6. Visualización enfocada a negocio.

El informe debe permitir responder, como mínimo, a estas preguntas:

- ¿Cuánto hemos vendido y cuánto hemos ganado en el periodo seleccionado?
- ¿Cuál es el margen real (no solo el nominal) y cómo evoluciona?
- ¿Qué categorías, subcategorías y productos aportan más ventas y más beneficio?
- ¿Qué regiones, estados y ciudades rinden mejor?
- ¿Qué segmento de cliente (Consumer / Corporate / Home Office) es más rentable?
- ¿Hay estacionalidad? ¿Crecimiento interanual?

## Recursos necesarios

El dataset ya está en esta carpeta, no hay que descargar nada.

| Recurso | Detalle |
|---|---|
| Power BI Desktop | Última versión estable (Microsoft Store o [powerbi.microsoft.com](https://powerbi.microsoft.com/)). |
| Dataset | `superstore.csv` (en esta misma carpeta, ~9.9k filas). |
| Fuente original (referencia) | [raw.githubusercontent.com/leonism/sample-superstore/master/data/superstore.csv](https://raw.githubusercontent.com/leonism/sample-superstore/master/data/superstore.csv). |
| Diccionario de columnas | `Row ID, Order ID, Order Date, Ship Date, Ship Mode, Customer ID, Customer Name, Segment, Country, City, State, Postal Code, Region, Product ID, Category, Sub-Category, Product Name, Sales, Quantity, Discount, Profit`. |

> El dataset solo contiene datos de **Estados Unidos**, así que las geografías se limitan a US States. Esto se tendrá en cuenta al diseñar los mapas y la dimensión geográfica.

## Lo que vas a practicar (nivel intermedio)

- Power Query: tipado, renombrado, columnas condicionales, *merge* y *reference*.
- Modelado en estrella: pasar de una tabla "plana" a un esquema con `Fact_Sales` + dimensiones `Dim_*`.
- Tabla calendario con `CALENDARAUTO()` y jerarquías.
- DAX: medidas básicas, `DIVIDE`, time intelligence (`SAMEPERIODLASTYEAR`, `TOTALYTD`).
- Formato condicional, bookmarks y diseño de informe.

---

## Paso a paso

### 1. Crear el archivo y conectar a la fuente

1. Abrir Power BI Desktop → **Obtener datos → Web**.
2. Pegar la URL: `https://raw.githubusercontent.com/leonism/sample-superstore/master/data/superstore.csv`.
3. Vista previa: comprobar 21 columnas y ~9.9k filas. Pulsar **Transformar datos** (no "Cargar", queremos entrar en Power Query).

> **Buena práctica**: en un proyecto real, todos los pasos de transformación se hacen en Power Query. La capa de informe solo consume el modelo ya limpio.

### 2. Limpieza y tipado en Power Query

Renombrar la consulta a `fct_sales_raw` y aplicar:

| Columna | Tipo | Notas |
|---|---|---|
| `Order Date` | Fecha | Formato origen: `M/d/yyyy` (US). |
| `Ship Date` | Fecha | Igual. |
| `Postal Code` | Texto | Los códigos postales de US pueden empezar por 0, **no** deben ser numéricos. |
| `Sales`, `Profit` | Número decimal fijo (Currency) | Usar tipo `Fixed decimal number` o `Currency`. |
| `Quantity`, `Discount` | Número decimal | `Discount` se queda como decimal (0.2 = 20%). |
| Resto de textos | Texto | `Trim` + `Clean` por si vienen espacios. |

Pasos recomendados:

- Eliminar duplicados por `Row ID` (es la clave única de la fila).
- Detectar y revisar nulos en `Postal Code` (algunos pedidos corporativos lo traen vacío).
- Renombrar todas las columnas a `snake_case` y sin acentos (`Order Date` → `order_date`). Esto evita problemas en DAX y en el modelo.

### 3. Columnas calculadas en Power Query (no en DAX)

Añadir estas columnas **dentro de Power Query**, porque son atributos de la línea de pedido que no cambian con el contexto de filtro:

- `ship_days` = `Date.From([ship_date]) - Date.From([order_date])` → duración del envío en días.
- `has_discount` = `if [discount] > 0 then "Con descuento" else "Sin descuento"` (tipo texto para usarlo en segmentaciones).
- `profit_margin_line` = `if [sales] = 0 then 0 else [profit] / [sales]` → margen a nivel de línea, útil para análisis granular.

> **Buena práctica**: lo que se pueda precalcular en Power Query, se precalcula allí. DAX queda para los agregados dinámicos.

### 4. Modelado en estrella

Una tabla plana con 21 columnas se puede "pintar" directamente, pero es un antipatrón: ocupa más, filtra peor y obliga a usar `DISTINCTCOUNT` para contar clientes o productos únicos.

**Estrategia**: dejar `fct_sales` como tabla de hechos (una fila por línea) y crear cuatro dimensiones por *reference*:

```
fct_sales (hechos)
├── dim_customer   (customer_id, customer_name, segment)
├── dim_product    (product_id, product_name, category, sub_category)
├── dim_geography  (city, state, region, postal_code, country)
└── dim_date       (date, year, month, quarter, month_name, year_month)
```

Pasos en Power Query:

1. Click derecho sobre `fct_sales_raw` → **Reference** → renombrar a `dim_customer`. Eliminar todas las columnas excepto `customer_id`, `customer_name`, `segment`. Quitar duplicados por `customer_id`.
2. Repetir para `dim_product` (quedarse con `product_id`, `product_name`, `category`, `sub_category`) y para `dim_geography` (`city`, `state`, `region`, `postal_code`, `country`).
3. Renombrar la consulta original a `fct_sales` y eliminar las columnas descriptivas que ya viven en las dimensiones (deja solo las claves y las medidas: `sales`, `quantity`, `discount`, `profit`, `ship_days`, `has_discount`, `profit_margin_line`).
4. Cerrar y aplicar.

En la vista de **Modelo** de Power BI Desktop, crear las relaciones 1:N desde cada `dim_*` hacia `fct_sales`. Todas con **dirección de filtro única** y **ambos lados activos**, salvo que se justifique lo contrario.

> **Buena práctica**: en un modelo en estrella, las dimensiones no se relacionan entre sí directamente. Si necesitas cruzar cliente y producto, lo haces con medidas, no con relaciones.

### 5. Tabla calendario

Crear `dim_date` con DAX (no en Power Query) para que el motor la gestione internamente como tabla de tiempo:

```dax
dim_date =
ADDCOLUMNS (
    CALENDARAUTO (),
    "year",      YEAR ( [Date] ),
    "month_num", MONTH ( [Date] ),
    "month_name", FORMAT ( [Date], "MMM" ),
    "quarter",   "Q" & QUARTER ( [Date] ),
    "year_month", FORMAT ( [Date], "YYYY-MM" )
)
```

Luego en la vista de modelo: click derecho sobre `dim_date` → **Marcar como tabla de fechas**. Esto habilita las funciones de *time intelligence* (`SAMEPERIODLASTYEAR`, `TOTALYTD`, etc.) sin tener que pasar la fecha manualmente.

Crear una jerarquía: `Año → Trimestre → Mes → Día`.

Relacionar `dim_date[Date]` con `fct_sales[order_date]` (1:N, unidireccional).

### 6. Medidas DAX

Crear una **tabla vacía** llamada `_medidas` (o usar la carpeta de medidas de la vista de modelo) para alojar todas las medidas en un único lugar. Esto facilita el mantenimiento.

Medidas base:

```dax
Total Sales = SUM ( fct_sales[sales] )

Total Profit = SUM ( fct_sales[profit] )

Total Orders = DISTINCTCOUNT ( fct_sales[order_id] )

Total Customers = DISTINCTCOUNT ( fct_sales[customer_id] )
```

Medidas de ratio (usar `DIVIDE` para evitar división por cero):

```dax
Profit Margin = DIVIDE ( [Total Profit], [Total Sales] )

Avg Order Value = DIVIDE ( [Total Sales], [Total Orders] )
```

Medidas de time intelligence:

```dax
Sales LY = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( dim_date[Date] ) )

Sales YoY % =
DIVIDE (
    [Total Sales] - [Sales LY],
    [Sales LY]
)

Sales YTD = TOTALYTD ( [Total Sales], dim_date[Date] )
```

> **Buena práctica**: todas las medidas van con formato (moneda, %, entero) y descripción. Una medida sin descripción es deuda técnica desde el día 1.

### 7. Diseño del informe

Estructura recomendada (una sola página para esta clase):

```
+--------------------------------------------------------------+
|  KPI: Ventas   |  KPI: Beneficio   |  KPI: Margen   |  KPI: Pedidos |
+--------------------------------------------------------------+
|  Slicers: Año, Región, Segmento, Categoría, Con/Sin descuento |
+--------------------------------------------------------------+
|  Línea: Sales por Mes (jerarquía Año→Trimestre→Mes)         |
+----------------------------------+---------------------------+
|  Barras: Ventas por Subcategoría | Mapa: Ventas por Estado   |
+----------------------------------+---------------------------+
|  Tabla detalle: Top 20 productos por beneficio               |
+--------------------------------------------------------------+
```

Tips de visualización:

- **KPI cards** formateados con `Sales` y `Sales YoY %` como *callout value*.
- En el **mapa**, usar `state` desde `dim_geography`. Confirmar que la categoría de datos esté como "State or Province".
- La **línea temporal** debe respetar la jerarquía: si el usuario hace drill-down de Año a Mes, el eje se reagrupa.
- En la **tabla de productos**, incluir barra de datos en `Profit` y formato condicional verde (>0) / rojo (<0).

### 8. Publicar y validar

- Guardar como `.pbix`.
- (Opcional) Publicar en *Power BI Service* en un *workspace* personal para probar el comportamiento en la nube.
- Validar con un usuario real: ¿se entiende sin necesidad de preguntar? ¿los filtros están claros?

---

## Explicación y buenas prácticas

### Por qué un modelo en estrella y no una tabla plana

Con 9.9k filas el rendimiento no se nota, pero los problemas aparecen en tres sitios:

1. **Conteo de clientes/productos únicos**: en una tabla plana necesitas `DISTINCTCOUNT` sobre `customer_id` en cada visual. En estrella, `DISTINCTCOUNT ( dim_customer[customer_id] )` es una operación simple y el optimizador la trata como medida de dimensión.
2. **Filtros cruzados**: si filtras por `State = "California"`, en el modelo plano arrastras toda la fila (incluyendo product_id, etc.) al motor. En estrella solo se propaga la relación 1:N hacia la tabla de hechos.
3. **Mantenimiento**: añadir un nuevo atributo (p. ej. `customer_email`) es crear una columna en `dim_customer`. En una tabla plana rompe potencialmente medidas y visuales.

### Por qué `DIVIDE` y no `/`

```dax
Bad :  Margin = [Total Profit] / [Total Sales]
Good:  Margin = DIVIDE ( [Total Profit], [Total Sales] )
```

`DIVIDE(a, b)` devuelve `BLANK()` cuando `b = 0`, mientras que `/` devuelve `error`. Los visuales gestionan `BLANK()` correctamente (lo ocultan o lo muestran como guion). Los errores, en cambio, rompen el visual entero. A nivel de buenas prácticas, **`BLANK()` es el "valor ausente natural" en DAX**, y `DIVIDE` es la única forma correcta de dividir.

### Por qué `CALENDARAUTO` y no una tabla hecha a mano

- Detecta el rango real de fechas de tu modelo (mínimo y máximo de todas las columnas fecha).
- Genera fechas contiguas, sin huecos, lo que es imprescindible para `SAMEPERIODLASTYEAR` y compañía.
- Marcarla como **tabla de fechas** activa el *time intelligence* automático.

Si en el futuro añades una nueva tabla con fechas (p. ej. `dim_ship_date`), la vuelves a marcar como tabla de fechas y el motor entiende que ambas son dimensiones de tiempo válidas.

### Por qué precalcular `ship_days` en Power Query

Es un atributo estático de la línea de pedido: una vez calculado, no cambia. Si lo hicieras en DAX con una columna calculada, recomputaría en cada刷新 y, además, ocuparía espacio en el modelo. La regla es:

- **Power Query**: transformaciones sobre filas individuales.
- **Columna calculada DAX**: solo cuando necesitas una expresión que involucre relaciones o un filtro de fila.
- **Medida DAX**: agregados dinámicos (SUM, COUNT, ratios, time intelligence).

### Formato y documentación de medidas

- Nombre: `Sales YoY %` y **no** `medida_17` ni `S1`. Si alguien abre el informe en seis meses, tiene que entender qué hace cada medida.
- Carpeta de medidas: agrupar por dominio (`_KPIs`, `_Tiempos`, `_Productos`).
- Descripción: una línea con la fórmula conceptual. Ej: *"Variación porcentual de ventas vs. el mismo periodo del año anterior"*.

### Errores comunes que se ven en este dataset

1. **Mapas que no pintan estados**: pasa cuando la columna está en formato texto con un valor no reconocido por Bing Maps (p. ej. "US-California" en lugar de "California"). Solución: limpiar en Power Query.
2. **`Sales YoY %` da error en el primer año**: porque `SAMEPERIODLASTYEAR` no tiene periodo anterior. Usa `IF ( ISBLANK ( [Sales LY] ), BLANK(), ... )` para esos casos.
3. **Margen = 0% en todas las filas**: probablemente estás dividiendo una columna calculada con un `CALCULATE` que no respeta el contexto. Asegúrate de que la medida use `[Total Sales]`, no `SUM(fct_sales[sales_margin])`.
4. **Postal Code aparece como `2.15E+08`**: la columna se quedó como número. Cámbiala a texto en Power Query.

---

## Checklist de autoevaluación

- [ ] Las 4 dimensiones están separadas y relacionadas con `fct_sales`.
- [ ] `dim_date` está marcada como tabla de fechas.
- [ ] Todas las medidas usan `DIVIDE` para divisiones.
- [ ] Las visualizaciones de tiempo responden a la jerarquía Año→Trimestre→Mes.
- [ ] El mapa pinta correctamente los estados de US.
- [ ] Los slicers filtran todas las visuales de la página.
- [ ] No hay columnas calculadas DAX que se podrían haber hecho en Power Query.
- [ ] El informe se entiende en menos de 30 segundos por una persona que no lo ha visto antes.

## Extensiones posibles (para después de la clase)

- Añadir una página de **análisis de devoluciones** (no hay devoluciones reales en el dataset, se puede simular con pedidos de `Standard Class` con `ship_days > 5`).
- Crear un **parámetro "What-if"** para simular distintos escenarios de descuento y ver el impacto en margen.
- Sustituir el CSV por una conexión a **PostgreSQL / SQL Server** usando *DirectQuery* para practicar el cambio de modo de conectividad.
- Crear **roles de seguridad** a nivel de fila para que un comercial regional solo vea sus estados.
