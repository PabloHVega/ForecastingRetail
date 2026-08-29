---
description: "Análisis Exploratorio de Datos (EDA) masivo e interactivo sobre el tablón de entrenamiento actual, tarea a tarea, con checklist en el chat, informe final en Markdown y actualización del estado del proyecto."
---

# INSTRUCCIONES: AGENTE A_03_EDA (adaptado a Claude Code)

> Adaptado del agente de GitHub Copilot `.github/agents/A_03_EDA.agent.md`. La lógica de EDA es la misma; lo que cambia es cómo se ejecuta el código y cómo se gestiona el plan, porque Claude Code no tiene un kernel de notebook en vivo ni una herramienta `todo` con estado persistente.
>
> **Diferencias clave frente al original:**
> - No hay `run_notebook_cell` / `read_notebook_cell_output`: el código se escribe en el notebook con `NotebookEdit` y se ejecuta con `jupyter nbconvert --to notebook --execute --inplace` vía Bash; los resultados se leen releyendo el `.ipynb` con `Read`.
> - Los gráficos, además de quedar en el notebook, se guardan también como PNG independiente (`plt.savefig(...)`) en una carpeta temporal para poder inspeccionarlos visualmente con `Read` (que sí puede ver imágenes).
> - No hay herramienta `todo`: el plan y su progreso se mantienen como una checklist en Markdown dentro del propio chat, reimprimiéndola cada vez que se cierra una tarea.
> - Este flujo se ejecuta en el hilo principal de la conversación (no como subagente aislado), precisamente para poder ser interactivo tarea a tarea como el original.

## Parámetros orientativos por defecto

- `umbral_alta_cardinalidad`: sin umbral fijo por defecto.
- `max_top_categorias_alta_cardinalidad`: 20.
- `max_variables_por_bloque_graficos`: sin límite fijo; se decide con un gráfico por fila.
- `max_categorias_grafico`: sin límite fijo por defecto.
- `umbral_missing_alerta`: heurística interna.
- `umbral_rare_labels`: heurística interna.

## 0. Contexto de entrada y dataframe de trabajo

Este flujo trabaja exclusivamente sobre el **tablón de entrenamiento actual** indicado en `.github/copilot-instructions.md`.

### PASO INICIAL OBLIGATORIO

1. Leer `.github/copilot-instructions.md` con `Read` y localizar la sección `## ESTADO ACTUAL DEL PROYECTO`.
2. Extraer de esa sección:
   - La ruta del **Dataframe actual** (relativa a `03_notebooks/`, p. ej. `../02_datos/03_Entrenamiento/02_train_tablon_calidad.pkl`).
   - La **Estructura del dataframe** (salida de `df.info()`).
3. Notebook de trabajo: `03_notebooks/03_EDA.ipynb`. Si no existe, crearlo con `NotebookEdit` (`edit_mode: insert`, `cell_type: code`, sin `cell_id`).
4. Insertar como primera celda de código:
```python
import pandas as pd
df = pd.read_pickle("[ruta_extraída_de_copilot-instructions]")
df.shape, df.dtypes.head()
```
5. Ejecutar el notebook completo con Bash:
```
jupyter nbconvert --to notebook --execute --inplace "03_notebooks/03_EDA.ipynb"
```
6. Releer el notebook con `Read` sobre `03_notebooks/03_EDA.ipynb` y confirmar en el chat que `df` se ha cargado correctamente (shape y dtypes).

**Reglas:**

- El dataframe principal de trabajo en el notebook debe llamarse siempre `df`.
- Si el usuario indica otra ruta, adaptarse, documentarlo, y dejar coherente la actualización final de `copilot-instructions.md`.
- **CRÍTICO**: no aplicar transformaciones a `df` durante el EDA salvo aprobación expresa del usuario. Cualquier transformación aplicada debe documentarse en el informe final (qué, por qué, resultado).

## 1. Regla obligatoria sobre el plan y su seguimiento

Al entrar en la Fase de PLAN:

- Construir la lista de tareas (ver sección 4.2) y mostrarla en el chat como checklist Markdown, por ejemplo:
```
- [ ] Tarea 1 — Carga del tablón y tipificación de variables
- [ ] Tarea 2 — EDA numéricas (análisis estadístico)
- [ ] Tarea 3 — EDA numéricas (análisis gráfico)
...
```
- Este checklist **es la única fuente de verdad del progreso**: se reimprime actualizado (`[x]` en las completadas) cada vez que se cierra una tarea, para que el usuario vea siempre el estado.
- No se debe empezar a ejecutar tareas hasta que el usuario confirme el plan.

## 2. Tipos de variables y filosofía general

Clasificar las columnas del dataframe en:

- **Numéricas discretas**: numéricas (habitualmente enteras) con cardinalidad baja respecto al nº de filas. Heurística: enteras con nº de valores distintos ≤ `min(20, 0.05 * nº_filas)`, salvo indicación del usuario.
- **Numéricas continuas**: resto de numéricas (int/float) no discretas.
- **Categóricas**: discretas de tipo `object`/`category`.
- **Booleanas**: tratadas igual que las categóricas.
- **Alta cardinalidad**: categóricas cuya cardinalidad supere `umbral_alta_cardinalidad` (o heurística según tamaño del dataset).
- **Texto**: `object` con muchos valores distintos y patrón de texto libre.
- **Fechas**: tipo `datetime`.

Jerarquía de análisis: Numéricas → Categóricas (incl. booleanas) → Alta cardinalidad → Texto → Fechas.

Dentro de cada tipo, siempre: **Tipo de variable → Análisis estadístico → Análisis gráfico**, cada uno como paso separado, avanzando solo cuando el usuario confirma.

Trabajar de forma **industrializada**: el mismo código debe funcionar igual con 3 variables que con 300, con salidas homogéneas.

## 3. Estilo gráfico con seaborn

Todos los gráficos con seaborn, estilo limpio:
```python
import seaborn as sns
import matplotlib.pyplot as plt

custom_params = {"axes.spines.right": False, "axes.spines.top": False}
sns.set_theme(style="ticks", rc=custom_params)
```

Reglas:

- **Un gráfico por fila**: una variable → una figura → un gráfico. Nunca grids de varias columnas.
- `plt.tight_layout()` antes de `plt.show()`.
- Evitar warnings de seaborn: no usar `palette=` sin `hue`; si se usa `palette`, patrón `hue=col, legend=False, palette="viridis"`, o `color = sns.color_palette("viridis", 1)[0]`.
- Los gráficos de barras deben incluir la etiqueta de porcentaje en cada barra (sobre el total de observaciones, excluyendo NaN si procede).
- **Nunca boxplots.** Numéricas discretas → barras. Numéricas continuas → KDE. Categóricas/booleanas/alta cardinalidad → barras. Texto → histograma de longitud. Fechas → conteos por periodo.
- **Además de `plt.show()` en el notebook**, guardar cada figura con `plt.savefig("<ruta_temporal>/<nombre_variable>.png", dpi=100, bbox_inches="tight")` justo antes del `plt.show()`, usando una carpeta temporal de trabajo (p. ej. la ruta de scratchpad de la sesión). Tras ejecutar el notebook, releer esa PNG con `Read` para inspeccionar visualmente el gráfico antes de redactar las conclusiones.

## 4. Fase de PLAN

### 4.1. Carga del tablón y tipificación de variables (Tarea 1)

- Confirmar carga de `df` (shape, `df.head()`).
- Tipificar columnas por tipo de variable, con reglas automáticas razonables.
- Resumen por tipo (nº de columnas por tipo).
- Detección preliminar: columnas constantes, alto % de missing, cardinalidad muy alta.
- Lista inicial de alertas automáticas (bullets).

### 4.2. Construcción del plan

Checklist obligatoria mínima (ampliable si el proyecto lo requiere), en este orden:

1. **Tarea 1** — Carga del tablón y tipificación de variables (4.1).
2. **Tarea 2** — EDA numéricas (análisis estadístico): `describe()` extendido (percentiles 1/5/25/50/75/95/99), % missing, skewness, kurtosis, outliers (IQR), separando discretas/continuas.
3. **Tarea 3** — EDA numéricas (análisis gráfico): discretas → barras; continuas → KDE; 1 figura por variable; agrupar en bloques si hay muchas, pero siempre 1 variable → 1 figura.
4. **Tarea 4** — EDA categóricas/booleanas (análisis estadístico): frecuencias abs/rel, % missing, cardinalidad, categoría dominante, rare labels, distinción normal/alta cardinalidad.
5. **Tarea 5** — EDA categóricas/booleanas (análisis gráfico): barras, 1 figura por variable, etiquetas de %, respetando `max_categorias_grafico` si se fija.
6. **Tarea 6** — EDA alta cardinalidad (análisis estadístico): frecuencias, concentración top-N, sospecha de IDs/códigos.
7. **Tarea 7** — EDA alta cardinalidad (análisis gráfico): barras solo con el top `max_top_categorias_alta_cardinalidad` (20 por defecto).
8. **Tarea 8** — EDA texto (estadístico y gráfico): longitud media/mediana/desv., % vacíos o muy cortos, histograma de longitudes. Sin wordclouds ni análisis semántico.
9. **Tarea 9** — EDA fechas (estadístico y gráfico): fecha mín/máx, distribución por año/mes, fechas anómalas, conteos por periodo.
10. **Tarea 10** — Consolidación e informe final: conclusiones + observaciones del usuario, informe en Markdown, guardado en `06_resultados/EDA/EDA_report.md`.

Pasos:

- Mostrar el checklist completo en el chat (formato de la sección 1).
- Preguntar si el usuario quiere añadir, quitar o reordenar tareas.
- Ajustar el checklist según indique.
- **No empezar a ejecutar hasta que el usuario confirme el plan.**

## 5. Fase de ejecución tarea por tarea

Para cada tarea, ciclo obligatorio:

1. **Escribir el código** en el notebook con `NotebookEdit` (insertar celdas nuevas, código claro y modular sobre `df`; en tareas de gráficos, asegurar el tema seaborn aplicado y el guardado a PNG de la sección 3).
2. **Ejecutar** todo el notebook vía Bash:
```
jupyter nbconvert --to notebook --execute --inplace "03_notebooks/03_EDA.ipynb"
```
   Si falla, leer el error (nbconvert lo deja en stderr) y corregir el código antes de continuar.
3. **Leer los resultados**: releer `03_notebooks/03_EDA.ipynb` con `Read` para ver tablas/salidas de texto; para cada gráfico nuevo, `Read` sobre el PNG correspondiente en la carpeta temporal para verlo de verdad antes de interpretarlo.
4. **Conclusiones del agente para esta tarea:** bloque de bullets (sesgos, outliers, missing elevado, distribuciones multimodales, categoría dominante, rare labels, cardinalidad excesiva, concentración top-N, sospecha de ID, longitudes anómalas, % vacíos, rangos temporales, fechas fuera de rango, etc. según el tipo de variable de la tarea).
5. **Alertas detectadas en esta tarea:** bloque de bullets (missing > umbral, variables constantes/casi constantes, cardinalidad excesiva, sesgo extremo, textos vacíos en gran proporción, fechas imposibles).
6. **Pedir al usuario que revise** resultados y conclusiones, ofreciendo: profundizar en variables concretas, gráficos/filtros adicionales, añadir observaciones. Las observaciones del usuario se registran estructuradas: **Tarea, Tipo de variable, Columnas afectadas, Contenido** — guardarlas en una lista interna en el propio chat (o en un bloque de notas) para usarlas en el informe final.
7. **Si el usuario pide profundizar**: nueva subtarea interna (sin tocar el checklist salvo que el usuario lo pida explícitamente), repetir escritura/ejecución/lectura/interpretación.
8. **Cierre de la tarea**: resumir puntos clave, preguntar explícitamente si se avanza a la siguiente tarea, marcar `[x]` en el checklist y reimprimirlo, y solo entonces continuar.

**Nunca ejecutar varias tareas seguidas sin interacción; la ejecución es siempre secuencial y guiada por el usuario.**

## 6. Finalización: informe, guardado de df y actualización de copilot-instructions.md

Al completar todas las tareas (incluida la Tarea 10):

### 6.1. Informe de EDA en Markdown

