# 01 — Contexto y Dominio

> Documento maestro del Valorizador CEN. Define el vocabulario, el modelo conceptual y las reglas del negocio. Toda decisión técnica posterior (backend, frontend, template Excel) debe ser consistente con este documento.

---

## 1. Propósito de la herramienta

El **Coordinador Eléctrico Nacional (CEN)** evalúa anualmente las obras del **Plan de Expansión** del sistema de transmisión. Cada obra requiere una **valorización**: la estimación determinística del costo de inversión (CAPEX) que servirá como referencia regulatoria.

El Valorizador CEN es la herramienta que permite al departamento de Ingeniería y Diseño:

1. **Modelar** la obra a partir de su descripción en el decreto.
2. **Valorizar** la obra agregando partidas unitarias desde un catálogo gobernado por el CEN.
3. **Exportar** la memoria de cálculo y un reporte ejecutivo trazables.

El método es **bottom-up determinístico**: dado un modelo de obra y un catálogo, el resultado es reproducible y auditable.

---

## 2. Glosario CEN

| Término | Definición |
|---|---|
| **Obra** | Proyecto del Plan de Expansión definido por un decreto. Puede involucrar 1 o más instalaciones. |
| **Instalación** | Unidad física valorizada (una subestación específica, un tramo de línea específico). Una obra puede contener instalaciones de tipos distintos. |
| **Componente tipificado** | Módulo estandarizado del catálogo CEN (paño, barra, vano de línea, transformador, equipo de compensación). Trae una composición predefinida de partidas. |
| **Partida unitaria** | Unidad atómica del costo: equipo, material, montaje, ingeniería o servidumbre/terreno. Tiene un Precio Unitario (PU) en el catálogo. |
| **PU (Precio Unitario)** | Valor monetario asociado a una partida, definido por el catálogo CEN y no editable por el modelador. |
| **CAPEX** | Costos de inversión. Único tipo de costo en el alcance del prototipo. |
| **COMA / AVI** | Costos de O&M y Anualidad del Valor de Inversión. **Fuera de alcance** en el prototipo. |
| **Tipo de obra** | Una de cuatro: `SE-nueva`, `SE-amp`, `L-nueva`, `L-amp`. |
| **Subtipo de obra** | Granularidad adicional dentro del tipo de obra (8 valores, ver §3.3.1). Indexa costos generales y factores. Derivable automáticamente o seleccionable. |
| **Equipo genérico** | Identificación canónica de un equipo (interruptor TV, desconectador trifásico SPT, TP, TC, …) independiente de su tensión. Vive en `equipos_genericos` del catálogo y es la llave compartida entre partidas, composiciones, fundaciones y estructuras. |
| **Cubicación** | Cantidad de materiales (m³ de hormigón, kg de enfierradura, kg de acero, un de pernos) que insume un componente físico. Se modela como filas de la composición del componente (fundación, estructura), no como código del motor. |
| **Zona** | Clasificación geográfica que determina los PU de terreno. |
| **Plazo** | Duración del proyecto, insumo para calcular costos generales (inspección, etc.) e intereses intercalarios. |
| **Decreto** | Acto regulatorio que formaliza la obra. Es la entrada base del modelado (en el prototipo se ingresa manualmente; futura integración con sistemas del CEN). |
| **Diagrama Unifilar (DU)** | Representación esquemática del sistema eléctrico que el editor canvas construye. Modo principal de visualización del prototipo. |
| **DEE Planta** | Vista de planta (topología física). Documentada como vista futura, no implementada en el prototipo. |
| **Catálogo** | Base de datos versionada de componentes tipificados, composiciones, partidas, PU y factores. Mantenida por el CEN, ingresada vía Excel del consultor externo. |
| **Versión del catálogo** | Snapshot inmutable del catálogo en una fecha de vigencia. Cada proyecto queda anclado a una versión específica. |
| **Curador del catálogo** | Rol que sube nuevas versiones del Excel y publica. |
| **Modelador** | Rol que crea proyectos y valoriza obras usando el catálogo. |
| **Factor** | Porcentaje aplicado sobre subtotales (GG, utilidades, contingencias, ITO). Tiene defaults por tipo de obra; editable a nivel proyecto. |
| **Partida ad-hoc** | Partida añadida manualmente al proyecto, no presente en el catálogo. Requiere justificación. |
| **Snapshot valorizado** | Estado inmutable de un proyecto al pasar a estado "Valorizado": congela inputs, versión del catálogo y resultados. |
| **AIS** | Subestación aislada en aire (convencional). |
| **GIS** | Subestación aislada en gas. En GIS, los módulos se valorizan en los paños; la "barra" agrupa obras de terreno (galpón, grúa). |
| **HIS** | Tecnología híbrida (equipos AIS + GIS). |

