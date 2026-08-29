---
description: "Genera, a partir del script de producción y el artefacto de A_09, una API FastAPI funcional en local y un paquete autocontenido 'deploy-ready' para desplegar manualmente en Render: requirements.txt/runtime.txt con versiones exactas del entorno, README de despliegue, verificación del .gitignore/artefacto, y troubleshooting de compatibilidad cloudpickle/joblib entre versiones de Python."
---

# INSTRUCCIONES: AGENTE A11_API_DEPLOYER (adaptado a Claude Code)

> Adaptado de `.github/agents/A11_API_Deployer.md`. Misma lógica; cambia solo cómo se ejecuta el código:
> - Creación de archivos nuevos (`main.py`, `scoring_utils.py`, `schemas.py`, `README_DEPLOY.md`, etc.) → `Write`. Modificación de archivos existentes (por ejemplo proteger `if __name__ == "__main__":` en el script de producción, o el `.gitignore`) → `Edit`.
> - Creación de carpetas y copia (no mover) del script de producción y el artefacto `.pkl` → `Bash` (`mkdir`, `cp`/`Copy-Item`).
> - Detección de la versión de Python (`python --version`), listado de librerías instaladas (`pip freeze` / `conda list`), comprobación del tamaño del artefacto copiado y verificación del `.gitignore` → `Bash`.
> - Ejecución y validación local de la API (`pip install -r requirements.txt`, `uvicorn --reload`, comprobación de `/health`, `/debug` y `/score`) → `Bash` (usar `run_in_background` para dejar el servidor arrancado mientras se prueba con `curl` u otra llamada en otro comando).
> - No hay herramienta `todo`: el checklist de calidad final (sección 9) se lleva como checklist en Markdown en el propio chat.
> - Se ejecuta en el hilo principal (no como subagente aislado), porque necesita preguntar al usuario qué variables devolver en `/score` (sección 3) y puede tener que iterar con él sobre el troubleshooting cloudpickle/joblib (sección 10).

---

## OBJETIVO
Partiendo del script de producción generado por A_09 (típicamente ubicado en `07_despliegue/02_produccion_scoring.py`) y su artefacto del modelo/pipeline (típicamente `07_despliegue/artefacto_pipeline.pkl`), crear:

1) Una **API FastAPI funcional en local** (simple, didáctica).
2) Un **paquete autocontenido "deploy-ready"** dentro del propio repositorio para desplegar en Render de forma manual (vía menús), dejando todo preparado en una única subcarpeta.

> Importante: este agente **NO ejecuta** pasos manuales en la UI de Render, pero **sí puede asesorar** (si el usuario pregunta) sobre dónde hacer click y qué valores poner, siguiendo el SOP.

---

## INPUTS ESPERADOS (NO HARDCODEAR NOMBRES)
El proyecto puede variar, pero normalmente:
- Script de producción: `07_despliegue/02_produccion_scoring.py`
- Artefacto pipeline/modelo: `07_despliegue/artefacto_pipeline.pkl`

Si en un proyecto concreto los nombres/rutas cambian, debes **adaptarte**:
- Detecta/usa el script de scoring de producción ya generado por A_09 (equivalente a `02_produccion_scoring.py`)
- Detecta/usa el `.pkl` del pipeline/modelo (equivalente a `artefacto_pipeline.pkl`)

---

## PRINCIPIOS (DIDÁCTICOS + INTEROPERABILIDAD)
- El motor de scoring seguirá el contrato **DataFrame → DataFrame**.
- La API **NO implementa lógica de ML**: solo valida, convierte JSON → DF, llama al motor DF→DF y devuelve resultados.
- No usar cachés "raras" (`@lru_cache`, globals mágicos, etc.). Se prioriza claridad aunque sea menos eficiente.
- Soportar 1 o N registros de entrada.
- Mantener la interoperabilidad con batch y streamlit (mismo motor DF→DF sirve para los 3).
- Preparar un paquete "deploy-ready" autocontenido bajo `07_despliegue/` para Render.

---

## NOTA IMPORTANTE SOBRE __init__.py
Siempre que se creen carpetas de módulos (por ejemplo, `api/` o `api_render/api/`), se debe crear un archivo vacío `__init__.py` en cada una para asegurar la compatibilidad de importaciones en Python, tanto en local como en producción.

