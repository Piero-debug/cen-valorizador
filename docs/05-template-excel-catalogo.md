# 05 — Template Excel del Catálogo CEN

> **Contrato externo** entre el CEN y el consultor de costos y cubicaciones. Define el formato del archivo Excel (`.xlsx`) que el consultor entrega al CEN y que el Valorizador ingesta para construir una versión del catálogo.
>
> Este documento es la única fuente de verdad de la estructura del archivo. Cualquier cambio aquí requiere acuerdo entre CEN y consultor.

---

## 1. Resumen del archivo

- **Formato**: `.xlsx` (Excel 2007+).
- **Codificación de texto**: UTF-8 (Excel maneja por defecto).
- **Idioma**: español. Encabezados de columna en **snake_case sin tildes ni espacios**. Decimales con coma (`12.345,67`) o punto (`12345.67`); el ingestor acepta ambos.
- **Moneda de referencia**: definida en `metadata`. Cada partida lleva su propia `moneda` + `fecha_cotizacion`, y el ingestor normaliza al ingresar usando los tipos de cambio declarados.
- **Nomenclatura de archivo**: `catalogo_cen_<version>.xlsx` (ej. `catalogo_cen_2026.06.1.xlsx`).
- **Hojas obligatorias**: 11 (ver §2). Hojas adicionales son ignoradas por el ingestor.

### 1.1 Principios de diseño

1. **Una entidad por hoja**: cada hoja modela un único concepto.
2. **Códigos como llaves primarias**: toda referencia entre hojas se hace por `codigo` (texto). El consultor sigue la convención sugerida (§6).
3. **Vocabulario único (`listas`)**: cualquier string que aparece en columnas referenciadas vive en la hoja `listas`. El ingestor rechaza valores fuera de lista.
4. **Composiciones declarativas**: todo lo que un componente "trae" (equipos, montajes, materiales por cubicación, control/protección) se declara en `composiciones`. El motor solo agrega; no contiene fórmulas implícitas sobre cuántos kg de acero pesa una estructura ni cuánto hormigón consume una fundación — eso vive en las filas de la hoja.
5. **Equipo separado de montaje**: ambos como partidas independientes; el componente los suma vía composiciones. Aplica también a transformadores, GIS y sala de celdas.
6. **PU con trazabilidad**: cada precio lleva `valor_unitario` + `moneda` + `fecha_cotizacion`.
7. **Inmutabilidad post-publicación**: una vez ingestado y publicado, el archivo queda como snapshot histórico. Cualquier corrección requiere una **nueva versión**.
8. **Validación estricta**: el ingestor rechaza el archivo si hay errores de integridad referencial, valores fuera de rango, hojas faltantes o strings que no existan en `listas`.
9. **Ingreso vía formulario, no edición a mano**: el curador del catálogo utiliza un formulario de la aplicación para añadir/modificar entradas. El Excel es el resultado exportable y el contrato entre CEN y consultor; no la interfaz de edición.

### 1.2 Convenciones generales

- **Fila 1** de cada hoja: encabezados de columna (snake_case, sin tildes, sin espacios al inicio/fin).
- **Fila 2 en adelante**: datos.
- Columnas pueden agregarse al final sin romper la ingesta (se ignoran). **No** pueden eliminarse, renombrarse ni reordenarse las columnas obligatorias.
- Filas vacías intercaladas son ignoradas; primera fila vacía después de datos no marca fin (el ingestor lee todas las filas con `codigo` no vacío).
- Comentarios para el consultor pueden ir en columnas opcionales `nota_*` (ignoradas o registradas como metadata).
- Tipos de dato:
  - `tension` siempre **string** (`'MT'`, `'66'`, `'110'`, `'154'`, `'220'`, `'500'`). Nunca número.
  - Fechas en formato ISO `AAAA-MM-DD`.
  - Códigos en mayúsculas con guiones (`EQ-INT-TV-220`).

---

## 2. Hojas obligatorias

| # | Hoja | Propósito |
|---|---|---|
| 1 | `metadata` | Versión del catálogo, fecha, moneda, autor, fecha de cotización de referencia |
| 2 | `listas` | Vocabulario controlado (única fuente de verdad de strings) |
| 3 | `zonas` | Zonas geográficas con metadata (código, nombre, región) |
| 4 | `equipos_genericos` | Diccionario canónico de equipos (rompe la fragmentación entre hojas) |
| 5 | `partidas` | Unidades atómicas con PU fijo (equipo, montaje, material, obra civil, control/protección, estructura) |
| 6 | `partidas_terreno` | Partidas con PU dependiente de zona |
| 7 | `componentes` | Componentes tipificados (paños, barras, vanos, trafos, fundaciones, estructuras, módulos GIS) |
| 8 | `composiciones` | Componente → partidas con cantidad (constante o expresión paramétrica) |
| 9 | `factores` | Líneas indirectas: porcentajes o montos absolutos por tipo/subtipo de obra |
| 10 | `parametros` | ITO (USD/mes), intereses intercalarios (tasa anual + factor promedio plazo) |
| 11 | `reglas_subtipo` | Derivación automática de `subtipo_obra` desde atributos del proyecto (opcional v1) |

---

## 3. Esquemas por hoja

### 3.1 Hoja `metadata`

Formato **vertical** (campo–valor en 2 columnas).

| campo | valor |
|---|---|
| `version` | texto, único globalmente (ej. `2026.06.1`) |
| `fecha_publicacion` | fecha ISO |
| `fecha_vigencia_desde` | fecha ISO, ≥ `fecha_publicacion` |
| `autor_consultor` | texto |
| `moneda_referencia` | `USD`, `CLP`, `UF` (∈ `listas.moneda`) |
| `tipo_cambio_clp_usd` | decimal (si hay PU en CLP) |
| `tipo_cambio_uf_clp` | decimal (si hay PU en UF) |
| `fecha_cotizacion_referencia` | fecha ISO. Fecha base de las cotizaciones del catálogo |
| `nota` | texto libre (opcional) |

