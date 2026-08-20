# Análisis de logs distribuidos — detección de incidentes

Análisis exploratorio sobre 5.795 logs de un sistema distribuido para localizar el momento crítico de una caída y explicar qué servicio, qué endpoint y qué mensajes estuvieron detrás.

**Stack:** Python 3 · pandas · matplotlib · Jupyter

---

## Cómo correrlo

```bash
pip install pandas matplotlib jupyter
jupyter notebook analisis.ipynb
```

Las celdas se ejecutan en orden. Los CSV están en el repo, no hace falta nada más.

---

## El problema

Dado un volcado de logs de varios servicios, encontrar **cuándo** se rompió el sistema y **qué** se rompió — sin saber de antemano que hubo un incidente ni a qué hora.

## El método

**1. Definiciones operativas.** Antes de medir nada, hay que decidir qué cuenta como falla. Un log es un `bad_event` si su severidad es `ERROR` o `CRITICAL` **o** su status code es 5xx. Es una definición explícita y discutible, pero fijarla por adelantado evita el sesgo de elegir el criterio que produce el resultado más vistoso.

**2. Ventanas de 5 minutos.** Cada log se asigna a un bin con `dt.floor("5min")`: un evento de las 19:24:48 cae en la ventana de las 19:20:00. Sin agregación temporal, 5.795 puntos sueltos no muestran ninguna tendencia.

**3. Detección del momento crítico.** Se calcula el `bad_rate` (proporción de eventos malos sobre el total) por ventana y se rankean las 5 peores, ordenando por `bad_rate`, después por cantidad de bad events y después por volumen total. La primera es el momento crítico. Con un filtro previo: **solo compiten ventanas con 20 o más eventos**.

**4. Diagnóstico.** Aislada la ventana crítica, se descompone: bad events por servicio, top 5 de mensajes de error, top 5 de endpoints comprometidos ordenados por cantidad de fallas, luego por 5xx y luego por latencia.

**5. Comparación contra baseline.** Todo lo que quedó **fuera** de la ventana crítica es el baseline. Se contrastan cuatro métricas —eventos totales, `bad_rate`, latencia promedio y % de 5xx— para cuantificar cuánto se desvió el incidente de la operación normal.

**6. Visualización.** Dos gráficos: conteo de eventos por severidad a lo largo del tiempo, y evolución del `bad_rate` con la ventana crítica sombreada.

---

## Los datos

| Archivo | Filas | Columnas |
|---|---:|---|
| `server_logs.csv` | 5.795 | 15 — timestamps, servicio, severidad, mensaje, trace/request id, método, endpoint, status code, latencia, host, env, región |
| `logs_livecoding.csv` | 200 | 4 — timestamp, servicio, status code, tiempo de respuesta |

---

## Estructura

```
analisis.ipynb          Análisis principal — 6 secciones, de exploración a conclusiones
server_logs.csv         Dataset principal
livecoding.ipynb        Ejercicio de livecoding: picos de tráfico en ventanas de 10 min
logs_livecoding.csv     Dataset del ejercicio
```

---

## Decisiones de análisis

**El umbral de 20 eventos por ventana.** Es la decisión que más cambia el resultado. Sin ese piso, una ventana con 2 logs de los cuales 1 es error tiene un `bad_rate` de 50% y le gana a la ventana real del incidente, que quizás tenga 40% sobre 300 eventos. El ruido estadístico de las ventanas chicas produce máximos espurios. El filtro descarta esas ventanas antes de buscar el máximo.

**`bad_rate` en lugar de conteo absoluto.** Una ventana de tráfico pico puede tener más errores en términos absolutos simplemente porque tiene más de todo. La proporción normaliza por volumen y responde la pregunta correcta: no "¿dónde hubo más errores?" sino "¿dónde falló una fracción anormal de los pedidos?".

**Validación de columnas antes de tocar los datos.** La primera celda de carga verifica que las 10 columnas obligatorias estén presentes y lanza `ValueError` con la lista de las que faltan si no. Es preferible fallar en la celda 4 con un mensaje claro que en la celda 20 con un `KeyError` opaco.

**Timestamps en UTC explícito.** `pd.to_datetime(..., utc=True)` en ambas columnas de fecha. Logs de servicios distribuidos pueden venir con zonas horarias distintas; normalizar en la carga evita que las ventanas de 5 minutos mezclen eventos de horas diferentes.

**El baseline es el complemento, no un período elegido.** Se define como todo lo que no está en la ventana crítica (`~in_critical_window`), en lugar de recortar a mano un tramo "tranquilo". Elegir el baseline a ojo es donde se cuela el sesgo: siempre se puede encontrar un tramo que haga ver peor al incidente.

**Dos criterios de ordenamiento para los endpoints.** El top 5 de endpoints comprometidos ordena por bad events, después por 5xx y después por latencia. El desempate importa: dos endpoints con la misma cantidad de fallas no son igual de graves si uno además está lento.

**Conclusiones generadas desde los datos.** La última celda no tiene texto escrito a mano: arma las siete conclusiones interpolando los valores calculados. Si el dataset cambia, las conclusiones cambian con él y no quedan afirmaciones desactualizadas en el notebook.

---

## Ejercicio de livecoding

`livecoding.ipynb` es un ejercicio aparte, resuelto contra reloj: detectar picos de tráfico en ventanas de 10 minutos e identificar qué ventana tuvo más tráfico, qué servicio la dominó y qué endpoint apareció más.

Usa un enfoque distinto al del análisis principal: `set_index` sobre el timestamp más `resample('10min')` en lugar de `floor` más `groupby`. `resample` es más directo para series temporales regulares; `floor` + `groupby` da más control cuando la ventana es una columna más que se cruza con otras dimensiones.

---

## Contexto

Challenge de análisis de datos con pandas. La consigna pedía partir de un volcado de logs, definir criterios operativos propios, detectar el momento crítico del sistema y sostener el diagnóstico con tablas y gráficos.

Cierra la secuencia que abre [5_Basic_Logs_Simulation](https://github.com/Nishikawaz/5_Basic_Logs_Simulation): allá se generan e ingestan los logs, acá se los interroga.

---

## Limitaciones conocidas

- **La ventana crítica es una sola.** El análisis asume un único incidente. Un dataset con dos caídas separadas mostraría solo la peor; habría que buscar todas las ventanas por encima de un umbral en vez del máximo.
- **El tamaño de ventana está fijo en 5 minutos.** Un incidente de 30 segundos se diluiría dentro de su bin. La elección del bin condiciona qué se puede detectar.
- **Umbral y criterio de `bad_event` sin análisis de sensibilidad.** No se verifica cuánto cambiaría el resultado con otro piso de eventos o incluyendo `WARNING` como evento malo.
