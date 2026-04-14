# 🏭 Microsoft Fabric IQ - Lab: Manufactura & Supply Chain

Escenario end-to-end de **Microsoft Fabric IQ (Preview)** aplicado a un escenario de manufactura y cadena de suministro mexicana. Cubre la creación de una Ontología a partir de un Semantic Model, la incorporación de telemetría de máquinas via Eventhouse, y la configuración de un Operations Agent para monitoreo en tiempo real.

> ⚠️ **Nota:** Todos los componentes de Fabric IQ están en **Public Preview**. La interfaz y comportamiento puede cambiar. Guía elaborada con base en la documentación oficial de Microsoft Learn (marzo 2026).

---

## 📋 Contenido del repo
```
fabric-iq-manufacturing/
├── data/
│   ├── DimSupplier.csv          # Catálogo de proveedores
│   ├── DimProduct.csv           # Catálogo de materias primas y componentes
│   ├── DimPlant.csv             # Plantas y centros de distribución
│   ├── DimMachine.csv           # Catálogo de maquinaria
│   ├── FactPurchaseOrder.csv    # Órdenes de compra (tabla de hechos)
│   └── MachineTelemetry.csv     # Telemetría de máquinas → Eventhouse
└── README.md                    # Esta guía
```

---

## 🧠 ¿Qué es Fabric IQ y por qué importa?

En una empresa manufacturera, el equipo de Compras define "proveedor crítico" de una forma, Operaciones de otra, y los agentes de IA ven solo tablas con columnas como `SUP_ID` o `fk_plant` — sin contexto de negocio. El resultado: reportes que no cuadran, agentes que no se pueden confiar, y analistas respondiendo las mismas preguntas una y otra vez.

**Microsoft Fabric IQ** resuelve esto con una **Ontología** — una capa semántica viva que define el vocabulario del negocio una sola vez y lo comparte con todas las herramientas: Power BI, agentes de IA, y aplicaciones. En lugar de que cada equipo tenga su propia definición de "Proveedor" o "Máquina", la Ontología lo define una vez y todos hablan el mismo idioma.
```
SIN FABRIC IQ                          CON FABRIC IQ
─────────────────────────────────      ─────────────────────────────────────
                                       
  Compras      Operaciones  TI               Ontología (vocabulario único)
     │               │       │          ┌──────────────────────────────────────────┐
     ▼               ▼       ▼          │  Producto → suministra → Proveedor       │
  tabla_prov    tbl_ops   dim_sup       │  Planta → opera → Maquina                │
                                        │  OrdenDeCompra ──cumple──────► Proveedor |
                                        |  OrdenDeCompra ──incluida_en─► Producto  │
                                        |  OrdenDeCompra ──recibe──────► Planta    │
  Agente IA:                            └──────────────────────────────────────────┘
  "¿Cuál es SUP_ID?"                            │
  "¿Qué es fk_plant?"                           ▼
  "No entiendo los joins"            ┌───────────────────────┐
                                     │  Data Agent           │
                                     │  Operations Agent     │
                                     │  Power BI             │
                                     │  → todos hablan el    │
                                     │    mismo idioma       │
                                     └───────────────────────┘
```

> 💡 La Ontología no reemplaza tu Lakehouse ni tu Semantic Model — se construye **encima** de ellos, añadiendo significado de negocio sin mover datos.

### ¿Por qué una Ontología y no solo un Lakehouse o Semantic Model?

Un Data Agent conectado a un Lakehouse o Semantic Model puede responder preguntas de aggregation y reporting, totales, promedios, rankings. Lo hace traduciendo lenguaje natural a SQL o DAX contra tablas con nombres técnicos como `factpurchaseorder` o `fk_plant_id`.

La Ontología resuelve un problema diferente: **dar contexto semántico a la IA**. En lugar de ver tablas y columnas, el agente ve entidades de negocio (Proveedor, Planta, Maquina) y relaciones con significado (opera, suministra, cumple). Esto le permite razonar sobre conexiones entre dominios que en el Lakehouse no existen explícitamente:

- ¿Qué proveedores abastecen plantas que tienen máquinas en riesgo?
- ¿Qué plantas con órdenes atrasadas también tienen baja eficiencia operativa?

Estas preguntas requieren traversal semántico, recorrer el grafo de negocio siguiendo relaciones con significado. Eso es lo que la Ontología habilita.

## 🎯 Escenario de negocio

