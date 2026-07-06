# Clase 4 · Producción, optimización y despliegue del informe

## Descripción del ejercicio

Llevar el modelo construido en la **Clase 3** desde un estado "funcional" hasta un estado **production-ready**. Un informe en Power BI no es de producción hasta que:

- Carga rápido (incluso con filtros pesados).
- Está protegido (cada usuario ve solo lo que le corresponde).
- Se puede navegar sin perder al usuario.
- Está desplegado en un entorno gobernado (Power BI Service + App + pipeline).
- Tiene monitorización y alertas configuradas.

Esta clase cubre el ciclo de vida desde "está en mi portátil" hasta "lo consume toda la empresa".

## Recursos necesarios

| Recurso | Detalle |
|---|---|
| `.pbix` de la Clase 3 | El modelo `fct_sales` + `dim_*` + `dim_date` con dimensiones de rol. |
| Power BI Desktop | Última versión. |
| Power BI Service | Cuenta Pro o Premium Per User (PPU). Se necesita para publicar y testear RLS. |
| DAX Studio (opcional) | [daxstudio.org](https://daxstudio.org/) — herramienta gratuita para analizar queries DAX. |
| SQL Server Profiler / Tabular Editor (opcional) | Para análisis profundo de modelo (opcional en esta clase). |

> **Dato**: la Clase 3 importó los CSVs desde `clases/class_03_modelado_dimensional_avanzado/data/`. Los datos siguen ahí, pero en Clase 4 ya trabajas con el `.pbix` resultado, no con los CSVs.

> Si no tienes cuenta Pro, puedes hacer toda la parte de Desktop (optimización, RLS en local con "View as Role", navegación). El bloque de publicación y App se puede simular guardando el `.pbix` y publicándolo en "Mi área de trabajo personal".

## Lo que vas a practicar

- **Performance Analyzer** integrado en Power BI Desktop.
- Optimización del modelo: cardinalidad, tipos, columnas calculadas vs. medidas.
- **Aggregations** (tablas de agregación).
- **Row-Level Security** (RLS) estática y dinámica con `USERNAME()` / `USERPRINCIPALNAME()`.
- Navegación avanzada: bookmarks, botones, tooltips, drill-through, parámetros de campo.
- Publicación en Power BI Service, **Apps**, suscripciones y alertas.
- **Deployment pipelines** (Dev → Test → Prod).
- Monitorización con el *metrics hub* y *refresh history*.

---

## Paso a paso

### 1. Auditoría con Performance Analyzer

1. En Power BI Desktop, ir a **Vista → Performance Analyzer** → **Iniciar grabación**.
2. Refrescar la página o interactuar con visuales.
3. Revisar la tabla: cada visual tiene una duración, dividida en *DAX query* y *Visual display*.
4. **Identificar cuellos de botella**: cualquier DAX query > 200 ms es candidata a optimizar.

Síntomas típicos y soluciones rápidas:

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Sales YTD` tarda 800 ms | `FILTER(ALL(...))` dentro de `CALCULATE` | Reescribir con `DATESYTD` o `TOTALYTD` |
| Visual de matriz lento con jerarquía | Iteración fila a fila | Cambiar a `SUMMARIZECOLUMNS` o a una columna calculada |
| Filtro de segmentación lento | Cardinalidad alta en la columna del slicer | Cambiar a una jerarquía o agrupar valores |
| Página entera > 5 s | Muchos visuales con `USERELATIONSHIP` pesados | Pre-calcular en `fct_sales` o reducir cardinalidad |

### 2. Reducir el tamaño del modelo

Abrir **Model view** y mirar el tamaño de cada tabla (esquina inferior derecha).

Acciones:

1. **Quitar columnas no usadas**: en Power Query, seleccionar y *Remove*. Esto reduce el modelo *antes* de cargar.
2. **Optimizar tipos**:
   - `Texto` con cardinalidad baja (1-10 valores únicos, p. ej. `Region`) → marcar como `No resumir` y, si es posible, convertir a `Entero` con un lookup.
   - Columnas decimales con 2 decimales → `Número decimal fijo` (4 bytes en vez de 8).
   - Fechas → `Date`, no `DateTime`.
3. **Evitar columnas calculadas DAX**: cada `CALUMN =` se materializa en el modelo. Si se puede hacer en Power Query, hazlo allí.
4. **Reducir precisión** de decimales: redondear a 2 decimales si no se necesita más precisión analítica. `ROUNDX` no cambia el tipo, así que hay que hacerlo en Power Query con `Number.Round`.

Validar el ahorro: anotar el tamaño antes y después. Una reducción del 30% es habitual.

### 3. Tablas de agregación (opcional)

Si `fct_sales` es muy grande (>5M filas), crear `fct_sales_agg` con agregados por `customer_key` + `product_key` + `month`. Mapear la relación `fct_sales_agg → dim_*` con la opción **"Agregación"** y marcar la granularidad.

Power BI usará automáticamente la tabla de agregación para visuales de alto nivel (matriz, KPI cards) y recurrirá a `fct_sales` solo para el detalle.

Para esta clase, **es opcional**. Si tu modelo es pequeño, sáltate este paso.

### 4. Row-Level Security (RLS)

#### RLS estática

1. **Modelado → Administrar roles → Crear**.
2. Nombre: `Salesperson East`.
3. Filtro DAX: `[region] = "East"`.
4. Guardar.

#### RLS dinámica (recomendada)

```dax
[region] = LOOKUPVALUE (
    dim_employee[region],
    dim_employee[employee_email], USERPRINCIPALNAME ()
)
```

Esto filtra `region` por la región del empleado cuyo email coincide con el usuario que está viendo el informe.

#### Testear en local

**Modelado → Ver como roles** → seleccionar `Salesperson East` + (opcional) un usuario concreto con *Other user*. Revisar que las visuales reflejan el filtro.

> **Importante**: `USERPRINCIPALNAME()` devuelve el UPN del usuario en Power BI Service (normalmente su email corporativo). En Desktop con "View as Role", el simulacro usa un usuario ficticio. Por eso **el test definitivo se hace en Service** tras publicar.

### 5. Navegación con bookmarks y botones

Una buena navegación convierte un informe de "colección de páginas" en "producto". Pasos:

1. Crear un **lienzo oculto** (tema oscuro, sin visuales) que actúe como navigation bar fija en todas las páginas. Marcarlo como **Page hidden**.
2. Crear **bookmarks** para cada estado de la navigation bar (botón "Inicio" activo, etc.).
3. Asignar **acciones** a los botones: `Button → Action → Bookmark → [correspondiente]`.
4. En cada página, vincular el botón de navegación al bookmark que lleva a la página correspondiente.

**Tooltips personalizados**: crear una página pequeña (200x200 px) con detalle al pasar el ratón. Marcar como `Allow use as tooltip`. En la visual principal, *Tooltip → Page → [la página tooltip]*.

**Drill-through vs drill-down vs bookmarks**:

| Mecanismo | Cuándo usarlo |
|---|---|
| Drill-down | Jerarquías del modelo (Año → Mes → Día). |
| Drill-through | Saltar a otra página con un contexto ya aplicado. |
| Bookmarks | Estados de UI (filtros aplicados, visibilidad de paneles). |
| Field parameters | Cambiar la métrica o dimensión analizada en un visual. |

### 6. Field parameters (opcional, DAX 2.x)

Permiten al usuario elegir qué métrica o dimensión se está analizando:

```dax
Metric Selector = {
    ("Sales", NAMEOF('fct_sales'[sales_amount]), 0),
    ("Profit", NAMEOF('fct_sales'[profit]), 1),
    ("Quantity", NAMEOF('fct_sales'[order_quantity]), 2)
}
```

Power BI genera automáticamente un slicer con las opciones. Ideal para que un usuario no técnico elija entre "ver ventas", "ver margen" o "ver unidades" sin tener que tocar el modelo.

### 7. Publicación y App

1. **Inicio → Publicar** → seleccionar workspace. Se recomienda un workspace dedicado (no "Mi área de trabajo personal") para gobernar el ciclo de vida.
2. Configurar la **programación de actualización** (Schedule refresh). Para orígenes que no son locales, configurar gateway si hace falta.
3. En el workspace → **Crear App**:
   - Nombre, icono, descripción.
   - Elegir qué contenido incluir.
   - Permisos: toda la organización / grupos de seguridad específicos.
4. **Suscripciones**: cada usuario puede suscribirse a una página y recibir un PDF/visor por email con la periodicidad que elija.
5. **Alertas**: en KPI cards y métricas concretas, los usuarios pueden configurar "alertarme cuando X > 1000".

### 8. Deployment pipeline

Power BI Premium / PPU incluye **Deployment Pipelines** (en workspaces Pro normales no, hay que tener licencias PPU o Premium).

Estructura típica: tres workspaces en distintas fases.

| Fase | Workspace | Quién accede |
|---|---|---|
| **Desarrollo** | `[empresa]-BI-DEV` | Solo el equipo de BI. |
| **Test** | `[empresa]-BI-TEST` | Key users / QA. |
| **Producción** | `[empresa]-BI-PROD` | Toda la organización. |

Flujo:

1. Crear los tres workspaces.
2. En el servicio, ir a **Deployment pipelines → Crear pipeline**.
3. Asignar cada workspace a una fase.
4. **Desplegar de Dev → Test** (botón): copia el contenido, mantiene las credenciales y reglas de RLS.
5. **Desplegar de Test → Prod** cuando QA haya validado.
6. En cada paso, revisar las **reglas de despliegue** (deployment rules): por ejemplo, que el dataset en Prod apunte a un origen de datos productivo, no al de test.

### 9. Monitorización

En el workspace de Prod:

- **Refresh history**: ¿cuándo se actualizó el modelo por última vez? ¿Ha fallado?
- **Usage metrics**: cuántas personas abren el informe, qué páginas son las más vistas, cuánto duran las sesiones.
- **Activity log** (en Microsoft 365 admin center): auditoría completa de accesos y cambios.
- **Alertas automáticas** configuradas a nivel de capacidad Premium (si aplica).

---

## Explicación y buenas prácticas

### Performance Analyzer: el "devtools" de Power BI

Es la herramienta mínima para empezar a optimizar. Graba cada visual con su tiempo de ejecución de query DAX y de render. Tres reglas:

1. **Optimiza primero el DAX, no el visual.** Cambiar de un gráfico de barras a uno de columnas no acelera el cálculo.
2. **Cuidado con las "medidas bonitas pero lentas".** Una medida con `FILTER(ALL(...), ...)` anidada puede funcionar con 10k filas y morir con 10M. Mide antes de celebrar.
3. **Las aggregations no son magia.** Solo aplican cuando los visuales están al nivel de la agregación. Si el usuario quiere ver líneas de pedido, no se pueden usar.

### RLS: el error que pone en riesgo datos

Los tres errores más comunes:

1. **Filtrar por `USERPRINCIPALNAME()` sin una tabla de mapping**: el UPN es el email, pero `fct_sales` probablemente no tiene email. Necesitas una dimensión con `employee_email` para cruzar.
2. **Olvidar `LOOKUPVALUE` y filtrar directamente con `USERPRINCIPALNAME()` sobre una columna que no existe en el modelo**: el filtro simplemente no aplica, y el usuario ve todos los datos sin saberlo.
3. **Asumir que la RLS se testea en Desktop**: en Desktop solo se simula. El test real es en Service con un usuario real.

> **Regla de oro**: si un informe tiene RLS, **documenta quién ve qué** y **revisar trimestralmente**. Las altas y bajas de personal son la causa #1 de fugas de datos por RLS mal mantenida.

### Aggregations: cuándo realmente valen la pena

- Tabla de hechos < 1M filas: no vale la pena. El optimizador lo maneja sin ayuda.
- 1M-50M filas: aggregations bien planteadas pueden recortar 5-10x el tiempo de carga.
- > 50M filas: necesitas aggregations + incremental refresh + modo `DirectQuery`/`Composite`. La optimización se vuelve una disciplina aparte.

### Deployment pipeline: lo que casi nunca se respeta

El pipeline existe porque **el mismo informe lo tocan varias personas**. Sin pipeline:

- Un usuario publica su versión sobre la de otro.
- Un cambio de DAX rompe un dashboard que usaba el equipo financiero.
- No hay forma de hacer rollback.

Aunque no tengas Premium, la disciplina mínima es:

1. Un workspace para desarrollo.
2. Antes de publicar a producción, **exportar y comparar** con la versión actual (Power BI tiene *Compare* en `Lineage view`).
3. **No publicar directamente** a producción sin que alguien valide.

### Bookmarks vs parámetros de campo

Los **field parameters** son *parámetros de modelo*: afectan a qué medida/dimensión se calcula. Los **bookmarks** son *estados de UI*: filtros, visibilidad, foco.

Si quieres que el usuario elija "ventas vs margen", usa field parameters. Si quieres que pueda guardar "mi vista favorita", usa bookmarks personales.

### Documentación del informe

Mínimo que debería tener cada informe en producción:

- **Descripción del dataset** (en el `.pbix` y en Service): qué contiene, de dónde viene, cada cuánto se actualiza.
- **Diccionario de medidas**: nombre, descripción, fórmula, formato, carpeta. Una tabla `_measures` con columnas `Measure`, `Description`, `Format`, `Category` ayuda.
- **Changelog**: qué cambió en cada despliegue. Un `.txt` o una página oculta en el informe.
- **Contacto**: a quién avisar si algo falla.

---

## Checklist de autoevaluación

- [ ] Performance Analyzer muestra que ninguna visual tarda > 500 ms en su DAX query.
- [ ] El modelo ocupa menos del 50% del tamaño que tenía al final de la Clase 3.
- [ ] La RLS dinámica filtra correctamente en Service con un usuario real (no "View as Role").
- [ ] Hay navegación con botones que llevan a cada página principal.
- [ ] El informe está publicado en un workspace dedicado, no en "Mi área personal".
- [ ] Hay una App creada con permisos específicos.
- [ ] Existe al menos un pipeline de despliegue (Dev → Prod como mínimo) o un proceso manual equivalente documentado.
- [ ] El refresh está programado y se ha ejecutado al menos una vez sin error.

## Extensiones posibles

- **Power BI embedded**: insertar el informe en una aplicación web propia con `embed for customer` (necesita capacidad dedicada).
- **Power Automate**: disparar un refresh cuando llega un nuevo fichero a SharePoint, o enviar un email cuando una métrica cae bajo umbral.
- **CI/CD con GitHub Actions + Power BI Project (.pbip)**: el nuevo formato `.pbip` permite versionar el informe como texto plano en Git. Es el siguiente nivel de "production-grade".
- **Integration with Dataverse / Fabric**: mover el modelo a un *lakehouse* en Microsoft Fabric y consumirlo vía *DirectLake* para datasets masivos.
