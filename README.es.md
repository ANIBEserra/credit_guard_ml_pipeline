<p align="center">
  <b>Idiomas:</b>
  <a href="README.md">🇧🇷 Português</a> | 
  <a href="README.en.md">🇺🇸 English</a> | 
  🇪🇸 Español
</p>

---

Acceda a Looker Studio para ver el tablero de monitoreo: [Credit Guard Pipeline](https://datastudio.google.com/s/oUAEZgpbSvY)

# Pipeline Serverless de Ingesta y Gobernanza de Datos de Registro CNPJ (GCP & GitHub Actions)

## 🗺️ Resumen de Navegación

* [📌 1. Propósito del Proyecto y Visión de Negocio](#-1-propósito-del-proyecto-y-visión-de-negocio)
* [🏗️ 2. Arquitectura de Medallón Serverless](#️-2-arquitectura-de-medallón-serverless)
* [📁 3. Organización del Repositorio y Estructura de Carpetas](#-3-organización-del-repositorio-y-estructura-de-carpetas)
* [🛡️ 4. Gobernanza de Datos y Gestión de Metadatos](#️-4-gobernanza-de-datos-y-gestion-de-metadatos)
* [📊 5. Monitoreo del Pipeline e Insights Operacionales](#-5-monitoreo-del-pipeline-e-insights-operacionales)

---

## 📌 1. Propósito del Proyecto y Visión de Negocio

En los entornos corporativos modernos, la consolidación de datos de registro provenientes de fuentes externas públicas (como la Recaudación Federal) suele enfrentar desafíos crónicos: payloads JSON complejos y anidados, falta de estandarización, ausencia de diccionarios de datos claros y riesgos de exposición de datos sensibles de personas físicas (socios).

El objetivo de este proyecto fue construir un **pipeline de datos 100% serverless y de bajo costo (FinOps)** para centralizar, estructurar y gobernar los datos de CNPJ y del Cuadro de Socios y Administradores (QSA) consumidos a través de *BrasilAPI*.

El proyecto se enfoca en la excelencia de la **Ingeniería y Gobernanza de Datos**, demostrando:
* **Automación Eficiente:** Ingesta programada mediante CI/CD sin la necesidad de mantener servidores propios encendidos las 24 horas, los 7 días de la semana.
* **Arquitectura Escalable:** Almacenamiento optimizado utilizando el formato de columnas `.parquet` para reducir los costos de escaneo analítico.
* **Cultura de Gobernanza:** Aplicación práctica de catálogo de datos, diccionario de variables y técnicas de privacidad (*Privacy by Design*) directamente en la capa de entrega.

## 🏗️ 2. Arquitectura de Medallón Serverless

La arquitectura del proyecto fue diseñada siguiendo los principios de desacoplamiento de capas (Medallón) y computación en la nube administrada. El flujo garantiza el procesamiento eficiente de grandes volúmenes de datos de forma asíncrona, enfocándose en la optimización de costos de almacenamiento y consulta.

```text
  [ BrasilAPI ]
        │ (Solicitud Asíncrona / Python)
        ▼
 ┌────────────────────────────────────────┐
 │             GitHub Actions             │ ◄── [ Disparador Cron / CI/CD ]
 └────────────────────────────────────────┘
        │
        ├─► [ 1. Extracción JSON ] ──► Google Cloud Storage (Bucket Raw / Bronze)
        │
        └─► [ 2. Conversión y    ] ──► Google Cloud Storage (Bucket Structured / Silver)
              Procesamiento .parquet
        │
        ▼
 ┌────────────────────────────────────────┐
 │         Google BigQuery (OLAP)         │ ◄── [ Almacenamiento de Tablas Silver ]
 └────────────────────────────────────────┘
        │
        ▼
 ┌────────────────────────────────────────┐
 │           Looker Studio (BI)           │ ◄── [ Entrega y Enmascaramiento de Datos ]
 └────────────────────────────────────────┘
 ```
 ### 🧱 Detalle Técnico de las Capas

* **Capa Bronze (Raw/JSON):** Los datos devueltos por la API en formato JSON bruto se guardan directamente en **Google Cloud Storage (GCS)**. Esto preserva la inmutabilidad del dato original y permite reprocesamientos futuros (*replayability*) sin penalizar la API de origen.
* **Procesamiento y Optimización (.parquet):** Utilizando **Python**, el payload JSON se parsea, limpia y convierte al formato de columnas **.parquet**. Este paso reduce drásticamente el tamaño del archivo final en el Data Lake y optimiza la velocidad de lectura para las etapas analíticas.
* **Capa Silver (Structured/BigQuery):** Los archivos `.parquet` estructurados en GCS se integran en **Google BigQuery**. En esta capa analítica, los datos adquieren un tipado estricto, esquemas bien definidos y quedan organizados de forma relacional (Tabla de Empresas y Tabla de Cuadro de Socios).
* **Capa de Entrega (BI/Looker Studio):** BigQuery expone las tablas directamente a **Looker Studio**, donde las métricas operacionales y volumétricas se muestran de forma fluida, eliminando la necesidad de procesamientos complejos en tiempo de ejecución en el tablero.

---

## 📁 3. Organización del Repositorio y Estructura de Carpetas

El proyecto fue estructurado de forma práctica y organizada, separando los scripts de ejecución, las configuraciones de metadados y las documentaciones de soporte.

```text
├── configs/
│   ├── __init__.py
│   └── mapping.py           # Diccionarios de mapeo (De/Para) y renombre de columnas
├── data/
│   └── input/
│       └── cnpjs.csv        # Lista inicial de CNPJs que sirve de insumo para la búsqueda en la API
│   ├── raw/                 # [Dinámico] Creado en la ejecución para almacenar temporalmente los archivos JSON brutos
│   └── silver/              # [Dinámico] Creado en la ejecución para almacenar temporalmente los archivos .parquet
├── docs/                    # Documentaciones auxiliares y diccionarios de datos del proyecto
├── notebooks/
│   ├── analyse_silver.ipynb # Análisis y validación de los datos estructurados en la capa Silver
│   └── brasil-api.ipynb     # Jupyter Notebook utilizado para la exploración inicial de BrasilAPI
├── scripts/
│   └── main.py              # Script principal que contiene las funciones de extracción, limpieza y carga en GCP
├── .gitignore               # Definición de archivos y credenciales locales que no deben versionarse
├── README.md                # Documentación completa del ecosistema
└── requirements.txt        # Dependencias del proyecto (pandas, pyarrow, google-cloud-storage, etc.)
```

### ⚙️ Gestión del Flujo y Archivos Temporales

* **Insumo de Entrada (`data/input/cnpjs.csv`):** Archivo estático que contiene la relación de CNPJs que el pipeline debe procesar en cada ejecución.
* **Ciclo de Vida Local y Efimeridad:** Para garantizar la resiliencia y el aislamiento de las etapas, el script crea las carpetas `data/raw/` y `data/silver/` en tiempo de ejecución para apoyar el procesamiento local. Sin embargo, al final del pipeline, la función `clean_local_temp_files` purga estos directorios. Esto garantiza que el runner de GitHub Actions no acumule basura digital y cumpla con las mejores prácticas de seguridad de datos.
* **Aislamiento de Reglas (`configs/`):** Centraliza los mapeos y renombres de columnas. En caso de que la API de origen cambie el nombre de algún campo, el mantenimiento se realiza únicamente en este archivo, manteniendo intacto el script principal.
* **Exploratorio (`notebooks/`):** Espacio enfocado en el desarrollo incremental, pruebas de conexiones y prototipado de transformaciones analíticas antes de integrarlas al código de producción.

---

## 🛡️ 4. Gobernanza de Datos y Gestión de Metadatos

La gobernanza de este pipeline fue diseñada para garantizar la trazabilidad y el control sobre el ciclo de vida del dato.

### ⏳ 1. Ciclo de Vida de los Datos (Data Lifecycle Management)

El pipeline adopta una política de persistencia para garantizar el cumplimiento de la LGPD (Ley General de Protección de Datos de Brasil) y las buenas prácticas de gobernanza de datos:
* **Ingesta y Descarte Local:** Los archivos generados en las carpetas `data/raw/` y `data/silver/` existen solo durante la ejecución del script en el runner y se eliminan inmediatamente después de la carga en la nube.
* **Persistencia en Capas (GCP):** El dato bruto permanece inmutable en Cloud Storage (Bronze) para fines de auditoría, mientras que la capa analítica (Silver) en BigQuery se encuentra actualizada para el consumo, garantizando un historial limpio y auditable.

### 📋 2. Matriz de Cumplimiento y Clasificación de Datos

| Dominio de los Datos | Atributos Principales | Finalidad de Uso | Base Legal (LGPD) | Clasificación de Seguridad |
| :--- | :--- | :--- | :--- | :--- |
| **Registro PJ** | `NRCNPJ`, `NMRAZSOC`, `NMFANT`, `VLCPTSOC`, `IDMTZFIL`, `DSIDMTZFIL` | Centralización, higienización y enriquecimiento de registros de personas jurídicas para la mitigación de riesgos operacionales, prevención de fraudes (KYC) y validación de datos en ecosistemas de negocios. | **Interés Legítimo**<br>(Art. 7º, IX)<br><br>**Cumplimiento de Obligación Legal o Regulatoria**<br>(Art. 7º, II - Conformidad con PLD/Compliance) | **Público**<br>(Datos institucionales de registro público ostensible en la Recaudación Federal) |
| **QSA** *(Cuadro de Socios)* | Nombre del Socio, CPF (enmascarado/parcial), Calificación del Socio | Evaluación de vínculos societarios, identificación de beneficiarios finales y generación de tableros analíticos de monitoreo en Looker Studio. | **Interés Legítimo**<br>(Art. 7º, IX - Protección del propio negocio)<br><br>**Cumplimiento de Obligación Legal o Regulatoria**<br>(Art. 7º, II - Normas de Compliance y Gobernanza) | **Personal / PII**<br>(Datos de personas físicas. Requiere enmascaramiento dinámico en la capa de visualización y control de efimeridad con descarte local mediante `clean_local_temp_files`). |

### 📖 3. Catálogo de Datos y Linaje

La estructura de datos se mapea de forma declarativa y centralizada:
* **Centralización de Esquemas:** Las definiciones de tipos, claves y relaciones de BigQuery se controlan mediante código, garantizando que el volumen ingerido respete estrictamente el contrato de datos establecido.
* **Separación de Contextos:** División clara entre la entidad de datos de registro de la Empresa y la entidad del Cuadro de Socios y Administradores (QSA).

### 📋 4. Tabla de Logs y Monitoreo

Para garantizar la observabilidad del pipeline, el script alimenta una tabla dedicada de logs operacionales en BigQuery en cada ejecución. Esta tabla registra:
* **Metadados de Ingesta:** Fecha y hora del procesamiento, volumen de CNPJs consultados con éxito, cantidad de fallas en la API y el estado final de la carga (Éxito/Error). Esto permite rastrear la salud del flujo analítico sin necesidad de abrir consolas de infraestructura.

### 🔤 5. Estandarización de Nomenclatura (Patrón DATASUS)

El diccionario de columnas en el archivo `configs/mapping.py` adopta la **metodología de compresión de vocales y prefijos taxonómicos inspirada en el DATASUS**:

El de/para mapea explícitamente los tipos y aliases cortos:
* `cnpj` ➔ `NRCNPJ` (Número CNPJ)
* `razao_social` ➔ `NMRAZSOC` (Nombre Razón Social)
* `nome_fantasia` ➔ `NMFANT` (Nombre Fantasía)
* `capital_social` ➔ `VLCPTSOC` (Valor Capital Social)
* `identificador_matriz_filial` ➔ `IDMTZFIL` (Identificador Matriz Filial)
* `descricao_identificador_matriz_filial` ➔ `DSIDMTZFIL` (Descripción Identificador Matriz Filial)

Este enfoque reduce drásticamente el ruido de los metadados, ahorra bytes de almacenamiento de columnas, estandariza las claves lógicas del Data Warehouse y asegura que el modelado relacional sea limpio y predecible para cualquier analista de la organización.

---

## 📊 5. Monitoreo del Pipeline e Insights Operacionales

La etapa final del ecosistema consiste en exponer los datos procesados y estructurados de la capa Silver (BigQuery) hacia **Looker Studio**, donde el flujo de ingesta y la volumetría empresarial se monitorean visualmente.

### 📈 Métricas de Monitoreo y Observabilidad

El tablero técnico consume directamente la tabla de logs generada por el pipeline, permitiendo el seguimiento de indicadores vitales de DataOps:
* **Tasa de Éxito de la Ingesta:** Porcentaje de solicitudes enviadas a BrasilAPI que respondieron con éxito (*Status 200*) vs. fallas de registro (CNPJs inválidos o inexistentes).
* **Volumetría Diaria:** Monitoreo del volumen de filas inyectadas por ejecución para garantizar que no haya caídas drásticas o anomalías en el flujo de procesamiento.
* **Métricas de FinOps:** Trazabilidad del tiempo de ejecución y volumen de datos traficados, ayudando a garantizar el funcionamiento del pipeline dentro del límite de gratuidad (*Free Tier*) de GCP.

### 💡 Extracción de Insights de Registro Básicos

Además del monitoreo del pipeline, el tablero consolida visiones generales sobre el portafolio de empresas procesadas para proporcionar inteligencia de negocios:
* **Distribución Geográfica:** Concentración de las empresas activas en el territorio nacional por Estado (UF) y municipio (utilizando el campo `cd_municipio` mapeado).
* **Análisis de Tamaño y Capital:** Agrupamiento del volumen financiero de las organizaciones con base en la distribución del campo `VLCPTSOC` (Capital Social).
* **Visión de Cuadro Societario:** Mapeamiento cuantitativo del número de socios por organización, permitiendo entender la densidad de vínculos societarios en el ecosistema consultado.