**FabricaMX** es una empresa manufacturera con 3 plantas de producción y 3 centros de distribución en México. El reto: los equipos de Compras, Operaciones y Mantenimiento trabajan con definiciones distintas de "proveedor crítico", "orden atrasada" y "máquina en riesgo". Los agentes de IA no pueden responder preguntas cross-funcionales porque ven tablas sin contexto de negocio.

**La solución con Fabric IQ:**
- Una **Ontología** unifica el vocabulario: Proveedor, Producto, Planta, Máquina, OrdenDeCompra
- El **Eventhouse** captura la telemetría en tiempo real de las máquinas
- Un **Operations Agent** monitorea temperatura y vibración y recomienda acciones
- Un **Data Agent** permite hacer preguntas en lenguaje natural sobre toda la cadena de suministro

### Modelo de entidades y relaciones
```
Producto      ──suministra──► Proveedor
Planta        ──opera──────► Maquina
OrdenDeCompra ──cumple──────► Proveedor
OrdenDeCompra ──incluida_en─► Producto
OrdenDeCompra ──recibe──────► Planta
```

---

## ✅ Prerrequisitos

### 1. Workspace de Fabric
- Workspace con capacidad Fabric habilitada (F2 o superior)
- **No usar** el workspace "My Workspace" (no soportado para Ontología)

### 2. Tenant settings — Habilitar en Admin Portal

Un administrador de Fabric debe habilitar los siguientes toggles en:
**Configuración → Portal de administración → Configuración del inquilino**

| Setting | Sección |
|---|---|
| Users can create Ontology items (preview) | IQ |
| User can create Graph (preview) | IQ |
| Users can create and share Data agent item types (preview) | Data Science |
| Users can use Copilot and other features powered by Azure OpenAI | Copilot |
| Data sent to Azure OpenAI can be processed outside your capacity's geographic region | Copilot |
| Data sent to Azure OpenAI can be stored outside your capacity's geographic region | Copilot |
| Operations agent (preview) | Real-Time Intelligence |

> 💡 Sin estos settings habilitados, los items de Ontología, Data Agent y Operations Agent no aparecerán en el workspace.

### 3. Teams
- Cuenta de Microsoft Teams (requerida para recibir notificaciones del Operations Agent)
- Instalar la app **Fabric Operations Agent** desde el Teams App Store

### 4. Power Automate
- Acceso a **Power Automate** con permisos para crear flujos
- El Operations Agent requiere al menos una acción conectada a un flujo de Power Automate via **Fabric Activator** para poder activarse
- El flujo debe tener acceso a Microsoft Teams para publicar mensajes en canales o chats

---

## 🚀 Paso a paso

### FASE 1 - Preparar los datos en el Lakehouse

**Objetivo:** Cargar los datos de referencia (suppliers, products, plants, machines, orders) en tablas Delta del Lakehouse.

1. En tu workspace, selecciona **+ New item → Lakehouse**
2. Nómbralo `ManufacturingLH`
   > ⚠️ Asegúrate de que la opción **"Lakehouse schemas (Public Preview)"** esté **deshabilitada** al crear el Lakehouse. La Ontología no es compatible con Lakehouse schemas.

3. Desde el ribbon del Lakehouse: **Get data → Upload files**
4. Sube los siguientes archivos de la carpeta `data/`:
   - `DimSupplier.csv`
   - `DimProduct.csv`
   - `DimPlant.csv`
   - `DimMachine.csv`
   - `FactPurchaseOrder.csv`

5. Para cada archivo, en el panel **Files**, haz clic en `...` → **Load to Tables → New table**. Mantén los nombres por defecto.

Al terminar debes tener estas tablas Delta en el Lakehouse:

| Tabla | Descripción |
|---|---|
| `dimsupplier` | Proveedores |
| `dimproduct` | Productos / materias primas |
| `dimplant` | Plantas y CEDIS |
| `dimmachine` | Maquinaria |
| `factpurchaseorder` | Órdenes de compra |

> ❌ **NO subas** `MachineTelemetry.csv` al Lakehouse. Este archivo va al Eventhouse en la Fase 2.

---

### FASE 2 - Preparar el Eventhouse (telemetría de máquinas)

**Objetivo:** Cargar la telemetría simulada de máquinas en una KQL Database, que servirá como fuente de datos en tiempo real para la Ontología y el Operations Agent.

1. En tu workspace: **+ New item → Eventhouse**
2. Nómbralo `ManufacturingEH`
   - Se crea automáticamente una KQL Database con el mismo nombre