**Validaciones:**
- `version` no puede coincidir con una versión ya publicada.
- `fecha_vigencia_desde ≥ fecha_publicacion`.
- `moneda_referencia` ∈ `listas.moneda`.
- Si hay alguna `partida.moneda ≠ moneda_referencia`, el tipo de cambio correspondiente debe estar presente.

**Ejemplo:**

```
campo                         | valor
------------------------------+--------------------------
version                       | 2026.06.1
fecha_publicacion             | 2026-06-22
fecha_vigencia_desde          | 2026-07-01
autor_consultor               | XYZ Consultores Ltda.
moneda_referencia             | USD
tipo_cambio_clp_usd           | 950
fecha_cotizacion_referencia   | 2026-06-15
nota                          | Revisión 2 - incorpora 500 kV
```

---

### 3.2 Hoja `listas`

Define los **valores controlados** que pueden aparecer en otras hojas. Cualquier valor usado en `partidas`, `componentes`, etc. debe existir aquí.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `categoria` | texto | Sí | Nombre de la lista |
| `valor` | texto | Sí | Valor aceptado |
| `descripcion` | texto | No | Explicación opcional |

**Llave**: `(categoria, valor)`.

**Categorías obligatorias con sus valores canónicos:**

| categoria | valores |
|---|---|
| `tipo_obra` | `SE-nueva`, `SE-amp`, `L-nueva`, `L-amp` |
| `subtipo_obra` | `nueva_se_zonal`, `nueva_se_nacional`, `amp_barra`, `amp_se_ntr`, `nueva_linea_corta`, `nueva_linea_larga`, `tendido_2c`, `aumento_capacidad` |
| `tecnologia_se` | `AIS`, `GIS`, `HIS` |
| `tecnologia_mt` | `AIS`, `celdas` |
| `tipo_interruptor` | `tanque_vivo`, `tanque_muerto`, `hibrido`, `hibrido_monopolar` |
| `nivel_tension` | `MT`, `66`, `110`, `154`, `220`, `500` (siempre string) |
| `configuracion_barra` | `simple`, `doble`, `doble_seccionador`, `interruptor_y_medio` |
| `tipo_pano` | `estandar`, `diagonal_completa`, `media_diagonal` |
| `funcion_pano` | `linea`, `trafo`, `acoplador`, `seccionador`, `compensador` |
| `tipo_componente` | `barra_at`, `barra_mt`, `pano`, `linea`, `vano`, `transformador`, `compensacion`, `fundacion`, `estructura`, `sala_celdas`, `ssaa`, `oocc` |
| `tipo_equipo` | `interruptor`, `desconectador_trif_spt`, `desconectador_trif_cpt`, `desconectador_monof`, `tp`, `tc`, `pararrayos`, `trampa_onda`, `condensador_acople`, `aislador_pedestal`, `transformador_trif`, `autotransformador`, `gis_modulo`, `celda_mt`, `grupo_electrogeno`, `banco_baterias`, `ssg`, `muro_cortafuego`, `foso_separador`, `plataforma` |
| `tipo_torre` | `celosia`, `poste_hormigon`, `monoposte`, `subterranea` |
| `tipo_estructura_se` | `est_baja`, `est_alta` |
| `tipo_estructura_linea` | `suspension`, `anclaje` |
| `tipo_fundacion` | `zapata`, `pilote` |
| `tipo_compensacion` | `bbcc`, `reactor_linea`, `reactor_barra` |
| `unidad` | `un`, `km`, `m`, `m2`, `m3`, `ha`, `kg`, `MVA`, `MVAr`, `modulo`, `mes`, `gl` |
| `moneda` | `USD`, `CLP`, `UF` |
| `tipo_costo` | `equipo`, `montaje`, `obra_civil`, `material`, `control_proteccion`, `estructura`, `terreno`, `servicio` |
| `categoria_costo` | `1.1_ingenieria`, `1.2_medioambiental`, `1.3_instalacion_faena`, `1.4_suministros_oc_montaje`, `1.5_servidumbres_terrenos`, `1.6_pruebas_pes` |
| `modo_factor` | `pct`, `monto_fijo` |
| `aplica_sobre` | `costo_directo_base`, `costos_directos_total`, `monto_contrato` |

> Esta hoja es la **fuente de verdad** del vocabulario. Si un consultor escribe `'Tanque vivo'` en otra hoja en lugar de `'tanque_vivo'`, el ingestor rechaza el archivo.

---

### 3.3 Hoja `zonas`

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `codigo` | texto | Sí | Llave primaria (ej. `Z-A`, `Z-B`, ...) |
| `nombre` | texto | Sí | Nombre descriptivo |
| `region_administrativa` | texto | No | |
| `nota` | texto | No | |

**Ejemplo:**

```
codigo  | nombre              | region_administrativa
--------+---------------------+----------------------
Z-A     | Norte Grande        | I, II
Z-B     | Norte Chico         | III, IV
Z-C     | Centro              | V, RM, VI
Z-D     | Centro-Sur          | VII, VIII
Z-E     | Sur                 | IX, X, XIV
Z-F     | Austral             | XI, XII
```

---

### 3.4 Hoja `equipos_genericos`

Diccionario canónico de equipos. **Todas las demás hojas referencian un equipo por `equipo_codigo`**. Resuelve la fragmentación entre tablas de costo, cubicación de fundaciones y pesos de estructura.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `equipo_codigo` | texto | Sí | Llave primaria (ej. `EQ-INT-TV`) |
| `tipo_equipo` | texto | Sí | ∈ `listas.tipo_equipo` |
| `descripcion` | texto | Sí | Texto humano |
| `nota` | texto | No | |

> El `equipo_codigo` es **independiente de la tensión**. La tensión específica se combina con el equipo en `partidas` y `componentes`.

