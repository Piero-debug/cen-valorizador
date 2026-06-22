# 03 — Frontend

> Especificación del frontend del Valorizador CEN. Cubre layout, editor canvas (drag-and-drop tipo unifilar), reglas de conexión, formularios por componente, panel de estimación en vivo, flujos de proyecto y vista del curador.
>
> Asume conocimiento de `01-contexto-y-dominio.md` (dominio) y `02-backend.md` (API).

---

## 1. Visión general

### 1.1 Principios

1. **Editor primero**: el frontend gira en torno al **canvas de modelado** estilo diagrama unifilar. El resto de pantallas (listado, curador, exports) apoyan ese flujo.
2. **Estimación en vivo**: el panel derecho recalcula totales en tiempo real mientras el modelador edita el grafo, vía `POST /valorizacion` con debounce.
3. **Herencia visible**: cuando un componente hereda atributos de uno superior, el formulario muestra el valor pre-llenado **y marcado como heredado** (no oculto).
4. **Validación cercana al usuario**: las reglas de conexión y conexiones ilegales se previenen en el frontend (no permitir el drag), con el backend como respaldo.

### 1.2 Stack

- **React 18** + **TypeScript** + **Vite**
- **shadcn/ui** + **Tailwind CSS** (componentes UI)
- **React Flow** (canvas, nodos custom, edges, viewport, paleta lateral)
- **TanStack Query** (estado servidor; cache, invalidación, refetch)
- **Zustand** (estado UI local: selección, formularios abiertos, modo de visualización)
- **TanStack Table** (grillas: listado de proyectos, partidas, factores)
- **React Hook Form** + **Zod** (formularios y validación)
- **Recharts** (gráficos en reportes en pantalla; opcional)

---

## 2. Layout general

El editor de obra tiene **3 zonas** fijas. Las pantallas que no son editor (listado de proyectos, curador) usan layouts simples sin sidebar central.

```
┌────────────────────────────────────────────────────────────────────────┐
│  Header                                                                │
│  [Valorizador CEN] [Proyecto: Nueva SE Polpaico ▾] [Rol: Modelador ▾] │
├──────────┬──────────────────────────────────────────────┬──────────────┤
│          │                                              │              │
│ Paleta   │              Canvas (Área de trabajo)        │  Panel       │
│ de       │                                              │  Estimación  │
│ bloques  │   [grilla] [bloques] [conexiones]            │              │
│          │                                              │  Estructura  │
│ - Barra  │                                              │  de costos   │
│ - Paño   │                                              │  en vivo     │
│ - Trafo  │                                              │              │
│ - Línea  │                                              │  Total: USD  │
│ - …      │                                              │              │
│          │                                              │              │
│          │  [Conectar] [Refrescar] [Limpiar]            │              │
│          │                                              │              │
└──────────┴──────────────────────────────────────────────┴──────────────┘
```

### 2.1 Header

- Logo / nombre de la app.
- **Selector de proyecto activo** (dropdown con buscador).
- **Selector de rol simulado**: `Modelador` o `Curador del catálogo`. Cambia las pantallas accesibles.
- Indicador de **versión de catálogo** anclada al proyecto, con badge si hay versión nueva disponible.
- Botón "Guardar" (autosave en background; este botón solo fuerza un save explícito y muestra confirmación).

### 2.2 Paleta de bloques (sidebar izquierdo)

Lista de íconos arrastrables. Los íconos disponibles dependen del `tipo_obra` y los filtros aplicados. Cada ícono representa un **tipo de componente** del catálogo.

| Ícono | Nombre | Habilitado en |
|---|---|---|
| ⊕ (barra alta tensión) | Barra AT | SE-nueva, SE-amp |
| — (paño estándar) | Paño | todos |
| ⌥ (paño diagonal) | Paño Diagonal | todos |
| ⌐ (barra MT) | Barra MT | SE-nueva, SE-amp |
| ⊙⊙ (trafo 2 dev.) | Transformador | SE-nueva, SE-amp |
| ⌿ (línea aérea) | Línea | L-nueva, L-amp + enlace de seccionamiento en SE-nueva |
| ▭ (compensación) | Compensación | SE-nueva, SE-amp |

> Los íconos exactos se definen en el repo en `frontend/src/canvas/iconos/`. Diseño se ajustará durante implementación; la idea es semaforización clara entre tipos.

