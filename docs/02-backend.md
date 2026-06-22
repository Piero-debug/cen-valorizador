# 02 — Backend

> Especificación del backend del Valorizador CEN. Cubre arquitectura modular, modelo de datos, motor de cálculo determinístico, ingesta del catálogo, contratos de API y generación de outputs.
>
> Asume conocimiento de `01-contexto-y-dominio.md` (modelo conceptual) y `05-template-excel-catalogo.md` (formato del Excel del consultor).

---

## 1. Visión general

### 1.1 Principios

1. **Motor de cálculo aislado**: el núcleo (módulo `valorizador.motor`) no depende de FastAPI, SQLAlchemy ni del sistema de archivos. Recibe estructuras Pydantic y devuelve estructuras Pydantic. Esto permite testearlo con fixtures puros y reutilizarlo desde notebooks, scripts CLI o un futuro worker.
2. **Determinismo**: dado el mismo `(modelo_obra, catalogo_version_id, factores_efectivos)`, el motor produce **exactamente** el mismo resultado. Cero aleatoriedad, cero dependencias temporales en el cálculo.
3. **Inmutabilidad del catálogo**: una vez publicada una versión, sus filas no se modifican. La capa de persistencia hace cumplir esto con constraints + lógica de aplicación.
4. **Trazabilidad**: cada snapshot valorizado guarda inputs, versión del catálogo y resultado intermedio detallado para auditar.

### 1.2 Capas

```
┌──────────────────────────────────────────────────┐
│  FastAPI (HTTP)                                  │  ← api/
│  - routers, schemas de request/response          │
│  - dependency injection (sesión DB, auth simul.) │
└────────────────────┬─────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────┐
│  Servicios de aplicación                         │  ← valorizador/servicios/
│  - orquestan motor + persistencia                │
│  - gestionan ciclo de vida del proyecto          │
└────────┬──────────────────────────────┬──────────┘
         │                              │
┌────────▼──────────┐         ┌─────────▼──────────┐
│ Dominio + Motor   │         │ Persistencia       │
│ (puros)           │         │ (SQLAlchemy)       │
│ - Pydantic models │         │ - modelos ORM      │
│ - reglas de cálc. │         │ - repositorios     │
│ - validación      │         │ - Alembic migrate  │
└───────────────────┘         └────────────────────┘
         ▲                              │
         │                              │
┌────────┴──────────┐                   │
│ Catálogo (ingesta)│                   │
│ - parser Excel    │                   │
│ - validadores     │                   │
│ - publicación     │                   │
└───────────────────┘                   │
                                        ▼
                                  ┌──────────┐
                                  │ SQLite   │
                                  └──────────┘
```

---

## 2. Estructura de módulos

```
backend/
├── pyproject.toml
├── alembic.ini
├── alembic/
│   └── versions/
├── api/                          # FastAPI app
│   ├── main.py
│   ├── routers/
│   │   ├── proyectos.py
│   │   ├── catalogo.py
│   │   ├── valorizacion.py
│   │   └── exports.py
│   ├── schemas/                  # Pydantic de request/response (HTTP)
│   └── deps.py                   # dependency injection
├── valorizador/                  # núcleo del dominio
│   ├── __init__.py
│   ├── dominio/                  # entidades Pydantic (modelo conceptual)
│   │   ├── obra.py
│   │   ├── instalacion.py
│   │   ├── componente.py
│   │   ├── partida.py
│   │   └── enums.py
│   ├── motor/                    # cálculo determinístico
│   │   ├── resolver_cantidades.py
│   │   ├── agregador_costos.py
│   │   ├── factores.py
│   │   └── plazo.py              # ITO + intereses intercalarios
│   ├── catalogo/                 # ingesta y versionado
│   │   ├── lector_excel.py
│   │   ├── validadores.py
│   │   ├── publicador.py
│   │   └── diff.py
│   ├── persistencia/             # SQLAlchemy
│   │   ├── modelos.py
│   │   ├── repositorios.py
│   │   └── sesion.py
│   ├── servicios/                # orquestación
│   │   ├── servicio_proyecto.py
│   │   ├── servicio_catalogo.py
│   │   └── servicio_exports.py
│   └── exports/                  # generadores de Excel y PDF
│       ├── memoria_excel.py
│       └── reporte_pdf.py
└── tests/
    ├── motor/                    # tests de fórmulas (regresión)
    ├── catalogo/                 # tests de ingesta y diff
    └── api/                      # tests end-to-end de endpoints
```

---

## 3. Modelo de dominio (Pydantic)

Los modelos Pydantic son el contrato interno entre capas. La capa de persistencia y la de API tienen sus propios modelos espejo.