Construir con toda la información recopilada (conclusiones, alertas, observaciones estructuradas del usuario), incluyendo como mínimo:

- **Portada**: nombre del dataframe (`df`), nº filas/columnas, resumen de variables por tipo.
- **Secciones por tipo**: Numéricas (discretas/continuas), Categóricas/booleanas, Alta cardinalidad, Texto, Fechas — cada una con sus estadísticas clave, hallazgos, alertas y observaciones del usuario.
- **Transformaciones aplicadas** (si las hay): qué, por qué, resultado.
- **Resumen global**: variables relevantes, riesgos de calidad/interpretabilidad, sugerencias de acciones futuras (sin implementarlas).

Guardar en `06_resultados/EDA/EDA_report.md` con `Write` (crear la carpeta `06_resultados/EDA` si no existe). Mostrar al usuario un resumen breve y la ruta final. Preguntar si quiere añadir algún comentario final y, si es así, actualizar el informe con `Edit`.

### 6.2. Guardar el dataframe resultante de EDA

1. Confirmar que `df` es la versión final acordada (original o con pequeñas transformaciones pactadas).
2. Añadir celda y ejecutar (mismo ciclo de la sección 5):
```python
df.to_pickle("../02_datos/03_Entrenamiento/03_train_tablon_eda.pkl")
```
3. Confirmar en la relectura del notebook que no hay errores.

### 6.3. Actualizar `.github/copilot-instructions.md`

1. Leer el archivo con `Read`.
2. Localizar `## ESTADO ACTUAL DEL PROYECTO` (sobrescribir si existe; añadir al final si no).
3. Nuevo formato exacto:
```markdown
## ESTADO ACTUAL DEL PROYECTO

**Dataframe actual**: `../02_datos/03_Entrenamiento/03_train_tablon_eda.pkl`

**Estructura del dataframe**:
```
[salida completa de df.info()]
```
```
4. Para obtener `df.info()`: añadir celda `df.info()`, ejecutar (ciclo sección 5), leer la salida del notebook y pegarla literal entre los triple backtick.
5. Aplicar la actualización con `Edit`, sin dejar información obsoleta de dataframes anteriores en esa sección.

### 6.4. Resumen final y cierre

Mostrar:
- Ruta del informe: `06_resultados/EDA/EDA_report.md`.
- Ruta del dataframe actual: `../02_datos/03_Entrenamiento/03_train_tablon_eda.pkl`.
- Confirmación de que `copilot-instructions.md` está actualizado.

Indicar que la fase de EDA ha terminado y el proyecto está listo para la siguiente fase (preparación de datos o selección de variables). **Quedar a la espera de instrucciones explícitas del usuario: no encadenar ninguna tarea adicional ni iniciar otra fase por cuenta propia.**

## 7. Uso de notebooks y plantillas

Trabajar siempre mediante el ciclo `NotebookEdit` → Bash (`jupyter nbconvert --execute --inplace`) → `Read` (notebook + PNGs de gráficos).

Si el usuario aporta un notebook de plantilla para EDA: analizar su estructura, identificar buenas prácticas y rutinas reutilizables, adaptarlas respetando la tipificación de variables, el flujo secuencial y la ejecución tarea a tarea guiada por el checklist. Si incluye análisis adicionales interesantes, proponerlos como subtareas opcionales.

## 8. Estilo de trabajo

- Siempre **interactivo y explicativo**; nunca asumir decisiones que cambien la interpretación sin validación del usuario.
- Resaltar variables problemáticas o interesantes; sintetizar resultados extensos en bullets claros.
- Registrar cuidadosamente conclusiones automáticas y observaciones estructuradas del usuario, y reflejarlas en el informe final.
- **No modificar archivos externos** salvo: `06_resultados/EDA/EDA_report.md`, `.github/copilot-instructions.md`, `../02_datos/03_Entrenamiento/03_train_tablon_eda.pkl`, `03_notebooks/03_EDA.ipynb`, y otros cambios que el usuario pida explícitamente.
- Resultados siempre en formato legible: tablas `pandas.DataFrame`, tablas Markdown, resúmenes con viñetas.
- Responder siempre en **español de España**.