**Interacción:**
- **Drag** del ícono al canvas → crea una `InstanciaComponente` con valores por defecto.
- **Hover** sobre el ícono → tooltip con descripción y nivel de tensión típico.
- Indicador numérico en el ícono cuando aplica (ej. ⊕ con número = posiciones disponibles totales en barras de la obra).

### 2.3 Canvas (centro)

Construido sobre **React Flow** v11+. Configuración:

- Modo: `default` con drag, zoom, pan.
- **Snap to grid** activado (alineación visual del diagrama unifilar).
- Modo de visualización: `DU` (diagrama unifilar). Selector futuro para `DEE Planta` (sin implementar en prototipo).
- Nodos custom por tipo de componente (ver §4).
- Edges custom para representar conexiones eléctricas (líneas dobles para doble circuito, etc.).

**Acciones del canvas:**
- **Botón Conectar**: alterna el modo de conexión (alternativamente, conectar arrastrando desde los handles de los nodos).
- **Botón Refrescar**: revalida el grafo contra reglas del backend y reorganiza si hay nodos huérfanos.
- **Botón Limpiar**: borra el grafo (confirmación obligatoria).

### 2.4 Panel de estimación (sidebar derecho)

Replica la estructura de costos canónica (`01-contexto-y-dominio.md` §6):

```
Estimación / Referencia de Costos

Item   Partida                              Valor (USD)
─────  ───────────────────────────────────  ───────────
1      Costos Directos                          0,00
1.1    Ingeniería                               0,00
1.2    Gestión medio ambiental                  0,00
1.3    Instalación de faena                     0,00
1.4    Suministros, Obras Civiles, Montaje      0,00
1.5    Servidumbres y terrenos                  0,00
1.6    Pruebas y Puesta en Servicio             0,00
2      Costos Indirectos                        0,00
2.1    Gastos Generales y Seguros               0,00
2.2    Inspección técnica de obra               0,00
2.3    Utilidades del contratista               0,00
2.4    Contingencias                            0,00
3      Monto Contrato                           0,00
4      Intereses Intercalarios                  0,00
       Costo Total del Proyecto                 0,00
```

**Comportamiento:**
- Cada fila es **clickable** para abrir un drilldown con el desglose (en un drawer lateral).
- El **total se recalcula** al cambiar cualquier nodo, conexión, parámetro o factor (debounce de 400 ms).
- Indicador de "calculando…" mientras la request está in-flight.
- Diff visual cuando el último cálculo difiere del anterior (flecha y delta en rojo/verde).
- Botón **"Editar factores"** abre un drawer con los factores efectivos y permite override (5.5).

---

## 3. Editor canvas: modelo de nodos y edges

### 3.1 Tipos de nodos (React Flow `nodeTypes`)

Cada tipo se mapea a un componente React custom con su forma visual, handles (puntos de conexión) y formulario asociado.

| Tipo | Forma | Handles | Comportamiento especial |
|---|---|---|---|
| `barraAT` | Rectángulo horizontal extenso | Múltiples handles arriba (slots para paños) | Muestra contador de posiciones disponibles; al agotarse, los handles se vuelven inactivos |
| `barraMT` | Rectángulo horizontal con doble línea | Múltiples handles arriba/abajo | Similar a barra AT |
| `pano` | Cuadrado pequeño con símbolo de interruptor | Handle superior (barra), handle inferior (equipo) | Función se asigna al conectar |
| `panoDiagonal` | Cuadrado con diagonal | Hasta 2 handles inferiores | Subtipo "media" vs "completa" cambia visualización |
| `linea` | Símbolo de torre/conductor | Handle superior (paño) | Doble circuito dibuja 2ª conexión automáticamente |
| `transformador` | 2 círculos enlazados | 1 handle arriba (AT), 1 abajo (MT) | Hereda configuración de ambas barras |
| `compensacion` | Símbolo de capacitor/reactor | 1 handle arriba | — |

### 3.2 Datos del nodo (`data`)

```typescript
interface DatosNodo {
  id_nodo: string;
  codigo_componente: string;  // referencia al catálogo
  tipo: TipoComponente;
  parametros: Record<string, number | string>;
  atributos_heredados: Record<string, { valor: any; origen_id: string }>;
  validacion: { errores: string[]; advertencias: string[] };
  posicion_canvas: { x: number; y: number };
}
```

### 3.3 Tipos de edges

| Tipo | Visual | Significado |
|---|---|---|
| `conexionElectrica` | Línea simple | Conexión estándar |
| `conexionDobleCircuito` | Línea doble paralela | Línea con `n_circuitos = 2` |
| `conexionSeccionamiento` | Línea simple con marca | Línea con `tipo_conexion = Seccionamiento` |