### 3.1 Enums centrales

```python
from enum import Enum

class TipoObra(str, Enum):
    SE_NUEVA = "SE-nueva"
    SE_AMP = "SE-amp"
    L_NUEVA = "L-nueva"
    L_AMP = "L-amp"

class SubtipoObra(str, Enum):
    NUEVA_SE_ZONAL = "nueva_se_zonal"
    NUEVA_SE_NACIONAL = "nueva_se_nacional"
    AMP_BARRA = "amp_barra"
    AMP_SE_NTR = "amp_se_ntr"
    NUEVA_LINEA_CORTA = "nueva_linea_corta"
    NUEVA_LINEA_LARGA = "nueva_linea_larga"
    TENDIDO_2C = "tendido_2c"
    AUMENTO_CAPACIDAD = "aumento_capacidad"

class TipoComponente(str, Enum):
    BARRA_AT = "barra_at"
    BARRA_MT = "barra_mt"
    PANO = "pano"           # incluye estandar, diagonal_completa, media_diagonal (via tipo_pano)
    LINEA = "linea"
    VANO = "vano"
    TRANSFORMADOR = "transformador"
    COMPENSACION = "compensacion"
    FUNDACION = "fundacion"
    ESTRUCTURA = "estructura"
    SALA_CELDAS = "sala_celdas"
    SSAA = "ssaa"
    OOCC = "oocc"

class ModoFactor(str, Enum):
    PCT = "pct"
    MONTO_FIJO = "monto_fijo"

class EstadoProyecto(str, Enum):
    BORRADOR = "borrador"
    EN_REVISION = "en_revision"
    VALORIZADO = "valorizado"
    ARCHIVADO = "archivado"

class TecnologiaSE(str, Enum):
    AIS = "AIS"
    GIS = "GIS"
    HIS = "HIS"

class CategoriaCosto(str, Enum):
    INGENIERIA = "1.1_ingenieria"
    MEDIOAMBIENTAL = "1.2_medioambiental"
    INSTALACION_FAENA = "1.3_instalacion_faena"
    SUMINISTROS_OC_MONTAJE = "1.4_suministros_oc_montaje"
    SERVIDUMBRES_TERRENOS = "1.5_servidumbres_terrenos"
    PRUEBAS_PES = "1.6_pruebas_pes"
```

### 3.2 Modelo del proyecto (lado modelador)

```python
from pydantic import BaseModel, Field
from typing import Optional
from uuid import UUID

class AntecedentesGenerales(BaseModel):
    tipo_obra: TipoObra
    subtipo_obra: Optional[SubtipoObra] = None  # opcional: derivado por reglas_subtipo o elegido por el modelador
    zona_codigo: str
    plazo_meses: int = Field(gt=0)
    nombre_obra: str
    numero_decreto: Optional[str] = None
    tipo_ampliacion_linea: Optional[str] = None  # input para derivar subtipo en L-amp
    nota: Optional[str] = None

class InstanciaComponente(BaseModel):
    """Una instancia concreta en el grafo de la obra."""
    id_nodo: UUID                       # id en el canvas
    codigo_componente: str              # referencia al catálogo
    parametros: dict[str, float] = {}   # longitud_km, capacidad_mva, etc.
    posicion_canvas: tuple[float, float] = (0, 0)  # coord para la UI

class Conexion(BaseModel):
    id_origen: UUID
    id_destino: UUID
    nota: Optional[str] = None

class PartidaAdHoc(BaseModel):
    descripcion: str
    unidad: str
    cantidad: float
    precio_unitario: float
    moneda: str
    categoria_costo: CategoriaCosto
    justificacion: str  # obligatoria

class ProyectoValorizable(BaseModel):
    """Input completo al motor de cálculo."""
    proyecto_id: UUID
    catalogo_version_id: UUID
    antecedentes: AntecedentesGenerales
    instancias: list[InstanciaComponente]
    conexiones: list[Conexion]
    factores_override: dict[str, float] = {}    # factor_codigo → valor_pct override
    partidas_ad_hoc: list[PartidaAdHoc] = []
```

### 3.3 Modelo del catálogo (lado curador)

Refleja 1-a-1 las hojas del Excel (ver `05-template-excel-catalogo.md` §3).