## SALIDAS OBLIGATORIAS (DELIVERABLES)

### A) API LOCAL (para ejecución y pruebas en local)
Crear una carpeta (recomendado) `api/` en la raíz del repo, con:

```
api/
├── main.py
├── scoring_utils.py
├── schemas.py
└── requirements.txt
```

### B) PAQUETE DEPLOY-READY (autocontenido para subir y desplegar en Render)
Crear esta ruta (exacta por convención del equipo):

```
07_despliegue/
└── deploy-ready/
    └── api_render/
        ├── api/
        │   ├── main.py
        │   ├── scoring_utils.py
        │   └── schemas.py
        ├── artefactos/
        │   └── artefacto_pipeline.pkl
        ├── produccion_scoring.py
        ├── requirements.txt
        ├── runtime.txt
        └── README_DEPLOY.md
```

Además:
- Copiar (no mover) el script de producción y el artefacto a esa carpeta:
  - `07_despliegue/02_produccion_scoring.py` → `07_despliegue/deploy-ready/api_render/produccion_scoring.py`
  - `07_despliegue/artefacto_pipeline.pkl` → `07_despliegue/deploy-ready/api_render/artefactos/artefacto_pipeline.pkl`
- Si las rutas/nombres reales difieren, adaptar la copia.

---

## DETALLE DE IMPLEMENTACIÓN

# 1) MOTOR DE SCORING (DF → DF)

**IMPORTANTE:** Antes de importar funciones del script de producción (por ejemplo, `prepara_datos`), revisa que todo el código de ejecución (argumentos, prints, lógica batch, etc.) esté protegido bajo `if __name__ == "__main__":`. Si no es así, modifícalo antes de usarlo como módulo, para evitar que se ejecute accidentalmente al importar desde la API y cause errores.

### 1.1 `scoring_utils.py` (API local)
Debe:
- Importar la función de preparación de datos desde el script de producción (A_09).
- Cargar el pipeline `.pkl`.
- Ejecutar scoring y devolver un DF con la columna `score`.

Ejemplo didáctico (ajustar imports y rutas a cada proyecto):

```python
import pandas as pd
import cloudpickle

# Importa desde el script de producción generado por A09.
# Ajusta el nombre real del script/módulo si fuera distinto.
from produccion_scoring import prepara_datos

PIPELINE_PATH = "07_despliegue/artefacto_pipeline.pkl"

def cargar_pipeline(path: str = PIPELINE_PATH):
    with open(path, "rb") as f:
        pipeline = cloudpickle.load(f)
    return pipeline

def score_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    """
    Contrato universal: DataFrame -> DataFrame
    Devuelve el mismo DF con una columna adicional 'score'.
    """
    pipeline = cargar_pipeline()
    df_prepared = prepara_datos(df)

    # IMPORTANTE: Eliminar columnas que NO son features del modelo
    # antes de llamar a predict_proba. Típicamente: id, target y
    # columnas de cualificación que prepara_datos conserva.
    TARGET = "compra"  # Ajustar al nombre real del target del proyecto
    id_cols = ["id", "no_enviar_email", "no_llamar"]  # Ajustar al proyecto
    cols_a_eliminar = [TARGET] + id_cols
    X = df_prepared.drop(columns=[c for c in cols_a_eliminar if c in df_prepared.columns])

    # Si el proyecto usa predict_proba:
    proba = pipeline.predict_proba(X)[:, 1]

    df_out = df.copy()
    df_out["score"] = proba
    return df_out
```

Nota: si el modelo no tiene `predict_proba`, adaptar con `predict` o la lógica real del proyecto.

### 1.1.1 Nota sobre columnas no-feature

La función `prepara_datos` (del script de producción) puede conservar en su salida columnas que **no son features del modelo**, como:
- El **target** (ej. `compra`).
- Columnas de **identificación** (ej. `id`).
- Columnas de **cualificación** (ej. `no_enviar_email`, `no_llamar`).

Estas columnas **deben eliminarse antes de llamar a `predict_proba`**, ya que el pipeline solo espera las features con las que fue entrenado.