3. Abre la KQL Database haciendo clic en su nombre en el Explorer

4. Desde el ribbon: **Get data → Local file**
5. Crea una **New table** llamada `MachineTelemetry`
6. Sube el archivo `data/MachineTelemetry.csv`
7. Confirma el schema (mantén defaults) y selecciona **Finish**

Al terminar verás la tabla `MachineTelemetry` con datos de temperatura, vibración, presión y OEE para 4 máquinas.

**Verifica** con esta query KQL desde el Eventhouse:
```kql
MachineTelemetry
| summarize count() by MachineId, Status
| order by MachineId asc
```

Debes ver registros con estados `Normal`, `Alerta` y `Critico`.

---

### FASE 3 - Crear el Semantic Model

**Objetivo:** Crear un Semantic Model Direct Lake desde el Lakehouse. Este modelo es el punto de partida para generar la Ontología automáticamente.

1. Desde el ribbon del Lakehouse `ManufacturingLH`: **New semantic model**
2. Configura:
   - **Name:** `ManufacturingSM`
   - **Tables:** Selecciona estas 5 tablas:
     - `dimsupplier`
     - `dimproduct`
     - `dimplant`
     - `dimmachine`
     - `factpurchaseorder`

3. Selecciona **Confirm**

4. Abre el Semantic Model en modo **Editing** y ve a **Manage relationships**

5. Crea las siguientes relaciones con **+ New relationship**:

| From table | From column | To table | To column | Cardinality | Active |
|---|---|---|---|---|---|
| `factpurchaseorder` | `SupplierId` | `dimsupplier` | `SupplierId` | Many to one (*:1) | ✅ |
| `factpurchaseorder` | `ProductId` | `dimproduct` | `ProductId` | Many to one (*:1) | ✅ |
| `factpurchaseorder` | `PlantId` | `dimplant` | `PlantId` | Many to one (*:1) | ✅ |
| `dimmachine` | `PlantId` | `dimplant` | `PlantId` | Many to one (*:1) | ✅ |

> ⚠️ **No crear** la relación `dimproduct → dimsupplier` en el Semantic Model — genera paths ambiguos con `factpurchaseorder → dimsupplier`. Esta relación se agrega directamente en la Ontología (Fase 4).

6. Selecciona **Close**

> 💡 El Semantic Model ahora refleja exactamente el modelo relacional del negocio. La Ontología tomará estas relaciones como punto de partida.

---

### FASE 4 - Generar la Ontología

**Objetivo:** Crear la Ontología de manufactura a partir del Semantic Model.

#### 4.1 Generar desde el Semantic Model

1. Desde el Semantic Model `ManufacturingSM`, en el ribbon superior: **Generate Ontology**
2. Configura:
   - **Workspace:** Tu workspace actual
   - **Name:** `ManufacturingOntology`
   > ⚠️ Solo letras, números y guiones bajos. Sin espacios ni guiones.
3. Selecciona **Create**

La Ontología abre cuando está lista. Debes ver las 5 entity types en el panel izquierdo.

#### 4.2 Renombrar entity types

Renombra cada entity type a un nombre de negocio más claro:

| Nombre original | Nombre nuevo |
|---|---|
| `factpurchaseorder` | `OrdenDeCompra` |
| `dimsupplier` | `Proveedor` |
| `dimproduct` | `Producto` |
| `dimplant` | `Planta` |
| `dimmachine` | `Maquina` |

**Para renombrar:** Selecciona el entity type → en el panel de configuración, haz clic en el ícono de edición junto al nombre.

#### 4.3 Agregar entity type keys faltantes

Verifica que cada entity type tenga su key asignado. Si falta alguno, agrégalo:

| Entity type | Key |
|---|---|
| `Proveedor` | `SupplierId` |
| `Producto` | `ProductId` |
| `Planta` | `PlantId` |
| `Maquina` | `MachineId` |
| `OrdenDeCompra` | `OrderId` |

> ⚠️ Configura los keys **antes** de crear o configurar cualquier relationship type. Si una relación ya fue creada con un key incorrecto, debes eliminarla, corregir el key y volver a crearla.

**Para agregar key:** Selecciona el entity type → panel derecho → ícono de edición junto a **Key** → selecciona la columna → **Save**. Si aparece más de una columna en la lista del key compuesto, elimina las que no correspondan dejando solo la indicada en la tabla.