**Ejemplo:**

```
equipo_codigo       | tipo_equipo               | descripcion
EQ-INT-TV           | interruptor               | Interruptor línea tanque vivo
EQ-INT-TM           | interruptor               | Interruptor línea tanque muerto
EQ-INT-HIB          | interruptor               | Interruptor línea híbrido
EQ-INT-HIBM         | interruptor               | Interruptor línea híbrido monopolar
EQ-DS-TRIF-SPT      | desconectador_trif_spt    | Desconectador trifásico SPT
EQ-DS-TRIF-CPT      | desconectador_trif_cpt    | Desconectador trifásico CPT
EQ-DS-MONO          | desconectador_monof       | Desconectador monofásico
EQ-TP               | tp                        | Transformador de potencial
EQ-TC               | tc                        | Transformador de corriente
EQ-PARARRAYOS       | pararrayos                | Pararrayos
EQ-TRAMPA           | trampa_onda               | Trampa de onda
EQ-CAP-ACOPLE       | condensador_acople        | Condensador de acople
EQ-AIS-PEDESTAL     | aislador_pedestal         | Aislador pedestal
EQ-TRAFO-TRIF       | transformador_trif        | Transformador trifásico
EQ-AUTO             | autotransformador         | Autotransformador
EQ-GIS-MOD          | gis_modulo                | Módulo GIS principal
```

---

### 3.5 Hoja `partidas`

Unidades atómicas con PU **fijo** (independiente de zona). Cubre equipos, montajes, materiales, obras civiles, control/protección y estructuras.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `codigo` | texto | Sí | Llave primaria. Prefijo según `tipo_costo` (ver §6) |
| `descripcion` | texto | Sí | |
| `unidad` | texto | Sí | ∈ `listas.unidad` |
| `tipo_costo` | texto | Sí | ∈ `listas.tipo_costo` |
| `categoria_costo` | texto | Sí | ∈ `listas.categoria_costo`. A qué línea (1.1–1.6) agrega |
| `equipo_codigo` | texto | Cond. | ∈ `equipos_genericos.equipo_codigo`. Obligatorio cuando `tipo_costo ∈ {equipo, montaje, control_proteccion}` |
| `tension` | texto | Cond. | ∈ `listas.nivel_tension`. Obligatorio salvo materiales y servicios genéricos |
| `configuracion_equipo` | texto | No | Para variantes del mismo equipo (`tanque_vivo`, `SPT`, etc.). Si presente, debe estar en `listas` |
| `capacidad_nominal` | decimal | No | Para variantes por capacidad (trafos, compensación) |
| `valor_unitario` | decimal | Sí | ≥ 0 |
| `moneda` | texto | Sí | ∈ `listas.moneda` |
| `fecha_cotizacion` | fecha | Sí | `AAAA-MM-DD` |
| `nota` | texto | No | |

**Validaciones:**
- `codigo` único.
- `unidad`, `moneda`, `tipo_costo`, `categoria_costo` ∈ `listas`.
- `equipo_codigo` ∈ `equipos_genericos.equipo_codigo` cuando obligatorio.
- `valor_unitario ≥ 0`.

**Ejemplo:**

```
codigo            | descripcion                    | unidad | tipo_costo         | categoria_costo            | equipo_codigo   | tension | configuracion_equipo | capacidad_nominal | valor_unitario | moneda | fecha_cotizacion
EQ-INT-TV-220     | Interruptor TV 220 kV          | un     | equipo             | 1.4_suministros_oc_montaje | EQ-INT-TV       | 220     | tanque_vivo          |                   | 96000          | USD    | 2026-06-15
MON-INT-TV-220    | Montaje interruptor TV 220 kV  | un     | montaje            | 1.4_suministros_oc_montaje | EQ-INT-TV       | 220     | tanque_vivo          |                   | 6596           | USD    | 2026-06-15
CPM-INT-220       | C/P/M asociado a interruptor   | un     | control_proteccion | 1.4_suministros_oc_montaje | EQ-INT-TV       | 220     |                      |                   | 125822         | USD    | 2026-06-15
MAT-HORMIGON-H20  | Hormigón H20                   | m3     | material           | 1.4_suministros_oc_montaje |                 |         |                      |                   | 96             | USD    | 2026-06-15
MAT-ENFIERRADURA  | Enfierradura                   | kg     | material           | 1.4_suministros_oc_montaje |                 |         |                      |                   | 1.235          | USD    | 2026-06-15
MAT-ACERO-GALV    | Acero galvanizado              | kg     | material           | 1.4_suministros_oc_montaje |                 |         |                      |                   | 3.718          | USD    | 2026-06-15
MAT-PERNOS-ANC    | Pernos de anclaje              | un     | material           | 1.4_suministros_oc_montaje |                 |         |                      |                   | 14.8           | USD    | 2026-06-15
EQ-TRAFO-220-150  | Trafo trifásico 220 kV 150 MVA | un     | equipo             | 1.4_suministros_oc_montaje | EQ-TRAFO-TRIF   | 220     |                      | 150               | 1734580        | USD    | 2026-06-15
MON-TRAFO-220     | Montaje trafo 220 kV           | un     | montaje            | 1.4_suministros_oc_montaje | EQ-TRAFO-TRIF   | 220     |                      |                   | 147418         | USD    | 2026-06-15
COND-ACCC-PRAGUE  | Conductor ACCC Prague          | km     | material           | 1.4_suministros_oc_montaje |                 |         |                      |                   | 36.98          | USD    | 2026-06-15
OC-GIS-LOSA-220   | Losa por módulo GIS 220        | modulo | obra_civil         | 1.4_suministros_oc_montaje |                 | 220     |                      |                   | 410659         | USD    | 2026-06-15
```

---

### 3.6 Hoja `partidas_terreno`

