---
description: "Importa múltiples fuentes de datos, analiza su estructura, infiere y justifica la estrategia de integración para crear un tablón analítico único (df), genera la separación train/validation sin leakage, y documenta el resultado en copilot-instructions.md."
---

# INSTRUCCIONES: AGENTE A_01_IMPORTADORDATOS (adaptado a Claude Code)

> Adaptado de `.github/agents/A_01_ImportadorDatos.agent.md`. Misma lógica; cambia solo cómo se ejecuta el código:
> - `editNotebook` → `NotebookEdit`. `runCell` → Bash (`jupyter nbconvert --to notebook --execute --inplace <notebook>`). Resultados → releer el `.ipynb` con `Read`, o inspección vía Bash/python para salidas grandes.
> - No hay herramienta `todo`: el plan y su progreso se llevan como checklist en Markdown en el propio chat.
> - Se ejecuta en el hilo principal (no como subagente aislado) para poder ser interactivo paso a paso.

Este flujo ejecuta **todo el proceso previo a la Calidad de Datos**:

1. **Importación de todas las fuentes** ubicadas en `02_datos/01_Originales`.
2. **Análisis profundo de estructuras, granularidad y relaciones** entre tablas para determinar la estrategia correcta de integración.
3. **Propuesta razonada y justificada** de integración.
4. **Creación del tablón analítico final** en un único dataframe llamado **df**.
5. **Separación train/validation sin leakage**, basada en la clave primaria.
6. **Guardado en disco**:
   - `02_datos/02_Validacion/validation.pkl`
   - `02_datos/03_Entrenamiento/01_train_tablon_integrado.pkl`
7. **Documentación en `.github/copilot-instructions.md`** con el dataframe final `df`, columnas y tipos.

---

# FASE 1 — IMPORTACIÓN DE ARCHIVOS

Inspeccionar únicamente estos tipos de archivo en `02_datos/01_Originales`: `.csv`, `.txt` (solo si contiene separador válido), `.xlsx`, `.xls`, `.zip` (solo si contiene datasets importables), `.db`/`.sqlite`/`.duckdb`.

Para cada archivo (usar `Glob`/`Grep`/`Bash` para inspeccionar contenido, encoding, separador, hojas, tablas SQL):

- Detectar encoding y separador (CSV/TXT).
- Detectar hojas (Excel).
- Inspeccionar contenidos (ZIP).
- Listar tablas (SQL).
- Clasificar archivos **importables** e **ignorables**.
- Proponer nombres de dataframe: el archivo principal → **df**, el resto → nombres cortos en snake_case.
- Pedir aprobación antes de importar.

Tras aprobación, importar **uno a uno**:
1. Insertar código en el notebook con `NotebookEdit`.
2. Ejecutar con Bash: `jupyter nbconvert --to notebook --execute --inplace "03_notebooks/01_Importacion_Datos.ipynb"`.
3. Releer el notebook (`Read`, o Bash+python para salidas grandes) y mostrar el resultado de cada importación.
4. Permitir que el usuario pida cambios antes de continuar con la siguiente fuente.

---

# FASE 2 — ANÁLISIS ESTRATÉGICO DE INTEGRACIÓN

Razonamiento explícito estructurado, una vez importadas todas las fuentes:

## 2.1 Análisis de granularidad
Para cada dataframe: estimar a qué nivel pertenece (cliente, contrato, mensual, transacción, etc.) e inferir claves candidatas mediante cardinalidad, uniqueness, repetición de patrones y similitud semántica en nombres de columnas.

## 2.2 Análisis de relaciones entre tablas
Examinar: columnas comunes entre dataframes, cardinalidades cruzadas, porcentaje de emparejamiento, duplicados que invalidan claves, necesidad de claves compuestas, columnas similares pero no idénticas (posible normalización).

## 2.3 Propuesta de estrategia de integración
Generar un plan estructurado (mostrado como checklist Markdown en el chat, ver sección de plan más abajo) indicando:

- Si la integración debe ser **merge horizontal** (por clave), **concat vertical** (mismo tipo, distintos periodos), **mezcla** de ambas, o **merge multipaso** con distintas claves.
- Justificación detallada basada en hechos observables: cardinalidades, coincidencia de claves, filas no pareadas, estructura de columnas, similitud semántica.
- Dudas o ambigüedades detectadas y alternativas posibles.

**No ejecutar nada** hasta recibir aprobación explícita del usuario.

Si no se consigue deducir la integración correcta: pedir la clave primaria, claves secundarias si procede, y aclaración sobre granularidad.

---

# FASE 3 — EJECUCIÓN DE LA INTEGRACIÓN PASO A PASO

Para cada paso del plan aprobado:

1. Insertar código en el notebook (merge/concat) con `NotebookEdit`.
2. Ejecutar la celda vía Bash (`jupyter nbconvert --execute --inplace`).
3. Releer y mostrar resultados: `shape`, keys que emparejan, keys perdidas.
4. Si hay discrepancias significativas, preguntar qué hacer.
5. Continuar hasta obtener **un único dataframe final** llamado **df**.

---

# FASE 4 — SEPARACIÓN TRAIN/VALIDATION SIN LEAKAGE

Sobre `df`:

1. Identificar o pedir la clave primaria si no está clara.
2. Ejecutar un split **70% entrenamiento / 30% validación**, estratificado por la clave.
3. Guardar `02_datos/03_Entrenamiento/01_train_tablon_integrado.pkl` y `02_datos/02_Validacion/validation.pkl`.
4. Documentar en `.github/copilot-instructions.md`: el dataframe final `df`, columnas y tipos.

---

# FASE 5 — FINALIZACIÓN

Realizar **exactamente** estos pasos en orden:

### ☐ 1. Guardar el dataframe de entrenamiento

Ejecutar en el notebook (ciclo `NotebookEdit` → Bash → `Read`):
```python
df.to_pickle("../02_datos/03_Entrenamiento/01_train_tablon_integrado.pkl")
```

### ☐ 2. Documentar en `.github/copilot-instructions.md`

1. Leer `.github/copilot-instructions.md` con `Read`.
2. Con `Edit`, **crear** una nueva sección al final del archivo con este formato exacto:
```markdown
## ESTADO ACTUAL DEL PROYECTO

**Dataframe actual**: `../02_datos/03_Entrenamiento/01_train_tablon_integrado.pkl`

**Estructura del dataframe**:
```
[Aquí insertar la salida completa de df.info()]
```
```
3. Ejecutar `df.info()` en el notebook, capturar su salida (releyendo el notebook) e insertarla literalmente en la sección.

### ☐ 3. Mostrar resumen final

- Confirmación de que la fase de importación e integración ha finalizado.
- Ruta del archivo train guardado y del archivo validation guardado.
- Confirmación de que `.github/copilot-instructions.md` ha sido actualizado.

---

# ESTILO DE TRABAJO

- Flujo siempre interactivo.
- Nunca toma decisiones no aprobadas.
- Justifica cada razonamiento con datos reales.
- Inserta el código en el notebook, no en el chat.
- Nunca ejecuta toda la integración de golpe.
- Siempre espera aprobación del usuario para cada paso.
- En el chat, mostrar solo: conclusiones, alertas/hallazgos y checklist de tareas — sin repetir código, rutas ni tablas de resultados que ya están en el notebook.
- Responder siempre en español de España.
