# Clase 3 · Modelado dimensional avanzado (Sales and Marketing Sample)

## Descripción del ejercicio

Trabajar con un dataset empresarial **multi-tabla** del que no se puede hacer un modelo en estrella trivial. El Sales and Marketing Sample de Microsoft (obviEnce) contiene hechos de ventas diarias, dimensiones geográficas, productos, fabricantes y una tabla de **sentimiento de mercado** (marketing) que se enlaza a las ventas. Es un caso real de "ventas + marketing" en el mismo modelo.

El objetivo es construir desde cero el modelo dimensional, configurar **dimensiones de rol** (porque `Date` da servicio tanto a `SalesFact` como a `Sentiment`), montar una **jerarquía padre-hijo** sobre fabricantes, analizar consecución de objetivos y simular escenarios con un parámetro *what-if*.

## Recursos necesarios

Los datos ya están en esta carpeta, no hay que descargar nada.

| Archivo | Tabla destino | Filas | Tamaño | Notas |
|---|---|---|---|---|
| `data/Date.csv` | `dim_date` | 6.209 | 345 KB | Calendario desde 2007 hasta 2018 aprox. |
| `data/Geo.csv` | `dim_geo` | 39.948 | 1,7 MB | Zip → City / State / Region / District (US). |
| `data/Manufacturer.csv` | `dim_manufacturer` | 14 | 227 B | 14 fabricantes. Uno (`VanArsdel`) es el de la propia empresa. |
| `data/Product.csv` | `dim_product` | 2.412 | 123 KB | Producto + categoría + segmento + ManufacturerID. |
| `data/SalesFact.csv` | `fct_sales` | **1.260.752** | 40 MB | Hechos de ventas: ProductID × Date × Zip con Units y Revenue. |
| `data/Sentiment.csv` | `fct_sentiment` | 21.473 | 944 KB | Sentimiento de mercado: Date × State × Manufacturer × Product. |

> **`SalesFact` tiene 1,26M filas.** Es un dataset de tamaño realista (no es de juguete). Si tu equipo va justo, puedes filtrarlo en Power Query al cargar (p. ej. solo 2015-2017, solo VanArsdel) para acelerar el ejercicio. Lo importante es la mecánica, no la cantidad exacta.

> **Nota sobre el `.pbix` original**: este ejercicio se hace con los CSVs. El `.pbix` original de Microsoft (con el informe pre-construido) NO está en la carpeta a propósito: el objetivo es **construir el modelo**, no abrir uno ya hecho.

## Modelo a construir (esquema lógico)

```
                          ┌──────────────┐
                          │  dim_date    │ 1
                          └──────┬───────┘
                                 │ N
                                 ▼
┌──────────────┐ 1    N ┌─────────────────┐ N    1 ┌────────────────┐
│  dim_geo     │────────│   fct_sales     │        │  dim_product   │
│  (Zip key)   │        │   SalesFact     │────────│  (ProductID)   │
└──────────────┘        └─────────────────┘        └────────┬───────┘
                                 ▲                          │ 1
                                 │ N                        ▼ N
                                 │                ┌──────────────────┐
                                 │                │  dim_manufacturer│
                                 │                │  (ManufacturerID)│
                                 │                └────────┬─────────┘
                                 │                         │ 1
                                 │ N                       ▼ N
                          ┌──────┴───────┐                (↑ jerarquía)
                          │fct_sentiment │────────────────┘
                          │   Sentiment  │
                          └──────┬───────┘
                                 │ 1
                                 ▼
                          dim_date (mismo, rol distinto)
```

Fíjate en el detalle: `fct_sales` y `fct_sentiment` se enlazan a `dim_date` por la misma columna `Date`. La **dimensión de rol** es lo que permite tratar "fecha de venta" y "fecha de sentimiento" como dimensiones independientes sin duplicar `dim_date`.

## Lo que vas a practicar

- Importar 6 CSVs de golpe y aplicar tipado correcto.
- **Modelado en estrella** con 4 dimensiones y 2 hechos.
- **Dimensiones de rol** con `USERELATIONSHIP()`.
- **Jerarquía padre-hijo** sobre `dim_manufacturer` usando la columna `MfgisVanArsdel` como discriminador.
- Análisis de consecución de objetivos (Sales vs Target).
- Parámetro **What-if** simulado a mano.
- Optimización básica del modelo (cardinalidad, tipos).

---

## GUÍA DE IMPORTACIÓN Y POWER QUERY