```python
class CatalogoVersion(BaseModel):
    id: UUID
    version: str
    fecha_publicacion: date
    fecha_vigencia_desde: date
    moneda_referencia: str
    tipo_cambio_clp_usd: Optional[float] = None
    tipo_cambio_uf_clp: Optional[float] = None
    autor_consultor: str
    nota: Optional[str] = None

class EquipoGenerico(BaseModel):
    equipo_codigo: str
    tipo_equipo: str
    descripcion: str

class Partida(BaseModel):
    codigo: str
    descripcion: str
    unidad: str
    tipo_costo: str          # equipo | montaje | obra_civil | material | control_proteccion | estructura | terreno | servicio
    categoria_costo: CategoriaCosto
    equipo_codigo: Optional[str] = None
    tension: Optional[str] = None
    configuracion_equipo: Optional[str] = None
    capacidad_nominal: Optional[float] = None
    valor_unitario: float
    moneda: str
    fecha_cotizacion: date

class PartidaTerreno(BaseModel):
    codigo: str
    descripcion: str
    unidad: str
    zona_codigo: str
    valor_unitario: float
    moneda: str
    fecha_cotizacion: date

class Componente(BaseModel):
    codigo: str
    tipo: TipoComponente
    descripcion: str
    tension: Optional[str] = None
    tecnologia: Optional[str] = None
    configuracion_barra: Optional[str] = None
    tipo_interruptor: Optional[str] = None
    tipo_pano: Optional[str] = None
    equipo_codigo: Optional[str] = None      # para fundacion y estructura
    tipo_fundacion: Optional[str] = None     # para fundacion
    tipo_estructura: Optional[str] = None    # para estructura
    capacidad_nominal: Optional[float] = None
    tipo_torre: Optional[str] = None
    tipo_conductor: Optional[str] = None
    n_conductores_fase: Optional[int] = None
    tipo_compensacion: Optional[str] = None

class FilaComposicion(BaseModel):
    componente_codigo: str
    partida_codigo: str
    cantidad_expr: str  # constante o expresión paramétrica

class Factor(BaseModel):
    tipo_obra: TipoObra
    subtipo_obra: Optional[SubtipoObra] = None  # vacío = aplica a todos los subtipos del tipo_obra
    factor_codigo: str
    descripcion: str
    modo: ModoFactor                            # pct | monto_fijo
    valor: float                                 # pct ∈ [0,100] o monto absoluto en moneda
    moneda: Optional[str] = None                 # obligatorio si modo=monto_fijo
    aplica_sobre: Optional[str] = None           # obligatorio si modo=pct
    fecha_cotizacion: Optional[date] = None      # obligatoria si modo=monto_fijo

class ReglaSubtipo(BaseModel):
    tipo_obra: TipoObra
    subtipo_obra: SubtipoObra
    condicion: Optional[str] = None  # expresión booleana; vacía = catch-all
    prioridad: int

class ParametroPlazo(BaseModel):
    parametro: str   # "ito" | "intereses_intercalarios"
    variable: str
    valor: float
    unidad: Optional[str] = None
```

### 3.4 Modelo del resultado de valorización

```python
class CostoPartida(BaseModel):
    partida_codigo: str
    descripcion: str
    unidad: str
    cantidad: float
    precio_unitario: float
    subtotal: float
    moneda: str
    componente_id: Optional[UUID] = None  # de qué instancia salió
    categoria_costo: CategoriaCosto
    es_ad_hoc: bool = False

class SubtotalCategoria(BaseModel):
    categoria: str       # "1.1_ingenieria", etc.
    subtotal: float

class ResultadoValorizacion(BaseModel):
    proyecto_id: UUID
    catalogo_version_id: UUID
    fecha_calculo: datetime
    moneda: str

    partidas_resueltas: list[CostoPartida]
    factores_efectivos: dict[str, float]

    # estructura canónica (ver 01-contexto-y-dominio §6)
    costos_directos: dict[str, float]    # por categoría 1.1 a 1.6
    total_costos_directos: float

    costos_indirectos: dict[str, float]  # por categoría 2.1 a 2.4
    total_costos_indirectos: float

    monto_contrato: float
    intereses_intercalarios: float
    costo_total_proyecto: float
```

---

## 4. Motor de cálculo

> El módulo `valorizador.motor` recibe un `ProyectoValorizable` + el `CatalogoVersion` cargado en memoria, y devuelve un `ResultadoValorizacion`. Sin efectos secundarios.

### 4.1 Función principal

```python
def calcular(
    proyecto: ProyectoValorizable,
    catalogo: CatalogoSnapshot,
) -> ResultadoValorizacion:
    ...
```

Donde `CatalogoSnapshot` agrupa en memoria todas las entidades del catálogo de la versión anclada (partidas, partidas de terreno, componentes, composiciones, factores, parámetros).

### 4.2 Pipeline interno