---

## 3. Alcance del dominio

### 3.1 Segmentos cubiertos

- **Transmisión nacional y zonal** del sistema eléctrico chileno.
- **No** se cubren distribución, generación, ni transmisión dedicada.

### 3.2 Niveles de tensión

| Nivel | Uso |
|---|---|
| 500 kV | Troncal nacional |
| 220 kV | Nacional/zonal |
| 154 kV | Zonal |
| 110 kV | Zonal |
| 66 kV | Zonal |
| MT (genérica) | Conexión a barra MT en subestaciones |

### 3.3 Tipos de obra

| Código | Nombre | Componentes característicos |
|---|---|---|
| `SE-nueva` | Nueva Subestación | SE completa (barras, paños, trafos, SSAA, obras civiles) + **enlace de seccionamiento** (línea corta de conexión al sistema existente) |
| `SE-amp` | Ampliación de Subestación | Extensión de barras + nuevos paños + **cambio de equipos** en paños existentes (normalización) |
| `L-nueva` | Nueva Línea | Tramo completo (estructuras, conductores, fundaciones, faja) + **paños en los extremos** (en SE existentes) |
| `L-amp` | Ampliación de Línea | **Cambio de equipos** en paños existentes (aumento capacidad) + **nuevo paño** en tendido de segundo circuito + extensión |

**Regla**: una obra puede incluir instalaciones de varios tipos. Por ejemplo, una `SE-nueva` siempre arrastra una línea corta (enlace de seccionamiento), y una `L-nueva` siempre arrastra paños en ambos extremos.

#### 3.3.1 Subtipos de obra (para indexar costos generales)

Algunos costos generales y factores varían dentro del mismo `tipo_obra` según atributos del proyecto. Para indexarlos se define `subtipo_obra` (8 valores), **derivado automáticamente** de las entradas del proyecto o elegido manualmente por el modelador:

| `tipo_obra` | `subtipo_obra` | Condición de derivación |
|---|---|---|
| `SE-nueva` | `nueva_se_zonal` | tensión máx. < 220 kV (default) |
| `SE-nueva` | `nueva_se_nacional` | tensión máx. ≥ 220 kV |
| `SE-amp` | `amp_barra` | sin transformador nuevo (default) |
| `SE-amp` | `amp_se_ntr` | incluye al menos un transformador nuevo |
| `L-nueva` | `nueva_linea_corta` | longitud total ≤ 200 km (default) |
| `L-nueva` | `nueva_linea_larga` | longitud total > 200 km |
| `L-amp` | `tendido_2c` | tipo de ampliación = tendido segundo circuito |
| `L-amp` | `aumento_capacidad` | tipo de ampliación = aumento de capacidad (default) |

La regla exacta y los umbrales se declaran en el catálogo (hoja `reglas_subtipo`, opcional). Si la hoja no está, el modelador elige el subtipo desde un selector en el formulario de antecedentes; el frontend sugiere un default con la heurística de la tabla anterior. Los 4 `tipo_obra` siguen siendo la primera elección visible para el modelador; los subtipos solo aparecen donde son relevantes (selector en antecedentes y línea por línea de costos generales en la memoria de cálculo).

---