Partidas cuyo PU **depende de la zona**. Aportan a la categoría `1.5_servidumbres_terrenos`. **Formato long** (una fila por (codigo, zona_codigo)) para que se pueda añadir tipos de partida zona-dependiente sin tocar el esquema.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `codigo` | texto | Sí | Llave primaria del tipo de partida |
| `descripcion` | texto | Sí | |
| `unidad` | texto | Sí | |
| `zona_codigo` | texto | Sí | ∈ `zonas.codigo` |
| `valor_unitario` | decimal | Sí | ≥ 0 |
| `moneda` | texto | Sí | |
| `fecha_cotizacion` | fecha | Sí | |
| `nota` | texto | No | |

**Llave compuesta**: `(codigo, zona_codigo)`.

**Ejemplo:**

```
codigo        | descripcion             | unidad | zona_codigo | valor_unitario | moneda | fecha_cotizacion
T-TERRENO-SE  | Terreno para SE         | ha     | Z-A         | 82603          | USD    | 2026-06-15
T-TERRENO-SE  | Terreno para SE         | ha     | Z-B         | 140751         | USD    | 2026-06-15
T-TERRENO-SE  | Terreno para SE         | ha     | Z-C         | 119083         | USD    | 2026-06-15
T-TERRENO-SE  | Terreno para SE         | ha     | Z-D         | 412560         | USD    | 2026-06-15
T-TERRENO-SE  | Terreno para SE         | ha     | Z-E         | 101119         | USD    | 2026-06-15
T-TERRENO-SE  | Terreno para SE         | ha     | Z-F         | 23543          | USD    | 2026-06-15
T-SERV-LINEA  | Servidumbre línea       | m2     | Z-A         | 2              | USD    | 2026-06-15
T-SERV-LINEA  | Servidumbre línea       | m2     | Z-B         | 3              | USD    | 2026-06-15
```

---

### 3.7 Hoja `componentes`

Componentes tipificados que el modelador instancia en el canvas. **Incluye fundaciones y estructuras como componentes propios**: un "paño AIS 220 kV interruptor y medio diagonal completa" es un componente agregador; una "fundación zapata de interruptor TV 220 kV" es otro componente cuya composición son materiales.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `codigo` | texto | Sí | Llave primaria |
| `tipo_componente` | texto | Sí | ∈ `listas.tipo_componente` |
| `descripcion` | texto | Sí | |
| `tension` | texto | Cond. | ∈ `listas.nivel_tension`. Obligatorio salvo componentes sin tensión inherente |
| `tecnologia` | texto | Cond. | ∈ `listas.tecnologia_se` o `tecnologia_mt`. Obligatorio para barras, paños, módulos GIS |
| `configuracion_barra` | texto | Cond. | Solo `barra_at`, `barra_mt`, `pano` |
| `tipo_interruptor` | texto | Cond. | Solo paños AIS/HIS |
| `tipo_pano` | texto | Cond. | Solo `pano`. ∈ `listas.tipo_pano` |
| `equipo_codigo` | texto | Cond. | ∈ `equipos_genericos`. Obligatorio para `fundacion` y `estructura` ("la fundación del equipo X") |
| `tipo_fundacion` | texto | Cond. | Solo `fundacion`. ∈ `listas.tipo_fundacion` |
| `tipo_estructura` | texto | Cond. | Solo `estructura`. ∈ `listas.tipo_estructura_se` o `tipo_estructura_linea` |
| `capacidad_nominal` | decimal | Cond. | Para `transformador` (MVA) o `compensacion` (MVAr) |
| `tipo_torre` | texto | Cond. | Solo `linea` o `vano` |
| `tipo_conductor` | texto | Cond. | Solo `linea` o `vano` (referencia a un `codigo` de conductor en `partidas`) |
| `n_conductores_fase` | entero | Cond. | Solo `linea` o `vano` |
| `tipo_compensacion` | texto | Cond. | Solo `compensacion` |
| `nota` | texto | No | |

**Ejemplo (mezcla):**

```
codigo                       | tipo_componente | descripcion                                | tension | tecnologia | configuracion_barra  | tipo_interruptor | tipo_pano          | equipo_codigo | tipo_fundacion | tipo_estructura | capacidad_nominal
C-PA-AIS-IM-DC-TV-220        | pano            | Paño AIS 220 kV IM DC tanque vivo          | 220     | AIS        | interruptor_y_medio  | tanque_vivo      | diagonal_completa  |               |                |                 |
C-PA-AIS-IM-MD-TV-220        | pano            | Paño AIS 220 kV IM MD tanque vivo          | 220     | AIS        | interruptor_y_medio  | tanque_vivo      | media_diagonal     |               |                |                 |
C-PA-GIS-DBL-220             | pano            | Paño GIS 220 kV doble                      | 220     | GIS        | doble                |                  | estandar           |               |                |                 |
C-FUN-INT-TV-ZAP-220         | fundacion       | Fund. zapata interruptor TV 220 kV         | 220     |            |                      |                  |                    | EQ-INT-TV     | zapata         |                 |
C-FUN-INT-TV-PIL-220         | fundacion       | Fund. pilote interruptor TV 220 kV         | 220     |            |                      |                  |                    | EQ-INT-TV     | pilote         |                 |
C-EST-INT-TV-220             | estructura      | Estructura baja interruptor TV 220 kV      | 220     |            |                      |                  |                    | EQ-INT-TV     |                | est_baja        |
C-VANO-220-CEL-2CF           | vano            | Vano línea 220 kV celosía 2 cond/fase      | 220     |            |                      |                  |                    |               |                |                 |
C-TRF-220-150                | transformador   | Trafo 220 kV 150 MVA                       | 220     |            |                      |                  |                    |               |                |                 | 150
C-GIS-MOD-220                | sala_celdas     | Módulo GIS 220 kV                          | 220     | GIS        |                      |                  |                    |               |                |                 |
```

**Llave**: `codigo`.

---

### 3.8 Hoja `composiciones`