#### 4.4 Renombrar y configurar relationship types

Después de generar desde el Semantic Model, verás relationship types con nombres automáticos. Renómbralos y configúralos:

El wizard de configuración de cada relación pide:
- **Source data table:** la tabla que contiene las FKs de ambas entidades
- **Source entity type** + **Source column:** la columna en la source data table que mapea al key de esa entidad
- **Target entity type** + **Source column:** la columna en la source data table que mapea al key de la entidad destino

| Nombre automático | Nombre nuevo | Source entity type | Source column | Entity type key | Target entity type | Source column | Entity type key | Source data table |
|---|---|---|---|---|---|---|---|---|
| `factpurchaseorder_has_dimsupplier` | `cumple` | `OrdenDeCompra` | `OrderId` | `OrderId` | `Proveedor` | `SupplierId` | `SupplierId` | `factpurchaseorder` |
| `factpurchaseorder_has_dimproduct` | `incluida_en` | `OrdenDeCompra` | `OrderId` | `OrderId` | `Producto` | `ProductId` | `ProductId` | `factpurchaseorder` |
| `factpurchaseorder_has_dimplant` | `recibe` | `OrdenDeCompra` | `OrderId` | `OrderId` | `Planta` | `PlantId` | `PlantId` | `factpurchaseorder` |
| `dimmachine_has_dimplant` | `opera` | `Maquina` | `MachineId` | `MachineId` | `Planta` | `PlantId` | `PlantId` | `dimmachine` |

> ⚠️ La relación `opera` se genera con nombre automático `dimmachine_has_dimplant` pero su **source data table debe cambiarse a `dimmachine`** — es la única tabla que contiene tanto `PlantId` como `MachineId`. Si se deja `dimplant` como source table, el graph no podrá popular los edges de esta relación.

> ⚠️ La relación `suministra` (Producto → Proveedor) no se genera automáticamente porque fue excluida del Semantic Model para evitar paths ambiguos. Créala manualmente con **Add relationship** en la Ontología:
> - **Nombre:** `suministra`
> - **Source entity type:** `Producto` — Source column: `ProductId` — Entity type key: `ProductId`
> - **Target entity type:** `Proveedor` — Source column: `SupplierId` — Entity type key: `SupplierId`
> - **Source data table:** `dimproduct`

#### 4.5 Agregar la entidad Maquina con binding de telemetría (Eventhouse)

La telemetría de máquinas está en el Eventhouse, no en el Lakehouse. Hay que agregar un **time series binding** a la entity type `Maquina`:

1. Selecciona la entity type `Maquina`
2. Ve al tab **Bindings → Add data to entity type**
3. Selecciona `ManufacturingEH` (KQL Database) como fuente → **Connect**
4. Selecciona la tabla `MachineTelemetry`
5. En **Binding type** selecciona **Timeseries (time series)**
6. Configura:
   - **Source data timestamp column:** `Timestamp`
   - En la sección **Static**, mapea:
     - `MachineId` → `MachineId` (key de identidad)
     - `PlantId` → `PlantId`
   - En la sección **Timeseries**, mapea únicamente estas columnas (borra `Timestamp` si aparece aquí — ya está declarado arriba):
     - `Temperature`, `Vibration`, `Pressure`, `OEE`, `Status`
7. Selecciona **Save**

> 💡 Este binding conecta cada instancia de máquina (definida en el Lakehouse) con su serie de tiempo de telemetría (en el Eventhouse).

#### 4.6 Refrescar el Graph Model

Después de configurar todos los bindings y relaciones, el Graph Model necesita ser refrescado para que las queries devuelvan instancias. El refresh **no se hace desde dentro de la Ontología** — se hace desde el workspace:

1. Ve a tu workspace
2. Localiza el item de tipo **Graph model** con nombre `ManufacturingOntology_graph_...`
3. Selecciona `...` → **Settings → Schedule**
4. Selecciona el botón **Refresh**
5. Espera a que termine (puedes verificar en **Last successful update**)

> 💡 La Ontología crea automáticamente un Graph model en el workspace al configurar los bindings. Este item es el que almacena las instancias y edges del grafo. Sin refrescarlo, las queries en el Relationship graph devolverán "Query result contains no data" aunque la configuración esté correcta.

---

### FASE 5 - Explorar la Ontología (Graph Preview)

**Objetivo:** Verificar que los datos están correctamente vinculados y explorar el grafo de negocio.

