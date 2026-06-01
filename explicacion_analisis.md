# Explicacion detallada del notebook `analisis.ipynb`

Este documento explica, bloque por bloque, que hace el notebook de analisis de logs distribuidos. La idea es que puedas entender no solo que calcula cada celda, sino tambien por que esos calculos sirven para responder el challenge.

El objetivo general del notebook es detectar el momento en el que el sistema estuvo peor, identificar que servicio y endpoint fueron los mas afectados, y comparar ese incidente contra el comportamiento normal del resto del dataset.

---

## Bloque 1: Titulo y objetivo del analisis

```python
# Analisis de logs distribuidos
```

Este bloque es una celda Markdown. No ejecuta codigo, pero cumple una funcion importante: deja claro el objetivo del notebook.

El notebook no busca agregar logs ni modificar el dataset. Su objetivo es usar los datos existentes para encontrar evidencia cuantitativa sobre:

- cuando ocurrio el peor momento del sistema;
- que servicio fue el mas afectado;
- que endpoint estuvo mas comprometido;
- que mensajes dominaron durante el incidente;
- que cambio durante el incidente respecto al baseline.

Esto ayuda a que el analisis sea reproducible: cualquier persona que ejecute el notebook deberia llegar a las mismas conclusiones.

---

## Bloque 2: Imports

```python
import pandas as pd
import matplotlib.pyplot as plt
```

Este bloque importa las dos librerias permitidas por el challenge.

`pandas` se usa para trabajar con el CSV como una tabla de datos. Permite cargar el archivo, convertir columnas, agrupar por ventanas de tiempo, contar eventos, calcular porcentajes y crear tablas resumen.

`matplotlib.pyplot` se usa para crear los dos graficos obligatorios. En este notebook se usa para visualizar tendencias temporales: conteos por severidad y bad rate por ventana de 5 minutos.

---

## Bloque 3: Carga y validacion de datos

```python
df = pd.read_csv("server_logs.csv")
```

Esta linea carga el archivo `server_logs.csv` en un DataFrame llamado `df`.

Un DataFrame es una tabla en memoria: cada fila representa un log y cada columna representa un campo del log, como `timestamp_event`, `service_name`, `severity`, `endpoint`, `status_code` o `latency_ms`.

El archivo se lee desde la misma carpeta donde esta el notebook. Por eso la ruta usada es simplemente `"server_logs.csv"`.

### Validacion de columnas obligatorias

```python
required_columns = [
    "timestamp_event",
    "received_at",
    "service_name",
    "severity",
    "message",
    "method",
    "endpoint",
    "status_code",
    "latency_ms",
    "trace_id",
]
```

Esta lista contiene las columnas minimas que el challenge dice que deben existir.

Luego se calcula si falta alguna:

```python
missing_columns = sorted(set(required_columns) - set(df.columns))
if missing_columns:
    raise ValueError(f"Faltan columnas obligatorias: {missing_columns}")
```

La expresion `set(required_columns) - set(df.columns)` compara las columnas esperadas contra las columnas reales del CSV.

Si falta alguna columna obligatoria, el notebook se detiene con un error claro. Esto evita producir conclusiones falsas o incompletas con un dataset que no cumple el contrato.

### Conversion de tipos

```python
df["timestamp_event"] = pd.to_datetime(df["timestamp_event"], utc=True)
df["received_at"] = pd.to_datetime(df["received_at"], utc=True)
df["status_code"] = pd.to_numeric(df["status_code"], errors="coerce")
df["latency_ms"] = pd.to_numeric(df["latency_ms"], errors="coerce")
```

Estas lineas preparan las columnas para el analisis.

`timestamp_event` y `received_at` se convierten a fechas reales de Pandas. Esto es necesario para poder ordenar cronologicamente, crear ventanas de 5 minutos y graficar en el tiempo.

El parametro `utc=True` deja las fechas normalizadas en UTC. Esto evita confusiones de zona horaria.

`status_code` y `latency_ms` se convierten a valores numericos. Esto permite hacer comparaciones como `status_code >= 500` y calcular promedios de latencia.

`errors="coerce"` significa que si algun valor no se puede convertir, Pandas lo transforma en `NaN` en lugar de romper inmediatamente. Es una forma controlada de manejar datos sucios.

### Orden cronologico

```python
df = df.sort_values("timestamp_event").reset_index(drop=True)
```

Esta linea ordena todos los logs por el momento en que ocurrieron.

Despues, `reset_index(drop=True)` reconstruye el indice de filas desde cero. Esto deja el DataFrame limpio y consistente despues del ordenamiento.

### Prints iniciales

```python
print(f"Dataset cargado: {len(df):,} logs")
print(f"Rango temporal: {df['timestamp_event'].min()} -> {df['timestamp_event'].max()}")
```

Estos prints muestran:

- cuantos logs hay en total;
- desde que fecha/hora hasta que fecha/hora cubre el dataset.

Sirven como primera verificacion de que el archivo se cargo correctamente.

---

## Bloque 4: Definiciones operativas obligatorias

Este bloque implementa exactamente las reglas que pide el challenge.

### Ventanas de 5 minutos

```python
df["window_start"] = df["timestamp_event"].dt.floor("5min")
```

Cada log se asigna a una ventana de 5 minutos usando `timestamp_event`.

Por ejemplo, si un evento ocurrio a las `11:12:34`, su `window_start` sera `11:10:00`. Si ocurrio a las `11:17:05`, su ventana sera `11:15:00`.

Esto permite agrupar eventos por bloques temporales comparables.

### Identificacion de errores 5xx

```python
df["is_5xx"] = df["status_code"] >= 500
```

Esta columna booleana marca si un log tiene un codigo HTTP mayor o igual a 500.

Los codigos 5xx suelen indicar errores del lado del servidor, por eso son relevantes para detectar incidentes.

### Definicion de bad event

```python
df["is_bad_event"] = df["severity"].isin(["ERROR", "CRITICAL"]) | df["is_5xx"]
```

Esta linea implementa la definicion obligatoria de `bad_event`.

Un evento se considera malo si cumple al menos una de estas condiciones:

- `severity` es `ERROR`;
- `severity` es `CRITICAL`;
- `status_code` es mayor o igual a `500`.

El operador `|` significa "o". Por lo tanto, alcanza con que una de las condiciones sea verdadera.

### DataFrame de eventos malos

```python
bad_df = df[df["is_bad_event"]].copy()
```

Esta linea crea un DataFrame nuevo que contiene solamente los bad events.

Se usa despues para calcular mensajes malos mas repetidos y diagnosticar el incidente.

### Bad rate global

```python
print(f"Bad events totales: {df['is_bad_event'].sum():,}")
print(f"Bad rate global: {df['is_bad_event'].mean():.2%}")
```

Como `is_bad_event` es una columna booleana, `sum()` cuenta cuantos valores son `True`.

Tambien, `mean()` calcula la proporcion de eventos malos sobre el total. Esa proporcion es el `bad_rate`.

---

## Bloque 5: Exploracion inicial

Este bloque responde las preguntas minimas de la seccion 6.1 del challenge.

```python
message_counts = df["message"].value_counts()
bad_message_counts = bad_df["message"].value_counts()
service_counts = df["service_name"].value_counts()
```

`value_counts()` cuenta cuantas veces aparece cada valor de una columna.

Aqui se calculan tres conteos:

- mensajes mas repetidos en todo el dataset;
- mensajes mas repetidos solo entre bad events;
- cantidad de logs por servicio.

Luego se arma una tabla:

```python
exploracion_inicial = pd.DataFrame(...)
```

La tabla responde:

- cuantos logs hay en total;
- que severidad aparece mas;
- que servicio genera mas logs;
- que servicio genera menos logs;
- cual es el mensaje mas repetido;
- cual es el mensaje malo mas repetido.

Esta tabla es una vista inicial del dataset antes de detectar el incidente.

---

## Bloque 6: Deteccion del momento critico

Este bloque encuentra la ventana de 5 minutos con peor comportamiento segun la regla del challenge.

### Agrupacion por ventana temporal

```python
window_summary = (
    df.groupby("window_start")
    .agg(
        total_events=("trace_id", "size"),
        bad_events=("is_bad_event", "sum"),
        avg_latency_ms=("latency_ms", "mean"),
        events_5xx=("is_5xx", "sum"),
    )
    .reset_index()
)
```

`groupby("window_start")` agrupa todos los logs que pertenecen a la misma ventana de 5 minutos.

Despues, `.agg(...)` calcula metricas por cada ventana:

- `total_events`: cantidad total de logs en la ventana;
- `bad_events`: cantidad de eventos malos;
- `avg_latency_ms`: latencia promedio;
- `events_5xx`: cantidad de respuestas HTTP 5xx.

`reset_index()` vuelve a dejar `window_start` como una columna normal.

### Calculo de bad rate y porcentaje 5xx

```python
window_summary["bad_rate"] = window_summary["bad_events"] / window_summary["total_events"]
window_summary["pct_5xx"] = window_summary["events_5xx"] / window_summary["total_events"]
```

`bad_rate` mide que proporcion de eventos de la ventana fueron malos.

`pct_5xx` mide que proporcion de eventos de la ventana tuvieron status HTTP 5xx.

Ambas metricas son proporciones entre `0` y `1`. Cuando se imprimen con formato porcentual, `0.582` se muestra como `58.20%`.

### Filtro obligatorio de volumen minimo

```python
eligible_windows = window_summary[window_summary["total_events"] >= 20].copy()
```