## 4. Modelo conceptual: jerarquía de 4 niveles

El núcleo del dominio es una jerarquía estricta de agregación bottom-up:

```
┌─────────────────────────────────────────────────────────┐
│ Obra                                                    │
│   (Plan de Expansión, definida por decreto)             │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │ Instalación (1..N)                              │   │
│   │   (subestación específica / tramo de línea)     │   │
│   │                                                 │   │
│   │   ┌─────────────────────────────────────────┐   │   │
│   │   │ Componente tipificado (1..N)            │   │   │
│   │   │   (paño, barra, vano, trafo, …)         │   │   │
│   │   │                                         │   │   │
│   │   │   ┌─────────────────────────────────┐   │   │   │
│   │   │   │ Partida unitaria (1..N)         │   │   │   │
│   │   │   │   (cantidad × PU)               │   │   │   │
│   │   │   └─────────────────────────────────┘   │   │   │
│   │   └─────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 4.1 Reglas de composición

- **Obra**: contenedor, no aporta costo directo. Hereda factores por defecto según `tipo_obra`.
- **Instalación**: agrupa componentes. Conoce su zona (para PU de terreno) y plazo.
- **Componente tipificado**: trae una **composición predefinida** de partidas con cantidades base, que se escalan según parámetros (longitud de vano, MVA del trafo, etc.).
- **Partida unitaria**: el único nivel que aporta costo monetario. `subtotal = cantidad_resuelta × PU`.

### 4.2 Origen de cada nivel

| Nivel | ¿Quién lo define? | ¿Editable por modelador? |
|---|---|---|
| Obra | Decreto + usuario | Sí (parámetros generales) |
| Instalación | Usuario al modelar | Sí (tipo, zona heredada, plazo heredado) |
| Componente | Catálogo CEN (set tipificado) | Selección sí; estructura interna no |
| Partida | Catálogo CEN (composición del componente) | PU **no**, cantidad sí cuando depende de parámetro |

### 4.3 Partidas ad-hoc

El modelador puede añadir partidas **fuera del catálogo** cuando una obra requiere algo no tipificado. Estas partidas:
- Quedan marcadas como `ad_hoc = true`.
- Requieren **justificación obligatoria** en texto libre.
- Aparecen separadas en la memoria de cálculo.

---

## 5. Tipificación de componentes

### 5.1 Antecedentes generales (inputs del proyecto)

Son los primeros tres parámetros que el modelador define al iniciar una obra. Determinan defaults y opciones disponibles aguas abajo.

| Parámetro | Valores | Impacto |
|---|---|---|
| **Tipo Proyecto** | `Nueva Línea`, `Nueva Subestación`, `Ampliación Subestación`, `Ampliación Línea` | Habilita/deshabilita componentes en la paleta; carga factores por defecto |
| **Zona** | Lista regional CEN | Determina los **PU de terreno y servidumbres** |
| **Plazo** | Meses | Determina **costos generales** (ITO, etc.) e **intereses intercalarios** |

### 5.2 Barra (AT)

Componente principal de una subestación AT. Una SE puede tener **múltiples barras** (Barra 1, Barra 2, …), modelando configuraciones de varios patios.

| Atributo | Valores |
|---|---|
| Tecnología | `AIS`, `GIS`, `HIS` |
| Tensión | 66 / 110 / 154 / 220 / 500 kV |
| Configuración | (depende del decreto: simple, doble, doble con seccionador, interruptor y medio, …) |
| Posiciones | Entero. Define cuántos paños caben en la barra. |
| Capacidad | Determina internamente el tipo de conductor de barra (sin elección del usuario) |

**Reglas:**
- Las posiciones tienen un **indicador visible** que decrementa a medida que se conectan paños.
- La configuración es **atributo del componente Barra**, no de la SE como tal. Una SE con dos patios puede tener configuraciones distintas en cada barra.
- En **GIS**, la barra agrupa solo obras de terreno (galpón, grúa, civiles); los módulos GIS se valorizan dentro de los paños.

### 5.3 Barra (MT)

Componente para conexión en media tensión, típicamente como secundario de transformador.

| Atributo | Valores |
|---|---|
| Tecnología | `AIS`, `celdas` |
| Configuración | (según norma técnica de la SE) |
| Posiciones | Entero |

**Regla de herencia**: al conectar un transformador a una Barra AT y una Barra MT, los **paños de trafo heredan** la tensión y configuración de las barras conectadas.

### 5.4 Paño

El paño es el "espacio" en una barra que se conecta a un equipo aguas abajo (línea, trafo, compensador) o entre barras (acoplador, seccionador).

#### 5.4.1 Paño (estándar)

| Atributo | Valores |
|---|---|
| Tipo de interruptor | Depende de tecnología de barra (ver 5.4.3) |
| Función | `línea`, `trafo`, `acoplador`, `seccionador`, `compensador` (se **determina al conectar**) |
| Conexiones | Acepta **1** conexión |

**Reglas:**
- La función del paño se asigna **automáticamente** al conectarlo a una línea, trafo o compensador.
- Si el usuario quiere dejar un paño solo "vestido" (sin conexión definitiva), existe la opción `click` para asignar función manualmente.
- En `SE-nueva`, los paños **acoplador** y **seccionador** se generan **automáticamente** según la configuración de la barra.
- En `SE-amp`, los paños acoplador y seccionador son **seleccionables manualmente** vía `click`.

#### 5.4.2 Paño Diagonal

| Atributo | Valores |
|---|---|
| Tipo de interruptor | Depende de tecnología de barra |
| Subtipo | `diagonal completa`, `media diagonal` |
| Función | `línea`, `trafo` |
| Conexiones aceptadas | 2 líneas / 1 línea + 1 trafo / 1 línea / 1 trafo. **2 trafos: prohibido**. |

**Reglas de interruptores en diagonales:**

| Caso | Interruptores |
|---|---|
| `SE-nueva` con media diagonal | **2** interruptores (siempre) |
| `SE-amp` o `L-nueva` con media diagonal | **1 o 2** interruptores (completar diagonal vía `click`) |
| Diagonal completa (cualquier tipo de obra) | **3** interruptores |

#### 5.4.3 Tipo de interruptor según tecnología de la barra

| Tecnología | Opciones |
|---|---|
| AIS | `tanque vivo`, `tanque muerto` |
| HIS | `híbrido`, `híbrido monopolar` |
| GIS | sin selección de tipo (todo es modular GIS) |

### 5.5 Línea

| Atributo | Valores |
|---|---|
| Tipo de conexión | `Línea`, `Seccionamiento` |
| Longitud | km |
| Cantidad de circuitos | 1, 2 |
| Capacidad | MVA |
| Interferencia | flag/parámetro |
| Cable guardia | tipo según catálogo |
| Tipo de torre | `celosía`, `poste hormigón`, `monoposte`, `subterránea` |
| Tipo de conductor | según catálogo |
| Tipo de ampliación (solo si `L-amp`) | `tendido segundo circuito`, `aumento de capacidad`, `extensión` |

**Reglas:**
- **Seccionamiento**: duplica las conexiones a la subestación; trae antecedentes de la línea existente (tipo de conductor, n° de conductores por fase, tipo de cable guardia). La longitud trae un default editable. En el prototipo estos antecedentes se ingresan manualmente; la integración con InfoTécnica queda para etapas posteriores.
- **Doble circuito**: el grafo dibuja una **segunda conexión** al paño y a la barra, automáticamente.
- **Futuro**: trazado geográfico con complemento de Google Earth.

### 5.6 Transformador

| Atributo | Valores |
|---|---|
| Tipo de trafo | según catálogo (relaciones primario/secundario, número de devanados) |
| Capacidad | MVA |
| Muro y foso | siempre incluido en SE-nueva; `click` en ampliaciones |
| Sistema contra incendio | `click` |

### 5.7 Compensación

| Atributo | Valores |
|---|---|
| Tipo | `BBCC` (banco de condensadores), `reactor de línea`, `reactor de barra` |
| Capacidad | MVAr |

### 5.8 Resumen de componentes habilitados por tipo de obra

| Componente | `SE-nueva` | `SE-amp` | `L-nueva` | `L-amp` |
|---|:---:|:---:|:---:|:---:|
| Barra AT | ● | ● (extensión) | – | – |
| Barra MT | ● | ● | – | – |
| Paño (estándar) | ● | ● | ● (en extremos) | ● (cambio equipos) |
| Paño Diagonal | ● | ● | ● (en extremos) | ● |
| Línea | ● (enlace de seccionamiento) | – | ● (tramo principal) | ● |
| Transformador | ● | ● | – | – |
| Compensación | ● | ● | – | – |

---

## 6. Estructura de costos (panel de estimación)

El motor agrega costos por estas líneas exactas. Esta estructura es **canónica** y debe respetarse en el panel derecho del editor, en la memoria de cálculo Excel y en el reporte PDF.

| # | Categoría | Cálculo |
|---|---|---|
| **1** | **Costos Directos** | Σ subcategorías 1.1 a 1.6 |
| 1.1 | Ingeniería | factor × costo directo base (o partidas directas si aplica) |
| 1.2 | Gestión medio ambiental | factor × costo directo base |
| 1.3 | Instalación de faena | factor × costo directo base |
| 1.4 | Suministros, Obras Civiles, Montaje | **núcleo bottom-up**: Σ partidas de componentes |
| 1.5 | Servidumbres y terrenos | Σ partidas de terreno (PU por zona × cantidad) |
| 1.6 | Pruebas y Puesta en Servicio | factor × costo directo base |
| **2** | **Costos Indirectos** | Σ subcategorías 2.1 a 2.4 |
| 2.1 | Gastos Generales y Seguros | factor × (1) |
| 2.2 | Inspección técnica de obra (ITO) | función de plazo |
| 2.3 | Utilidades del contratista | factor × (1) |
| 2.4 | Contingencias | factor × (1) |
| **3** | **Monto Contrato** | (1) + (2) |
| **4** | **Intereses Intercalarios** | función de plazo y (3) |
| | **Costo Total del Proyecto** | (3) + (4) |

**Notas:**
- **1.4** es la línea principal alimentada por el motor bottom-up: cada componente tipificado aporta sus partidas valoradas.
- Los factores aplicados a 1.1, 1.2, 1.3, 1.6, 2.1, 2.3, 2.4 vienen del catálogo con defaults por `tipo_obra`. El modelador puede sobrescribirlos a nivel proyecto.
- **2.2 (ITO)** y **4 (Intereses Intercalarios)** son función del **plazo**. La fórmula exacta se define en `02-backend.md`.
- Solo **CAPEX**. Sin COMA/AVI.

---

## 7. Reglas transversales

### 7.1 Herencia top-down

Las selecciones del modelador en niveles superiores se propagan como **defaults editables** hacia abajo:

```
Proyecto (tipo obra, zona, plazo)
   ↓