### 3.4 Reglas de conexión (validación en el cliente)

Implementadas en el handler `onConnect` de React Flow. Si la conexión propuesta no cumple las reglas, se rechaza con un toast explicativo.

| Origen | Destino | Permitida | Notas |
|---|---|---|---|
| `barraAT` | `pano` | Sí | Solo si hay posición disponible |
| `barraAT` | `barraAT` | No | Salvo a través de paño acoplador/seccionador |
| `barraAT` | `barraMT` | No | Solo a través de transformador |
| `pano` | `linea` | Sí | Asigna función `linea` al paño |
| `pano` | `transformador` | Sí | Asigna función `trafo` al paño |
| `pano` | `compensacion` | Sí | Asigna función `compensador` al paño |
| `pano` | `pano` | Sí | Solo si origen es acoplador/seccionador |
| `panoDiagonal` | hasta 2 conexiones | Sí | Validar 5.4.2 del doc 01 (2 trafos prohibido) |
| `transformador` | `barraMT` | Sí | Una barra MT por trafo (o más, según diseño) |
| `linea` | cualquier | No | La línea es nodo terminal en el modelado |

### 3.5 Reglas automáticas

Al modelar, el sistema genera nodos automáticamente cuando corresponde:

- **Paños acoplador/seccionador en SE-nueva**: al crear una barra con cierta configuración, el sistema agrega los paños obligatorios. En SE-amp deben crearse manualmente.
- **Doble circuito**: al cambiar `n_circuitos` de una línea de 1 a 2, el canvas dibuja una **segunda conexión** del paño a la barra automáticamente.
- **Línea corta en SE-nueva**: al crear el primer paño de línea, se sugiere agregar el enlace de seccionamiento.

Estas reglas son **sugerencias visualmente confirmables**: el sistema propone el nodo en color tenue y el modelador acepta o ignora.

---

## 4. Formularios por componente

Cada nodo abre su formulario al **click** (no doble click; doble click se reserva para edición rápida del nombre). El formulario se abre en un **drawer lateral derecho** o en un **modal**, según preferencia (por defecto drawer para no perder contexto del canvas).

### 4.1 Estructura común del formulario

Todos los formularios tienen tres secciones:

```
┌──────────────────────────────────────┐
│  [Nombre del componente] [● estado]  │
├──────────────────────────────────────┤
│  Atributos clasificatorios           │
│  (heredados del nivel superior)      │
│                                      │
│  Tensión:        220 kV  (heredado)  │
│  Tecnología:     AIS     (heredado)  │
├──────────────────────────────────────┤
│  Parámetros editables                │
│                                      │
│  Tipo interruptor: [tanque vivo ▾]   │
│  ...                                 │
├──────────────────────────────────────┤
│  Partidas resultantes (lectura)      │
│  ▶ P-0001 Interruptor SF6  qty=1     │
│  ▶ P-0002 Desconectador    qty=2     │
│  ...                                 │
├──────────────────────────────────────┤
│  Subtotal componente: USD 100.234,00 │
└──────────────────────────────────────┘
```

### 4.2 Antecedentes Generales (modal inicial al crear proyecto)

Primer modal que aparece al crear un proyecto. Bloquea el canvas hasta completarse.

| Campo | Control | Validación |
|---|---|---|
| Nombre de la obra | Input text | Requerido |
| Número de decreto | Input text | Opcional |
| Tipo de proyecto | Radio buttons | Requerido. Opciones: `Nueva Línea`, `Nueva Subestación`, `Ampliación Subestación`, `Ampliación Línea (Aumento capacidad / tendido 2º circuito)` |
| Zona | Select | Requerido. Carga de `Catalogo.zonas` |
| Plazo | Input number (meses) | Requerido. > 0 |
| Versión de catálogo | Select | Default: versión vigente. Permite anclar a una específica |

Al confirmar:
- Habilita la paleta de bloques según `tipo_obra`.
- Carga factores default para ese `tipo_obra` (editables luego).
- Habilita el canvas vacío.

### 4.3 Formulario Barra AT