```
1. resolver_subtipo(proyecto, catalogo.reglas_subtipo)
       → si antecedentes.subtipo_obra está vacío, lo deriva evaluando
         las reglas en orden de prioridad contra atributos del proyecto
         (tiene_transformador, tension_max, longitud_total_km,
         tipo_ampliacion_linea). Si no hay reglas, requiere que el
         modelador lo haya elegido (selector manual del frontend).
2. resolver_cantidades(proyecto, catalogo)
       → expande cada InstanciaComponente en una lista de CostoPartida
       (evalúa expresiones paramétricas, resuelve PU de terreno por zona)
3. agregar_partidas_ad_hoc(proyecto.partidas_ad_hoc)
4. agrupar_por_categoria_costo()
       → produce costos_directos[1.1 .. 1.6]
5. aplicar_factores_directos(catalogo.factores, override, subtipo_obra)
       → para cada factor con factor_codigo ∈ {1.1, 1.2, 1.3, 1.6}:
            * filtra por (tipo_obra, subtipo_obra) (con subtipo vacío como wildcard)
            * si modo=pct:     valor / 100 × aplica_sobre
            * si modo=monto_fijo: suma directa (en moneda referencia)
6. total_costos_directos = Σ(1.1..1.6)
7. aplicar_factores_indirectos(catalogo.factores, override, subtipo_obra, total_directos)
       → 2.1, 2.3, 2.4 con misma lógica dual modo
8. calcular_ito(plazo_meses, catalogo.parametros)
       → 2.2 = ito.costo_mensual × plazo_meses
9. monto_contrato = total_costos_directos + total_costos_indirectos
10. calcular_intereses_intercalarios(monto_contrato, plazo, catalogo.parametros)
11. costo_total = monto_contrato + intereses_intercalarios
```

### 4.3 Fórmulas explícitas

#### 4.3.1 Resolución de cantidades

Para cada `InstanciaComponente`:
- Busca el `Componente` en el catálogo por `codigo_componente`.
- Para cada fila de `Composiciones` asociada al componente:
  - Evalúa `cantidad_expr` con el contexto de `instancia.parametros` (longitud_km, capacidad_mva, etc.).
  - Resuelve PU:
    - Si `partida_codigo` está en `Partidas`: usa el PU directamente.
    - Si está en `Partidas_Terreno`: busca por `(partida_codigo, antecedentes.zona_codigo)`.
  - `subtotal = cantidad_resuelta × precio_unitario_en_moneda_referencia`.

#### 4.3.2 Estructura de costos directos

Los factores soportan dos modos (`pct` y `monto_fijo`). El motor selecciona la fila aplicable por `(tipo_obra, subtipo_obra, factor_codigo)` — una fila con `subtipo_obra` vacío actúa como wildcard (aplica a todos los subtipos del `tipo_obra`).

```
costo_directo_base = Σ partidas con categoria_costo = "1.4_suministros_oc_montaje"
                   + Σ partidas con categoria_costo = "1.5_servidumbres_terrenos"
                   (más partidas ad-hoc en estas categorías)

aplicar_factor(f, base) =
    si f.modo == "pct":          f.valor / 100 × base
    si f.modo == "monto_fijo":   convertir(f.valor, f.moneda → moneda_ref)

Para cada factor_codigo en {1.1, 1.2, 1.3, 1.6, admin_supervision}:
    f = factores[(tipo_obra, subtipo_obra, factor_codigo)]
    costo_categoria += aplicar_factor(f, costo_directo_base)
    (si hay partidas directas con esa categoría, se suman)

costos_directos["1.4"] = Σ partidas categoria_costo "1.4"
costos_directos["1.5"] = Σ partidas categoria_costo "1.5"
costos_directos["1.1"] = Σ aplicar_factor(f, costo_directo_base) para factores con categoria 1.1
                         + Σ partidas directas en 1.1
costos_directos["1.2"] = análogo
costos_directos["1.3"] = análogo
costos_directos["1.6"] = análogo

total_costos_directos = Σ costos_directos[1.1..1.6]
```

> **Nota**: las partidas pueden venir como directas (en `partidas` con `categoria_costo` = 1.4 o 1.5) o como factores. El consultor decide qué modelar como uno u otro. Si una partida está como directa **y** existe un factor para la misma categoría, **se suman**. La columna `aplica_sobre` no se usa cuando `modo=monto_fijo` (el monto es absoluto en moneda).

#### 4.3.3 Costos indirectos

```
Para cada factor_codigo en {2.1_gastos_generales, 2.1_seguros_financieros, 2.3_utilidades, 2.4_contingencias}:
    f = factores[(tipo_obra, subtipo_obra, factor_codigo)]
    base = total_costos_directos  (o el aplica_sobre declarado en la fila)
    costos_indirectos[linea] += aplicar_factor(f, base)
```

#### 4.3.4 ITO (2.2)