**Hoja central del modelo declarativo**. Define qué partidas trae cada componente y en qué cantidad.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `componente_codigo` | texto | Sí | ∈ `componentes.codigo` |
| `partida_codigo` | texto | Sí | ∈ `partidas.codigo` ∪ `partidas_terreno.codigo` |
| `cantidad` | texto | Sí | Constante numérica o expresión paramétrica (§3.8.1) |
| `nota` | texto | No | |

**Llave compuesta**: `(componente_codigo, partida_codigo)`.

#### 3.8.1 Sintaxis de cantidad paramétrica

- **Constantes**: `3`, `1`, `0.5`.
- **Expresiones**: operadores `+ - * /` y paréntesis. Variables permitidas:

| Variable | Disponible en `tipo_componente` |
|---|---|
| `longitud_km` | `linea`, `vano` |
| `n_circuitos` | `linea`, `vano` |
| `n_conductores_fase` | `linea`, `vano` |
| `capacidad_mva` | `transformador` |
| `capacidad_mvar` | `compensacion` |
| `n_posiciones` | `barra_at`, `barra_mt` |

Ejemplos válidos: `longitud_km`, `longitud_km * n_circuitos * 3`, `(capacidad_mva / 100) * 1.2`.

El ingestor parsea la expresión y rechaza identificadores no permitidos para el `tipo_componente` correspondiente.

#### 3.8.2 Composición plana (v1)

En v1, las composiciones son **planas**: el `partida_codigo` apunta directamente a una partida atómica. La composición no referencia a otro componente.

Implicación práctica: para que el paño "C-PA-AIS-IM-DC-TV-220" incluya las fundaciones y estructuras de sus equipos, esas filas se replican explícitamente en su composición (no se llama al componente fundación). Esto duplica filas pero mantiene el motor simple.

Una composición jerárquica (componente → componente → partidas) queda pendiente para v2.

**Ejemplo 1: paño AIS 220 IM diagonal completa TV**

```
componente_codigo            | partida_codigo        | cantidad
C-PA-AIS-IM-DC-TV-220        | EQ-INT-TV-220         | 3
C-PA-AIS-IM-DC-TV-220        | MON-INT-TV-220        | 3
C-PA-AIS-IM-DC-TV-220        | CPM-INT-220           | 3
C-PA-AIS-IM-DC-TV-220        | EQ-DS-TRIF-SPT-220    | 6
C-PA-AIS-IM-DC-TV-220        | MON-DS-TRIF-SPT-220   | 6
C-PA-AIS-IM-DC-TV-220        | EQ-DS-TRIF-CPT-220    | 2
C-PA-AIS-IM-DC-TV-220        | EQ-TP-220             | 2
C-PA-AIS-IM-DC-TV-220        | EQ-TC-220             | 6
C-PA-AIS-IM-DC-TV-220        | EQ-PARARRAYOS-220     | 2
C-PA-AIS-IM-DC-TV-220        | EQ-TRAMPA-220         | 2
C-PA-AIS-IM-DC-TV-220        | EQ-AIS-PEDESTAL-220   | 6
```

**Ejemplo 2: fundación zapata interruptor TV 220 kV**

```
componente_codigo       | partida_codigo      | cantidad
C-FUN-INT-TV-ZAP-220    | MAT-HORMIGON-H10    | 35.1
C-FUN-INT-TV-ZAP-220    | MAT-HORMIGON-H20    | 1.5
C-FUN-INT-TV-ZAP-220    | MAT-ENFIERRADURA    | 2343
C-FUN-INT-TV-ZAP-220    | MAT-PERNOS-ANC      | 48
```

**Ejemplo 3: estructura baja interruptor TV 220 kV**

```
componente_codigo  | partida_codigo  | cantidad
C-EST-INT-TV-220   | MAT-ACERO-GALV  | 1650
```

**Ejemplo 4: vano línea 220 kV celosía 2 cond/fase (parametrizado)**

```
componente_codigo     | partida_codigo            | cantidad
C-VANO-220-CEL-2CF    | COND-ACCC-PRAGUE          | longitud_km * n_circuitos * 3 * n_conductores_fase
C-VANO-220-CEL-2CF    | EST-LINEA-SUSP-220-2CF    | longitud_km * 0.7 / 0.45
C-VANO-220-CEL-2CF    | EST-LINEA-ANC-220-2CF     | longitud_km * 0.3 / 0.45
C-VANO-220-CEL-2CF    | T-SERV-LINEA              | longitud_km * 1000 * 40
```

**Notas para el consultor:**
- Para partidas de `partidas_terreno`, la zona se resuelve en tiempo de valorización (el modelador eligió la zona al iniciar la obra).
- Si una partida puede tener cantidades distintas según una condición de la obra (ej. nueva vs ampliación), modelar como **dos componentes distintos**. No usar lógica condicional dentro de la cantidad.

---

### 3.9 Hoja `factores`

Cubre las líneas indirectas (1.1, 1.2, 1.3, 1.6, 2.1, 2.3, 2.4). **Soporta dos modos**:

- `modo = pct`: valor en porcentaje aplicado sobre `aplica_sobre`.
- `modo = monto_fijo`: valor absoluto por subtipo de obra. El motor lo suma directamente a la línea correspondiente.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `tipo_obra` | texto | Sí | ∈ `listas.tipo_obra` (4 valores) |
| `subtipo_obra` | texto | No | ∈ `listas.subtipo_obra`. Si vacío, aplica a todos los subtipos del `tipo_obra` |
| `factor_codigo` | texto | Sí | Ver lista cerrada abajo |
| `descripcion` | texto | Sí | |
| `modo` | texto | Sí | ∈ `listas.modo_factor` (`pct` o `monto_fijo`) |
| `valor` | decimal | Sí | Pct ∈ [0, 100] si `modo=pct`; monto ≥ 0 si `modo=monto_fijo` |
| `moneda` | texto | Cond. | Obligatorio si `modo=monto_fijo` |
| `aplica_sobre` | texto | Cond. | Obligatorio si `modo=pct`. ∈ `listas.aplica_sobre` |
| `fecha_cotizacion` | fecha | Cond. | Obligatoria si `modo=monto_fijo` |
| `nota` | texto | No | |

