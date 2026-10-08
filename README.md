Categoría: Casos de Uso Empresariales | Centro de Excelencia CX (Telecomunicaciones)

# Executive Operations & BI Analytics Dashboard (R&D-I)

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Data%20Analysis%20Expressions-orange?style=for-the-badge)
![Microsoft Lists](https://img.shields.io/badge/Microsoft%20Lists-Single%20Source%20of%20Truth-6A1B9A?style=for-the-badge&logo=sharepoint&logoColor=white)
![Power Query](https://img.shields.io/badge/Power%20Query-ETL%20%26%20Data%20Prep-2C8E4E?style=for-the-badge)
![Data Modeling](https://img.shields.io/badge/Modelado-In--Memory%20DAX%20Model-blue?style=for-the-badge)

> **Tablero de control analítico y seguimiento operativo para la gestión de solicitudes de Research, Diseño e Implementación (R&D-I). Modelo analítico construido en Power BI a partir de una lista única en Microsoft Lists / SharePoint Online, incorporando tabla de calendario autónoma y medidas DAX para la medición de SLAs, tiempos de resolución (MTTR) y demanda por segmento.**

[![Demo en Vivo](https://img.shields.io/badge/Demo%20Interactiva-GitHub%20Pages-amber?style=for-the-badge)](https://cassedev.github.io/ops-bi-analytics-dashboard/)
---

## 📌 Contexto y Arquitectura de Datos

La arquitectura técnica de este proyecto resuelve un escenario corporativo habitual: los datos operativos se centralizan en una única fuente plana (**Microsoft Lists**), sin servidores de bases de datos intermedias.

El rol de Power BI en este ecosistema consiste en:
1. **Conexión Nativa:** Ingesta directa desde la lista de SharePoint Online / Microsoft Lists.
2. **Transformación (Power Query):** Limpieza de tipos de datos, tipificación de fechas y descarte de metadatos internos de SharePoint.
3. **Modelado Analítico en DAX:** Creación autónoma de la tabla de inteligencia temporal (`Dim_Calendario`) y de una tabla dedicada para centralizar todas las métricas del negocio (`_Medidas`), evitando columnas calculadas innecesarias y optimizando el motor de almacenamiento de Power BI (VertiPaq).

---

## 📐 Modelado In-Memory y Medidas DAX Implementadas

Toda la inteligencia analítica del tablero se genera dentro del propio modelo mediante DAX:

### 1. Generación Autónoma de Dim_Calendario
Para evitar depender de fechas discontinuas de la lista y habilitar funciones de inteligencia temporal (como comparativas mensuales MoM), se genera una tabla de fechas pura en el modelo:

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

*Nota de buenas prácticas: Marcada como "Tabla de Fechas" (Mark as Date Table) y vinculada a la fecha de ingreso de la lista.*

---

### 2. Tabla Centralizada de Métricas (_Medidas)

* **Volumen Total:**
  Total Solicitudes = COUNTROWS('Fact_Solicitudes_Lists')

* **Cumplimiento Global de SLA (%):** Proporción de tickets cerrados dentro del plazo comprometido sobre el total de completados.
  SLA Compliance % = 
  DIVIDE(
      CALCULATE([Total Solicitudes], 'Fact_Solicitudes_Lists'[Cumple_SLA] = TRUE(), 'Fact_Solicitudes_Lists'[Estado] = "Completada"),
      CALCULATE([Total Solicitudes], 'Fact_Solicitudes_Lists'[Estado] = "Completada"),
      0
  )

* **Tiempo Medio de Resolución (MTTR en días):** Cálculo dinámico de días hábiles/naturales transcurridos entre la validación técnica y el cierre.
  MTTR (Dias) = 
  AVERAGEX(
      FILTER('Fact_Solicitudes_Lists', NOT(ISBLANK('Fact_Solicitudes_Lists'[Fecha_Cierre]))),
      DATEDIFF('Fact_Solicitudes_Lists'[Fecha_Aprobacion], 'Fact_Solicitudes_Lists'[Fecha_Cierre], DAY)
  )

* **Variación Mensual (MoM %):**
  Solicitudes MoM % = 
  VAR MesActual = [Total Solicitudes]
  VAR MesAnterior = CALCULATE([Total Solicitudes], DATEADD('Dim_Calendario'[Date], -1, MONTH))
  RETURN
  DIVIDE(MesActual - MesAnterior, MesAnterior, 0)

* **Tasa de Solicitudes Bloqueadas (%):** Monitoreo de desvíos y cuellos de botella por dependencias externas.
  Blocked Rate % = 
  DIVIDE(
      CALCULATE([Total Solicitudes], 'Fact_Solicitudes_Lists'[Estado] = "Bloqueada"),
      [Total Solicitudes],
      0
  )

* **Carga Activa (Work In Progress):** Tickets en cola de trabajo o en ejecución.
  Active Workload = 
  CALCULATE(
      [Total Solicitudes],
      'Fact_Solicitudes_Lists'[Estado] IN { "Backlog", "En Curso" }
  )

---

## 📊 Visualizaciones y Valor de Negocio

El tablero interactivo ofrece:
* **Filtros Dinámicos Cruzados:** Segmentación inmediata por cliente (**B2B** vs **B2C**) y por naturaleza del servicio (**Research**, **Diseño**, **Implementación**).
* **Control de Capacidad:** Comparativa entre solicitudes nuevas versus capacidad de resolución dentro de SLA.
* **Seguimiento del Embudo Operativo:** Visualización clara de la distribución de tickets en Backlog, En Curso, Bloqueadas y Completadas.