1. Desde la Ontología, selecciona cualquier entity type y ve al ribbon: **Entity type overview**
2. Verifica que los **Entity instances** muestren datos (conteos de registros)
3. Para la entity type `Maquina`, cambia el rango de tiempo:
   - **Custom range:** 2026-03-23 00:00 → 2026-03-27 00:00
   - **Time granularity:** 1 minute

> 💡 **Volumen mínimo de datos para charts de timeseries:** los charts de Temperature, Vibration, Pressure y OEE en Entity type overview requieren suficiente densidad de datos históricos para renderizar. Cargar al menos varios meses de telemetría — los archivos de este repo cubren agosto 2025 a marzo 2026.
   

4. Para explorar el Relationship graph con instancias:
   - Selecciona una entity type → **Entity type overview** → en el tile **Relationship graph** selecciona **Expand**
   - En **Components** selecciona los **Nodes** y **Edges** que quieres visualizar
   - Selecciona **Run query** para cargar las instancias 

> 💡 Los filtros se aplican sobre la entidad que tiene la propiedad como campo directo. Por ejemplo, para filtrar máquinas por planta, usa **Add filter → Maquina → PlantId = PLT-001** (no sobre `Planta`), ya que `PlantId` es una propiedad del binding de `dimmachine`.
   

   **Query 1:** Máquinas de la Planta Monterrey
   - En **Components** activa: Nodes → `Maquina`, Edges → `opera`
   - **Add filter → Maquina → PlantId = PLT-001**
   - Selecciona **Run query**

   **Query 2:** Órdenes actualmente En Transito
   - Selecciona entity type `OrdenDeCompra` → **Entity type overview** → **Expand**
   - En **Components** activa: Nodes → `OrdenDeCompra`
   - **Add filter → OrdenDeCompra → Status = En Transito**
   - Selecciona **Run query**   
     
> 💡 Si en **Diagram view** los nodos aparecen como "NULL", cambia a **Card view** usando el selector en el ribbon — los datos están correctos, es solo un comportamiento visual del preview cuando el instance display name resuelve a un campo nulo.


---

### FASE 6 - Crear el Data Agent

**Objetivo:** Habilitar consultas en lenguaje natural sobre toda la Ontología.

1. En tu workspace: **+ New item → Data agent (preview)**
2. Nómbralo `ManufacturingAgent`
3. Selecciona **Add a data source**
4. Busca `ManufacturingOntology` y selecciona **Add**
5. El agente abre cuando está listo. Verás las entity types en el Explorer.

**Configura las instrucciones del agente** (campo "Instructions"):
```
## Objetivo
Eres un asistente de inteligencia de manufactura para FabricaMX.
Responde siempre en español. Usa los nombres de las entidades de negocio 
(Proveedor, Planta, Maquina, OrdenDeCompra, Producto) al referirte a los datos.

## Valores exactos de campos categóricos — úsalos siempre al filtrar
- OrdenDeCompra.Status: 'Entregado', 'En Transito', 'Pendiente'
- Planta.PlantType: 'Manufactura', 'Almacen'
- Maquina.MachineType: 'Prensa', 'Torno', 'Soldadora', 'Inyectora' — tipo de equipo, NO estado operativo

## Direcciones de relaciones en GQL — úsalas exactamente así
- MATCH (o:`OrdenDeCompra`)-[:`recibe`]->(p:`Planta`)
- MATCH (o:`OrdenDeCompra`)-[:`cumple`]->(s:`Proveedor`)
- MATCH (o:`OrdenDeCompra`)-[:`incluida_en`]->(prod:`Producto`)
- MATCH (p:`Planta`)-[:`opera`]->(m:`Maquina`)
- MATCH (prod:`Producto`)-[:`suministra`]->(s:`Proveedor`)

Support group by in GQL

```

> 💡 Las direcciones de las relaciones en GQL son críticas — el LLM puede inferirlas incorrectamente basándose en semántica del lenguaje natural. Por eso las instructions incluyen los patrones `MATCH` explícitos para cada relación. `MATCH` es la sentencia principal de **GQL (Graph Query Language)**, el estándar ISO para consultar bases de datos de grafos — equivalente a `SELECT` en SQL. Cada patrón define la dirección exacta de traversal entre entidades. Para más contexto: [GQL Language Guide - Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/graph/gql-language-guide)

**Prueba estas preguntas de ejemplo:**