> Esta sección es el "cómo" del ejercicio. Léela entera antes de empezar. Si solo te interesan los principios (no el caso concreto), tienes una versión más genérica en `referencias/importacion_y_power_query.md`.

### A. Conceptos básicos que necesitas tener claros

| Concepto | Qué es |
|---|---|
| **Query** | Una tabla lógica, resultado de una receta de transformaciones. Vive en Power Query. |
| **Step (Paso aplicado)** | Una transformación discreta (`Quitados duplicados`, `Tipo cambiado`, etc.). |
| **Refresh (Actualizar)** | Ejecuta la receta. La query vuelve a leer la fuente y aplica los pasos. |
| **Cargar** | Materializa la query en el modelo (caché en memoria). |
| **Query folding** | Si la fuente es una base de datos, Power Query intenta "empujar" las transformaciones al motor remoto (SQL, etc.). Si pierdes el folding, todo se procesa en local. |
| **Reference vs Duplicate** | `Reference` crea una nueva query que apunta a la misma lógica. `Duplicate` copia los pasos. La primera es más eficiente y la que se usa casi siempre. |

### B. Importar los 6 CSVs de la Clase 3

1. Abrir Power BI Desktop → **Inicio → Obtener datos → Carpeta**.
2. Seleccionar la carpeta `data/` que está junto a este README.
3. Power BI ofrece combinar y transformar. Pulsar **Combinar y transformar datos** (o **Transformar datos** si quieres ver la lista primero).
4. En Power Query, eliminar la columna `Content` y la columna `Name` y dejar solo `Name` como nombre y `Data` como contenido. O más sencillo: hacer **Combinar archivos** y borrar las columnas sobrantes en un solo paso.

> Truco: **Obtener datos → Carpeta** es la mejor forma de cargar N archivos del mismo esquema. Si te equivocas, siempre puedes ir paso a paso con **Obtener datos → Texto/CSV** archivo a archivo.

5. Power BI genera una query `Transformar archivo de muestra` por cada CSV. Renombrarlas:

| Query generada | Renombrar a |
|---|---|
| `Date.csv` | `dim_date` |
| `Geo.csv` | `dim_geo` |
| `Manufacturer.csv` | `dim_manufacturer` |
| `Product.csv` | `dim_product` |
| `SalesFact.csv` | `fct_sales` |
| `Sentiment.csv` | `fct_sentiment` |

6. **Tipar todas las columnas** correctamente. En cada query, abrir el menú de tipo de cada columna (icono `ABC123` a la izquierda del nombre) y elegir:

| Tabla | Columnas clave | Tipo a aplicar |
|---|---|---|
| `dim_date` | `Date` | Fecha |
| `dim_date` | `MonthNo`, `Year`, etc. | Entero |
| `dim_date` | `MonthName`, `Quarter` | Texto |
| `dim_geo` | `Zip` | **Texto** (códigos postales US pueden empezar por 0) |
| `dim_geo` | `City`, `State`, `Region`, `District` | Texto |
| `dim_manufacturer` | `ManufacturerID` | Entero |
| `dim_manufacturer` | `Manufacturer` | Texto |
| `dim_manufacturer` | `MfgisVanArsdel` | Texto (Yes/No) |
| `dim_product` | `ProductID`, `ManufacturerID` | Entero |
| `dim_product` | `Category`, `Segment`, `Product` | Texto |
| `dim_product` | `isVanArsdel`, `IsCompete`, `IsCompeteHide` | True/False |
| `fct_sales` | `ProductID` | Entero |
| `fct_sales` | `Date` | Fecha |
| `fct_sales` | `Zip` | **Texto** |
| `fct_sales` | `Units` | Entero |
| `fct_sales` | `Revenue` | Número decimal fijo |
| `fct_sentiment` | `DateID` | Entero (formato `YYYYMMDD`) |
| `fct_sentiment` | `StateID` | Entero |
| `fct_sentiment` | `ManufacturerID` | Entero |
| `fct_sentiment` | `ProductID` | Entero |
| `fct_sentiment` | `Score` | Número decimal |
| `fct_sentiment` | `Date` | Fecha |
| `fct_sentiment` | `State` | Texto |
| `fct_sentiment` | `zip` | Texto |

7. **Optimizar antes de cargar**: para `fct_sales` (1,26M filas), no cargues columnas que no vayas a usar. Si el plan de análisis es solo Revenue/Units/Date/Product/Zip, elimina `Date` si prefieres mantener la fecha como `DateID`. (Para este ejercicio mantenla como fecha).