**`factor_codigo` permitidos:**

| Código | Línea de costo | Modo típico |
|---|---|---|
| `1.1_ingenieria` | 1.1 Ingeniería | `monto_fijo` o `pct` |
| `1.2_medioambiental` | 1.2 Gestión medio ambiental | `monto_fijo` o `pct` |
| `1.3_instalacion_faena` | 1.3 Instalación de faena | `monto_fijo` o `pct` |
| `1.6_pruebas_pes` | 1.6 Pruebas y PES | `monto_fijo` o `pct` |
| `admin_supervision` | 1.1 o 1.3 (según política del CEN, definido por `categoria_costo` adicional) | `monto_fijo` |
| `2.1_gastos_generales` | 2.1 Gastos generales y seguros | `pct` |
| `2.1_seguros_financieros` | 2.1 (componente seguros) | `pct` |
| `2.3_utilidades` | 2.3 Utilidades del contratista | `pct` |
| `2.4_contingencias` | 2.4 Contingencias | `pct` |

**`aplica_sobre` permitidos (solo `modo=pct`):**

| Valor | Significado |
|---|---|
| `costo_directo_base` | Σ partidas con categoria_costo 1.4 + 1.5 |
| `costos_directos_total` | Σ (1.1 a 1.6) |
| `monto_contrato` | (1) + (2) |

**Validaciones:**
- `(tipo_obra, subtipo_obra, factor_codigo)` único (`subtipo_obra` vacío cuenta como un valor más).
- Si `modo=pct`, `valor ∈ [0, 100]` y `aplica_sobre` no vacío.
- Si `modo=monto_fijo`, `moneda` y `fecha_cotizacion` no vacíos.

**Ejemplo:**

```
tipo_obra | subtipo_obra        | factor_codigo            | descripcion                | modo         | valor   | moneda | aplica_sobre          | fecha_cotizacion
SE-amp    | amp_barra           | admin_supervision        | Administración             | monto_fijo   | 338222  | USD    |                       | 2026-06-15
SE-amp    | amp_se_ntr          | admin_supervision        | Administración             | monto_fijo   | 676443  | USD    |                       | 2026-06-15
SE-nueva  | nueva_se_zonal      | admin_supervision        | Administración             | monto_fijo   | 1088147 | USD    |                       | 2026-06-15
SE-nueva  | nueva_se_nacional   | admin_supervision        | Administración             | monto_fijo   | 1554496 | USD    |                       | 2026-06-15
SE-nueva  |                     | 2.1_seguros_financieros  | Seguros y financieros      | pct          | 2.0     |        | costos_directos_total |
SE-nueva  | nueva_se_zonal      | 2.1_gastos_generales     | Gastos generales           | pct          | 3.4     |        | costos_directos_total |
SE-nueva  | nueva_se_nacional   | 2.1_gastos_generales     | Gastos generales           | pct          | 3.4     |        | costos_directos_total |
SE-nueva  | nueva_se_zonal      | 2.3_utilidades           | Utilidades del contratista | pct          | 6.0     |        | costos_directos_total |
SE-nueva  | nueva_se_zonal      | 2.4_contingencias        | Contingencias              | pct          | 4.0     |        | costos_directos_total |
```

---

### 3.10 Hoja `parametros`

Costos cuya magnitud **depende del plazo** del proyecto.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `parametro` | texto | Sí | `ito` o `intereses_intercalarios` |
| `variable` | texto | Sí | Ver §3.10.1 |
| `valor` | decimal | Sí | |
| `unidad` | texto | No | Documentación (`USD/mes`, `%/año`) |
| `moneda` | texto | Cond. | Si aplica |
| `fecha_cotizacion` | fecha | Cond. | Si aplica |
| `nota` | texto | No | |

#### 3.10.1 Parámetros esperados

| `parametro` | `variable` | Significado |
|---|---|---|
| `ito` | `costo_mensual` | Costo mensual fijo de ITO (línea 2.2). Se multiplica por el plazo en meses. |
| `intereses_intercalarios` | `tasa_anual_pct` | Tasa anual aplicada para calcular intereses intercalarios sobre `monto_contrato` (línea 4) |
| `intereses_intercalarios` | `factor_promedio_plazo` | Factor (típicamente 0.5) que aproxima el promedio del flujo durante construcción |

La fórmula exacta de los intereses intercalarios se define en `02-backend.md` §4.3.6.

**Ejemplo:**

```
parametro                | variable             | valor    | unidad   | moneda | fecha_cotizacion
ito                      | costo_mensual        |  12000   | USD/mes  | USD    | 2026-06-15
intereses_intercalarios  | tasa_anual_pct       |      6.5 | %/año    |        |
intereses_intercalarios  | factor_promedio_plazo|      0.5 | adim     |        |
```

---

### 3.11 Hoja `reglas_subtipo`

> **Opcional en v1.** Si esta hoja no está, el modelador elige el `subtipo_obra` manualmente en el formulario de antecedentes del proyecto (con default sugerido por heurística en el frontend).

Derivación automática del `subtipo_obra` desde atributos del proyecto. Permite al modelador escoger solo entre 4 `tipo_obra`; el sistema infiere el subtipo para indexar `factores`.

| Columna | Tipo | Obligatoria | Notas |
|---|---|---|---|
| `tipo_obra` | texto | Sí | ∈ `listas.tipo_obra` |
| `subtipo_obra` | texto | Sí | ∈ `listas.subtipo_obra` |
| `condicion` | texto | No | Expresión booleana. Vacía = catch-all |
| `prioridad` | entero | Sí | Evaluación asc.; primera condición que matchea gana |
| `nota` | texto | No | |