```
ito_mensual = catalogo.parametros["ito"]["costo_mensual"]   # en moneda referencia
costos_indirectos["2.2"] = ito_mensual × plazo_meses

total_costos_indirectos = Σ costos_indirectos[2.1..2.4]
```

#### 4.3.5 Monto contrato

```
monto_contrato = total_costos_directos + total_costos_indirectos
```

#### 4.3.6 Intereses intercalarios (4)

```
tasa_anual = catalogo.parametros["intereses_intercalarios"]["tasa_anual_pct"] / 100
factor_promedio = catalogo.parametros["intereses_intercalarios"]["factor_promedio_plazo"]  # ej. 0.5
plazo_anios = plazo_meses / 12

intereses_intercalarios = monto_contrato × tasa_anual × plazo_anios × factor_promedio
```

#### 4.3.7 Costo total

```
costo_total_proyecto = monto_contrato + intereses_intercalarios
```

### 4.4 Resolución de la expresión `cantidad_expr`

`cantidad_expr` es una cadena que el consultor define en el Excel (ver `05-template-excel-catalogo.md` §3.7.1). El motor la evalúa con un parser restringido:

- Operadores permitidos: `+ - * / ( )`.
- Identificadores permitidos: `longitud_km`, `n_circuitos`, `capacidad_mva`, `capacidad_mvar`, `n_posiciones`.
- Solo identificadores válidos para el `tipo` del componente.
- Implementación: `ast.parse(expr, mode="eval")` + walker que solo acepta `BinOp`, `Num`, `Name` (filtrado), `UnaryOp`, `Constant`.

Si la expresión es solo un número, se evalúa como constante. Cualquier identificador o llamada no permitida lanza `ExpresionCantidadInvalida` en ingesta (no en cálculo: las expresiones se validan al publicar el catálogo).

### 4.5 Reglas de conversión de moneda

- Todos los cálculos internos se hacen en la `moneda_referencia` del catálogo (típicamente USD).
- PU en otra moneda (CLP, UF) se convierten al ingestar usando `metadata.tipo_cambio_*`.
- Si `tipo_cambio_*` no está y se requiere conversión, el ingestor **rechaza** el catálogo.
- Cada partida lleva su propia `fecha_cotizacion`. El ingestor normaliza a `metadata.fecha_cotizacion_referencia` si fuera necesario (en v1 no hay reajuste por fecha; se documenta la fecha pero se asume cotización homogénea dentro de un catálogo).
- Los outputs se generan en la moneda de referencia. Conversión a CLP para reportes ejecutivos es responsabilidad de la capa de export, no del motor.

### 4.6 Manejo de partidas ad-hoc

- Se suman a las categorías de costo declaradas por el modelador.
- Aparecen marcadas con `es_ad_hoc = true` en `CostoPartida`.
- En la memoria de cálculo Excel se listan en una sección aparte para auditoría.

### 4.7 Determinismo y reproducibilidad

- El motor no debe leer de la BD ni del filesystem durante el cálculo. Recibe todo en `ProyectoValorizable` + `CatalogoSnapshot`.
- Orden de iteración estable: los items del resultado se ordenan por `(codigo_componente, partida_codigo)` para que un diff binario sea posible.
- Sin `random`, sin `datetime.now()` dentro del cálculo (la `fecha_calculo` se inyecta como parámetro).

---

## 5. Ingesta del catálogo desde Excel

> El módulo `valorizador.catalogo` toma un archivo `.xlsx` que cumple `05-template-excel-catalogo.md` y produce un `CatalogoSnapshot` listo para persistir y publicar.

### 5.1 Pipeline

```
1. lector_excel.cargar(path: Path) -> CatalogoCrudo
       lee con openpyxl las 11 hojas (metadata, listas, zonas, equipos_genericos,
       partidas, partidas_terreno, componentes, composiciones, factores,
       parametros, reglas_subtipo); parsea cada hoja a list[dict] sin validar
2. validadores.validar_estructural(crudo) -> list[ErrorEstructural]
       valida nombres de hojas, encabezados, tipos básicos, snake_case sin tildes
3. validadores.validar_vocabulario(crudo) -> list[ErrorVocabulario]
       cada string en columnas controladas debe existir en listas; cualquier
       desviación de spelling (mayúsculas, espacios) es error bloqueante
4. validadores.validar_dominio(crudo) -> list[ErrorDominio]
       unicidad, integridad referencial, expresiones, modo_factor vs columnas
       condicionales (monto_fijo ⇒ moneda+fecha; pct ⇒ aplica_sobre)
5. validadores.validar_cobertura(crudo) -> list[ErrorCobertura]
       cada tipo_obra tiene los factores indirectos obligatorios; cada componente
       tiene composición; equipos_genericos cubre las referencias de partidas
       y componentes; parametros tiene ito + intereses_intercalarios completos
6. diff.calcular(crudo, version_vigente) -> Diff
       muestra cambios vs versión vigente (para mostrar al curador)
7. publicador.publicar(crudo, decision_usuario) -> CatalogoVersion
       persiste atómicamente (transacción) con id nuevo
```