8. **Cerrar y aplicar** (Home → Close & Apply). El modelo se carga. Tardará unos segundos por el `SalesFact`.

> **Error típico**: dejar `Zip` como número. Los ZIPs de la costa este de US empiezan por 0 (`02134 = Boston`). Si lo dejas como Entero, se pierden los ceros a la izquierda y los joins fallan silenciosamente.

### C. Crear el modelo (vista de modelo)

Una vez cargadas las 6 tablas, ir a la **vista de Modelo** (icono de la izquierda) y arrastrar para crear las relaciones:

| Desde | Hacia | Cardinalidad | Dirección filtro | Activa |
|---|---|---|---|---|
| `dim_date[Date]` | `fct_sales[Date]` | 1:N | Unidireccional | **Sí** |
| `dim_date[Date]` | `fct_sentiment[Date]` | 1:N | Unidireccional | **No** (rol) |
| `dim_geo[Zip]` | `fct_sales[Zip]` | 1:N | Unidireccional | Sí |
| `dim_geo[Zip]` | `fct_sentiment[zip]` | 1:N | Unidireccional | No (rol) |
| `dim_product[ProductID]` | `fct_sales[ProductID]` | 1:N | Unidireccional | Sí |
| `dim_product[ProductID]` | `fct_sentiment[ProductID]` | 1:N | Unidireccional | No (rol) |
| `dim_manufacturer[ManufacturerID]` | `dim_product[ManufacturerID]` | 1:N | Unidireccional | Sí |
| `dim_manufacturer[ManufacturerID]` | `fct_sentiment[ManufacturerID]` | 1:N | Unidireccional | Sí |

Hay 3 **dimensiones de rol** en este modelo. La activa de cada una depende de qué hecho sea el "principal" en el informe (en este caso `fct_sales`).

### D. Marcar la tabla de fechas

Click derecho sobre `dim_date` → **Marcar como tabla de fechas** → seleccionar la columna `Date`. Esto activa el *time intelligence* automático en DAX.

### E. "Web scraping" en Power Query — guía detallada

> "Scraping" en Power BI no es lo mismo que con BeautifulSoup o Selenium. Aquí se hace con el conector **Web** o con código **M** que llama a `Web.Contents`. Es limitado (no ejecuta JavaScript, no maneja sesiones complejas), pero sirve para muchísimos casos reales.

#### E.1. Caso fácil: tabla HTML estática

1. **Obtener datos → Web**.
2. Pegar la URL (por ejemplo, una página de Wikipedia con tablas).
3. Power BI detecta las tablas y te las lista. Seleccionar la que quieres.
4. En Power Query, limpiar y tipar como cualquier otra fuente.

**Ejemplo**: una página de la CIA con datos de países. URL: `https://www.cia.gov/the-world-factbook/field/population/country-comparison/`. (Ojo: la CIA cambió la web, este ejemplo puede no funcionar literalmente. La idea es la misma: una URL con tablas HTML accesibles.)

#### E.2. Caso medio: API REST que devuelve JSON

La mayoría de APIs modernas devuelven JSON. Power Query tiene `Json.Document()` para parsearlos.

Ejemplo con la API pública [jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com):

```m
let
    Source = Json.Document(
        Web.Contents(
            "https://jsonplaceholder.typicode.com/users",
            [
                Headers = [ #"Accept" = "application/json" ]
            ]
        )
    ),
    #"Converted to Table" = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Expanded Column1" = Table.ExpandRecordColumn(#"Converted to Table", "Column1", {"id", "name", "username", "email"}, {"id", "name", "username", "email"})
in
    #"Expanded Column1"
```

Cómo usarlo:

1. En Power Query → **Obtener datos → Consulta en blanco**.
2. Se abre el editor avanzado (botón "Editor avanzado" en la cinta). Pegar el código M.
3. Power BI ejecuta la llamada. Si la API requiere autenticación, ver E.3.

#### E.3. Caso con autenticación

```m
let
    Source = Json.Document(
        Web.Contents(
            "https://api.example.com/v1/sales",
            [
                Headers = [
                    #"Authorization" = "Bearer TU_TOKEN_AQUI",
                    #"Accept"        = "application/json"
                ],
                Query = [
                    #"year"  = "2025",
                    #"month" = "06"
                ]
            ]
        )
    )
in
    Source
```

**Cómo funciona**:

- `Web.Contents(URL, options)` — la única función que hace HTTP en M.
- `Headers` — diccionario de cabeceras HTTP. `Authorization: Bearer ...` es el estándar OAuth2.
- `Query` — parámetros que se añaden a la URL como `?year=2025&month=06`.
- `RelativePath` — para APIs con URL base fija.
- `Content` — para POST/PUT. Recibe un binario (hay que construirlo con `Text.ToBinary` o `Json.FromValue`).

#### E.4. Paginación

Las APIs rara vez devuelven 100.000 registros de golpe. Hay que paginar. Patrón típico:

```m
let
    // Páginas a llamar
    Pages = List.Transform(
        {1, 2, 3, 4, 5},
        each Json.Document(
            Web.Contents(
                "https://api.example.com/v1/sales",
                [
                    Query = [ page = Text.From(_) ]
                ]
            )
        )
    ),
    // Combinar todas las páginas en una tabla
    Combined = Table.FromList(Pages, Splitter.SplitByNothing(), {"Page"}),
    Expanded = Table.ExpandListColumn(Combined, "Page")
in
    Expanded
```

Para casos reales con paginación dinámica (siguiente enlace en la respuesta), necesitas un bucle con `List.Generate` o `Function.InvokeAfter` con throttling.

#### E.5. Errores comunes en web scraping con Power Query

| Error | Causa | Solución |
|---|---|---|
| `DataSource.Error: Web.Contents failed` | La URL no responde o no tienes permisos. | Abre la URL en un navegador primero. Si pide login, necesitas autenticación. |
| `We cannot convert a value of type Record to Table` | La respuesta JSON es un objeto, no un array. | Usa `Record.ToTable()` o expande campos uno a uno. |
| `Formula.Firewall: query references other queries...` | Estás mezclando datos de privacidad distinta (p. ej. CSV local + API web). | Configura los niveles de privacidad en `Archivo → Opciones y configuración → Opciones → Privacidad`. |
| La tabla sale vacía | La web renderiza la tabla con JavaScript. Power Query **no ejecuta JS**. | Usa una API si la web la tiene. Si no, considera otro origen. |
| `429 Too Many Requests` | Estás llamando demasiado rápido. | Añade `Function.InvokeAfter(() => ..., #duration(0,0,0,1))` entre llamadas. |
| Datos cortados en 1.000 filas | Algunas APIs paginan automáticamente pero Power Query solo lee la primera página. | Implementa la paginación manualmente (ver E.4). |
| Columnas con tipos mezclados | La API devuelve un campo como string en algunos registros y número en otros. | `Table.TransformColumns` con `each if _ is null then null else Value.FromText(_)` o cambia el tipo a texto. |

#### E.6. Cuándo NO usar Power Query para scraping

- La web es **SPA (single page application)** y el contenido se renderiza con JS. Power Query solo descarga el HTML inicial.
- Necesitas **interacción**: clicks, scrolls, login con captcha.
- El sitio tiene **rate limiting agresivo** sin API key.
- Necesitas **scraping masivo** (>100k páginas).

En esos casos, scrappea con Python (BeautifulSoup, Scrapy, Selenium) o Node (Puppeteer, Playwright) y exporta a CSV. Luego lo carga Power BI como un CSV más.

#### E.7. Query folding: el detalle que nadie cuenta

**Query folding** = Power Query traduce tus pasos M a una sola query SQL (o similar) que se ejecuta en la base de datos origen. Solo aplica a orígenes SQL, OData, ODBC, etc. NO aplica a CSV, Excel, JSON, Web.

```m
// Ejemplo de pasos que SÍ hacen folding:
SQL.Source   → "SELECT * FROM Sales WHERE year = 2024"
             → Table.SelectRows(... [year] = 2024)   ← plegable
             → Table.Group(...)                       ← plegable

// Ejemplo de pasos que ROMPEN el folding:
             → Table.AddColumn(..., each MyCustomFunction(x))   ← no plegable
```

**Reglas de oro**:

- Cuantos más pasos al principio, más probable que se mantenga el folding.
- Cualquier `Table.AddColumn` con una función personalizada rompe el folding en la primera columna que la use.
- Si rompes el folding, todo se procesa en local. Con 1M filas no pasa nada. Con 100M filas, tu informe tarda 10 minutos en refrescar.

Para verificar: click derecho sobre la query en Power Query → **Ver consulta nativa**. Si la query se ve, hay folding.

---

## Paso a paso del ejercicio