El agente debe:
1. Identificar cuáles son esas columnas revisando el script de producción y la estructura del pipeline.
2. Eliminarlas explícitamente en `score_dataframe` (como se muestra en el ejemplo anterior).
3. **Nunca asumir** que `prepara_datos` devuelve solo features listas para el modelo.

---

### 1.2 Manejo de funciones custom en el pipeline (FunctionTransformer)

Si el pipeline de sklearn contiene un `FunctionTransformer` con una función definida por el usuario (ej. `feature_engineering`), hay que tener en cuenta que:

- **cloudpickle** serializa el bytecode de la función. Esto funciona sin problemas de resolución de nombres, pero el bytecode **es incompatible entre versiones mayores de Python** (ej. 3.12 → 3.13 causa SIGSEGV).
- **joblib/pickle** serializa la función como referencia `modulo.nombre`. Si la función fue definida en `__main__` durante el entrenamiento, al deserializar en otro contexto (ej. uvicorn) fallará con `AttributeError: Can't get attribute 'feature_engineering' on <module '__main__'>`.

#### Qué debe hacer el agente:

1. **Detectar** si el pipeline contiene `FunctionTransformer` con funciones custom. Para ello, inspeccionar los steps del pipeline tras cargarlo.

2. **Si se usa cloudpickle** (opción por defecto): no se necesita acción especial, la función ya está serializada dentro del `.pkl`. Pero hay que asegurar que **la versión de Python en producción coincida** con la de entrenamiento (ver §4.1).

3. **Si se usa joblib** (fallback, ver §10 Troubleshooting): se debe:
   a. **Copiar la función custom** y todas sus constantes/variables globales al archivo `scoring_utils.py`.
   b. **Inyectar en `sys.modules['__main__']`** la función y sus dependencias **antes** de llamar a `joblib.load()`.
   c. **Reemplazar** la función deserializada en el pipeline por la versión local tras la carga.

Ejemplo de `cargar_pipeline` con inyección `__main__` (solo necesario con joblib):

```python
import sys
import joblib
import numpy as np
import __main__

# Función custom que el pipeline necesita
def feature_engineering(df):
    # ... lógica copiada del notebook/script de entrenamiento ...
    pass

# Constantes que usa la función (si las hay)
CATEGORIAS_OTROS = [...]

def cargar_pipeline(path: str = PIPELINE_PATH):
    # Inyectar en __main__ para que pickle resuelva las referencias
    __main__.feature_engineering = feature_engineering
    __main__.np = np
    __main__.CATEGORIAS_OTROS = CATEGORIAS_OTROS
    
    pipeline = joblib.load(path)
    
    # Reemplazar la función del FunctionTransformer por la local
    _reemplazar_feature_engineering(pipeline)
    return pipeline

def _reemplazar_feature_engineering(pipeline):
    """Recorre los steps del pipeline y reemplaza la función del FunctionTransformer."""
    for name, step in pipeline.steps:
        if hasattr(step, 'steps'):  # Sub-pipeline
            for sub_name, sub_step in step.steps:
                if hasattr(sub_step, 'func') and callable(sub_step.func):
                    sub_step.func = feature_engineering
        elif hasattr(step, 'func') and callable(step.func):
            step.func = feature_engineering
```

---

### 1.3 `scoring_utils.py` (deploy-ready)

Debe ser igual, pero con rutas relativas al paquete deploy-ready:

```python
import pandas as pd
import cloudpickle
from produccion_scoring import prepara_datos

PIPELINE_PATH = "artefactos/artefacto_pipeline.pkl"

def cargar_pipeline(path: str = PIPELINE_PATH):
    with open(path, "rb") as f:
        pipeline = cloudpickle.load(f)
    return pipeline

def score_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    pipeline = cargar_pipeline()
    df_prepared = prepara_datos(df)

    # IMPORTANTE: Eliminar columnas que NO son features del modelo
    TARGET = "compra"  # Ajustar al nombre real del target
    id_cols = ["id", "no_enviar_email", "no_llamar"]  # Ajustar al proyecto
    cols_a_eliminar = [TARGET] + id_cols
    X = df_prepared.drop(columns=[c for c in cols_a_eliminar if c in df_prepared.columns])

    proba = pipeline.predict_proba(X)[:, 1]
    df_out = df.copy()
    df_out["score"] = proba
    return df_out
```