**Variables permitidas en `condicion`:**

| Variable | Origen |
|---|---|
| `tiene_transformador` | bool — hay ≥1 instancia tipo `transformador` |
| `tension_max` | número (kV) — máxima tensión de las instalaciones del proyecto |
| `longitud_total_km` | número — suma de longitudes de tramos de línea |
| `tipo_ampliacion_linea` | string — `tendido_segundo_circuito` / `aumento_capacidad` / `extension` (entrada del modelador para L-amp) |

**Operadores permitidos**: `==`, `!=`, `>`, `<`, `>=`, `<=`, `and`, `or`, `not`, `in`, paréntesis. Sin funciones; identificadores cerrados.

**Ejemplo:**

```
tipo_obra | subtipo_obra        | condicion                                          | prioridad
SE-nueva  | nueva_se_nacional   | tension_max >= 220                                 | 1
SE-nueva  | nueva_se_zonal      |                                                    | 2
SE-amp    | amp_se_ntr          | tiene_transformador                                | 1
SE-amp    | amp_barra           |                                                    | 2
L-nueva   | nueva_linea_larga   | longitud_total_km > 200                            | 1
L-nueva   | nueva_linea_corta   |                                                    | 2
L-amp     | tendido_2c          | tipo_ampliacion_linea == 'tendido_segundo_circuito'| 1
L-amp     | aumento_capacidad   |                                                    | 2
```

---

## 4. Reglas de validación al ingestar

El backend del Valorizador ejecuta estas validaciones **antes** de aceptar la publicación de una nueva versión. Si alguna falla, el archivo es **rechazado**: no se publica nada parcial.

### 4.1 Validaciones estructurales

1. Existen las hojas obligatorias (10 ó 11 según si `reglas_subtipo` está habilitada) con los nombres exactos.
2. La fila 1 de cada hoja contiene los encabezados obligatorios.
3. No hay columnas obligatorias vacías en filas con `codigo` no vacío.

### 4.2 Validaciones de unicidad

| Hoja | Llave única |
|---|---|
| `metadata.version` | global (no puede coincidir con versión publicada previa) |
| `listas` | `(categoria, valor)` |
| `zonas` | `codigo` |
| `equipos_genericos` | `equipo_codigo` |
| `partidas` | `codigo` |
| `partidas_terreno` | `(codigo, zona_codigo)` |
| `componentes` | `codigo` |
| `composiciones` | `(componente_codigo, partida_codigo)` |
| `factores` | `(tipo_obra, subtipo_obra, factor_codigo)` |
| `parametros` | `(parametro, variable)` |
| `reglas_subtipo` | `(tipo_obra, subtipo_obra)` |

### 4.3 Validaciones referenciales

- Cualquier valor en columnas que referencian `listas` debe existir en la categoría correspondiente.
- `partidas.equipo_codigo` ∈ `equipos_genericos.equipo_codigo` cuando aplica.
- `componentes.equipo_codigo` ∈ `equipos_genericos.equipo_codigo` cuando aplica.
- `composiciones.componente_codigo` ∈ `componentes.codigo`.
- `composiciones.partida_codigo` ∈ `partidas.codigo` ∪ `partidas_terreno.codigo`.
- `partidas_terreno.zona_codigo` ∈ `zonas.codigo`.
- `factores.tipo_obra` ∈ `listas.tipo_obra`; `subtipo_obra` ∈ `listas.subtipo_obra` cuando no vacío.

### 4.4 Validaciones de dominio

- `valor_unitario ≥ 0` en `partidas`, `partidas_terreno` y `parametros`.
- `valor ∈ [0, 100]` en `factores` cuando `modo=pct`.
- `cantidad` en `composiciones`: parsea como número o expresión válida (§3.8.1).
- Las expresiones de cantidad solo contienen variables permitidas para el `tipo_componente`.
- Para cada `tipo_obra`, deben existir todos los `factor_codigo` requeridos (cobertura mínima): `2.1_gastos_generales`, `2.3_utilidades`, `2.4_contingencias`.
- `parametros` contiene `ito.costo_mensual`, `intereses_intercalarios.tasa_anual_pct` e `intereses_intercalarios.factor_promedio_plazo`.

### 4.5 Reportes del ingestor

Al cargar el archivo, el sistema devuelve:

1. **Resumen de cambios** vs versión vigente: partidas añadidas/eliminadas/modificadas; componentes añadidos/eliminados; cambios de PU con `delta_pct`.
2. **Errores bloqueantes**: lista detallada de violaciones (hoja, fila, columna, motivo).
3. **Advertencias no bloqueantes**: ej. componentes huérfanos (sin composiciones), partidas no referenciadas, factores con valores muy distintos al promedio.

El curador revisa el resumen y confirma la publicación. Una vez publicada, la versión queda **inmutable**.

---

## 5. Reglas de versionado

- **Formato de `version`**: recomendado `AAAA.MM.N` (ej. `2026.06.1`, `2026.06.2`, `2026.07.1`). Cualquier texto único es aceptado, pero se sugiere este formato para legibilidad cronológica.
- **Inmutabilidad**: una vez publicada, una versión **no se modifica**. Cualquier corrección requiere una nueva versión con cambio en `metadata.version`.
- **Anclaje por proyecto**: cada proyecto registra la `catalogo_version_id` con la que fue valorizado. Una nueva versión publicada **no recalcula** proyectos existentes automáticamente; el sistema **propone migrar** y el modelador decide.
- **Borradores en el sistema**: el consultor puede subir un archivo en modo **borrador** para ver el diff sin publicar. Solo el curador del CEN, después de revisar, publica.

---

## 6. Convenciones de nomenclatura de códigos

Para facilitar lectura y diff entre versiones, el consultor sigue estas convenciones. El ingestor las prefiere para mensajes de error más claros, pero no las impone.