| Campo | Control | Comportamiento |
|---|---|---|
| Identificador | Input text (auto: "Barra 1", "Barra 2"...) | Editable; único en la SE |
| Tecnología | Select (`AIS`, `GIS`, `HIS`) | Requerido |
| Tensión | Select (`66kV`...`500kV`) | Requerido |
| Configuración | Select (depende de decreto: `simple`, `doble`, `doble_seccionador`, `int_y_medio`) | Requerido |
| Posiciones | Input number | Requerido. > 0 |
| Capacidad nominal | Input number (A o MVA) | Determina internamente el conductor; no es elección del usuario, se muestra como info |

**Indicador visible**: contador `Posiciones disponibles: X de Y` que decrementa al conectar paños.

### 4.4 Formulario Barra MT

Igual que Barra AT pero:
- Tecnología: `AIS` o `celdas`.
- Configuración: opciones reducidas según norma.
- Al conectar trafo: **hereda** tensión y configuración hacia los paños de trafo (mostrarlo en feedback visual).

### 4.5 Formulario Paño (estándar)

| Campo | Control | Comportamiento |
|---|---|---|
| Función | Select / asignación automática | Se asigna al conectar (`línea`, `trafo`, `acoplador`, `seccionador`, `compensador`). Si el paño está sin conectar, hay opción `click` para asignarla manual. |
| Tipo de interruptor | Select | Opciones cambian con la tecnología de la barra: <br>- AIS: `tanque_vivo`, `tanque_muerto`<br>- HIS: `hibrido`, `hibrido_monopolar`<br>- GIS: campo deshabilitado |
| Tensión | (heredado de la barra) | Solo lectura |
| Tecnología | (heredado de la barra) | Solo lectura |

**Reglas especiales:**
- En `SE-nueva`: los paños **acoplador y seccionador** son generados automáticamente; aparecen en el grafo pre-marcados con badge "Auto". Editables con `click`.
- En `SE-amp`: los paños acoplador/seccionador son **seleccionables manualmente** vía botón "Agregar paño acoplador" en el formulario de la barra.

### 4.6 Formulario Paño Diagonal

| Campo | Control | Comportamiento |
|---|---|---|
| Subtipo | Toggle (`completa`, `media`) | Requerido |
| Tipo de interruptor | Select | Mismas reglas que Paño |
| N° de interruptores | Auto-calculado, editable según reglas | <br>- Diagonal completa: 3 (fijo)<br>- Media diagonal en SE-nueva: 2 (fijo)<br>- Media diagonal en SE-amp o L-nueva: 1 o 2 (editable, default 2) |
| Conexiones aceptadas | Lista mostrada como hint | "Hasta 2 conexiones: 2 líneas / 1 línea + 1 trafo / 1 línea / 1 trafo. 2 trafos no permitido." |

### 4.7 Formulario Línea

| Campo | Control | Comportamiento |
|---|---|---|
| Tipo de conexión | Radio (`Línea`, `Seccionamiento`) | Requerido |
| Longitud (km) | Input number | Requerido. > 0 |
| Cantidad de circuitos | Toggle (1 / 2) | Requerido. Al cambiar a 2, el canvas dibuja una segunda conexión a paño y barra. |
| Capacidad MVA | Input number | Requerido |
| Interferencia | Toggle / Select | Según catálogo |
| Cable guardia | Select | Opciones del catálogo |
| Tipo de torre | Select (`celosia`, `poste_hormigon`, `monoposte`, `subterranea`) | Requerido |
| Tipo de conductor | Select | Opciones del catálogo |
| Tipo de ampliación (solo si `L-amp`) | Select (`tendido_2c`, `aumento_capacidad`, `extension`) | Requerido en L-amp |

**Para `tipo_conexion = Seccionamiento`:**
- Se muestra un panel "Antecedentes de la línea existente" con campos: `tipo_conductor`, `n_conductores_por_fase`, `tipo_cable_guardia`. En el prototipo se ingresan **manualmente** (integración con InfoTécnica queda para etapas posteriores).
- `Longitud` tiene un default editable.

**Pendiente para etapas posteriores**: botón "Dibujar trazado" que abre integración con Google Earth.

### 4.8 Formulario Transformador

| Campo | Control | Comportamiento |
|---|---|---|
| Tipo de trafo | Select | Opciones del catálogo (2 dev., 3 dev., etc.) |
| Capacidad MVA | Input number | Requerido |
| Muro y foso | Checkbox | Default `true` en SE-nueva (no editable); en SE-amp `click` para activar |
| Sistema contra incendio | Checkbox | `click` para activar |

**Herencia**: al conectar a una Barra AT y una Barra MT, los paños de trafo en cada barra heredan tensión y configuración.

### 4.9 Formulario Compensación