### 1. Cargar las 6 tablas (sección B de la guía)

Seguir la sección B. Al terminar tendrás 6 queries en Power Query, todas tipadas.

### 2. Crear el modelo (vista de modelo)

Seguir la sección C. 8 relaciones en total, 3 de ellas inactivas (dimensiones de rol).

### 3. Crear `dim_date` con jerarquías y marcarla como tabla de fechas

La tabla `Date.csv` ya tiene una jerarquía temporal rica (`Month`, `Quarter`, `Year`, etc.), así que no necesitas `CALENDARAUTO`. Solo:

1. Click derecho sobre `dim_date` → **Marcar como tabla de fechas** → columna `Date`.
2. Crear jerarquía: click derecho sobre `Year` → **Nueva jerarquía** → añadir `Quarter` → `Month` → `Date`.

### 4. Dimensión de rol: ventas vs sentimiento

Ya creadas en el modelo (relación activa a `fct_sales[Date]`, inactiva a `fct_sentiment[Date]`). Crear las medidas:

```dax
Total Revenue = SUM ( fct_sales[Revenue] )
Total Units   = SUM ( fct_sales[Units] )

Avg Sentiment =
AVERAGE ( fct_sentiment[Score] )

Sentiment by Sales Date =
CALCULATE (
    [Avg Sentiment],
    USERELATIONSHIP ( dim_date[Date], fct_sentiment[Date] )
)
```

> Truco: la medida `Sentiment by Sales Date` cambia el contexto temporal: "sentimiento medido en la fecha de la venta", no en la fecha del sentimiento. Es una de las preguntas reales que un equipo de marketing se hace: "¿cómo se sentía el mercado cuando vendimos?".

### 5. Jerarquía padre-hijo sobre Manufacturer

En este dataset, la jerarquía no es de empleados sino de fabricantes, pero el patrón es el mismo. Aquí "VanArsdel" es nuestra propia empresa y los demás son competencia.

```dax
Manufacturer Path =
IF (
    dim_manufacturer[MfgisVanArsdel] = "Yes",
    "VanArsdel",
    dim_manufacturer[Manufacturer]
)
```

Como solo hay 14 fabricantes, no necesitas `PATH`/`PATHITEM` — con la columna `MfgisVanArsdel` basta para crear una jerarquía simple "VanArsdel / Competencia". Si quisiéramos analizar una jerarquía más profunda (e.g. multi-nivel), usaríamos `PATH()`.

Crear una medida:

```dax
Revenue VanArsdel =
CALCULATE (
    [Total Revenue],
    dim_manufacturer[MfgisVanArsdel] = "Yes"
)

Revenue Competitors =
CALCULATE (
    [Total Revenue],
    dim_manufacturer[MfgisVanArsdel] = "No"
)
```

### 6. Análisis Sales vs Target

En este dataset **no hay** tabla de objetivos. La construimos nosotros para practicar.

En Power Query:

1. Click derecho sobre `fct_sales` → **Reference** → renombrar a `fct_targets`.
2. Eliminar todas las columnas excepto `Date`, `Zip` y `Revenue`.
3. Agrupar por `Year` y `ManufacturerID` y hacer la suma: `Agrupar por → Agrupar por [Year, ManufacturerID] → Suma de Revenue → renombrar columna a target_amount`.
4. Ajustar el target: para que la consecución tenga sentido, vamos a poner el target como `110% de las ventas reales del año anterior`. En Power Query, esto se puede hacer con M, pero es más simple en DAX como medida. Cargamos los targets "reales" (revenue) y la lógica del % la pone DAX.

```dax
Target % =
VAR vCurrent =
    CALCULATE ( [Total Revenue], ALLEXCEPT ( dim_manufacturer, dim_manufacturer[ManufacturerID] ) )
VAR vPriorYear =
    CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( dim_date[Date] ), ALLEXCEPT ( dim_manufacturer, dim_manufacturer[ManufacturerID] ) )
RETURN vPriorYear * 1.10

Attainment % =
DIVIDE ( [Total Revenue], [Target %] )
```

Formato condicional en el visual: rojo si `< 0.85`, ámbar si `0.85–1.0`, verde si `>= 1.0`.

### 7. Parámetro What-if

Simular "qué pasaría si el sentimiento mejora un X%". Crear tabla desconectada:

```dax
WhatIf Sentiment Boost = GENERATESERIES ( 0, 0.50, 0.05 )
```