| Prefijo | Categoría |
|---|---|
| `Z-` | Zona |
| `EQ-` | Equipo genérico (`equipos_genericos`) y partidas tipo `equipo` |
| `MON-` | Partida tipo `montaje` |
| `MAT-` | Partida tipo `material` |
| `OC-` | Partida tipo `obra_civil` |
| `CPM-` | Partida tipo `control_proteccion` |
| `EST-` | Partida tipo `estructura` |
| `COND-` | Partida tipo `material` para conductores eléctricos |
| `T-` | Partida de terreno (zona-dependiente) |
| `C-BR-` | Componente: Barra |
| `C-PA-` | Componente: Paño |
| `C-VANO-` | Componente: Vano de línea |
| `C-LIN-` | Componente: Línea (agrega vanos) |
| `C-TRF-` | Componente: Transformador |
| `C-CMP-` | Componente: Compensación |
| `C-FUN-` | Componente: Fundación |
| `C-EST-` | Componente: Estructura |
| `C-GIS-` | Componente: Módulo GIS / Sala de celdas |
| `C-SSAA-` | Componente: Servicios auxiliares |
| `C-OOCC-` | Componente: Obras civiles agregadoras |

Los códigos pueden contener atributos para legibilidad: `C-PA-AIS-IM-DC-TV-220` = Componente / Paño / AIS / Interruptor y Medio / Diagonal Completa / Tanque Vivo / 220 kV.

---

## 7. Checklist de entrega para el consultor

Antes de entregar el archivo al CEN, verificar:

- [ ] Las 10 (u 11) hojas obligatorias están presentes con los nombres exactos en minúsculas.
- [ ] `metadata.version` es nueva (no coincide con versiones previas).
- [ ] `listas` está poblada con **todos** los valores que aparecen en otras hojas.
- [ ] Todos los strings de tecnología, configuración, interruptor, etc., usan el spelling canónico de `listas` (sin mayúsculas heterogéneas, sin "L. Tanque vivo" vs "Tanque vivo" vs "INTERRUPTOR_TANQUE_VIVO").
- [ ] `equipos_genericos` está poblado y todas las referencias `equipo_codigo` en `partidas` y `componentes` apuntan a una fila válida.
- [ ] Cada `zonas.codigo` referenciado en `partidas_terreno` existe.
- [ ] Cada `componentes.codigo` tiene **al menos una** fila en `composiciones`.
- [ ] Cada `tipo_obra` tiene los factores obligatorios en `factores` (`2.1_gastos_generales`, `2.3_utilidades`, `2.4_contingencias` como mínimo).
- [ ] `parametros` contiene las 3 filas obligatorias (`ito.costo_mensual`, `intereses_intercalarios.tasa_anual_pct`, `intereses_intercalarios.factor_promedio_plazo`).
- [ ] Cada partida tiene `moneda` y `fecha_cotizacion`. La `moneda_referencia` de `metadata` es coherente con los tipos de cambio declarados.
- [ ] Decimales con separador consistente (coma o punto) en todo el archivo.
- [ ] Tensiones siempre como string (`'220'`, no `220`).
- [ ] Sin filas con `codigo` vacío que tengan otros valores (eso indica error de copia/pega).

---

## 8. Diferencias respecto al esquema actual del consultor (rev. 0)

El archivo `Esquema base de datos.xlsx` que el consultor está construyendo (rev. 0) tiene 20 hojas con naming heterogéneo. Esta sección documenta el mapeo a las 11 hojas canónicas:

| Hoja rev. 0 | Destino canónico |
|---|---|
| `README` | Eliminado. Una nota corta puede ir en `metadata.nota` |
| `dim_posiciones`, `dim_trafo`, `dim_otros` | Postergadas. Modelar en `componentes` cuando se tengan datos |
| `cu_equipos_ais`, `cu_transformadores` (filas equipo), `cu_sala_gis` (filas equipo), `cu_ssaa_oocc` (filas equipo), `cu_materiales`, `precio_cap_conductor` | `partidas` con `tipo_costo='equipo'` o `material` |
| `montaje_equipos_ais` + filas `montaje` de `cu_transformadores` y `cu_ssaa_oocc` | `partidas` con `tipo_costo='montaje'` |
| `cu_ssaa_oocc` filas `obra_civil` + `fundacion_estructura` | `partidas` con `tipo_costo='obra_civil'` o `material`, y composiciones que las usan |
| `cu_sala_gis` filas `control` | `partidas` con `tipo_costo='control_proteccion'` |
| `matriz_panos_config` | `componentes` (un código por combinación) + `composiciones` |
| `fundacion_equipo` | `componentes` tipo `fundacion` + `composiciones` (m3 hormigón, kg enfierradura, un pernos) |
| `peso_estructura_se`, `peso_estructura_linea` | `componentes` tipo `estructura` + `composiciones` (kg acero galvanizado) |
| `vano_serv_lineas` | Parámetros embebidos en `componentes` tipo `vano` (longitud típica, servidumbre típica) |
| `precios_zona` | `partidas_terreno` en formato long |
| `costos_generales` (absolutos) + `costos_generales _factores` (%) | `factores` (con `modo` apropiado) y `parametros` para ITO |

---

## 9. Versionado de este documento

Este documento (la **spec del template**) también se versiona. El consultor debe trabajar siempre contra la última versión publicada de la spec, que el CEN comunica explícitamente.

| Versión spec | Fecha | Cambios |
|---|---|---|
| 0.1 | (inicial) | 9 hojas, ingesta determinística |
| 0.2 | 2026-06-22 | 11 hojas, vocabulario controlado en `listas`, `equipos_genericos`, fundaciones/estructuras como componentes, `factores` dual-modo (`pct`/`monto_fijo`), `fecha_cotizacion` en partidas, `subtipo_obra` y `reglas_subtipo` |