### 5.2 Errores bloqueantes vs advertencias

- **Bloqueantes** (impiden publicar): hojas faltantes, columnas obligatorias ausentes, llaves duplicadas, referencias rotas (incluye strings que no existan en `listas`), `equipo_codigo` referenciado pero no presente en `equipos_genericos`, expresiones de cantidad inválidas, `valor` fuera de rango, factor obligatorio ausente para algún `tipo_obra`, factor con `modo=monto_fijo` sin `moneda` o `fecha_cotizacion`, factor con `modo=pct` sin `aplica_sobre`, partida sin `fecha_cotizacion`.
- **Advertencias** (se muestran, no bloquean): componentes huérfanos (sin composiciones), partidas no referenciadas por ningún componente, cambios de PU > 30% vs versión vigente, factores con valores muy distintos al promedio del catálogo, fechas de cotización con dispersión > 12 meses dentro de la misma versión.

### 5.3 Diff vs versión vigente

El diff se calcula a nivel de fila por hoja, comparando llaves primarias:

```python
class DiffHoja(BaseModel):
    hoja: str
    nuevas: list[dict]
    eliminadas: list[dict]
    modificadas: list[CambioFila]

class CambioFila(BaseModel):
    llave: dict
    campos_modificados: dict[str, tuple[Any, Any]]  # nombre → (antes, después)
```

Se presenta al curador antes de publicar. El curador confirma o rechaza.

### 5.4 Atomicidad de la publicación

La publicación es transaccional:
- Se inserta `CatalogoVersion` con todos los hijos en una sola transacción.
- Si falla la inserción de cualquier fila, rollback completo.
- Una vez `commit`, la versión queda inmutable (constraint `is_inmutable = true` que la app respeta).

---

## 6. Persistencia (SQLAlchemy + SQLite)

### 6.1 Tablas principales

| Tabla | Notas |
|---|---|
| `catalogo_version` | Header de versión. `is_inmutable` se setea al publicar. |
| `catalogo_lista` | FK a `catalogo_version`. Vocabulario controlado. |
| `catalogo_zona` | FK a `catalogo_version`. |
| `catalogo_equipo_generico` | FK a `catalogo_version`. Diccionario canónico de equipos. |
| `catalogo_partida` | FK a `catalogo_version`. |
| `catalogo_partida_terreno` | FK a `catalogo_version`. |
| `catalogo_componente` | FK a `catalogo_version`. |
| `catalogo_composicion` | FK a `catalogo_version`. |
| `catalogo_factor` | FK a `catalogo_version`. Soporta `modo ∈ {pct, monto_fijo}`. |
| `catalogo_parametro` | FK a `catalogo_version`. |
| `catalogo_regla_subtipo` | FK a `catalogo_version`. Opcional. |
| `proyecto` | Header con `estado`, `catalogo_version_id` anclado. |
| `proyecto_instancia` | Instancias de componente en el grafo. |
| `proyecto_conexion` | Conexiones del grafo. |
| `proyecto_partida_ad_hoc` | Partidas no tipificadas. |
| `proyecto_factor_override` | Overrides de factores por proyecto. |
| `valorizacion_snapshot` | Resultado inmutable al estado "valorizado". |

### 6.2 Reglas de integridad

- `proyecto.catalogo_version_id` apunta a una `catalogo_version` con `is_inmutable = true`.
- En estado `valorizado`, todas las tablas hijas del proyecto son **read-only** (validación de aplicación).
- Cambios en `proyecto` mientras está en estado `borrador` o `en_revision` no requieren snapshot.

### 6.3 Migraciones

- **Alembic** maneja el esquema. La primera migración crea todas las tablas.
- Migraciones idempotentes y reversibles cuando es posible.
- Comando: `alembic upgrade head` al iniciar la app si la BD no está al día.

---

## 7. API REST

> Estilo: REST con sustantivos en plural, snake_case en JSON, errores en formato Problem Details (`application/problem+json`).
>
> Prefijo: `/api/v1`. Documentación auto-generada en `/docs` (Swagger UI).
>
> Auth: ninguna en prototipo. Header `X-Rol-Simulado: curador_catalogo | modelador` para simular roles.

### 7.1 Catálogo

#### `GET /api/v1/catalogo/versiones`
Lista versiones publicadas, ordenadas desc por `fecha_vigencia_desde`.

