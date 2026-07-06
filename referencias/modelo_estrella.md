# Modelo en estrella · Dimensiones, hechos y relaciones

> Referencia completa sobre el **esquema en estrella** (star schema), el patrón de modelado dimensional más usado en Power BI, data warehouses y BI en general. Si después de leer esto quieres ver la mecánica aplicada a datos reales, mira las Clases 1 y 3 del curso.

## Índice

1. [¿Qué es el modelo en estrella?](#1-qué-es-el-modelo-en-estrella)
2. [Componentes fundamentales](#2-componentes-fundamentales)
3. [Tabla de hechos (Fact Table)](#3-tabla-de-hechos-fact-table)
4. [Tabla de dimensión (Dimension Table)](#4-tabla-de-dimensión-dimension-table)
5. [Relaciones entre tablas](#5-relaciones-entre-tablas)
6. [Variantes del esquema en estrella](#6-variantes-del-esquema-en-estrella)
7. [Dimensiones avanzadas](#7-dimensiones-avanzadas)
8. [Cómo se monta en Power BI](#8-cómo-se-monta-en-power-bi)
9. [Ejemplo aplicado: Superstore (Clase 1)](#9-ejemplo-aplicado-superstore-clase-1)
10. [Ejemplo aplicado: Sales and Marketing (Clase 3)](#10-ejemplo-aplicado-sales-and-marketing-clase-3)
11. [Buenas prácticas](#11-buenas-prácticas)
12. [Anti-patrones](#12-anti-patrones)
13. [Glosario rápido](#13-glosario-rápido)
14. [Referencias](#14-referencias)

---

## 1. ¿Qué es el modelo en estrella?

El modelo en estrella es un patrón de **modelado dimensional** donde:

- Una **tabla central de hechos** almacena los eventos medibles del negocio (ventas, transacciones, clics, etc.).
- Varias **tablas de dimensión** rodeándola almacenan el "quién, qué, cuándo, dónde, cómo" de esos eventos.
- Las relaciones son siempre **1:N** desde la dimensión al hecho, formando una **estrella** visual.

Fue popularizado por **Ralph Kimball** en los 90 como alternativa a los esquemas normalizados (3NF) usados en sistemas operacionales. La idea central:

> "Un data warehouse se diseña para **consultar**, no para escribir. Optimiza para que las preguntas de negocio se respondan con joins simples y agregaciones rápidas."

### ¿Por qué funciona tan bien en Power BI?

| Característica | Beneficio |
|---|---|
| Pocas tablas, todas relacionadas directamente con el hecho | El motor VertiPaq comprime y consulta rápido. |
| 1:N con dirección única | El filter context fluye de forma predecible. |
| Medidas aisladas del cálculo | DAX puede agregar/filtrar sin reescribir la lógica. |
| Dimensiones pequeñas (decenas a millones de filas) | Caben en memoria; las joins son O(1) en la práctica. |
| Estructura estable | El modelo se entiende y se extiende sin reescribir. |

### ¿Cuándo NO usarlo?

- Cuando los datos son **poco estructurados** (logs, eventos, JSON jerárquico) → considera un modelo de evento / wide table.
- Cuando necesitas **transacciones ACID** → no es un sistema transaccional, es analítico.
- Cuando el dominio es muy **graph-like** (redes sociales, knowledge graphs) → considera un grafo.

---

## 2. Componentes fundamentales

### Diagrama genérico

```
                  dim_date
                     │ 1
                     │
                     ▼ N
   dim_customer ───► ┌──────────┐ ◄─── dim_product
        1           │  fct_    │  1           1
                    │  sales   │
                    └──────────┘
                     ▲ N
                     │
                     │ 1
                  dim_store
```

La tabla central es la **tabla de hechos** (`fct_sales`). Las cuatro satélites son **dimensiones** (`dim_date`, `dim_customer`, `dim_product`, `dim_store`). Cada dimensión está conectada a `fct_sales` con una relación 1:N (una fila de la dimensión puede estar en muchas filas del hecho).

### Roles

| Tabla | Rol | Tamaño típico | Cardinalidad |
|---|---|---|---|
| **Hechos** (fact) | Almacena eventos y métricas | Millones a billones de filas | Alta |
| **Dimensiones** (dim) | Describe los ejes de análisis | Cientos a millones | Baja a media |

---

## 3. Tabla de hechos (Fact Table)

### Definición

Una tabla que almacena **eventos medibles del negocio**. Cada fila es un hecho: una venta, un clic, una transacción, una lectura de sensor.

### Características

- **Estrecha** (pocas columnas): típicamente entre 5 y 20.
- **Larga** (muchas filas): desde miles hasta billones.
- Contiene **claves foráneas** a las dimensiones y **métricas numéricas** (Sales, Quantity, Revenue, etc.).
- Las métricas son **aditivas** o casi siempre (más sobre esto abajo).
- La clave primaria suele ser **compuesta** (concatenación de varias FKs) o un **surrogate key** numérico.

### Ejemplo

| order_id | order_date | customer_id | product_id | store_id | quantity | sales | discount | profit |
|---|---|---|---|---|---|---|---|---|
| CA-2017-152156 | 2017-11-08 | CG-12520 | FUR-BO-10001798 | STORE-1 | 2 | 261.96 | 0.0 | 41.91 |
| CA-2017-152156 | 2017-11-08 | CG-12520 | FUR-CH-10000454 | STORE-1 | 3 | 731.94 | 0.0 | 219.58 |
| CA-2017-138688 | 2017-06-12 | DV-13045 | OFF-LA-10000240 | STORE-3 | 2 | 14.62 | 0.0 | 6.87 |

Una fila = una línea de pedido (una combinación de producto vendido a un cliente en una fecha en una tienda).

### Tipos de tablas de hechos

#### 3.1 Transaccional (Transaction Fact)

- Una fila por evento.
- Es el tipo más común. Las ventas, los clics, los pagos.
- **Ejemplo**: cada línea de pedido en `fct_sales`.

#### 3.2 Snapshot periódico (Periodic Snapshot)

- Una fila por **periodo + entidad**.
- Típicamente para inventarios, balances, métricas de salud.
- **Ejemplo**: una fila por producto por día con stock disponible.

| snapshot_date | product_id | store_id | units_in_stock | units_on_order |
|---|---|---|---|---|
| 2025-01-01 | P-001 | S-1 | 50 | 10 |
| 2025-01-01 | P-002 | S-1 | 200 | 0 |
| 2025-01-02 | P-001 | S-1 | 48 | 0 |

#### 3.3 Snapshot acumulativo (Accumulating Snapshot)

- Una fila por **entidad con ciclo de vida** (un pedido, un préstamo, un ticket).
- Las fechas de etapas se actualizan conforme la entidad avanza.
- **Ejemplo**: una fila por pedido con columnas para `order_date`, `ship_date`, `delivery_date`, `payment_date`.

| order_id | order_date | ship_date | delivery_date | current_status |
|---|---|---|---|---|
| ORD-1 | 2025-01-10 | 2025-01-12 | 2025-01-15 | Delivered |
| ORD-2 | 2025-01-11 | NULL | NULL | Awaiting Shipment |

> Las columnas de fecha se quedan en BLANK hasta que ocurre el evento. Esto es importante: si usas `BLANK()` correctamente en DAX, una medida tipo "tiempo medio de entrega" funciona sin más.

### Grano (Grain) — el concepto más importante

El **grano** de una tabla de hechos es el nivel de detalle de una fila. Definirlo mal es el error #1 en modelado.

> **Pregunta clave**: "¿Qué representa una fila de esta tabla?"

Para `fct_sales` del Superstore, una fila = **una línea de pedido**. Esto define qué se puede agregar:
- ✅ Sumar `sales` por cliente (porque cada línea pertenece a un cliente).
- ❌ Sacar el "precio unitario" promedio sin saber qué incluye (puede haber descuentos variables).

Si defines el grano mal (por ejemplo, "una fila por pedido en lugar de línea de pedido"), perderás la capacidad de calcular cosas como "producto más vendido" sin reescribir el modelo.

### Tipos de medidas según la aditividad

| Tipo | Definición | Ejemplos | Comportamiento al agregar |
|---|---|---|---|
| **Aditiva** | Se puede sumar en cualquier dimensión | `sales`, `quantity`, `profit` | Suma directa |
| **Semi-aditiva** | Se puede sumar en algunas dimensiones pero no en otras | `balance_cuenta`, `unidades_en_stock`, `cabeza_ganado` | No sumar en dimensión tiempo (sería doble conteo) |
| **No aditiva** | No se puede sumar en ninguna dimensión | `ratio`, `precio_unitario`, `margen_pct`, `temperatura` | Calcular siempre desde hechos aditivos (`DIVIDE([Profit], [Sales])`) |

> **Regla**: las medidas **no aditivas** casi nunca deberían ser columnas de la tabla de hechos. Es mejor guardarlas como cálculo DAX sobre medidas aditivas.

---

## 4. Tabla de dimensión (Dimension Table)

### Definición

Una tabla que **describe las entidades** de los hechos. Contiene los atributos por los que el usuario filtra, agrupa o etiqueta.

### Características

- **Ancha** (muchas atributos, decenas a cientos).
- **Corta** (relativamente pocas filas, miles a millones).
- **Alta densidad descriptiva**: texto, categorías, jerarquías, flags.
- Una **clave primaria** única (la surrogate key) que se referencia desde los hechos.
- **No se agregan**: las dimensiones no se suman, se filtran o se usan para agrupar.

### Ejemplo: `dim_product`

| product_id (PK) | product_name | category | sub_category | manufacturer | launch_date | is_own_brand |
|---|---|---|---|---|---|---|
| P-001 | Bush Somerset Bookcase | Furniture | Bookcases | Bush | 2018-03-01 | false |
| P-002 | Hon Deluxe Stacking Chair | Furniture | Chairs | Hon | 2019-05-15 | false |
| P-003 | Acme 8-Outlet Surge | Office Supplies | Appliances | Acme | 2020-01-10 | true |

### Componentes típicos de una dimensión

| Componente | Ejemplo | Notas |
|---|---|---|
| **Surrogate key** (PK) | `product_id = 1, 2, 3...` | Entero, sin significado de negocio. |
| **Natural key** | SKU, código externo | A veces se incluye como atributo. |
| **Atributos descriptivos** | `category`, `sub_category`, `color` | Texto o categoría. |
| **Jerarquías** | `category > sub_category > product` | Power BI las explota en drill-down. |
| **Atributos calculados** | `is_active`, `price_tier` | Pre-calculados en SQL o Power Query. |
| **Metadatos de auditoría** | `created_at`, `updated_at` | Útiles para SCD. |

### Surrogate key vs natural key

| Aspecto | Surrogate key | Natural key |
|---|---|---|
| Naturaleza | Entero autogenerado (1, 2, 3...) | Código de negocio (SKU, "FUR-BO-10001798") |
| Estabilidad ante cambios | No cambia si el negocio renombra | Cambia si se renombra o se reasigna |
| Espacio | 4 bytes | Hasta 50+ bytes |
| Velocidad de join | O(1) | Más lento, comparación de strings |
| Riesgo | Pierde significado de negocio | Es legible, fácil de debuggear |

> **Recomendación**: en el modelo dimensional, usa siempre **surrogate key** como PK de la dimensión y como FK en el hecho. Conserva la natural key como atributo de la dimensión.

### Tipos de dimensiones

Ver [sección 7](#7-dimensiones-avanzadas) para detalle de cada tipo.

---

## 5. Relaciones entre tablas

### Cardinalidad

| Cardinalidad | Notación | Cuándo se usa |
|---|---|---|
| **1:1** | 1──1 | Rara. Para拆分 una tabla por seguridad o rendimiento. |
| **1:N** (uno a muchos) | 1──<N | **El pan de cada día** en el modelo en estrella. |
| **N:M** (muchos a muchos) | N──<M | Evitar. Requiere tabla puente. |

### Dirección del filtro cruzado (Cross-filter direction)

En Power BI, cuando filtras por una dimensión, el filtro se propaga al hecho. Eso es la **dirección del filtro**. Hay tres opciones:

#### 5.1 Single direction (1 ──► N)

```
dim_date ──► fct_sales
```

Filtrar por `dim_date` afecta a `fct_sales`. Filtrar por `fct_sales` no afecta a `dim_date` (lo cual no tiene sentido hacerlo igualmente).

> **Esto es lo correcto en 95% de los casos.** El filtro fluye de la dimensión al hecho.

#### 5.2 Both directions (bidireccional)

```
dim_date ◄──► fct_sales
```

Filtrar por `dim_date` afecta a `fct_sales` Y al revés: filtrar por una medida calculada sobre `fct_sales` podría filtrar `dim_date`.

> **Úsalo solo cuando lo necesites explícitamente.** Por ejemplo, una tabla "puente" entre dos dimensiones. **Nunca** "por si acaso" — genera ambigüedad y bugs sutiles.

#### 5.3 None

No hay propagación del filtro. La relación existe para navegación pero no filtra.

> Muy raro. Se usa en relaciones decorativas o en escenarios avanzados con `CALCULATE` y `CROSSFILTER`.

### Relaciones activas vs inactivas

Solo puede haber **una relación activa** entre dos tablas. Las demás son **inactivas** y se activan con `USERELATIONSHIP()` en medidas.

Caso típico: **dimensión de rol**.

```
                    ┌──── dim_date[Date]
                    │      (PK)
                    │
            ┌───────┴────────┐
            │                │
   ACTIVA   │                │  INACTIVA
            ▼                ▼
   fct_sales[order_date]   fct_sales[ship_date]
```

Para usar la fecha de envío como filtro:

```dax
Sales by Ship Date =
CALCULATE (
    [Total Sales],
    USERELATIONSHIP ( dim_date[Date], fct_sales[ship_date] )
)
```

> **Más detalle**: en el README de la Clase 3 se trabaja con 3 dimensiones de rol (Date, Zip, ProductID). Léelo si quieres ver el patrón en acción.

### Integridad referencial

Una buena relación en estrella tiene **integridad referencial**: cada FK del hecho existe en la PK de la dimensión. Si no:

- Filas "huérfanas" se excluyen del join.
- En Power BI: el visual "filtra a los que sí matchean" silenciosamente.
- Solución: limpiar las FKs en Power Query antes de cargar.

---

## 6. Variantes del esquema en estrella

### 6.1 Snowflake (copo de nieve)

La dimensión está **normalizada** en varias tablas. Por ejemplo, `dim_product` se separa en `dim_product`, `dim_subcategory`, `dim_category`, relacionadas entre sí.

```
dim_category ──► dim_subcategory ──► dim_product ──► fct_sales
```

| Cuándo se usa | Por qué evitarlo en Power BI |
|---|---|
| Cargas muy grandes de dimensiones (millones de filas) | Más joins = más lento. El optimizador de VertiPaq no lo aprovecha. |
| Reutilización de subcategorías en otros hechos | Más complejidad de modelo. |

> **En Power BI, preferimos siempre la estrella "pura"**. Si `dim_category` tiene 5 filas, déjala dentro de `dim_product` como columna.

### 6.2 Galaxy / Constellation (múltiples hechos)

Varias tablas de hechos comparten dimensiones. Es el caso del Sales and Marketing Sample (Clase 3):

```
dim_date ──► fct_sales
dim_date ──► fct_sentiment
dim_product ──► fct_sales
dim_product ──► fct_sentiment
dim_geo ──► fct_sales
dim_geo ──► fct_sentiment
```

> Las dimensiones son **compartidas** (a veces llamadas "conformadas" si son idénticas en ambos hechos).

### 6.3 Star con dimensión degenerada

Una "dimensión" que en realidad es solo un identificador en la tabla de hechos, sin tabla aparte.

Ejemplo: `fct_sales` tiene `order_id` y `invoice_number` directamente. No tienen descripción asociada (o si la tienen, está en otra parte).

```
fct_sales:
  order_id (PK degenerada)
  customer_id ──► dim_customer
  product_id  ──► dim_product
  sales, quantity, profit
```

> Útil para identificadores operacionales (número de pedido, número de transacción) que el usuario quiere ver/filtrar pero que no justifican una tabla aparte.

---

## 7. Dimensiones avanzadas

### 7.1 Dimensiones conformadas (Conformed Dimensions)

Una dimensión **compartida por varios hechos** con la **misma clave y los mismos atributos**. Ejemplo: `dim_date` usada por `fct_sales` y `fct_sentiment`. Permite análisis cruzados consistentes.

### 7.2 Dimensiones de rol (Role-Playing Dimensions)

Una dimensión que se conecta al mismo hecho **varias veces**, cada vez con un rol distinto.

Ejemplo: `dim_date` → `fct_sales[order_date]` (rol "Order Date") y `dim_date` → `fct_sales[ship_date]` (rol "Ship Date"). Una sola tabla, dos relaciones inactivas/activas.

### 7.3 Junk Dimensions

Una dimensión que agrupa **flags y categorías de baja cardinalidad** que no justifican una tabla propia. Por ejemplo, `dim_junk`:

| junk_id | payment_method | order_channel | is_gift | is_priority |
|---|---|---|---|---|
| 1 | Credit Card | Web | false | false |
| 2 | Cash | Store | false | false |
| 3 | Credit Card | Web | true | true |

> Reduce el número de dimensiones y FKs en el hecho. Útil cuando tienes 5-10 flags que no forman un dominio coherente.

### 7.4 Dimensiones degeneradas (Degenerate Dimensions)

Identificadores que viven en la tabla de hechos (ver 6.3). `order_id`, `invoice_number`, `transaction_id`.

### 7.5 Slowly Changing Dimensions (SCD)

Cómo manejar cambios en atributos de dimensión a lo largo del tiempo.

| Tipo | Comportamiento | Ejemplo | Cuándo usar |
|---|---|---|---|
| **SCD Tipo 1** | Sobrescribe el valor antiguo. No hay historia. | Un cliente cambia de teléfono → se actualiza. | Cuando no importa la historia. |
| **SCD Tipo 2** | Crea una nueva fila con fechas de vigencia. | Cliente cambia de segmento → fila nueva con `valid_from = hoy`. | Cuando importa la historia (e.g. "cuánto facturó este cliente cuando estaba en segmento X"). |
| **SCD Tipo 3** | Añade columna con el valor anterior. | Cliente tiene `current_segment` y `previous_segment`. | Cuando solo necesitas "antes/después", no toda la historia. |
| **SCD Tipo 4** | Tabla aparte con la historia. | `dim_customer` actual + `dim_customer_history` con cambios. | Cuando la historia es muy densa y no quieres inflar la dimensión. |

En Power BI, el más usado es **SCD Tipo 2**, porque se puede modelar con columnas `valid_from` y `valid_to` y aplicar filtro temporal con `CALCULATE + FILTER`.

### 7.6 Jerarquías

Atributos relacionados que el usuario quiere drill-down. Power BI las soporta nativamente:

- **Naturales**: `Año > Trimestre > Mes > Día` (temporales).
- **Basadas en atributos**: `Categoría > Subcategoría > Producto`.
- **Padre-hijo** (parent-child): organigramas, jerarquías de empleados. Se modelan con `PATH`, `PATHITEM`, `PATHCONTAINS`. Más complejo, requiere medidas cuidadosas.

---

## 8. Cómo se monta en Power BI

### Pasos prácticos

1. **Identificar el grano del hecho**. ¿Una fila = qué exactamente?
2. **Identificar las métricas**. ¿Qué se suma? ¿Qué se promedia? ¿Qué se cuenta?
3. **Listar los ejes de análisis**. ¿Por qué atributos se va a filtrar/agrupar?
4. **Importar los datos en Power Query** (CSVs, Excel, BD). Tipar correctamente.
5. **Crear surrogate keys** si no las hay (o usar las que vengan).
6. **Crear las dimensiones por Reference** en Power Query. Quitar duplicados.
7. **Crear el modelo en la vista de Modelo** con relaciones 1:N unidireccionales.
8. **Marcar la tabla de fechas** si la hay.
9. **Ocultar las FKs** en la tabla de hechos (no se agregan, solo filtran).
10. **Crear medidas** en una tabla `_measures`.

### Cómo verificar que el modelo es correcto

- [ ] La tabla de hechos **no tiene** columnas de descripción (atributos van en dimensiones).
- [ ] Las dimensiones **no tienen** medidas (sumables). Solo atributos.
- [ ] Todas las relaciones son **1:N** unidireccionales (salvo casos justificados).
- [ ] Cada FK del hecho **existe** en la PK de su dimensión.
- [ ] La tabla de fechas está **marcada** y todas las relaciones de fecha la usan.
- [ ] Las medidas DAX usan **filter context** correctamente (no `FILTER(ALL(...))` sin razón).
- [ ] El modelo es **navegable mentalmente**: en 5 segundos puedes explicar cada tabla.

---

## 9. Ejemplo aplicado: Superstore (Clase 1)

### Diagrama

```
                dim_date
                   │ 1
                   │
                   ▼ N
dim_customer ──► ┌──────────┐ ◄─── dim_product
     1           │  fct_    │  1           1
                 │  sales   │
                 └──────────┘
                   ▲ N
                   │
                   │ 1
                dim_geo
```

### `fct_sales` (tabla de hechos)

| Columna | Tipo | Rol |
|---|---|---|
| `row_id` | Entero | PK surrogate (también degenerada) |
| `order_id` | Texto | Dimensión degenerada (folio del pedido) |
| `order_date` | Fecha | FK → `dim_date` |
| `ship_date` | Fecha | FK → `dim_date` (rol "Ship Date", inactiva) |
| `customer_id` | Texto | FK → `dim_customer` |
| `product_id` | Texto | FK → `dim_product` |
| `city`, `state`, `region`, `country` | Texto | **Debería** ser FK → `dim_geo`. En el modelo simple de Clase 1 se queda en el hecho (anti-patrón menor). |
| `postal_code` | Texto | **Debería** ser FK → `dim_geo`. |
| `sales` | Decimal fijo | Medida aditiva |
| `quantity` | Entero | Medida aditiva |
| `discount` | Decimal | Atributo de la línea (no siempre se agrega) |
| `profit` | Decimal fijo | Medida aditiva |

### Dimensiones

- **`dim_date`**: PK = `Date`. Atributos: `Year`, `Quarter`, `Month`, `MonthName`, `YearMonth`, `DayOfWeek`, `IsWeekend`.
- **`dim_customer`**: PK = `CustomerID`. Atributos: `CustomerName`, `Segment`.
- **`dim_product`**: PK = `ProductID`. Atributos: `ProductName`, `Category`, `SubCategory`.
- **`dim_geo`**: PK = `PostalCode`. Atributos: `City`, `State`, `Region`, `Country`.

### Medidas

- `[Total Sales] = SUM(fct_sales[sales])` — aditiva.
- `[Total Profit] = SUM(fct_sales[profit])` — aditiva.
- `[Profit Margin %] = DIVIDE([Total Profit], [Total Sales])` — **no aditiva**, se calcula.
- `[Sales YoY %]` — time intelligence, depende de `dim_date`.

---

## 10. Ejemplo aplicado: Sales and Marketing (Clase 3)

### Diagrama (constelación de hechos)

```
                  dim_date
                  (PK: Date)
                  /        \
              1  /            \  1
                /              \
               ▼ N            N ▼
        fct_sales           fct_sentiment
        (1.26M filas)       (21k filas)
            │  N                │  N
            │ 1                 │ 1
            ▼                   ▼
       dim_geo              dim_manufacturer
       (PK: Zip)            (PK: ManufacturerID)
                                ▲
                                │ 1
                                ▼ N
                            dim_product
                            (PK: ProductID)
```

### Hechos

- **`fct_sales`**: 1.26M filas. Una fila = venta diaria de un producto en un ZIP. Métricas: `Units`, `Revenue`.
- **`fct_sentiment`**: 21k filas. Una fila = score de sentimiento de mercado de un producto de un fabricante en un estado en una fecha. Métrica: `Score` (semi-aditiva, se promedia).

### Dimensiones

- **`dim_date`**: compartida por ambos hechos con roles distintos.
- **`dim_geo`**: compartida (PK: Zip en `fct_sales`, `zip` en `fct_sentiment`).
- **`dim_manufacturer`**: PK `ManufacturerID`. Atributo clave: `MfgisVanArsdel` (Yes/No), que distingue "nuestra marca" vs "competencia".
- **`dim_product`**: PK `ProductID`. Atributos: `Category`, `Segment`, `ManufacturerID` (FK a `dim_manufacturer`).

### Dimensiones de rol

- `dim_date[Date]` → `fct_sales[Date]` (rol "Order Date", **activa**).
- `dim_date[Date]` → `fct_sentiment[Date]` (rol "Sentiment Date", **inactiva**).
- `dim_geo[Zip]` → `fct_sales[Zip]` (**activa**).
- `dim_geo[Zip]` → `fct_sentiment[zip]` (**inactiva**).

### Medidas clave

```dax
Total Revenue = SUM ( fct_sales[Revenue] )
Avg Sentiment = AVERAGE ( fct_sentiment[Score] )
Sentiment by Sales Date = CALCULATE ( [Avg Sentiment], USERELATIONSHIP ( dim_date[Date], fct_sentiment[Date] ) )
```

---

## 11. Buenas prácticas

### Naming

| Elemento | Convención | Ejemplo |
|---|---|---|
| Tabla de hechos | `fct_` + singular | `fct_sales`, `fct_orders` |
| Tabla dimensión | `dim_` + singular | `dim_customer`, `dim_product` |
| Tabla calendario | `dim_date` | `dim_date` |
| Tabla auxiliar | nombre descriptivo sin prefijo estándar | `_measures`, `TopN Params`, `WhatIf Discount` |
| Columna PK | `<table>_id` o `<table>_key` | `customer_id`, `product_key` |
| Columna FK | `<referenced_table>_id` | `customer_id` en `fct_sales` |
| Medida | nombre de negocio | `Total Sales`, `Profit Margin %` |
| Columna | `snake_case`, sin acentos | `order_date`, `customer_id` |

### Estructura

- **Una sola tabla de hechos por "hecho de negocio"**. Mezclar hechos de naturalezas distintas (ventas + objetivos + inventario) en la misma tabla es anti-patrón.
- **Una sola tabla de fechas**. Dimensiones de rol con `USERELATIONSHIP`, no duplicar.
- **Surrogate keys** en dimensiones, referenciadas desde el hecho.
- **Atributos descriptivos** solo en dimensiones, no en el hecho.

### Relaciones

- **1:N unidireccional** por defecto.
- **Bidireccional** solo cuando sepas exactamente por qué y qué ambigüedades introduces.
- **Inactiva** cuando hay varias FKs del hecho a la misma dimensión.
- **`USERELATIONSHIP`** en medidas, no a nivel de relación.

### Documentación

- Descripción del **grano** de cada tabla de hechos.
- Lista de **atributos** de cada dimensión.
- **Diccionario de medidas** con nombre, descripción, formato, carpeta.
- Notas sobre **decisiones no obvias** (por qué se eligió SCD Tipo 2 aquí, por qué esta relación es bidireccional, etc.).

---

## 12. Anti-patrones

### ❌ Tabla plana (single big table)

```
fct_sales_full:
  row_id, order_id, order_date, customer_id, customer_name,
  customer_segment, product_id, product_name, category,
  sub_category, city, state, region, sales, quantity, profit
```

**Por qué es malo**:

- `DISTINCTCOUNT(customer_id)` requiere iterar por toda la tabla.
- Filtrar por `category` arrastra `sales` y `quantity` al cache aunque no los uses.
- Cualquier cambio de nombre de categoría requiere actualizar miles de filas.
- El modelo es grande y lento.

### ❌ Snowflake innecesario

Normalizar `dim_product` en `dim_product`, `dim_subcategory`, `dim_category` cuando la cardinalidad es de decenas de filas. Más joins, sin beneficio.

### ❌ Muchas dimensiones de fechas (sin usar `USERELATIONSHIP`)

Crear `dim_order_date`, `dim_ship_date`, `dim_due_date` con datos duplicados. Malgasta memoria y rompe time intelligence.

### ❌ Bidireccional "por si acaso"

Relaciones bidireccionales que generan ambigüedad y resultados incorrectos difíciles de debuggear. Casi siempre hay una solución con `USERELATIONSHIP` o `CROSSFILTER` en medidas.

### ❌ Many-to-many "naturales"

Relaciones N:M directas entre dos tablas. Casi siempre se resuelven con una **tabla puente** (bridge table) o cambiando el modelo.

### ❌ Medidas hardcodeadas en la tabla de hechos

Una columna `profit_margin = profit / sales` como columna calculada del hecho. Es una métrica derivada, debería ser una medida DAX.

### ❌ Sin surrogate key

Usar el código de negocio como PK. Si el código cambia (renombrado, reasignación), todo el modelo se rompe.

### ❌ Mezclar hechos en una tabla

```
sales_and_targets:
  date, product_id, region, sales, target, inventory, ...
```

- Distintos granos (sales es transaccional, inventory es snapshot).
- Distintas cadencias de actualización.
- Distintos permisos de seguridad.

Cada hecho a su tabla.

---

## 13. Glosario rápido

| Término | Definición breve |
|---|---|
| **Grano (Grain)** | Nivel de detalle que representa una fila. |
| **Hecho (Fact)** | Evento medible del negocio. |
| **Dimensión (Dimension)** | Tabla que describe los ejes de análisis. |
| **Surrogate key** | Clave numérica autogenerada, sin significado de negocio. |
| **Natural key** | Clave de negocio (SKU, código de cliente). |
| **Aditiva** | Medida que se puede sumar en cualquier dimensión. |
| **Semi-aditiva** | Medida que se puede sumar en algunas dimensiones pero no en todas (típicamente no en tiempo). |
| **No aditiva** | Medida que no se puede sumar (ratios, promedios). Se calcula con DAX. |
| **Dimensión de rol** | Una dimensión usada varias veces con roles distintos en el mismo hecho. |
| **Dimensión conformada** | Dimensión compartida por varios hechos con la misma PK. |
| **Junk dimension** | Dimensión que agrupa flags y categorías de baja cardinalidad. |
| **Slowly Changing Dimension (SCD)** | Patrón para manejar cambios en atributos de dimensión. |
| **SCD Tipo 1** | Sobrescribe el valor antiguo. Sin historia. |
| **SCD Tipo 2** | Crea fila nueva con fechas de vigencia. Mantiene historia. |
| **SCD Tipo 3** | Columna adicional con valor anterior. |
| **Snowflake** | Dimensión normalizada en varias tablas. (Evitar en Power BI). |
| **Galaxy / Constellation** | Múltiples hechos compartiendo dimensiones. |
| **Query folding** | Power Query traduce pasos M a SQL que se ejecuta en la fuente. |
| **Star schema** | Patrón de modelado con una tabla de hechos central y dimensiones alrededor. |
| **USERELATIONSHIP** | Función DAX que activa una relación inactiva en una medida. |
| **Cross-filter direction** | Sentido en que se propaga el filtro entre tablas. |
| **Tabla puente (Bridge table)** | Tabla que media una relación N:M. |
| **Time intelligence** | Conjunto de funciones DAX para comparar periodos (YoY, YTD, etc.). |

---

## 14. Referencias

Para profundizar:

- **Ralph Kimball** — *The Data Warehouse Toolkit* (3rd edition). El libro de referencia del modelado dimensional.
- **Chris Webb** — *Designing Tabular Models* (blog y libros). Específico de Analysis Services / Power BI.
- **Marco Russo** (SQLBI) — *The Definitive Guide to DAX*. Para entender el filter context y las medidas en un modelo en estrella.
- **Power BI documentation** — [Star schema guidance](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema).
- **Documentación TMDL / VertiPaq** — para entender cómo Power BI almacena el modelo internamente.

Si quieres ver el modelo en estrella aplicado:

- **Clase 1** del curso: Superstore (modelo simple, 1 hecho + 4 dimensiones).
- **Clase 3** del curso: Sales and Marketing (modelo complejo, 2 hechos + 4 dimensiones + dimensiones de rol).