El challenge pide elegir el momento critico solo entre ventanas con al menos 20 eventos.

Este filtro evita elegir una ventana con muy pocos logs donde el porcentaje podria ser alto por casualidad.

### Top 5 ventanas criticas

```python
top_5_windows = (
    eligible_windows.sort_values(
        ["bad_rate", "bad_events", "total_events"],
        ascending=[False, False, False],
    )
    .head(5)
    [["window_start", "total_events", "bad_events", "bad_rate"]]
    .reset_index(drop=True)
)
```

Esta tabla muestra las 5 peores ventanas.

El orden principal es `bad_rate` descendente. Es decir, la ventana con mayor proporcion de eventos malos queda primera.

Si hay empate, se desempata por:

1. mayor cantidad de `bad_events`;
2. mayor cantidad de `total_events`.

Esto hace que la seleccion sea estable y reproducible.

### Momento critico

```python
critical_window_start = top_5_windows.loc[0, "window_start"]
critical_window_end = critical_window_start + pd.Timedelta(minutes=5)
```

El momento critico es la primera fila del top 5.

`critical_window_start` guarda el inicio de la ventana.

`critical_window_end` calcula el final sumando 5 minutos.

---

## Bloque 7: Diagnostico dentro del momento critico

Este bloque analiza solo lo que paso dentro de la ventana critica.

El criterio declarado en el notebook es:

```text
Por cantidad de bad events.
```

Eso significa que el servicio y endpoint mas afectados se eligen segun cuantos eventos malos tuvieron, no por latencia ni por cantidad total de requests.

### Separacion entre incidente y baseline

```python
in_critical_window = (df["timestamp_event"] >= critical_window_start) & (df["timestamp_event"] < critical_window_end)
incident_df = df[in_critical_window].copy()
baseline_df = df[~in_critical_window].copy()
bad_incident_df = incident_df[incident_df["is_bad_event"]].copy()
```

`in_critical_window` es una mascara booleana. Marca `True` para los logs que ocurrieron dentro de la ventana critica.

`incident_df` contiene solo los eventos del momento critico.

`baseline_df` contiene todo el resto del dataset, es decir, todo lo que ocurrio fuera del momento critico.

`bad_incident_df` contiene solo los eventos malos dentro del momento critico.

Esta separacion es clave para poder comparar incidente contra baseline.

---

## Bloque 8: Bad events por servicio

```python
bad_events_by_service = (
    bad_incident_df.groupby("service_name")
    .size()
    .rename("bad_events")
    .sort_values(ascending=False)
    .reset_index()
)
```

Este bloque cuenta cuantos bad events tuvo cada servicio dentro del momento critico.

`groupby("service_name")` agrupa por servicio.

`.size()` cuenta cuantas filas hay por servicio.

`.sort_values(ascending=False)` ordena de mayor a menor.

El primer servicio de esta tabla es el servicio mas afectado segun el criterio elegido.

---

## Bloque 9: Top 5 mensajes en bad events

```python
top_5_bad_messages = (
    bad_incident_df["message"]
    .value_counts()
    .head(5)
    .rename_axis("message")
    .reset_index(name="bad_events")
)
```

Este bloque identifica los mensajes mas repetidos entre los bad events del momento critico.

Sirve para explicar cual fue el sintoma dominante del incidente.

Por ejemplo, si un mensaje de timeout aparece muchas veces, eso sugiere que el problema estuvo relacionado con demoras o bloqueos.

---

## Bloque 10: Top 5 endpoints mas comprometidos

```python
top_5_compromised_endpoints = (
    bad_incident_df.groupby("endpoint")
    .agg(
        bad_events=("endpoint", "size"),
        events_5xx=("is_5xx", "sum"),
        avg_latency_ms=("latency_ms", "mean"),
    )
    .sort_values(["bad_events", "events_5xx", "avg_latency_ms"], ascending=[False, False, False])
    .head(5)
    .round({"avg_latency_ms": 2})
    .reset_index()
)
```

Este bloque agrupa los bad events del incidente por endpoint.

Para cada endpoint calcula:

- `bad_events`: cantidad de eventos malos;
- `events_5xx`: cantidad de errores HTTP 5xx;
- `avg_latency_ms`: latencia promedio.

El orden principal es por cantidad de bad events. Si hay empate, usa cantidad de 5xx y luego latencia promedio.

El primer endpoint de esta tabla es el endpoint mas comprometido segun el criterio definido.

---

## Bloque 11: Comparacion incidente vs baseline

Este bloque responde que cambio durante el incidente respecto al resto del dataset.

### Funcion de resumen

```python
def resumen_periodo(data):
    return pd.Series(
        {
            "total_events": len(data),
            "bad_rate": data["is_bad_event"].mean(),
            "avg_latency_ms": data["latency_ms"].mean(),
            "%_5xx": data["is_5xx"].mean(),
        }
    )
```