| Pregunta | Qué demuestra |
|---|---|
| "¿Qué productos suministra Acero del Norte SA?" | Traversal: Producto → suministra → Proveedor |
| "¿Qué plantas recibieron órdenes pendientes de entrega?" | Traversal: OrdenDeCompra → recibe → Planta + filtro por Status |

> 💡 Si ves errores de "no data" en las primeras consultas, espera 2-3 minutos para que el agente se inicialice y vuelve a intentar.

6. Cuando estés satisfecho con las respuestas: **Publish** → **Publish**

---

### FASE 7 - Configurar el Operations Agent

**Objetivo:** Crear un agente que monitoree en tiempo real la telemetría de las máquinas y notifique cuando detecte condiciones de riesgo.

> ⚠️ **Prerrequisitos:**
> - Tener la app **Fabric Operations Agent** instalada en Microsoft Teams (búscala en el Teams App Store)
> - Tener acceso a **Power Automate** — se requiere para conectar al menos una acción antes de poder activar el agente

1. En tu workspace: **+ New item → Real-Time Intelligence → Operations agent**
2. Nómbralo `MachineMonitorAgent`

#### 7.1 Configurar el agente

> ⚠️ **Importante:** El Operations Agent actualmente solo soporta instrucciones y goals en **inglés**. Según la [documentación oficial de Microsoft](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/operations-agent-limitations), el LLM que genera el playbook no procesa correctamente instrucciones en otros idiomas. Asegúrate de escribir los campos **Business goals** y **Agent instructions** en inglés.

**Business goals:**
```
Ensure manufacturing machines in the "MachineTelemetry" table operate within safe temperature and vibration thresholds to prevent unplanned downtime.
```

**Agent instructions:**
```
Monitor the "MachineTelemetry" table in the KQL database.
Each machine is identified by "MachineId" and located in a plant identified by "PlantId".

Rules:
- When "Temperature" > 80, alert the operator for inspection
- When "Status" = 'Critico', recommend preventive shutdown
- When "Vibration" > 3.0, recommend reducing machine workload

Always include "MachineId", "PlantId", "Temperature", "Vibration", and "Status" in every recommendation.
```

**Knowledge source:**
- Selecciona **+ Add data** → selecciona `ManufacturingEH` (KQL Database)

> 💡 El agente selecciona el Eventhouse completo como fuente — no se selecciona una tabla específica. El agente determina qué tablas consultar basándose en las instructions.

#### 7.2 Configurar las acciones y conectarlas a Power Automate

> ⚠️ El agente requiere **al menos una acción conectada** a Power Automate para poder activarse. Sin esta conexión, el botón **Start** dará error de estado.

**Paso 1 — Agregar la acción `notify_operator` y conectarla**

Selecciona **+ Add action** y configura:

- **Action name:** `notify_operator`
- **Action description:** `Send alert to plant operator when a machine enters alert state`
- **Parameters:**

| Name | Description |
|---|---|
| `machine_id` | Unique identifier of the machine in alert state |
| `plant_id` | Identifier of the plant where the machine is located |
| `temperature` | Current temperature value recorded on the machine |
| `vibration` | Current vibration value recorded on the machine |
| `status` | Current machine status: Alerta or Critico |

Luego selecciona **Connect** junto a la acción recién creada. En el wizard **Configure custom action**:
1. **Step 1:** Selecciona tu workspace, ejemplo: `WS_Fabric_IQ` — en el dropdown de **Activator** escribe un nombre como `ManufacturingActivator` y créalo desde ahí mismo
2. **Step 2:** Selecciona **Create a connection** — se genera automáticamente
3. **Step 3:** Copia el **Connection string** generado
4. **Step 4:** Selecciona **Open flow builder** — se abre Power Automate con el trigger **"When an activator rule is triggered"** precargado

**Paso 2 - Configurar el flujo en Power Automate**

En Power Automate:
1. El trigger **"When an activator rule is triggered"** ya viene precargado con el connection string
2. Selecciona **+** para agregar una acción: **Post message in a chat or channel** (Teams)
3. Configura:
   - **Post as:** Flow bot
   - **Post in:** Channel
   - **Team:** el equipo de Teams donde quieres recibir las alertas (ej. `Plant-Operators`)
   - **Channel:** el canal dentro del equipo (ej. `Operators`)
   - **Message:** `Fabric Operation Agent Message`
4. **Save** el flujo
5. Vuelve al wizard en Fabric — el status debe cambiar a **Connected**
6. Selecciona **Apply**