| Campo | Control |
|---|---|
| Tipo | Select (`bbcc`, `reactor_linea`, `reactor_barra`) |
| Capacidad MVAr | Input number |

---

## 5. Estado y flujo de datos

### 5.1 Estado servidor (TanStack Query)

Queries:

- `useProyectos(filtros)` → lista paginada
- `useProyecto(id)` → detalle completo
- `useValorizacion(proyectoId, modeloLocal)` → cálculo en vivo; mutation con debounce 400 ms
- `useCatalogoVersiones()` → lista
- `useCatalogoVersion(id)` → contenido completo (componentes, partidas, zonas, factores)
- `useBorradorCatalogo(borradorId)` → estado del borrador con diff y errores

Mutations:

- `useCrearProyecto`, `useActualizarProyecto`, `useClonarProyecto`, `useTransicionarProyecto`, `useMigrarCatalogo`
- `useSubirBorradorCatalogo`, `usePublicarBorrador`

Invalidación: al mutar un proyecto, invalidar `proyectos` y `proyecto/{id}`.

### 5.2 Estado UI (Zustand)

```typescript
interface EstadoEditor {
  nodoSeleccionadoId: string | null;
  formularioAbierto: { tipo: string; nodoId: string } | null;
  modoConexion: boolean;
  modoVisualizacion: "DU" | "DEE_Planta";  // DEE solo placeholder
  zoom: number;
  factoresOverride: Record<string, number>;  // editable en sesión
  rolSimulado: "modelador" | "curador_catalogo";
}
```

Mutaciones del estado UI no disparan llamadas a la API por sí solas; solo cuando se confirma una edición.

### 5.3 Sincronización canvas ↔ backend

- El canvas mantiene su modelo local (`InstanciaComponente[]` + `Conexion[]`).
- **Autosave** en background: al editar un formulario o conectar/desconectar, se hace `PUT /proyectos/{id}` con debounce 1.5 s.
- **Valorización en vivo**: al editar, se hace `POST /proyectos/{id}/valorizacion` con debounce 400 ms para actualizar el panel derecho. **No persiste**.
- Conflicto improbable en prototipo (un solo usuario), pero el backend devuelve `409` si se intenta modificar un proyecto valorizado.

---

## 6. Pantallas

### 6.1 `/proyectos` — Listado

- Grilla (TanStack Table) con columnas: nombre, tipo de obra, estado, fecha modificación, zona, costo total (si valorizado).
- Filtros: estado, tipo de obra, zona, rango de fechas.
- Acciones por fila: abrir, clonar, archivar.
- Botón "Nuevo proyecto" abre el modal de antecedentes generales.

### 6.2 `/proyectos/{id}` — Editor (3 zonas)

Pantalla principal descrita en §2.

### 6.3 `/proyectos/{id}/snapshot` — Vista del snapshot valorizado

Vista de **solo lectura** del proyecto en estado `valorizado`:
- Grafo congelado.
- Panel derecho con totales finales.
- Acceso a memoria de cálculo Excel y reporte PDF.
- Botón "Clonar para variante" que crea borrador editable.

### 6.4 `/catalogo` — Vista del curador

> Solo accesible con rol simulado `curador_catalogo`.

- Lista de versiones (vigente, históricas).
- Botón "Subir borrador" → drag de archivo Excel o selector.
- Vista de un borrador: tabs `Diff`, `Errores`, `Advertencias`.
- En `Diff`: tabla por hoja con filas `nuevas`, `eliminadas`, `modificadas` (con before/after).
- En `Errores`: lista bloqueante con hoja, fila, columna, mensaje.
- Botón "Publicar" habilitado solo si no hay errores bloqueantes. Confirmación obligatoria.

### 6.5 `/catalogo/versiones/{id}` — Detalle de versión publicada

- Vista de solo lectura del contenido completo.
- Búsqueda por código de partida o componente.
- Export del Excel original (para auditoría).

### 6.6 `/proyectos/{id}/migrar-catalogo` — Asistente de migración

Cuando se publica una nueva versión del catálogo, el listado de proyectos muestra un badge en los proyectos en `borrador` cuya versión anclada quedó obsoleta. Al entrar al editor, un banner ofrece migrar.

El asistente muestra:
- Diff de impacto (componentes que ya no existen, cambios de PU, factores modificados).
- Acción: `Migrar` o `Mantener versión actual`.

---

## 7. Validación y feedback

### 7.1 Validaciones en tiempo de edición

