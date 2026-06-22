# Tareas pendientes — Revisión de los documentos de referencia

> Los 5 documentos del sprint inicial están escritos y entregados. Falta tu auditoría página por página antes de iniciar el desarrollo del prototipo.
>
> Sigue este orden: cada documento depende de los anteriores, así que validarlos secuencialmente evita re-trabajo.

---

## Orden de auditoría

### 1. `01-contexto-y-dominio.md` — Documento maestro
- [ ] Auditar
- Es el doc del que dependen los otros 4. Cualquier cambio aquí se propaga.
- **Revisar especialmente**:
  - Glosario (§2): ¿la terminología CEN está completa y exacta?
  - Jerarquía de 4 niveles (§4): ¿modela correctamente cómo se valoriza?
  - Tipificación de componentes (§5): ¿faltan parámetros, valores o reglas que tus pantallazos no alcanzaron a cubrir?
  - Estructura de costos (§6): ¿las fórmulas para 1.1–1.6, 2.1–2.4, 3, 4 son las que usa el CEN?
  - Reglas de conexión (§7.2): ¿hay combinaciones que omití o prohibí incorrectamente?

### 2. `05-template-excel-catalogo.md` — Contrato del consultor
- [ ] Auditar
- Bloquea al consultor externo. Una vez aprobado, comunícaselo formalmente.
- **Revisar especialmente**:
  - Las 9 hojas obligatorias (§2): ¿alguna sobra o falta?
  - Esquemas (§3): ¿los nombres de columna y valores controlados calzan con cómo el consultor está trabajando?
  - Sintaxis de cantidad paramétrica (§3.7.1): ¿las variables permitidas cubren todos los casos? ¿O hay componentes que necesitan otras variables (ej. `tension_kv`, `n_paños`, etc.)?
  - Decisión de **componentes separados vs IFs**: validé esto contigo informalmente (modelar `C-PD-220-AIS-MED-NEW` y `C-PD-220-AIS-MED-AMP` por separado en vez de lógica condicional). Confirmar.
  - Checklist de entrega (§7).

### 3. `02-backend.md` — Especificación del backend
- [ ] Auditar
- **Revisar especialmente**:
  - Modelo de datos (§3): ¿los modelos Pydantic reflejan correctamente el dominio?
  - Fórmulas explícitas (§4.3): las inventé bottom-up; valida que el resultado coincida con cómo el CEN valoriza hoy en Excel manual.
  - Política de partidas directas vs factores (§4.3.2): asumí que si una partida está como directa **y** existe un factor para la misma categoría, **se suman**. Confirmar política correcta.
  - Endpoints REST (§7): ¿faltan endpoints? ¿alguno sobra?
  - Migración de versión de catálogo (§7.5): el flujo "propone migrar, el modelador decide".

### 4. `03-frontend.md` — Especificación del frontend
- [ ] Auditar
- **Revisar especialmente**:
  - Layout de 3 zonas (§2): valida que coincide con tus pantallazos.
  - Paleta de bloques (§2.2): la lista de íconos habilitados por `tipo_obra` es interpretación mía; valida la matriz.
  - Reglas de conexión (§3.4): tabla completa de origen → destino. Valida que cubra todos los casos de tus diagramas.
  - Reglas automáticas (§3.5): paños acoplador/seccionador auto-generados en SE-nueva, doble circuito que dibuja 2ª conexión, etc.
  - Formularios por componente (§4): cada uno con campos, validaciones y reglas de herencia. Es la traducción más densa de tus pantallazos.

### 5. `04-arquitectura-sistema.md` — Visión transversal
- [ ] Auditar
- Último porque consolida todo. Sin embargo, tiene contenido propio que requiere validación.
- **Revisar especialmente**:
  - Stack consolidado (§2): tecnologías ya confirmadas, pero verifica que no haya quedado nada que el equipo del CEN prefiera distinto.
  - Cómo correr en local (§5): pasos pensados para tu entorno Windows + Python + Node.
  - Decisiones postergadas (§6): la lista de lo que **no** está en el prototipo. Si alguna debería entrar, registrarlo aquí.
  - Roadmap de implementación post-documentos (§7): orden sugerido de fases. Validar que el orden tenga sentido para cómo quieres demostrar avance al equipo.

---

## Cómo registrar feedback

Tres opciones, según preferencia:

1. **Anotaciones directas en el .md**: edita el archivo y deja comentarios en `> TODO:` o `> @piero:` para que los procese.
2. **Lista en este archivo**: agrega notas debajo de cada documento en formato:
   ```
   - [ ] §4.3.6 — La fórmula de intereses intercalarios usa flujo lineal; el CEN usa una curva específica que documentar.
   ```
3. **Conversación**: cuando estés revisando, abrime una sesión y voy aplicando los cambios doc por doc.

---

## Cuando cierres la auditoría

Una vez los 5 documentos estén validados, el siguiente paso es **iniciar el desarrollo del prototipo** siguiendo el roadmap de `04-arquitectura-sistema.md` §7.

Antes de iniciar:

- [ ] Confirmar nombre del repo y crear `.gitignore`, `README.md`, `pyproject.toml` y `package.json` esqueleto.
- [ ] Comunicar al consultor externo la versión final del Excel template (`05`).
- [ ] Decidir si el desarrollo lo lleva tu equipo, externo, o mixto.
- [ ] Definir cadencia de demos (sugerencia: una al cierre de cada fase del roadmap).