#### 7.3 Configurar el aprobador de acciones

1. Selecciona el ícono de **Settings** (engranaje ⚙️) en el ribbon
2. En el panel que se abre, selecciona **Agent behavior**
3. En **Agent action approver**, agrega tu cuenta — este es el usuario que recibirá los mensajes en Teams con las recomendaciones del agente para aprobar o rechazar
4. Cierra el panel de settings

#### 7.4 Generar playbook y activar el agente

1. Selecciona **Save** en el ribbon para guardar la configuración
2. Selecciona **Generate playbook** — el agente genera automáticamente las reglas y condiciones basándose en los goals, instructions y knowledge configurados
3. Revisa el playbook generado en el panel derecho — verifica que las condiciones reflejen los umbrales de temperatura y vibración definidos. Si no refleja lo esperado, ajusta los goals o instructions y genera el playbook de nuevo
4. Cuando estés satisfecho con el playbook, selecciona **Save** y luego **Start** en el ribbon para activar el agente
5. El agente comenzará a monitorear el Eventhouse cada 5 minutos y enviará mensajes en Teams cuando detecte condiciones que coincidan con las reglas del playbook

> 💡 **Best practices para generar el playbook correctamente:** El Operations Agent puede fallar al generar el playbook si las instrucciones no están bien estructuradas. Sigue estas recomendaciones de la [documentación oficial de Microsoft](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/operations-agent-limitations):
> - Escribe siempre en **inglés** — el LLM no soporta otros idiomas
> - Encierra los nombres de columnas en comillas dobles: `"Temperature"`, `"MachineId"`
> - Especifica explícitamente la tabla que el agente debe monitorear
> - Define umbrales concretos y cuantificables en lugar de descripciones vagas
> - Describe cada regla en una línea o bullet point separado
> - Si el playbook no se genera, ajusta las instrucciones y vuelve a intentarlo — el LLM es probabilístico y puede requerir varios intentos

**Para probar:** Inserta un registro que active solo la regla de temperatura, sin cruzar los otros umbrales simultáneamente — así evitas recibir múltiples alertas para la misma máquina:
```kql
.ingest inline into table MachineTelemetry <|
2026-03-27T16:00:00Z,MCH-002,PLT-001,83.5,1.2,4.8,0.85,Alerta
```

Este registro activa solo la regla de temperatura (> 80) sin disparar las de vibración ni status crítico. En Teams recibirás el mensaje en los próximos 1-3 minutos con la acción recomendada. Puedes seleccionar **Yes** para aprobar o **No** para rechazar.

---

