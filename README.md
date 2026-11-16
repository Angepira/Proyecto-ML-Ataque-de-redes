# Deteccion de ataques de red en dos etapas

Proyecto final de Machine Learning (MLEA_M), 2026-2.
Escuela Colombiana de Ingenieria Julio Garavito.

Equipo: Angelica, Hildebrando Pena, Isaac, Jared Farfan.

## 1. Que hace

Detectar ataques en trafico de red en dos pasos separados:

1. **Etapa 1**: decidir si un flujo es raro. Se entrena **solo con trafico
   benigno**, sin mostrarle ni un ataque.
2. **Etapa 2**: si salio raro, ponerle nombre al tipo de ataque. Si no se parece
   a nada de lo que vio, responde `Desconocido` en vez de inventar.

La razon de separarlo es que un clasificador solo reconoce lo que ya vio
entrenando, y en seguridad el ataque que importa es el que todavia no esta en
ningun dataset. Ademas se parece mas a la realidad: una empresa si tiene
registros de su trafico normal, pero no tiene ataques etiquetados de su red.

## 2. Los datos

**BigFlow-NIDS V2**, de Mendeley Data. El repo trae `data/merged_part_1.parquet`.

| | |
|---|---|
| Filas | 1.000.000 |
| Variables | 55 |
| Clases presentes | 4: `Benign`, `DDoS`, `xss`, `password` |
| Balance | 42,2% benigno, 57,8% ataque |
| Nulos | ninguno |

**Esto es una parte del dataset, no el dataset.** El diccionario de datos de
BigFlow-NIDS habla de 66.935.021 flujos y 32 clases; el archivo de este repo
tiene 1.000.000 de flujos y 4 clases. Nada de lo que esta aca se puede presentar
como valido para las 32.

## 3. Lo que encontramos

### Hay fuga de informacion por los puertos

| Variables | Columnas | Accuracy |
|---|---|---|
| Solo puertos y marcas de tiempo | 4 | **0.9774** |
| Solo comportamiento | 47 | 0.9695 |

Cuatro columnas que no dicen nada de como se comporta el flujo le ganan a las 47
que si lo dicen. No es que el puerto sea una buena variable: en este dataset
codifica que experimento se estaba corriendo. En una red real el atacante escoge
el puerto que quiera.

Las secciones 7 a 11 del notebook trabajan sin esas variables. Las metricas bajan
y son las que se pueden sostener.

### Nombrar el ataque es facil; detectarlo sin etiquetas es dificil

Etapa 1, todos entrenados solo con trafico benigno. Test de 120.000 flujos
(60.000 benignos y 20.000 de cada ataque). El umbral se fija sobre un conjunto de
calibracion, nunca sobre el test.

| Modelo | ROC-AUC | PR-AUC | Recall @ FPR 1% |
|---|---|---|---|
| Random Forest *(referencia supervisada)* | 0.9918 | 0.9902 | **71,5%** |
| Local Outlier Factor | 0.8096 | 0.8475 | **27,7%** |
| One-Class SVM | 0.7252 | 0.7478 | 3,9% |
| Isolation Forest | 0.9305 | 0.8622 | 3,5% |
| Autoencoder (MLP) | 0.9456 | 0.9122 | 3,5% |

La diferencia entre 27,7% y 71,5% es el precio de no tener ataques etiquetados,
medido y no supuesto.

Etapa 2, con las 3 clases de ataque:

| Modelo | F1 macro |
|---|---|
| Random Forest | **0.9975** |
| MLP | 0.9954 |
| Regresion logistica *(linea base)* | 0.9915 |

### El mejor ROC-AUC no es el mejor detector

El Local Outlier Factor tiene el ROC-AUC mas bajo de los no supervisados (0.81) y
es el que mas ataques atrapa en el punto de operacion (27,7%). El Isolation Forest
y el autoencoder ordenan mejor en general (0.93 y 0.95) pero en el 1% superior,
que es donde se decide si suena la alarma, la cola de los benignos se les mezcla
con los ataques y casi no atrapan nada.

Por eso el modelo se elige mirando el recall al 1% y no el AUC.

### El promedio escondia lo importante

La cascada completa detecta el 27,8% de los ataques con 96,5% de precision en la
alerta. Pero repartido por tipo:

| Real | Detectado | |
|---|---|---|
| Benigno | 600 de 60.000 generan alerta | 1,0% de falsas alarmas |
| DDoS | 12.701 de 20.000 | **63,5%** |
| password | 3.450 de 20.000 | 17,3% |
| xss | 502 de 20.000 | **2,5%** |