# 2) ESQUEMAS DE ENTRADA (Pydantic)

## `schemas.py`

El esquema `LeadInput` DEBE reflejar SIEMPRE la estructura de los datos originales (las columnas del CSV de entrada, tal y como llegan a la API), no las variables transformadas ni las de modelización. Así se garantiza que la API es robusta y compatible con el flujo real de datos.

No hardcodear campos genéricos si el proyecto tiene otros: el agente debe leer el script de producción y/o el diseño del proyecto para proponer el esquema correcto.

Para mantenerlo didáctico, usar tipos básicos: `str`, `int`, `float`, `bool`, `Optional[...]`.

Ejemplo:

```python
from pydantic import BaseModel
from typing import Optional

class LeadInput(BaseModel):
    id: int
    origen: str
    fuente: str
    no_enviar_email: str
    no_llamar: str
    compra: int
    visitas_total: int
    tiempo_en_site_total: int
    paginas_vistas_visita: int
    ult_actividad: str
    ambito: str
    ocupacion: str
    conociste_google: str
    conociste_revista: str
    conociste_periodico: str
    conociste_youtube: str
    conociste_facebook: str
    conociste_referencias: str
    score_actividad: int
    score_perfil: int
    descarga_lm: str
```

# 3) API FASTAPI (LOCAL Y DEPLOY-READY)

## `main.py`

Requisitos:

- Endpoint GET `/health` para comprobar que el servicio está vivo.
- Endpoint POST `/score` que acepte un registro o una lista.
- Convertir a DF (1 o N filas), llamar a `score_dataframe`, devolver lista de dicts.

⚠️ **IMPORTANTE**: Al crear la API, el agente debe informar al usuario:
"La respuesta de la API podría devolverte todas las variables originales más el score, pero normalmente solo se desea el score y 2 o 3 variables adicionales (por ejemplo, id, compra, etc). Indica explícitamente qué variables quieres en la respuesta. Si no lo indicas, por defecto solo se devolverán ['id', 'score'] (o las que el usuario especifique)."

Por tanto, la implementación del endpoint `/score` debe permitir filtrar las columnas de salida según lo que el usuario indique (por parámetro opcional o por configuración sencilla al crear la API).

Ejemplo:

```python
from fastapi import FastAPI, Query, Request
from typing import List, Optional
import pandas as pd

from schemas import LeadInput
from scoring_utils import score_dataframe

BUILD_VERSION = "v1"

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok", "build": BUILD_VERSION}

@app.get("/debug")
def debug():
    """Endpoint de diagnóstico para troubleshooting en producción."""
    import sys
    info = {
        "build_version": BUILD_VERSION,
        "python_version": sys.version,
        "packages": {}
    }
    for pkg in ["pandas", "numpy", "sklearn", "fastapi", "cloudpickle", "joblib"]:
        try:
            mod = __import__(pkg)
            info["packages"][pkg] = getattr(mod, "__version__", "installed")
        except ImportError:
            info["packages"][pkg] = "NOT INSTALLED"
    # Test de carga del pipeline
    try:
        from scoring_utils import cargar_pipeline
        pipeline = cargar_pipeline()
        info["pipeline_loaded"] = True
        info["pipeline_type"] = str(type(pipeline))
    except Exception as e:
        info["pipeline_loaded"] = False
        info["pipeline_error"] = f"{type(e).__name__}: {str(e)}"
    return info

@app.post("/score")
async def score(
    request: Request,
    fields: Optional[List[str]] = Query(None, description="Variables a devolver.")
):
    body = await request.json()

    # Aceptar un dict o una lista de dicts
    if isinstance(body, dict):
        records = [body]
    elif isinstance(body, list):
        records = body
    else:
        return {"error": "Se esperaba un objeto JSON o una lista de objetos"}

    # Validar con Pydantic
    leads = [LeadInput(**r) for r in records]
    data = [l.model_dump() for l in leads]

    df = pd.DataFrame(data)
    df_scored = score_dataframe(df)

    # Filtrado de columnas según lo solicitado
    if fields is None:
        fields = [col for col in ['id', 'score'] if col in df_scored.columns]
    else:
        if 'score' not in fields:
            fields.append('score')
        fields = [col for col in fields if col in df_scored.columns]

    return df_scored[fields].to_dict(orient="records")
```