**Response 200:**
```json
{
  "items": [
    {
      "id": "uuid",
      "version": "2026.06.1",
      "fecha_publicacion": "2026-06-15",
      "fecha_vigencia_desde": "2026-07-01",
      "moneda_referencia": "USD",
      "autor_consultor": "XYZ Consultores",
      "es_vigente": true
    }
  ]
}
```

#### `POST /api/v1/catalogo/borradores`
Sube un archivo Excel en modo borrador (no publica). Devuelve diff + errores.

**Request**: `multipart/form-data` con archivo `.xlsx`.

**Response 200:**
```json
{
  "borrador_id": "uuid",
  "metadata": { "version": "2026.06.2", ... },
  "errores_bloqueantes": [],
  "advertencias": [
    {"hoja": "Componentes", "fila": 12, "tipo": "huerfano", "detalle": "..."}
  ],
  "diff": {
    "Partidas": { "nuevas": [...], "modificadas": [...], "eliminadas": [...] },
    "Componentes": { ... }
  }
}
```

**Response 422** (errores bloqueantes): mismo body con lista de errores.

#### `POST /api/v1/catalogo/borradores/{borrador_id}/publicar`
Publica el borrador como versión inmutable. Requiere rol curador.

**Response 201:**
```json
{
  "id": "uuid",
  "version": "2026.06.2",
  "fecha_publicacion": "2026-06-18"
}
```

#### `GET /api/v1/catalogo/versiones/{id}`
Devuelve el contenido completo de una versión (para hidratar el frontend del editor o exportar).

### 7.2 Proyectos

#### `GET /api/v1/proyectos`
Lista proyectos con filtros opcionales (`?estado=borrador`, `?tipo_obra=SE-nueva`).

#### `POST /api/v1/proyectos`
Crea un proyecto nuevo en estado `borrador`.

**Request:**
```json
{
  "nombre_obra": "Nueva SE Polpaico 220 kV",
  "numero_decreto": "DS 123/2026",
  "antecedentes": {
    "tipo_obra": "SE-nueva",
    "zona_codigo": "Z-CENTR",
    "plazo_meses": 24
  },
  "catalogo_version_id": "uuid"
}
```

**Response 201:** el proyecto creado.

#### `GET /api/v1/proyectos/{id}`
Detalle del proyecto: antecedentes, instancias, conexiones, factores override, partidas ad-hoc.

#### `PUT /api/v1/proyectos/{id}`
Actualiza antecedentes y estructura del grafo. Solo en estado `borrador`.

**Request**: cuerpo completo del modelo `ProyectoValorizable` (sin `catalogo_version_id`, que es inmutable).

#### `POST /api/v1/proyectos/{id}/transiciones`
Cambia el estado del proyecto.

**Request:**
```json
{ "nuevo_estado": "en_revision" }
```

Transiciones permitidas (validadas en backend):
- `borrador → en_revision`
- `en_revision → borrador`
- `en_revision → valorizado` (dispara snapshot)
- `valorizado → archivado`
- `archivado → borrador` (genera variante automáticamente; ver `/clonar`)

#### `POST /api/v1/proyectos/{id}/clonar`
Clona el proyecto en estado `borrador`.

**Response 201:** el nuevo proyecto con nuevo `id`.

### 7.3 Valorización (cálculo)

#### `POST /api/v1/proyectos/{id}/valorizacion`
Ejecuta el cálculo y devuelve el resultado **sin persistir snapshot** (preview en vivo).

**Response 200:**
```json
{
  "resultado": {
    "moneda": "USD",
    "costos_directos": {"1.1_ingenieria": 12345, ...},
    "total_costos_directos": 123456,
    "costos_indirectos": {"2.1_gastos_generales": 12345, ...},
    "total_costos_indirectos": 23456,
    "monto_contrato": 145000,
    "intereses_intercalarios": 8000,
    "costo_total_proyecto": 153000,
    "partidas_resueltas": [
      {"partida_codigo": "P-0001", "cantidad": 1, "subtotal": 85000, ...}
    ]
  }
}
```

Este endpoint **no escribe** en BD. Es el que alimenta el panel derecho del editor en vivo, llamado con debounce desde el frontend.

#### `POST /api/v1/proyectos/{id}/transiciones` con `nuevo_estado = valorizado`
Internamente: ejecuta el motor + persiste `valorizacion_snapshot`.

### 7.4 Exports

#### `GET /api/v1/proyectos/{id}/exports/memoria-excel`
Devuelve el archivo `.xlsx` con la memoria de cálculo. Si el proyecto está `valorizado`, usa el snapshot; si es borrador, recalcula al vuelo.