## 📊 Arquitectura
```
┌────────────────────────────────────────────────────────────────────┐
│                      Microsoft Fabric Workspace                    │
│                                                                    │
│  ┌──────────────────┐   ┌─────────────────┐   ┌─────────────────┐  │
│  │ ManufacturingLH  │   │ ManufacturingEH │   │ ManufacturingSM │  │
│  │   (Lakehouse)    │   │ (Eventhouse)    │   │ (Semantic Model)│  │
│  │                  │   │                 │   │                 │  │
│  │ DimSupplier      │   │ MachineTelemetry│   │ Direct Lake     │  │
│  │ DimProduct       │   │ (KQL Database)  │   │ sobre Lakehouse │  │
│  │ DimPlant         │   └────────┬────────┘   └────────┬────────┘  │
│  │ DimMachine       │            │                     │           │
│  │ FactPurchaseOrder│            │                     │           │
│  └──────────────────┘            │                     │           │
│           │                      │                     │           │
│           └──────────────────────┴─────────────────────┘           │
│                                  │                                 │
│                    ┌─────────────▼──────────┐                      │
│                    │  ManufacturingOntology │                      │
│                    │      (Ontology)        │                      │
│                    │                        │                      │
│                    │  Entidades:            │                      │
│                    │  Proveedor, Producto   │                      │
│                    │  Planta, Maquina       │                      │
│                    │  OrdenDeCompra         │                      │
│                    └───────────┬────────────┘                      │
│                                │                                   │
│            ┌───────────────────┼                                   │ 
│            │                   │                                   │
│  ┌─────────▼────────┐  ┌───────▼───────────┐                       │
│  │ManufacturingAgent│  │MachineMonitorAgent│                       │
│  │  (Data Agent)    │  │(Operations Agent) │                       │
│  │                  │  │                   │                       │
│  │ NL queries sobre │  │ Monitoreo RT +    │                       │
│  │ la Ontología     │  │ Teams alerts      │                       │
│  └──────────────────┘  └───────────────────┘                       │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Preguntas sugeridas (Data Agent)

Estas preguntas están diseñadas para mostrar el valor cross-domain de la Ontología:

**Supply Chain:** — traversal multi-entidades estáticas
- "¿Qué proveedores cumplieron órdenes recibidas por la planta Monterrey?"
- "¿Qué productos están incluidos en órdenes recibidas por plantas de tipo manufactura?"

**Operaciones & Mantenimiento:** — traversal estático + timeseries
- "¿Cuál es el OEE promedio de las máquinas que opera la planta Monterrey?"
- "¿Qué máquinas opera la planta de Queretaro?"

**Cross-domain (transversal):**
- "¿Qué plantas recibieron órdenes de compra pendientes y también operan máquinas de tipo Prensa?"
- "¿Qué máquinas que opera la planta Monterrey tuvieron temperatura máxima mayor a 80 grados en el 2025?"

> 💡 Estas preguntas demuestran el valor único de la Ontología — traversal semántico entre entidades de negocio. Un Data Agent sobre Lakehouse o Semantic Model podría responder preguntas de aggregation simples, pero no puede razonar sobre conexiones entre dominios como Compras, Operaciones y Mantenimiento sin joins técnicos explícitos.

---

## ⚠️ Limitaciones conocidas (Preview)

| Limitación | Impacto | Workaround |
|---|---|---|
| Decimal type no soportado en Graph | Columnas como `UnitPrice` y `TotalAmount` pueden mostrar NULL en el grafo | Usar el Data Agent para consultas numéricas (o vincula con el Lakehouse directamente) |
| Lakehouse con OneLake Security no soportado | Si el LH tiene RLS habilitado, no se puede usar como fuente | Desactivar OneLake Security en el LH del demo |
| Column mapping automático en nombres especiales | Columnas con espacios o caracteres especiales rompen el grafo | Los CSVs de este repo ya tienen nombres limpios |
| Data Agent requiere inicialización (~2-3 min) | Las primeras consultas pueden dar error de "no data" | Esperar y reintentar |
| Operations Agent requiere Teams | Sin Teams no se reciben las notificaciones | Instalar la app en Teams antes de activar el agente |
| Ontología no se genera desde "My Workspace" | Si el SM está en My Workspace, falla la generación | Usar un workspace compartido con capacidad |
| Graph model requiere refresh manual desde workspace | Sin refresh el Relationship graph devuelve "Query result contains no data" | Refrescar el item Graph model desde el workspace antes de usar el graph |

---

## 🔭 Próximos pasos (fuera de scope de este demo)

### Plan (Planning Sheets)
El workload **Plan** permite hacer presupuestación y forecasting colaborativo directamente sobre el Semantic Model. Requiere adicionalmente:
- Una **Fabric SQL Database** para almacenar los metadatos de planeación
- Configurar Planning Sheets conectadas al `ManufacturingSM`

Caso de uso natural para este escenario: presupuesto de compras por proveedor y planta para el siguiente trimestre.

Documentación: [Planning sheets in plan (preview)](https://learn.microsoft.com/en-us/fabric/iq/plan/planning-overview)

### Fabric Activator
Para que el Operations Agent ejecute acciones automáticas (no solo notificaciones), conectar un **Activator** item que dispare workflows en Power Automate o llame APIs externas.

### MCP Integration
La Ontología es accesible via **MCP (Model Context Protocol)**, lo que permite que agentes externos (como Azure AI Foundry Agents) consuman el mismo vocabulario de negocio sin reconfiguración.

---

## 📚 Referencias

- [What is Fabric IQ (preview)?](https://learn.microsoft.com/en-us/fabric/iq/overview)
- [What is Ontology (preview)?](https://learn.microsoft.com/en-us/fabric/iq/ontology/overview)
- [Ontology tutorial — Introduction](https://learn.microsoft.com/en-us/fabric/iq/ontology/tutorial-0-introduction)
- [Operations Agent](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/operations-agent)
- [Data Agent](https://learn.microsoft.com/en-us/fabric/data-science/concept-data-agent)
- [Planning sheets](https://learn.microsoft.com/en-us/fabric/iq/plan/planning-overview)

---

## 🤝 Contribuciones

Este repo es un punto de partida. Si encuentras bugs en los pasos o cambios en la UI del preview, abre un issue o PR.