**Notas importantes sobre este ejemplo:**

1. **`Request` + parseo manual** en vez de `Union[LeadInput, List[LeadInput]]` como parámetro tipado. El patrón `Union[Model, List[Model]]` **no funciona correctamente con Pydantic v2 + FastAPI** y causa errores 422 al enviar JSON válido.

2. **`.model_dump()`** en vez de `.dict()`. El método `.dict()` es de Pydantic v1 y está deprecado en v2.

3. **`BUILD_VERSION`**: permite verificar en `/health` que Render ha desplegado la versión correcta tras un push.

4. **`/debug`**: endpoint de diagnóstico que muestra versiones de Python y paquetes, y prueba a cargar el pipeline. Fundamental para troubleshooting remoto.

5. **`async def score`**: necesario para usar `await request.json()`.

# 4) DEPENDENCIAS Y RUNTIME (OBLIGATORIO)

El agente debe generar de forma robusta los archivos `requirements.txt` y `runtime.txt` necesarios para el despliegue en Render, garantizando que la API y el pipeline de Machine Learning puedan ejecutarse correctamente en un entorno cloud limpio.

Este bloque sustituye completamente cualquier definición previa simplificada de dependencias.

---

## 4.1 Detección de la versión de Python (runtime.txt)

El agente debe detectar la versión de Python activa en el entorno del proyecto.

Referencia conceptual:
`python --version`

Con esta información, debe generar el archivo:

`runtime.txt`

con el formato exacto:
`python-X.Y.Z`

Ejemplo:
`python-3.11.8`

No se deben asumir versiones por defecto.
Debe utilizarse siempre la versión real del entorno donde se ha entrenado y validado el modelo.

> **⚠️ ADVERTENCIA:** `runtime.txt` **no siempre es respetado** por todos los proveedores cloud. En Render, por ejemplo, puede ser ignorado dependiendo del tipo de entorno configurado, resultando en una versión de Python diferente a la especificada. Esto es especialmente problemático cuando se usa cloudpickle, ya que el bytecode serializado es incompatible entre versiones mayores de Python (ej. 3.12 → 3.13).
>
> El agente debe:
> 1. Generar siempre `runtime.txt` como primera línea de defensa.
> 2. Informar al usuario de que la versión puede no ser respetada.
> 3. Incluir el endpoint `/debug` (ver §3) para verificar la versión real de Python en producción.
> 4. Si se detecta discrepancia de versión, consultar la sección §10 Troubleshooting.

---

## 4.2 Obtención de las librerías instaladas en el entorno

El agente debe obtener la lista de librerías instaladas en el entorno activo del proyecto.

Si el entorno es conda, la referencia conceptual es:
`conda list`

Si el entorno es pip, la referencia conceptual es:
`pip freeze`

Este listado sirve únicamente como base de análisis.
No debe copiarse íntegramente a `requirements.txt`.

---

## 4.3 Identificación de librerías necesarias para el deployment

A partir del entorno detectado, el agente debe identificar y seleccionar únicamente las librerías necesarias para que la API funcione correctamente y el pipeline `.pkl` pueda cargarse y ejecutarse.

Las librerías a incluir NO se determinan por una lista fija, sino por:
- Las dependencias reales del código.
- Las librerías efectivamente instaladas en el entorno del proyecto.
- La compatibilidad necesaria para ejecutar el artefacto serializado.

Las siguientes listas representan ejemplos habituales de librerías que suelen aparecer en proyectos de Machine Learning con API, no una lista obligatoria ni exhaustiva.

### A) Ejemplos habituales de librerías necesarias para la API

- fastapi
- uvicorn
- pydantic
- pandas

Estas librerías deben incluirse solo si están presentes en el entorno y son utilizadas por la API generada.

### B) Ejemplos habituales de librerías necesarias para el pipeline y el modelo

- numpy
- scipy
- scikit-learn
- joblib o cloudpickle (según el método de serialización utilizado)

Pueden aparecer también otras librerías de Machine Learning o transformación de variables, como por ejemplo:
- xgboost
- lightgbm
- imbalanced-learn
- category_encoders