Instalación (hereda zona, plazo)
   ↓
Barra (hereda tensión cuando es relevante)
   ↓
Paño (hereda tecnología de la barra → opciones de interruptor)
   ↓
Conexión (línea/trafo/compensador hereda configuración de paño)
```

### 7.2 Reglas de conexión en el canvas

| Origen | Destino | ¿Permitido? |
|---|---|---|
| Barra AT | Paño | Sí |
| Paño | Línea | Sí |
| Paño | Trafo | Sí |
| Paño | Compensación | Sí (excepto paño acoplador/seccionador) |
| Paño Diagonal | hasta 2 conexiones (líneas/trafo según 5.4.2) | Sí |
| Trafo | Barra MT | Sí |
| Línea | Línea | No |
| Barra | Barra | Solo vía transformador y sus respectivos paños |

### 7.3 Inmutabilidad del catálogo y anclaje

- Cada **versión publicada** del catálogo es **inmutable**.
- Cada proyecto registra su `catalogo_version_id` al momento de cálculo.
- Una nueva versión publicada **no altera** proyectos existentes hasta que el modelador decida migrar.
- Al haber nueva versión, el sistema **propone migrar** en la UI; el modelador acepta o rechaza.

### 7.4 Reproducibilidad

Dado el mismo `(modelo_de_obra, catalogo_version_id, factores_efectivos)`, el resultado de la valorización es **determinístico**. Esto debe poder verificarse con tests de regresión.

---

## 8. Ciclo de vida del proyecto

```
   ┌──────────┐    avanzar    ┌──────────────┐    valorizar    ┌─────────────┐
   │ Borrador │─────────────▶│ En revisión  │─────────────────▶│ Valorizado  │
   └────┬─────┘               └──────┬───────┘   (snapshot)    └──────┬──────┘
        │                            │                                │
        │                            ▼                                ▼
        │                       ┌─────────┐                      ┌──────────┐
        └──────clonar───────────│ Variante│                      │Archivado │
                                └─────────┘                      └──────────┘
