# Referencia de elementos visuales de Power BI

> Catálogo razonado de los visuales disponibles en Power BI, con guía de uso y errores comunes. Incluye los visuales **nativos** (los que vienen con Power BI Desktop) y los **personalizados** más comunes del AppSource. Para los visuales de R/Python, hay una sección aparte.

## Índice

1. [Cómo elegir el visual correcto](#1-cómo-elegir-el-visual-correcto)
2. [Visuales nativos de comparación](#2-visuales-nativos-de-comparación)
3. [Visuales nativos de tendencia](#3-visuales-nativos-de-tendencia)
4. [Visuales nativos de composición](#4-visuales-nativos-de-composición)
5. [Visuales nativos de distribución y correlación](#5-visuales-nativos-de-distribución-y-correlación)
6. [Visuales nativos geográficos](#6-visuales-nativos-geográficos)
7. [Visuales nativos de KPI y métricas](#7-visuales-nativos-de-kpi-y-métricas)
8. [Visuales nativos de tabla y matriz](#8-visuales-nativos-de-tabla-y-matriz)
9. [Visuales nativos de filtro y navegación](#9-visuales-nativos-de-filtro-y-navegación)
10. [Visuales de IA](#10-visuales-de-ia)
11. [Visuales de R / Python](#11-visuales-de-r--python)
12. [Visuales personalizados (AppSource)](#12-visuales-personalizados-appsource)
13. [Buenas prácticas de visualización](#13-buenas-prácticas-de-visualización)

---

## 1. Cómo elegir el visual correcto

| Pregunta de negocio | Visual recomendado |
|---|---|
| ¿Cómo evoluciona X en el tiempo? | Line chart, Area chart, Ribbon |
| ¿Cómo se compara X entre categorías? | Bar / column chart, Lollipop |
| ¿Cuál es la composición de un total? | Stacked bar/column 100%, Treemap, Pie/Donut (pocas categorías) |
| ¿Cómo se distribuyen los datos? | Histogram, Box plot (custom), Scatter |
| ¿Hay correlación entre dos variables? | Scatter chart, Bubble chart |
| ¿Cómo varía geográficamente? | Map, Filled map |
| ¿Cuál es la consecución de un objetivo? | KPI, Gauge, Bar con línea de referencia |
| ¿Qué está causando una variación? | Decomposition tree, Key influencers |
| ¿Qué pasaría si...? | What-if parameter + cualquier visual |
| Detalle fila a fila | Table, Matrix, Table con Drill-through |

> **Regla de oro**: la pregunta es la que manda. Si dudas entre dos visuales, elige el más simple.

---

## 2. Visuales nativos de comparación

### Stacked column chart / Clustered column chart

- **Uso**: comparar una métrica entre categorías, con o sin dimensión secundaria apilada.
- **Bueno para**: ventas por categoría y subcategoría, ventas por región y año.
- **Evitar**: si hay > 10-15 categorías (se vuelve ilegible).
- **Tips**: ordenar el eje X por valor descendente. Considera 100% stacked si la pregunta es composición.

### Bar chart (horizontal)

- **Uso**: mismo que column, pero con etiquetas largas. Categorías con nombres largos en el eje Y.
- **Bueno para**: ranking de productos, top ciudades.
- **Tips**: siempre ordenar por valor, no alfabético.

### Line and clustered column chart / Line and stacked column

- **Uso**: dos métricas con escalas distintas (e.g. ventas en barras, % margen en línea).
- **Bueno para**: ventas + ratio en el mismo visual.
- **Tips**: usar la propiedad de eje secundario para la línea si las escalas son muy diferentes.

---

## 3. Visuales nativos de tendencia

### Line chart

- **Uso**: evolución de una o varias series a lo largo del tiempo.
- **Bueno para**: ventas mensuales, tendencia de usuarios activos.
- **Tips**: respeta la jerarquía temporal. Si hay 4+ series, considera filtrar por categoría principal.

### Area chart / Stacked area chart

- **Uso**: igual que línea, pero destacando la magnitud acumulada.
- **Bueno para**: evolución de cuota de mercado, composición cambiante en el tiempo.
- **Tips**: stacked area puede falsear si las series no suman un total comparable.

### Ribbon chart

- **Uso**: ranking cambiante en el tiempo (qué categoría sube, qué baja).
- **Bueno para**: top 10 productos por año, ranking de regiones trimestre a trimestre.
- **Tips**: máximo 5-7 categorías o se vuelve confuso.

---

## 4. Visuales nativos de composición

### Pie chart / Donut chart

- **Uso**: composición de un total en 2-5 categorías.
- **Bueno para**: cuota de mercado, mix de ventas.
- **Evitar**: más de 5 categorías. Mejor treemap o bar.
- **Tips**: ordenar de mayor a menor en sentido horario. Total en el centro (donut).

### Treemap

- **Uso**: jerarquía de partes (categoría → subcategoría → producto) con tamaño proporcional.
- **Bueno para**: Pareto, mix de cartera, drill-down geográfico.
- **Tips**: usar color con criterio (no todos los cuadrados del mismo color).

### Waterfall chart

- **Uso**: análisis de variación entre un valor inicial y final, mostrando contribuciones intermedias.
- **Bueno para**: bridge de beneficio (precio - coste - descuento = margen), variaciones de presupuesto.
- **Tips**: ordena las barras de mayor a menor impacto para narrativa clara.

### Funnel chart

- **Uso**: etapas de un proceso secuencial con drop-off.
- **Bueno para**: conversión de embudo (lead → MQL → SQL → cliente),pipeline de ventas.
- **Tips**: las etapas deben ser estrictamente secuenciales. No usar para categorías paralelas.

---

## 5. Visuales nativos de distribución y correlación

### Scatter chart

- **Uso**: relación entre dos variables continuas.
- **Bueno para**: ventas vs descuento, edad vs ingresos, precio vs unidades.
- **Tips**: añade una línea de tendencia. Color por categoría para tercer eje categórico.

### Bubble chart

- **Uso**: scatter con una tercera variable como tamaño del punto.
- **Bueno para**: ventas (X) vs margen % (Y) vs cantidad (tamaño) por producto.
- **Tips**: las burbujas grandes tapan a las pequeñas. Limita a 20-30 puntos.

### Dot plot chart

- **Uso**: distribución de una métrica por categoría (alternativa a box plot).
- **Bueno para**: ventas por región, distribución de márgenes por categoría.

---

## 6. Visuales nativos geográficos

### Map (Bing Maps)

- **Uso**: puntos geográficos (ciudades, direcciones).
- **Bueno para**: ventas por ciudad, puntos de venta, incidentes.
- **Tips**: la categoría de datos debe estar bien reconocida por Bing. Lat/Lon con precisión.

### Filled map (choropleth)

- **Uso**: áreas geográficas coloreadas por intensidad.
- **Bueno para**: ventas por estado, densidad por país, KPI por región.
- **Tips**: usar escalas de color perceptualmente uniformes (e.g. Viridis). Pocos bins (4-6).

---

## 7. Visuales nativos de KPI y métricas

### Card

- **Uso**: un único número grande. El KPI más importante de la página.
- **Bueno para**: total ventas, total beneficio, % consecución.
- **Tips**: añadir comparación debajo (`vs LY`, `vs target`) con `CALL OUT VALUE` o segunda card.

### Multi-row card

- **Uso**: varios KPIs apilados con etiqueta.
- **Bueno para**: dashboard resumen con 3-5 KPIs.

### KPI

- **Uso**: valor actual vs objetivo vs tendencia.
- **Bueno para**: monitorización de objetivos (ventas vs quota, NPS vs target).
- **Tips**: el trend axis debe ser temporal. Sin tendencia, es solo un gauge.

### Gauge

- **Uso**: progreso hacia un valor objetivo, con rangos (rojo, ámbar, verde).
- **Bueno para**: consecución de cuota, % capacidad utilizada.
- **Evitar**: muchos gauges en la misma página. Distrae.

---

## 8. Visuales nativos de tabla y matriz

### Table

- **Uso**: detalle fila a fila, sin agregación visible.
- **Bueno para**: lista de pedidos, líneas de factura, detalle de cliente.
- **Tips**: formato condicional en background (data bars) para resaltar valores.

### Matrix

- **Uso**: tabla con filas y columnas dinámicas (pivot table).
- **Bueno para**: ventas por categoría × mes, consecución por región × trimestre.
- **Tips**: subtotales arriba vs abajo (configurable). Drill-down con cuidado.

---

## 9. Visuales nativos de filtro y navegación

### Slicer

- **Tipos**: lista, dropdown, range, before/after, relative date, hierarchy.
- **Uso**: filtrar el resto de visuales por valores.
- **Bueno para**: filtros de página o de informe.
- **Tips**: el slicer de fecha con "Relative date" (últimos 30 días) es muy potente para self-service.

### Button

- **Uso**: disparar acciones (bookmark, drill-through, page navigation, web URL, Q&A).
- **Bueno para**: navegación, reset de filtros, drill-through manual.
- **Tips**: tooltip con texto explicativo. Estado hover visible.

### Shapes / Text box / Image

- **Uso**: decoración, separadores, logos, anotaciones.
- **Tips**: usarlos para mejorar la lectura, no para llenar espacio.

---

## 10. Visuales de IA

### Key influencers

- **Uso**: explicar qué campos y valores influyen en una métrica.
- **Bueno para**: "¿qué hace que un cliente sea rentable?", "¿por qué suben las devoluciones?".
- **Tips**: la métrica objetivo debe ser binaria o categórica.

### Decomposition tree

- **Uso**: drill-down AI-driven. Power BI elige el siguiente nivel que más explica la variación.
- **Bueno para**: "explícame por qué las ventas bajaron en este segmento".
- **Tips**: útil en manos expertas, confuso para usuario novel.

### Smart narrative

- **Uso**: texto auto-generado que resume el visual.
- **Bueno para**: accesibilidad, informe ejecutivo narrado.
- **Tips**: revisar el texto. A veces dice obviedades o se equivoca.

### Q&A visual

- **Uso**: preguntas en lenguaje natural sobre los datos.
- **Bueno para**: self-service, exploración libre.
- **Tips**: el modelo debe tener sinónimos definidos en las columnas.

### Anomaly detection

- **Uso**: detección automática de outliers en series temporales.
- **Bueno para**: monitorización de KPIs que pueden fallar silenciosamente.
- **Tips**: requiere suficiente histórico (mínimo 2 ciclos completos).

---

## 11. Visuales de R / Python

### R visual / Python visual

- **Uso**: gráficos, modelos, análisis que Power BI nativo no soporta.
- **Bueno para**: clustering, statistical plots, modelos predictivos, geoespacial avanzado.
- **Tips**:
  - Limita el dataset que se pasa al script (filtros, agregaciones previas).
  - Documenta bien el script. Cualquiera debe entender qué hace.
  - El visual es lento porque hay un spin-up de R/Python. No abusar.

### Cuándo evitar R/Python

- ❌ Para gráficos que Power BI ya tiene (línea, bar, scatter).
- ❌ En páginas que se ven en mobile (render es lento).
- ❌ Cuando el usuario no puede mantener el script.
- ❌ Si los datos no caben en memoria del script.

---

## 12. Visuales personalizados (AppSource)

Hay cientos. Los más usados:

| Visual | Uso principal |
|---|---|
| **Charticulator** | Gráficos personalizados sin código (radial, sankey-like). |
| **Sankey** | Flujos entre categorías (e.g. origen → destino de clientes). |
| **Sunburst** | Jerarquía radial multi-nivel. |
| **Bullet chart** | Variante del gauge más densa en información. |
| **Box and whisker** | Distribución estadística (cuartiles, outliers). |
| **Histogram** | Distribución de frecuencias. |
| **Word cloud** | Frecuencia de términos. |
| **Chiclet slicer** | Slicer visual tipo botones en lugar de lista. |
| **HTML Content** (con cuidado) | Contenido HTML/CSS custom. ⚠️ Riesgo de seguridad. |
| **Mapbox / ArcGIS** | Mapas más avanzados que Bing. |

> **Advertencia**: cada visual custom es un proveedor externo. Revisa su mantenimiento, soporte, RGPD y si tiene coste. Para informes críticos, prefiere nativos.

---

## 13. Buenas prácticas de visualización

### Principios generales

1. **La pregunta es la que manda.** Si dudas entre dos visuales, el más simple.
2. **Menos es más.** Quita decoración que no codifique información.
3. **Ordena con propósito.** Categorías por valor, no alfabético. Tiempo cronológico, no inverso.
4. **El color comunica.** No decora. Una paleta discreta (5-7 colores) suele bastar.
5. **Ejes en 0 en barras.** En líneas, el ajuste es legítimo. En pie, no aplica.
6. **Títulos que dicen la conclusión.** "Las ventas caen un 12% en Q2" > "Ventas por mes".
7. **Tooltip con información adicional.** No repitas el visual. Añade contexto (vs LY, ranking, etc.).
8. **Accesibilidad**: contraste suficiente, no depender solo del color, tamaño de fuente legible.

### Anti-patrones

- ❌ **3D.** Distorsiona y no aporta.
- ❌ **Pie chart con > 5 categorías.** Ilegible.
- ❌ **Dual axis sin necesidad.** Confunde al usuario.
- ❌ **Ejes que no empiezan en 0** en gráficos de barra. Falsea la comparación.
- ❌ **Mismos datos en muchos visuales** sin justificación. Está gastando atención.
- ❌ **Color por defecto de Power BI** sin tonarlo a la marca. Parece informe amateur.
- ❌ **Visual sobrecargado de anotaciones.** Mejor una página limpia que 10 callouts.

### Checklist rápida antes de publicar una página

- [ ] ¿Se entiende la conclusión principal en 5 segundos?
- [ ] ¿Los slicers afectan a todo?
- [ ] ¿Los títulos dicen la conclusión, no la métrica?
- [ ] ¿La paleta de color es consistente con el resto del informe?
- [ ] ¿Los tooltips aportan información adicional, no duplican el visual?
- [ ] ¿El visual es responsive (se ve bien en pantallas pequeñas)?
- [ ] ¿El formato condicional está aplicado donde aporta (no por decorar)?