Esta funcion recibe un DataFrame y devuelve las metricas necesarias para compararlo:

- `total_events`: cantidad de logs;
- `bad_rate`: proporcion de eventos malos;
- `avg_latency_ms`: latencia promedio;
- `%_5xx`: proporcion de eventos con status code mayor o igual a 500.

La funcion se usa tanto para el incidente como para el baseline.

### Tabla comparativa

```python
incident_vs_baseline = pd.DataFrame(
    {
        "momento_critico": resumen_periodo(incident_df),
        "baseline": resumen_periodo(baseline_df),
    }
).T
```

Esta tabla tiene dos filas:

- `momento_critico`;
- `baseline`.

Y compara las mismas metricas en ambos periodos.

Esta es una de las tablas mas importantes del notebook, porque muestra con numeros concretos si el incidente fue realmente peor que el comportamiento normal.

---

## Bloque 12: Grafico 1 - eventos por severidad en bins de 5 minutos

```python
severity_order = ["INFO", "WARN", "ERROR", "CRITICAL"]
severity_time = (
    df.groupby(["window_start", "severity"])
    .size()
    .unstack(fill_value=0)
    .reindex(columns=severity_order, fill_value=0)
)
```

Este bloque crea una tabla temporal donde cada fila es una ventana de 5 minutos y cada columna es una severidad.

`unstack(fill_value=0)` transforma los valores de severidad en columnas. Si en una ventana no hubo eventos de cierta severidad, coloca `0`.

Despues se grafica:

```python
ax = severity_time.plot(figsize=(12, 5), linewidth=1.4)
```

Este es el primer grafico obligatorio.

Permite ver si durante el incidente aumentaron los eventos `ERROR` o `CRITICAL`, y tambien como se comportaron `INFO` y `WARN` a lo largo del tiempo.

---

## Bloque 13: Grafico 2 - bad rate en bins de 5 minutos

```python
bad_rate_time = window_summary.set_index("window_start")["bad_rate"]
```

Esta linea toma la columna `bad_rate` calculada por ventana y la prepara como serie temporal.

Despues se grafica:

```python
ax = bad_rate_time.plot(figsize=(12, 5), linewidth=1.6, color="tab:red")
ax.axvspan(critical_window_start, critical_window_end, color="tab:orange", alpha=0.25, label="Momento critico")
```

Este es el segundo grafico obligatorio.

La linea roja muestra el bad rate por cada ventana de 5 minutos.

`axvspan` resalta visualmente la ventana critica. Esto ayuda a conectar la tabla del top 5 con la serie temporal.

---

## Bloque 14: Conclusiones finales

Este bloque genera conclusiones reproducibles a partir de las tablas ya calculadas.

```python
servicio_mas_afectado = bad_events_by_service.loc[0, "service_name"] if not bad_events_by_service.empty else "Sin bad events"
endpoint_mas_comprometido = top_5_compromised_endpoints.loc[0, "endpoint"] if not top_5_compromised_endpoints.empty else "Sin bad events"
mensaje_dominante = top_5_bad_messages.loc[0, "message"] if not top_5_bad_messages.empty else "Sin bad events"
```

Estas lineas toman el primer valor de cada ranking:

- servicio mas afectado;
- endpoint mas comprometido;
- mensaje dominante.

Se usa una condicion `if not ...empty` para evitar errores si no hubiera bad events.

Luego se toman las metricas del incidente y del baseline:

```python
incident_metrics = incident_vs_baseline.loc["momento_critico"]
baseline_metrics = incident_vs_baseline.loc["baseline"]
```

Finalmente se imprimen 8 lineas de conclusion:

- momento critico exacto;
- cantidad de eventos y bad rate del incidente;
- servicio mas afectado;
- endpoint mas comprometido;
- mensaje dominante;
- comparacion de bad rate contra baseline;
- comparacion de latencia promedio;
- comparacion de porcentaje 5xx.

Esto cumple el requisito final del challenge: cerrar el notebook con conclusiones claras, basadas en datos y reproducibles.

---

## Resumen de la logica completa

El flujo del notebook es:

1. Cargar el CSV.
2. Validar que existan las columnas obligatorias.
3. Convertir fechas y numeros.
4. Crear ventanas de 5 minutos.
5. Marcar bad events segun severidad o status 5xx.
6. Calcular bad rate por ventana.
7. Elegir la peor ventana con al menos 20 eventos.
8. Analizar bad events dentro de esa ventana.
9. Comparar incidente contra baseline.
10. Graficar severidades y bad rate en el tiempo.
11. Imprimir conclusiones finales.

La parte mas importante es que todas las conclusiones salen de calculos hechos en el notebook. No se inventan columnas, no se inventan datos y no se elige manualmente el incidente.