```

| Estado | Quién edita | Qué se persiste |
|---|---|---|
| **Borrador** | Modelador | Modelo de obra mutable. Recalculo en vivo. |
| **En revisión** | Lectura, modelador puede revertir a Borrador | Bloqueo blando contra ediciones accidentales |
| **Valorizado** | Nadie (inmutable) | **Snapshot completo**: modelo + `catalogo_version_id` + factores efectivos + resultado |
| **Archivado** | Solo lectura | Mantiene snapshot, oculto del listado activo |

**Clonación**: cualquier proyecto (en cualquier estado) puede clonarse para generar una **variante** en estado Borrador. Caso de uso típico: comparar AIS vs GIS, ruta A vs B.

---

## 9. Outputs

| Output | Formato | Contenido |
|---|---|---|
| Vista web (panel derecho) | en pantalla | Estructura de costos en vivo, recalculada al editar el grafo |
| Memoria de cálculo | **Excel** (`.xlsx`) | Hojas por instalación, desglose hasta partida, factores aplicados, referencia a `catalogo_version_id` |
| Reporte ejecutivo | **PDF** | Total CAPEX, desglose por instalación, supuestos clave, decreto referido |

El export JSON estructurado y la comparación lado a lado se postergan para etapas posteriores.

---

## 10. Roles

| Rol | Permisos en prototipo |
|---|---|
| **Curador del catálogo** | Sube nuevas versiones del Excel; valida y publica versiones; **no** modifica proyectos |
| **Modelador** | Crea, edita, valoriza, clona, archiva proyectos; **no** modifica el catálogo |

En el prototipo no hay autenticación: el rol se simula con un selector en la cabecera. La gobernanza real se implementará al integrar SSO en etapas posteriores.

---

## 11. Lo que **no** está en alcance (prototipo)

| Tema | Razón |
|---|---|
| Autenticación / SSO | Etapa posterior; el prototipo valida dominio y UX |
| Despliegue cloud / Docker / TLS | Etapa posterior |
| Integración con InfoTécnica | Etapa posterior; antecedentes se cargan manualmente |
| Vista DEE Planta | Documentada como vista futura; el prototipo cubre solo DU |
| Trazado de línea con Google Earth | Etapa posterior |
| Comparación lado a lado de proyectos | v2 |
| Export JSON estructurado | Etapa posterior |
| COMA / AVI | Fuera de alcance del producto (solo CAPEX) |
| Distribución / generación | Fuera del segmento del producto |

---

## 12. Referencias internas

- **`02-backend.md`** — implementación del modelo de datos, fórmulas del motor y contratos de API.
- **`03-frontend.md`** — editor canvas, formularios, panel de estimación, flujos de proyecto.
- **`04-arquitectura-sistema.md`** — stack, repo y ejecución local.
- **`05-template-excel-catalogo.md`** — contrato del Excel del consultor (insumo del catálogo).