Detectar sin etiquetas funciona para ataques que cambian la forma del trafico, y
no funciona para los que viajan dentro de trafico de forma normal. Un `xss` va
dentro de una peticion web que a nivel de flujo se ve igual que cualquier otra.
Para esos haria falta mirar el contenido, no el resumen del flujo.

### El sistema sabe decir "no se"

Para cada tipo de ataque se borra por completo del entrenamiento, se entrena con
los que quedan y se le presentan los flujos del que se escondio:

| Ataque escondido | Va a `Desconocido` |
|---|---|
| xss | 100,0% |
| DDoS | 96,4% |
| password | 86,8% |

Promedio: **94,4%** de los flujos de un ataque que nunca vio terminan marcados
como desconocidos en vez de recibir una etiqueta inventada.

## 4. Como esta organizado el notebook

| Seccion | Que hay |
|---|---|
| 1 a 5 | librerias, carga, preprocesamiento, nulos y balanceo |
| Correlacion y pairplot | exploracion de variables |
| Lasso | seleccion de variables con L1 sobre 50 submuestras |
| Regresion logistica | clasificacion binaria, curva ROC y ajuste de umbral |
| Random Forest y Gradient Boosting | con busqueda en grilla |
| Multiclase | 4 clases, incluyendo benigno |
| **6** | variables que son fuga de informacion |
| **7** | etapa 1, detectar sin etiquetas |
| **8** | etapa 2, tipo de ataque. 8.1 hiperparametros, 8.2 umbral de rechazo |
| **9** | que pasa con un ataque que nunca se vio |
| **10** | cascada, las dos etapas juntas |
| **11** | conclusiones y limitaciones |

Las secciones 6 a 11 usan solo variables de comportamiento. Las de arriba
incluyen los puertos, asi que sus metricas son optimistas; la seccion 6 lo mide y
lo explica.

## 5. Como correrlo

```bash
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
jupyter lab Proyecto.ipynb
```

Corre de arriba a abajo sin tocar nada. Toma unos 49 minutos, y de esos **36 son
la celda del Lasso multiclase**, que ajusta 50 regresiones con `saga`. Pide cerca
de 1,5 GB de RAM.

Las semillas estan fijas (`SEMILLA = 42`) en el muestreo, las particiones y los
modelos.

El bloque de PyCaret del final necesita el otro entorno, el que esta descrito en
`requirements.txt`. El resto del notebook corre sin PyCaret.

## 6. Decisiones que hay que poder defender

| Pregunta | Donde esta la respuesta |
|---|---|
| ¿Por que no usaron las 32 clases? | Seccion 2. El archivo del repo tiene 4 |
| ¿Por que quitaron los puertos? | Seccion 6, con el experimento que lo mide |
| ¿Por que no deciden con accuracy? | Seccion 8, se decide con F1 macro |
| ¿Como eligieron los umbrales? | Seccion 7 (FPR 1%) y 8.2 (cobertura del 95%), los dos sobre calibracion |
| ¿Y si llega un ataque nuevo? | Seccion 9 |
| ¿Por que dos etapas y no una? | Seccion 1 y seccion 11 |
| ¿Por que el LOF si tiene el peor AUC? | Seccion 7 |
| ¿Por que la etapa 2 da 0.99 y la multiclase de arriba 0.78? | Nota al inicio de la seccion 8: son 4 clases contra 3, con puertos contra sin ellos, y sin escalar contra escalado |

## 7. Limitaciones

1. Es una parte del dataset: 1.000.000 de flujos y 4 de las 32 clases.
2. Con 3 clases de ataque, la prueba de la seccion 9 es mas facil de lo que seria
   con el dataset completo: al esconder una, el modelo queda con dos, y distinguir
   entre dos es mas facil que entre tres.
3. Una sola particion, sin validacion cruzada repetida en las metricas finales. No
   hay intervalos de confianza, asi que las diferencias pequenas entre modelos no
   se pueden afirmar.
4. Los modelos que dependen de distancias o kernels se entrenaron sobre
   submuestras por costo. Queda dicho en cada celda.
5. La etapa 1 habria que reentrenarla con el trafico normal de la red donde se
   instale. No se trasplanta.
6. Ninguna metrica mide que pasa si el atacante sabe que hay un modelo y trata de
   evadirlo.

## 8. Fuente de los datos

Uddin, Mohammed Borhan; Arefin, Mohammad Shamsul (2026), *BigFlow-NIDS*,
Mendeley Data, V2.

Uddin, M. B., Arefin, M. S., & Hussain, M. M. M. (2026). *BigFlow-NIDS: A
large-scale dataset for network intrusion detection.*