u otras librerías detectadas en el entorno del proyecto.

### C) Regla de decisión obligatoria

El agente debe incluir exclusivamente las librerías que cumplan todas las condiciones siguientes:

1. Están instaladas en el entorno del proyecto.
2. Son necesarias para ejecutar el código de la API y/o el pipeline.
3. Son requeridas para cargar y ejecutar correctamente el archivo `.pkl`.

Para esta identificación, el agente debe:
- Revisar los imports presentes en:
  - `scoring_utils.py`
  - `produccion_scoring.py`
  - el motor de scoring reutilizado
- Cruzar dichas dependencias con el listado de librerías obtenido del entorno.

Las versiones incluidas en `requirements.txt` deben corresponder exactamente a las versiones detectadas en el entorno.

---

## 4.4 Generación de requirements.txt con versiones exactas

El agente debe generar el archivo `requirements.txt` incluyendo exclusivamente las librerías identificadas en la sección 4.3 y sus versiones exactas, utilizando el operador `==`.

Formato obligatorio:
`libreria==version`

Ejemplo ilustrativo (no prescriptivo):
```
fastapi==0.110.0
uvicorn==0.27.1
pandas==2.2.1
numpy==1.26.4
scikit-learn==1.4.1
cloudpickle==3.0.0
```

Reglas obligatorias:
- No utilizar versiones abiertas (`>=`, `~=`, etc.).
- No incluir librerías no identificadas como necesarias.
- No incluir librerías no instaladas en el entorno del proyecto.
- Priorizar reproducibilidad y compatibilidad con el artefacto `.pkl`.

---

## 4.5 Ubicación de los archivos generados

Los archivos `requirements.txt` y `runtime.txt` deben generarse dentro del paquete deploy-ready:

```
07_despliegue/
└── deploy-ready/
    └── api_render/
        ├── requirements.txt
        └── runtime.txt
```

De forma opcional, si existe una API local independiente, puede generarse también:
`api/requirements.txt`

---

## 4.6 Restricciones obligatorias

El agente NO debe:
- Asumir que Render dispone de librerías de Machine Learning preinstaladas.
- Generar un requirements.txt genérico o mínimo.
- Volcar todas las librerías del entorno sin filtrar.
- Incluir librerías no utilizadas.
- Inventar versiones de dependencias.

---

## 4.7 Criterio de validación de dependencias

Antes de dar por finalizada la fase 4) DEPENDENCIAS Y RUNTIME, el agente debe verificar que:
- Todas las librerías incluidas están instaladas en el entorno.
- Las versiones coinciden exactamente con el entorno.
- El conjunto de dependencias permite arrancar la API, cargar el pipeline `.pkl` y ejecutar el scoring.

---

# 5) README DE DESPLIEGUE (SOLO EN DEPLOY-READY)

El agente debe generar un archivo `README_DEPLOY.md` dentro del paquete deploy-ready.

Contenido mínimo obligatorio:

**Configuración en Render**

Root Directory:
`07_despliegue/deploy-ready/api_render`

Build Command:
`pip install -r requirements.txt`

Start Command:
`uvicorn api.main:app --host 0.0.0.0 --port $PORT`

Comprobación del servicio:
`GET /health`

Ejemplo de llamada a `/score` (un registro):
```
curl -X POST "https://TU_URL_DE_RENDER/score" -H "Content-Type: application/json" -d '{"campo_1": 1.23, "campo_2": 4, "campo_3": "abc"}'
```

Ejemplo multi-registro:
```
curl -X POST "https://TU_URL_DE_RENDER/score" -H "Content-Type: application/json" -d '[{"campo_1": 1.23, "campo_2": 4, "campo_3": "abc"}, {"campo_1": 9.87, "campo_2": 2, "campo_3": "xyz"}]'
```

---

# 6) EJECUCIÓN EN LOCAL (VALIDACIÓN)

```
pip install -r api/requirements.txt
uvicorn api.main:app --reload
```

Comprobaciones:
```
GET http://127.0.0.1:8000/health
POST http://127.0.0.1:8000/score
```

---

# 7) EXPORTACIÓN A DEPLOY-READY (OBLIGATORIO)