```dax
Revenue Simulated =
VAR vBoost = SELECTEDVALUE ( 'WhatIf Sentiment Boost'[WhatIf Sentiment Boost Value], 0 )
VAR vBase = [Total Revenue]
VAR vSentimentFactor = 1 + ( [Avg Sentiment] / 100 ) * vBoost
RETURN vBase * vSentimentFactor
```

Visualizar: slicer con el what-if + una card con `Revenue Simulated` comparada con `Total Revenue`.

### 8. Optimización del modelo

Antes de pasar a Clase 4:

1. Abrir **Vista de modelo** → mirar tamaños de tabla abajo a la derecha. `fct_sales` será la dominante (~30-50 MB).
2. Confirmar tipos:
   - Todas las columnas `Date` como `Fecha` (no `DateTime`).
   - `Zip` como `Texto` en `fct_sales` y en `dim_geo`.
   - `Score` como `Decimal fijo` (no `Decimal`).
3. Marcar columnas que no se agreguen como `No resumir` (click derecho sobre la columna en la vista de modelo).

---

## Explicación y buenas prácticas

### Por qué este modelo tiene 3 dimensiones de rol

`Date` da servicio a `fct_sales` (fecha de venta) y a `fct_sentiment` (fecha de medición del sentimiento). Si crearas `dim_sales_date` y `dim_sentiment_date`, tendrías un producto cartesiano al cruzar ambas fechas. Una sola tabla con dos relaciones (una activa, otra inactiva) es más limpio y consume menos memoria.

### Sentimiento y ventas: un join sutil

`fct_sentiment` tiene **tres claves** que pueden enlazarse a `dim_geo`:
- `State` (texto)
- `zip` (texto)
- `StateID` (entero, código del estado)

`fct_sales` solo tiene `Zip`. Por eso elegimos relacionar por `Zip` en ambos. **No** por `State` — un mismo estado tiene cientos de ZIPs y unirlos así generaría ambigüedad.

### El "truco" del target que es 110% del año anterior

En un proyecto real, los objetivos no suelen estar en el mismo dataset: los marca el equipo de finanzas en otro sistema. Aquí los derivamos en DAX para practicar el patrón "medir contra benchmark histórico". En la vida real:

1. Recibes un Excel/CSV con objetivos por [Manufacturador × Mes × Año].
2. Lo importas como una tabla de hechos aparte (`fct_targets`).
3. Creas una dimensión de fecha compartida (o duplicada con `USERELATIONSHIP`).
4. La medida de consecución compara `Total Revenue` contra `SUM(fct_targets[target])`.

### Qué cubre la clase 3 que la 1 y 2 no cubrían

| Tema | Clase |
|---|---|
| Importar CSV/Excel, modelar básico, DAX básico | 1 |
| DAX avanzado, patrones analíticos | 2 |
| **Multi-tabla, dimensiones de rol, parent-child, what-if, optimización inicial** | **3** |
| Producción, RLS, despliegue, App | 4 |

La Clase 3 es la "antesala" de la 4: si entiendes el modelo y el DAX de la 3, la 4 se centra en lo que rodea al modelo (rendimiento, seguridad, navegación, despliegue).

---

## Checklist de autoevaluación

- [ ] Las 6 tablas están cargadas y tipadas.
- [ ] `Zip` está como **Texto** en `fct_sales` y en `dim_geo`.
- [ ] Las 8 relaciones del modelo están creadas (3 inactivas).
- [ ] `dim_date` está marcada como tabla de fechas.
- [ ] La medida `Sentiment by Sales Date` devuelve un valor coherente con la realidad.
- [ ] La consecución de objetivos muestra variación entre fabricantes.
- [ ] El slicer de "What-If Sentiment" cambia la card de `Revenue Simulated`.
- [ ] El modelo ocupa menos de 60 MB.

## Extensiones posibles

- Implementar un **parámetro dinámico** para comparar dos años lado a lado (slicer de año A y año B con un KPI de delta).
- Sustituir el parent-child simulado por uno real con `PATH`/`PATHITEM` si consigues una jerarquía de empleados.
- Añadir **anomaly detection** en la serie de revenue para detectar caídas sospechosas.
- Calcular **correlación** entre `Score` (sentimiento) y `Revenue` con la función `CORRELX` o un Python visual con `pandas`.
- Conectar este modelo a una **API pública** (precios del petróleo, tipo de cambio, etc.) usando la sección E de la guía y crear una dimensión "Contexto de mercado".
