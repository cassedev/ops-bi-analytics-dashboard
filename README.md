# Executive Operations & BI Analytics Dashboard (R&D-I)

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Data%20Analysis%20Expressions-orange?style=for-the-badge)
![SQL](https://img.shields.io/badge/SQL-Relational%20Queries-00758F?style=for-the-badge&logo=mysql&logoColor=white)
![Microsoft Lists](https://img.shields.io/badge/Microsoft%20Lists-Central%20Data%20Source-6A1B9A?style=for-the-badge&logo=sharepoint&logoColor=white)
![Star Schema](https://img.shields.io/badge/Architecture-Dimensional%20Star%20Schema-blue?style=for-the-badge)

> **Tablero de control analítico y seguimiento operativo para la gestión de solicitudes de Research, Diseño e Implementación (R&D-I). Modelo dimensional optimizado para evaluar cumplimiento de acuerdos de nivel de servicio (SLA), distribución de carga por segmento corporativo (B2B vs B2C) y tiempos medios de resolución (MTTR).**

---

## 📌 Propósito y Caso de Negocio

Este proyecto constituye la capa de visualización e inteligencia de negocios del ecosistema automatizado de ingesta de requerimientos. Transforma los registros transaccionales recopilados y sincronizados de forma continua en **Microsoft Lists** en un modelo analítico estructurado que permite:

* **Monitoreo de SLAs:** Detección de desviaciones entre tiempos comprometidos y plazos reales de entrega.
* **Segmentación de Demanda:** Comparación del volumen y naturaleza de requerimientos entre clientes corporativos (**B2B**) y masivos (**B2C**).
* **Identificación de Cuellos de Botella:** Medición del porcentaje de solicitudes en estado `Bloqueada` por dependencias externas y cálculo del volumen neto de trabajo en curso (*Work In Progress*).

---

## 🏗️ Arquitectura de Datos y Modelado Dimensional

El modelo se diseñó bajo un esquema en estrella (**Star Schema**) para garantizar un alto rendimiento en el cálculo de métricas tabulares y evitar dependencias de tablas desnormalizadas:

* **Tabla de Hechos (`Fact_Solicitudes_Lists`):**
  * Contiene los registros individuales sincronizados desde Microsoft Lists.
  * Claves foráneas: `ID_Fecha_Ingreso`, `ID_Servicio`, `ID_Segmento`, `ID_Estado`.
  * Métricas base: Tiempos de resolución (días), flags de cumplimiento de SLA (`TRUE/FALSE`).

* **Tablas de Dimensiones:**
  * `Dim_Calendario`: Creada de forma autónoma mediante funciones DAX para inteligencia temporal.
  * `Dim_Servicio`: Clasificación (`Research`, `Diseño`, `Implementación`).
  * `Dim_Segmento`: Tipo de cliente (`B2B`, `B2C`).
  * `Dim_Estado`: Ciclo operativo (`Backlog`, `En Curso`, `Bloqueada`, `Completada`).

---

## 📐 Métricas y Modelado Analítico Implementado

### 1. Inteligencia Temporal y Dim_Calendario (DAX)
Para habilitar comparaciones mensuales consistentes (MoM) y análisis cronológico sin huecos temporales, se implementó una tabla de calendario analítica:

Dim_Calendario = 
VAR MinDate = MIN('Fact_Solicitudes_Lists'[Fecha_Ingreso])
VAR MaxDate = TODAY()
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Año", YEAR([Date]),
    "Mes_Nro", MONTH([Date]),
    "Mes_Nombre", FORMAT([Date], "MMMM"),
    "Año_Mes", FORMAT([Date], "YYYY-MM"),
    "Trimestre", "T" & FORMAT([Date], "Q")
)

### 2. Medidas DAX Principales

* **Cumplimiento de SLA (%):** Proporción de requerimientos cerrados dentro de la fecha límite estipulada sobre el total de completados.
  SLA Compliance % = 
  DIVIDE(
      CALCULATE([Total Solicitudes], 'Fact_Solicitudes_Lists'[Cumple_SLA] = TRUE(), 'Fact_Solicitudes_Lists'[Estado] = "Completada"),
      CALCULATE([Total Solicitudes], 'Fact_Solicitudes_Lists'[Estado] = "Completada"),
      0
  )

* **Tiempo Medio de Resolución (MTTR en días):**
  MTTR (Dias) = 
  AVERAGEX(
      FILTER('Fact_Solicitudes_Lists', NOT(ISBLANK('Fact_Solicitudes_Lists'[Fecha_Cierre]))),
      DATEDIFF('Fact_Solicitudes_Lists'[Fecha_Aprobacion], 'Fact_Solicitudes_Lists'[Fecha_Cierre], DAY)
  )

* **Variación Mensual de Solicitudes (MoM %):**
  Solicitudes MoM % = 
  VAR MesActual = [Total Solicitudes]
  VAR MesAnterior = CALCULATE([Total Solicitudes], DATEADD('Dim_Calendario'[Date], -1, MONTH))
  RETURN
  DIVIDE(MesActual - MesAnterior, MesAnterior, 0)

* **Tasa de Bloqueo Operativo (%):**
  Blocked Rate % = 
  DIVIDE(
      CALCULATE([Total Solicitudes], 'Fact_Solicitudes_Lists'[Estado] = "Bloqueada"),
      [Total Solicitudes],
      0
  )

* **Carga Activa de Trabajo (Work In Progress):**
  Active Workload = 
  CALCULATE(
      [Total Solicitudes],
      'Fact_Solicitudes_Lists'[Estado] IN { "Backlog", "En Curso" }
  )

---

## 💻 Consulta de Consolidación Relacional (SQL)

Para entornos donde el modelo tabular consume datos desde una réplica relacional o base SQL centralizada, se desarrolló la consulta de agregación equivalente:

SELECT 
    s.servicio,
    s.segmento,
    COUNT(s.ticket_id) AS total_solicitudes,
    ROUND(AVG(DATEDIFF(day, s.fecha_aprobacion, s.fecha_cierre)), 1) AS mttr_dias,
    ROUND(100.0 * SUM(CASE WHEN s.cumple_sla = 1 THEN 1 ELSE 0 END) / NULLIF(COUNT(s.ticket_id), 0), 2) AS pct_sla_compliance
FROM fact_solicitudes_lists s
WHERE s.estado = 'Completada'
GROUP BY s.servicio, s.segmento
ORDER BY total_solicitudes DESC;

---

## 📊 Visualizaciones y Métricas del Tablero

1. **Tarjetas de KPIs Superiores:** Monitoreo instantáneo de volumen total, % de SLA, MTTR en días, tasa de tickets bloqueados y solicitudes activas en simultáneo.
2. **Segmentadores Interactivos:** Filtros combinados por segmento de negocio (B2B, B2C) y tipo de servicio (Research, Diseño, Implementación).
3. **Tendencia Temporal:** Comparativa gráfica mensual entre ingresos de nuevos tickets y resoluciones exitosas bajo SLA.
4. **Distribución de Demanda:** Gráfico de partición que ilustra el peso relativo de solicitudes corporativas frente a masivas.
5. **Carga Operativa por Servicio:** Gráfico de barras apiladas que desglosa el estado de avance específico de cada disciplina.