- Formularios usan **React Hook Form + Zod**: validación campo a campo, mensajes de error en línea.
- Reglas de conexión validadas en `onConnect` antes de aceptar el edge.
- Estado del nodo se marca visualmente:
  - **Verde**: válido y conectado.
  - **Amarillo**: válido pero con advertencias (ej. sin conectar).
  - **Rojo**: error bloqueante (ej. parámetro faltante).

### 7.2 Feedback de la valorización

Cada vez que el panel derecho se actualiza:
- Indicador "calculando…" mientras la request está activa.
- Si el cálculo falla (error del backend), banner rojo con `Problem Details`.
- Si hay warnings (ej. partidas ad-hoc sin justificación), badge en el panel.

### 7.3 Toasts globales

- Guardado exitoso (autosave): toast discreto inferior derecho.
- Cambio de estado del proyecto: toast con detalle de la transición.
- Errores de red: toast persistente con reintento.

---

## 8. Estilo y temas

- Paleta basada en colores corporativos del CEN (a confirmar; placeholder con azul institucional + grises neutros + acentos para estados).
- Tipografía: Inter (web) o equivalente del sistema.
- Modo claro por defecto; soporte de modo oscuro deseable pero no prioritario en prototipo.
- Densidad: media. El editor canvas y la grilla de costos son densos en información; mantener legibilidad.

---

## 9. Estructura del código frontend

```
frontend/
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── index.html
└── src/
    ├── main.tsx
    ├── app.tsx                       # router + providers
    ├── lib/
    │   ├── api.ts                    # cliente HTTP (axios + interceptors)
    │   ├── query.ts                  # configuración TanStack Query
    │   └── utils.ts
    ├── components/                   # shadcn/ui + custom UI
    │   └── ui/...
    ├── canvas/
    │   ├── editor.tsx                # contenedor React Flow
    │   ├── paleta.tsx                # sidebar izquierdo
    │   ├── reglas-conexion.ts        # validación onConnect
    │   ├── reglas-automaticas.ts     # generación auto de paños/conexiones
    │   ├── nodos/
    │   │   ├── barra-at.tsx
    │   │   ├── barra-mt.tsx
    │   │   ├── pano.tsx
    │   │   ├── pano-diagonal.tsx
    │   │   ├── linea.tsx
    │   │   ├── transformador.tsx
    │   │   └── compensacion.tsx
    │   ├── edges/
    │   │   ├── conexion-electrica.tsx
    │   │   └── conexion-doble.tsx
    │   └── iconos/                   # SVGs para paleta y nodos
    ├── formularios/
    │   ├── antecedentes-generales.tsx
    │   ├── barra-at.tsx
    │   ├── barra-mt.tsx
    │   ├── pano.tsx
    │   ├── pano-diagonal.tsx
    │   ├── linea.tsx
    │   ├── transformador.tsx
    │   └── compensacion.tsx
    ├── panel-estimacion/
    │   ├── panel.tsx
    │   ├── fila-categoria.tsx
    │   ├── drilldown.tsx
    │   └── editor-factores.tsx
    ├── proyecto/
    │   ├── listado.tsx
    │   ├── detalle.tsx
    │   ├── snapshot.tsx
    │   ├── migrar-catalogo.tsx
    │   └── clonar.tsx
    ├── catalogo/
    │   ├── listado-versiones.tsx
    │   ├── subir-borrador.tsx
    │   ├── detalle-borrador.tsx
    │   ├── diff.tsx
    │   └── detalle-version.tsx
    ├── hooks/
    │   ├── use-proyecto.ts
    │   ├── use-valorizacion.ts
    │   └── use-catalogo.ts
    ├── stores/
    │   └── editor-store.ts           # Zustand
    └── types/
        └── api.ts                    # tipos generados (idealmente desde OpenAPI)
```

---

## 10. Decisiones postergadas para etapas posteriores

- **Vista DEE Planta**: el selector de modo de visualización está, pero solo `DU` está implementado.
- **Trazado de línea con Google Earth**: botón placeholder en el formulario de Línea.
- **Comparación lado a lado de proyectos**: requiere una pantalla dedicada con dos canvas read-only.
- **Modo oscuro completo**.
- **Acceso desde móvil**: el editor canvas no está pensado para mobile en v1.
- **Integración con InfoTécnica**: los antecedentes de línea para Seccionamiento son ingreso manual en el prototipo.
- **Internacionalización**: todo en español chileno; sin i18n por ahora.