**Response 200:** `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.

#### `GET /api/v1/proyectos/{id}/exports/reporte-pdf`
Devuelve el reporte ejecutivo en PDF.

**Response 200:** `application/pdf`.

### 7.5 Migración de versión de catálogo

#### `POST /api/v1/proyectos/{id}/migrar-catalogo`
Cambia el `catalogo_version_id` del proyecto a otra versión. Solo en estado `borrador`.

**Request:**
```json
{ "nuevo_catalogo_version_id": "uuid" }
```

**Response 200:** preview del impacto (componentes que ya no existen, cambios de PU, etc.) + el proyecto actualizado.

### 7.6 Errores

Formato Problem Details:

```json
{
  "type": "https://valorizador.cen/errors/expresion-invalida",
  "title": "Expresión de cantidad inválida",
  "status": 422,
  "detail": "El identificador 'longitud_m' no está permitido para tipo 'linea'. Use 'longitud_km'.",
  "instance": "/api/v1/catalogo/borradores",
  "campo": "Composiciones[15].cantidad"
}
```

Códigos de error principales:
- `400` request malformada
- `403` rol simulado no autorizado (ej. modelador intenta publicar catálogo)
- `404` recurso no existe
- `409` conflicto de estado (ej. modificar proyecto valorizado)
- `422` validación de dominio (ingesta de Excel, transición no permitida)

---

## 8. Generación de outputs

### 8.1 Memoria de cálculo (Excel)

Estructura del archivo generado:

| Hoja | Contenido |
|---|---|
| `Resumen` | Datos de la obra (decreto, tipo, zona, plazo) + estructura de costos canónica + total |
| `Antecedentes` | Catálogo usado, fecha de cálculo, factores efectivos, parámetros |
| `Por_Instalacion` | Una sección por instalación, con desglose de componentes y partidas |
| `Partidas_Detalle` | Tabla plana de todas las partidas resueltas |
| `Partidas_AdHoc` | Solo partidas ad-hoc (auditoría) |
| `Factores_Aplicados` | Default vs override por proyecto |

Implementado con `openpyxl`. Formato con bordes, totales en negrita, moneda alineada a la derecha.

### 8.2 Reporte ejecutivo (PDF)

Una página por sección:
1. Portada (nombre obra, decreto, fecha)
2. Resumen ejecutivo (total + desglose 1/2/3/4)
3. Por instalación (resumen)
4. Supuestos clave (versión catálogo, factores efectivos, plazo, zona)

Implementado con `weasyprint` (HTML + CSS) por flexibilidad de layout. Template Jinja2.

---

## 9. Tests

### 9.1 Tests del motor (críticos)

- **Regresión por fixtures**: cada combinación `(tipo_obra, set_componentes, factores)` tiene un fixture con resultado esperado. Test compara cada campo del `ResultadoValorizacion` con tolerancia de 1e-6.
- **Tests unitarios** por subfunción: `calcular_ito`, `calcular_intereses_intercalarios`, `resolver_expresion`, etc.
- **Tests de propiedad** (con Hypothesis): variaciones de parámetros generan resultados monótonos donde corresponda (ej. duplicar longitud de línea duplica cantidad de conductor).

### 9.2 Tests de ingesta

- Archivos Excel válidos: ingestan sin errores.
- Archivos con errores estructurales: cada error tipo lanza un error bloqueante específico.
- Diff entre dos versiones produce el resultado esperado.

### 9.3 Tests de API

- Crear → editar → valorizar → exportar.
- Intento de modificar proyecto valorizado → 409.
- Modelador intentando publicar catálogo → 403.
- Migración de catálogo con preview.

---

## 10. Configuración y ejecución

- Variables de entorno principales (con defaults razonables para prototipo):
  - `VALORIZADOR_DB_PATH` (default `data/valorizador.db`)
  - `VALORIZADOR_UPLOADS_PATH` (default `data/uploads/`)
- Inicio: `uvicorn api.main:app --reload --port 8000`
- Migraciones: `alembic upgrade head` al arrancar (hook).
- Bootstrap mínimo: comando `python -m valorizador bootstrap` que carga un catálogo de ejemplo si la BD está vacía.

---

## 11. Decisiones postergadas para escalado

- Cambio de SQLite a PostgreSQL: trivial vía SQLAlchemy.
- Auth con SSO: middleware en `api/deps.py`, reemplaza `X-Rol-Simulado`.
- Cache de catálogo en memoria (Redis o similar) cuando crezca el tamaño.
- Cálculo asíncrono con worker (Celery / RQ) si el cálculo se vuelve lento.
- Integración con InfoTécnica: módulo `integraciones/infotecnica.py`.