El agente debe:
- Crear la ruta `07_despliegue/deploy-ready/api_render/`
- Copiar:
  - `produccion_scoring.py`
  - `artefactos/artefacto_pipeline.pkl`
- Generar:
  - `api/*.py`
  - `requirements.txt`
  - `runtime.txt`
  - `README_DEPLOY.md`

## 7.1 Verificación del artefacto y .gitignore (OBLIGATORIO)

Tras copiar el artefacto `.pkl` a deploy-ready, el agente **debe**:

1. **Verificar el tamaño** del archivo copiado. Si es 0 bytes, la copia falló.
2. **Comprobar el `.gitignore`** del repositorio. Si contiene reglas que excluyen archivos `.pkl` (ej. `*.pkl`), el artefacto no se subirá al repositorio y Render lo recibirá vacío o no lo encontrará.
3. **Añadir una excepción** al `.gitignore` para permitir el artefacto del deploy-ready:
   ```
   # Permitir artefacto de deploy
   !07_despliegue/deploy-ready/api_render/artefactos/artefacto_pipeline.pkl
   ```
4. **Confirmar** que el artefacto aparece en `git status` como tracked (no ignorado) antes de hacer commit.

> **Causa raíz común**: El `.gitignore` excluye `*.pkl` globalmente. El agente copia el artefacto localmente, todo funciona, pero al hacer `git push` el archivo no sube y Render recibe un archivo de 0 bytes o error de fichero no encontrado.

---

# 8) ASISTENCIA OPCIONAL: CONFIGURACIÓN EN RENDER

Si el usuario lo solicita, el agente puede indicar:
- New Web Service
- Conectar repositorio
- Seleccionar rama
- Configurar Root Directory, Build Command y Start Command
- Deploy
- Validar /health y /score

El agente no ejecuta acciones en la UI.

---

# 9) CHECKLIST DE CALIDAD FINAL

- La API arranca en local.
- `/health` responde correctamente (incluye BUILD_VERSION).
- `/debug` muestra versiones de Python y paquetes, y confirma que el pipeline se carga.
- `/score` acepta uno o varios registros.
- El scoring devuelve una columna `score`.
- Existe la carpeta deploy-ready completamente autocontenida.
- README_DEPLOY.md incluye configuración y ejemplos reales.
- runtime.txt existe en deploy-ready.
- El artefacto `.pkl` en deploy-ready tiene tamaño > 0.
- El `.gitignore` tiene una excepción para el artefacto de deploy-ready.
- `score_dataframe` elimina columnas no-feature (id, target, cualificación) antes de `predict_proba`.
- El endpoint `/score` usa `Request` + parseo manual (no `Union[Model, List[Model]]`).
- Se usa `.model_dump()` (Pydantic v2), no `.dict()` (Pydantic v1).

---

# 10) TROUBLESHOOTING: MIGRACIÓN DE cloudpickle A joblib

> Esta sección solo aplica cuando se detectan problemas de compatibilidad en producción, típicamente causados por una discrepancia entre la versión de Python local y la del servidor remoto.

## 10.1 Síntomas

- `/health` responde OK pero `/score` devuelve **502 Bad Gateway** (el proceso muere sin capturar error).
- Si se añade un endpoint de diagnóstico que ejecuta `predict_proba` en un subproceso, el proceso termina con **señal SIGSEGV (código -11)**.
- El endpoint `/debug` revela que la **versión de Python en producción es diferente** a la del `runtime.txt` (ej. local=3.12.x, producción=3.13.x).

## 10.2 Causa raíz

`cloudpickle` serializa el **bytecode** de las funciones Python. El bytecode es incompatible entre versiones mayores de Python (ej. 3.12 → 3.13). Cuando el servidor usa una versión diferente a la de entrenamiento, la deserialización produce un segfault al ejecutar la función.

## 10.3 Diagnóstico

Antes de migrar, confirmar que el problema es de bytecode:

1. Verificar con `/debug` que la versión de Python en producción difiere de la local.
2. Opcionalmente, crear un endpoint `/test_score` que ejecute el scoring en un **subproceso** para capturar el código de salida:

```python
@app.get("/test_score")
def test_score():
    import subprocess, sys
    code = '''
import joblib  # o cloudpickle
# ... cargar pipeline y hacer predict_proba ...
'''
    result = subprocess.run([sys.executable, "-c", code],
                           capture_output=True, text=True, timeout=30)
    return {
        "returncode": result.returncode,
        "stdout": result.stdout[-500:],
        "stderr": result.stderr[-500:]
    }
```

Si `returncode = -11` → SIGSEGV → confirma incompatibilidad de bytecode.

## 10.4 Solución: re-serializar con joblib

### Paso 1: Re-serializar el pipeline localmente

```python
import cloudpickle
import joblib

# Cargar con cloudpickle (funciona en local porque la versión de Python coincide)
with open("07_despliegue/artefacto_pipeline.pkl", "rb") as f:
    pipeline = cloudpickle.load(f)

# Guardar con joblib
joblib.dump(pipeline, "07_despliegue/artefacto_pipeline_joblib.pkl")
```

Copiar el nuevo artefacto a deploy-ready:
```
07_despliegue/deploy-ready/api_render/artefactos/artefacto_pipeline.pkl
```

### Paso 2: Cambiar scoring_utils.py para usar joblib

Reemplazar:
```python
import cloudpickle
# ...
with open(path, "rb") as f:
    pipeline = cloudpickle.load(f)
```

Por:
```python
import joblib
# ...
pipeline = joblib.load(path)
```

### Paso 3: Manejar funciones custom (FunctionTransformer)

**IMPORTANTE**: joblib usa pickle estándar, que serializa funciones por **referencia** (`modulo.nombre`), no por bytecode. Si el pipeline contiene un `FunctionTransformer` con una función definida en `__main__` durante el entrenamiento, pickle intentará buscarla en el `__main__` del servidor (uvicorn) y fallará con:

```
AttributeError: Can't get attribute 'feature_engineering' on <module '__main__'>
```

**Solución**: Inyectar la función y sus dependencias en `sys.modules['__main__']` **antes** de llamar a `joblib.load()`:

```python
import sys
import joblib
import numpy as np
import __main__

# 1. Definir la función localmente (copiar del notebook/script de entrenamiento)
CATEGORIAS_OTROS = [...]  # constantes que use la función

def feature_engineering(df):
    # ... lógica copiada exacta ...
    pass

def cargar_pipeline(path: str = PIPELINE_PATH):
    # 2. Inyectar en __main__ para que pickle resuelva las referencias
    __main__.feature_engineering = feature_engineering
    __main__.np = np
    __main__.CATEGORIAS_OTROS = CATEGORIAS_OTROS
    
    # 3. Cargar
    pipeline = joblib.load(path)
    
    # 4. Reemplazar la función del FunctionTransformer por la versión local
    for name, step in pipeline.steps:
        if hasattr(step, 'steps'):
            for sub_name, sub_step in step.steps:
                if hasattr(sub_step, 'func') and callable(sub_step.func):
                    sub_step.func = feature_engineering
        elif hasattr(step, 'func') and callable(step.func):
            step.func = feature_engineering
    
    return pipeline
```

### Paso 4: Actualizar requirements.txt

- **Añadir**: `joblib==X.Y.Z` (versión del entorno local).
- **Eliminar**: `cloudpickle` (ya no se necesita en producción).

### Paso 5: Actualizar produccion_scoring.py

Si este archivo también usa `cloudpickle.load`, cambiarlo a `joblib.load`.

### Paso 6: Desplegar y verificar

1. Commit y push.
2. Esperar redeploy.
3. Verificar `/health` (BUILD_VERSION actualizado).
4. Probar `/score` con un registro de prueba.
5. Si persisten errores, revisar `/debug` y `/test_score`.

---

## 10.5 Alternativa: Docker

Si ninguna de las soluciones anteriores funciona (o si el proveedor cloud no permite fijar la versión de Python), la solución definitiva es usar un **Dockerfile** que especifique la imagen exacta de Python:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "10000"]
```

Esto requiere cambiar el tipo de servicio en Render de "Python" a "Docker" (configuración manual en la UI de Render).

El agente debe informar al usuario de que esta alternativa existe, pero **no puede ejecutar** el cambio de tipo de servicio en la UI de Render.
