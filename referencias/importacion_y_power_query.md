# Importación de datos y Power Query · Guía de referencia

> Esta es la versión "standalone" de la guía que aparece en la Clase 3. Cubre los principios generales de importación y web scraping en Power Query. Si ya leíste la sección de la Clase 3, esta te sirve de consulta rápida; si no, léela entera.

## Índice

1. [Conceptos básicos de Power Query](#1-conceptos-básicos-de-power-query)
2. [Importar CSV](#2-importar-csv)
3. [Importar Excel](#3-importar-excel)
4. [Importar desde base de datos](#4-importar-desde-base-de-datos)
5. [Importar desde Web (Web connector)](#5-importar-desde-web-web-connector)
6. [Web.Contents en M: el conector programático](#6-webcontents-en-m-el-conector-programático)
7. [Web scraping práctico: HTML y JSON](#7-web-scraping-práctico-html-y-json)
8. [Autenticación](#8-autenticación)
9. [Paginación](#9-paginación)
10. [Errores comunes](#10-errores-comunes)
11. [Query folding: la optimización que no se ve](#11-query-folding-la-optimización-que-no-se-ve)
12. [Buenas prácticas generales](#12-buenas-prácticas-generales)

---

## 1. Conceptos básicos de Power Query

| Concepto | Qué es |
|---|---|
| **Query** | Una tabla lógica, resultado de una receta de transformaciones. Vive en Power Query. |
| **Step (Paso aplicado)** | Una transformación discreta que ves en el panel derecho. Cada paso cambia la tabla. |
| **Refresh (Actualizar)** | Ejecuta la receta. La query vuelve a leer la fuente y aplica los pasos en orden. |
| **Cargar** | Materializa la query en el modelo (caché en memoria / VertiPaq). |
| **Query folding** | Si la fuente es una BD, Power Query intenta "empujar" las transformaciones al motor remoto. Si lo pierdes, todo se procesa en local. |
| **Reference** | Crea una nueva query que apunta a la misma lógica. Cambias la referencia, no duplicas. |
| **Duplicate** | Copia los pasos. Útil para variantes que van a divergir mucho. |

> **Regla clave**: usa **Reference** por defecto. **Duplicate** solo cuando las dos tablas van a tener vidas completamente independientes.

## 2. Importar CSV

### Caso típico

**Obtener datos → Texto/CSV** → seleccionar el archivo → **Transformar datos** (no Cargar directamente).

### Pasos críticos

1. **Verifica la codificación** abajo a la derecha de la vista previa. Si salen caracteres raros (acentos, eñes), cambia a `65001 (Unicode UTF-8)` o `1252 (Windows-1252)`.
2. **Verifica el delimitador**: coma, punto y coma, tab, pipe. En CSV español es habitual `;` en lugar de `,`.
3. **Tipar todas las columnas** en el primer momento. Una columna mal tipada contamina el modelo.
4. **Encoding de fecha**: si vienen como `M/d/yyyy` (US), `d/M/yyyy` (ES) o `yyyy-MM-dd` (ISO), configura el *locale* en cada columna tipo fecha. Esto evita que `01/03/2025` se interprete como "1 de marzo" o "3 de enero" según el locale.

### Errores frecuentes con CSV

| Problema | Causa | Solución |
|---|---|---|
| Acentos rotos | Codificación incorrecta | Cambiar el encoding al importar |
| Todas las filas en una columna | Delimitador equivocado | Forzar delimitador manualmente |
| Fechas mal interpretadas | Locale por defecto | Cambiar locale en la columna |
| Última fila con totales del Excel | El CSV viene con footer | Quitar las últimas N filas o filtrar |
| Códigos postales con ceros a la izquierda cortados | Tipado numérico | Forzar tipo **Texto** siempre |

## 3. Importar Excel

### Opciones al importar

1. **Obtener datos → Libro de Excel** → seleccionar el archivo.
2. Aparece el **Navigator**: lista de tablas, hojas y rangos del workbook.
3. Puedes seleccionar uno o varios a la vez. Si seleccionas varios, se crea una query por cada uno.

### Para cada hoja/tabla

- **Tabla con nombre definido** (`Insertar → Tabla` en Excel) → Power BI la reconoce como tabla. Recomendable.
- **Hoja suelta** → te pregunta si quieres usar la primera fila como encabezado.
- **Rango con nombre** → aparece listado en el Navigator.

### Patrones útiles

- **Múltiples hojas con la misma estructura**: usar `Combinar archivos` o crear una query por hoja y luego `Append`.
- **Múltiples archivos en una carpeta**: `Obtener datos → Carpeta`. Power BI genera una query `Transformar archivo de muestra` y una `Transformar archivo` que las une todas.
- **Power Pivot en el Excel**: si el workbook tiene un modelo de Power Pivot, Power BI lo detecta y lo trae. Pero el modelo a veces tiene errores; mejor rehacerlo.

### Errores frecuentes con Excel

| Problema | Causa | Solución |
|---|---|---|
| No aparece la hoja | Hoja oculta | Desocultar la hoja en Excel |
| Columnas vacías al final | El rango es mayor que los datos | Especificar el rango exacto o filtrar nulos |
| Fórmulas rotas al refrescar | Las celdas eran fórmulas que ahora son errores | Limpiar en Power Query con `try...otherwise` |

## 4. Importar desde base de datos

Conectores nativos: SQL Server, PostgreSQL, MySQL, Oracle, Snowflake, BigQuery, Databricks, etc. **Aquí aplica el query folding**, que es donde se gana el rendimiento.

### Pasos típicos

1. **Obtener datos → Base de datos SQL Server** (o el conector que sea).
2. Introducir servidor y base de datos.
3. Elegir entre:
   - **Importar** (copia los datos al modelo, refresca periódicamente).
   - **DirectQuery** (consulta la BD en cada interacción; datos siempre actualizados, pero más lento).
   - **Modo mixto (Composite model)** para tablas grandes en DirectQuery y dimensiones en Import.
4. En el Navigator, seleccionar las tablas o escribir una **query SQL nativa** (botón "Opciones avanzadas" → escribir SQL).

### Cuándo usar DirectQuery

- Tabla > 100M filas y necesitas datos siempre frescos.
- La fuente tiene seguridad a nivel de fila y no se puede replicar en Power BI.
- Refresh en Power BI Service no es viable (latencia < 1h).

### Cuándo NO usar DirectQuery

- Si necesitas cálculos DAX pesados (relación 1:1 con aggregates) → Import es 10-100x más rápido.
- Si el modelo tiene muchas medidas que iteran con `FILTER` → Import.

## 5. Importar desde Web (Web connector)

### Caso fácil: tabla HTML estática

1. **Obtener datos → Web**.
2. Pegar la URL.
3. Power BI muestra las tablas detectadas. Seleccionar.
4. Transformar en Power Query como cualquier otra fuente.

### Limitaciones

- **No ejecuta JavaScript**. Si la página es una SPA (React, Angular, Vue), ves la tabla vacía.
- **No maneja login visual** (OAuth con redirect). Solo autenticación básica, bearer, API keys.
- **Sigue redirecciones automáticamente** pero puede fallar si hay captcha o cookies.

### Truco: la URL puede ser también de descarga directa

Si la URL devuelve un CSV, JSON o XML, Power BI lo detecta y lo trata como tal. Útil para endpoints públicos como `datos.gob.es`, APIs de bancos centrales, etc.

## 6. Web.Contents en M: el conector programático

`Web.Contents` es la única función M que hace HTTP. Todo el "scraping" serio pasa por aquí.

### Firma

```m
Web.Contents(
    url as text,
    optional options as record
) as binary
```

### Opciones disponibles

| Opción | Tipo | Uso |
|---|---|---|
| `Headers` | record | Cabeceras HTTP (`Authorization`, `Accept`, etc.) |
| `Query` | record | Parámetros que se añaden como `?key=value` |
| `RelativePath` | text | Path que se concatena a `url` |
| `Content` | binary | Body para POST/PUT (normalmente JSON) |
| `ManualStatusHandling` | list | Status codes a no tratar como error (e.g. `{404}`) |
| `Timeout` | duration | Timeout de la llamada |

### Ejemplo mínimo

```m
let
    Source = Web.Contents("https://api.example.com/data")
in
    Source
```

### Ejemplo con headers y query

```m
let
    Source = Web.Contents(
        "https://api.example.com/v1/sales",
        [
            Headers = [
                #"Authorization" = "Bearer ABC123",
                #"Accept"        = "application/json"
            ],
            Query = [
                #"year"  = "2025",
                #"limit" = "1000"
            ]
        ]
    ),
    Json = Json.Document(Source)
in
    Json
```

## 7. Web scraping práctico: HTML y JSON

### Parsear HTML con `Html.Table`

Si el Web connector no detecta la tabla automáticamente:

```m
let
    Source = Web.Contents("https://example.com/page"),
    Html = Html.Document(Source),
    Tabla = Html.Table(
        Html,
        {
            {"Columna1", "table tr > td:nth-child(1)", null},
            {"Columna2", "table tr > td:nth-child(2)", null}
        },
        [RowSelector = "table tr"]
    )
in
    Tabla
```

`Html.Table` usa selectores CSS. Útil cuando el Web connector se atasca.

### Parsear JSON

```m
let
    Source    = Web.Contents("https://jsonplaceholder.typicode.com/users"),
    AsJson    = Json.Document(Source),
    AsTable   = Table.FromList(AsJson, Splitter.SplitByNothing()),
    Expanded  = Table.ExpandRecordColumn(
        AsTable, "Column1",
        {"id", "name", "email", "address"},
        {"id", "name", "email", "address"}
    ),
    // Expandiendo un sub-objeto
    AddCity = Table.ExpandRecordColumn(
        Expanded, "address", {"city"}, {"city"}
    )
in
    AddCity
```

### Patrón: API con estructura anidada

```m
// Respuesta: { "data": [ { "id": 1, "user": { "name": "Ana" } }, ... ] }
let
    Source = Json.Document(Web.Contents("https://api.example.com/list")),
    // Navegar hasta el array
    Data   = Source[data],
    AsTable = Table.FromList(Data, Splitter.SplitByNothing()),
    Expanded = Table.ExpandRecordColumn(
        AsTable, "Column1", {"id", "user"}, {"id", "user"}
    ),
    AddUser = Table.ExpandRecordColumn(
        Expanded, "user", {"name", "email"}, {"name", "email"}
    )
in
    AddUser
```

## 8. Autenticación

| Tipo | Cómo se implementa |
|---|---|
| **Sin auth (público)** | Solo `Web.Contents(url)`. |
| **API Key en query** | `Query = [#"api_key" = "ABC"]`. ⚠️ Poca seguridad. |
| **API Key en header** | `Headers = [#"X-API-Key" = "ABC"]`. Lo más común. |
| **Basic Auth** | `Headers = [#"Authorization" = "Basic " & Binary.ToText(Text.ToBinary("user:pass"), BinaryEncoding.Base64)]`. |
| **Bearer Token (OAuth2)** | `Headers = [#"Authorization" = "Bearer " & token]`. El token se renueva con otra llamada. |
| **OAuth2 con flujo de código** | Power BI no lo soporta directamente. Solución: un proxy (Azure Function, AWS Lambda) que haga el flujo y exponga un endpoint con API key. |

> **Cuidado con los secretos**: nunca pegues un token en una query M que se publique. Usa **Power BI Parameters** o una puerta de enlace con credenciales. Para producción, usa Azure Key Vault + Dataflow con service principal.

## 9. Paginación

### Estilo 1: page=n en la URL

```m
let
    Pages = List.Transform(
        {1..10},
        (p) => Json.Document(
            Web.Contents(
                "https://api.example.com/v1/items",
                [Query = [page = Text.From(p), limit = "100"]]
            )
        )
    ),
    Combined = Table.FromList(Pages, Splitter.SplitByNothing(), {"Page"}),
    Expanded = Table.ExpandListColumn(Combined, "Page")
in
    Expanded
```

### Estilo 2: cursor en la respuesta (más moderno)

```m
let
    fnGetPage = (cursor as text) as list =>
        let
            url = "https://api.example.com/v1/items?cursor=" & cursor,
            response = Json.Document(Web.Contents(url)),
            items = response[data],
            nextCursor = response[next_cursor]
        in
            items,

    // Iterar hasta que next_cursor sea null
    AllPages = List.Generate(
        () => [c = "", page = fnGetPage("")],
        each [c] <> null,
        each [c = page{0}[next_cursor], page = fnGetPage([c])],
        each [page]
    )
in
    AllPages
```

### Estilo 3: header `Link` (estilo GitHub)

GitHub devuelve una cabecera `Link: <url>; rel="next"`. Power Query puede parsearla, pero es complejo. Más fácil: extraer el número de página y paginar como en estilo 1.

## 10. Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `DataSource.Error: Web.Contents failed` | URL caída o sin permisos | Probar la URL en un navegador |
| `We cannot convert a value of type Record to Table` | La respuesta es un objeto, no un array | Usar `Record.ToTable()` |
| `Formula.Firewall` | Mezclas de privacidad (local + web) | Configurar `Archivo → Opciones → Privacidad` |
| Tabla vacía | La web renderiza con JS | Usar API en su lugar |
| `429 Too Many Requests` | Rate limit | Añadir `Function.InvokeAfter(..., #duration(0,0,0,1))` |
| Solo 1000 filas devueltas | API pagina, no se ha implementado | Implementar paginación manual |
| Tipos mezclados en una columna | La API devuelve string o number según registro | `Table.TransformColumns(..., each try Value.FromText(_) otherwise _)` |
| `DataSource.Error: The credentials provided...` | La API requiere auth que no has pasado | Revisar headers, añadir Authorization |
| `OutOfMemory` al expandir JSON gigante | El JSON tiene listas de 50k elementos | `Table.AddColumn` con extracción selectiva, no expandir todo |

## 11. Query folding: la optimización que no se ve

**Query folding** = Power Query traduce tus pasos M a una sola query SQL (o similar) que se ejecuta en la BD origen. Solo aplica a orígenes SQL/OData/ODBC. **NO** aplica a CSV, Excel, JSON, Web.

### Ejemplo visual

```m
// Pasos que SÍ hacen folding:
SQL.Source                              → "SELECT * FROM Sales"
   → Table.SelectRows(..., [year]=2024)  → "SELECT * FROM Sales WHERE year = 2024"
   → Table.Group(..., {"category"})      → "SELECT category, SUM(...) FROM ... GROUP BY category"

// Pasos que ROMPEN el folding:
   → Table.AddColumn(..., each MyFn([x]))  → Power Query no sabe traducir MyFn a SQL
```

### Cómo verificar

Click derecho sobre la query en Power Query → **Ver consulta nativa**. Si ves el SQL, hay folding. Si no, todo se procesa en local.

### Por qué importa

- **Con folding**: 1M filas se filtran en SQL → solo 5k llegan a Power BI → modelo pequeño, refresh rápido.
- **Sin folding**: 1M filas llegan a Power BI → 1M filas se procesan en local → modelo grande, refresh lento.

### Reglas para mantener el folding

- Pon los filtros lo antes posible.
- No uses `Table.AddColumn` con funciones personalizadas hasta el final.
- Si rompes el folding, **documenta** por qué (a veces la transformación lo requiere).

## 12. Buenas prácticas generales

1. **Tipar siempre** desde el primer paso. Un tipo mal asignado genera bugs silenciosos.
2. **Renombrar a `snake_case`** sin acentos. Evita comillas y errores en DAX.
3. **`Reference` por defecto**, no `Duplicate`. Más eficiente y mantenible.
4. **Eliminar columnas/filas no usadas** antes de cargar. Cada columna que llega al modelo ocupa memoria.
5. **Nombra los pasos** con frases: `"Filtrados años >= 2015"`, no `"Filas filtradas1"`.
6. **Parametriza lo variable**: rutas, años, regiones, etc. Convierte lo que se repite en parámetro.
7. **Verifica el query folding** en orígenes de BD. Si lo pierdes, es la principal causa de lentitud.
8. **No hagas en Power Query lo que es trabajo de DAX** (totales, ratios) y viceversa.
9. **Cuidado con el privacidad** (`Archivo → Opciones → Privacidad`). Mezclar local + web puede requerir configuración explícita.
10. **Documenta queries complejas** con un comentario al inicio o una descripción en el panel.
