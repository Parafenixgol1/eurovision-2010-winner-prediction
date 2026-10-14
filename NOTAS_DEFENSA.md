# Notas de defensa (no se entrega)

Notebook entregado: `20261010_LuisDioses.ipynb` (v1). Todas las cifras de estas notas salen de outputs reales del notebook o de exploraciones hechas durante el desarrollo (marcadas como "exploración").

Estructura: una sección por sección del notebook. En cada decisión: qué se hizo, por qué, alternativas y por qué se descartaron, y puntos débiles honestos.

---

## 0. Enunciado y entrega

- Enunciado: predecir quién puede ganar Eurovisión 2010. Requisitos mínimos: (a) resumen ejecutivo, (b) descripción del modelo y objetivos, (c) comprensión de datos con gráficos, estadísticos y distribuciones, (d) preparación, (e) modelización, (f) conclusiones. Se valora originalidad, funciones genéricas y conclusiones "en términos del enunciado". 2h20 de tiempo.
- Mapeo: (a) sección 0, (b) secciones 1 y 5, (c) sección 3, (d) secciones 2 y 4, (e) secciones 5-7, (f) sección 8.
- Funciones genéricas: `clean`, `add_features`, `make_pipeline`, `semi_hits`, `final_ranking_metrics`, `coefficients`.
- Formato de nombre `fecha_NombreCandidato.ipynb` cumplido. Solo Python.
- No revisé `Plantilla_Soluciones_Senior.xlsm` ni `Cuestionario_Acceso_Senior.pdf`.

---

## 1. Planteamiento del problema

### Decisión: dos modelos encadenados (semifinal → final)
- Por qué: la hoja 2010 tiene 39 países y solo 25 llegan a la final (5 directos + 10 de cada semifinal). Un modelo solo de finales trataría a los 39 como finalistas.
- Alternativas descartadas:
  - Solo modelo de final, asumiendo que los 39 llegan: más simple, pero sobrevalora a países que probablemente no pasan.
  - Simular el festival voto a voto (Monte Carlo sobre una matriz de votos): más realista, demasiado complejo para 2 horas y difícil de explicar.
- Débil: la combinación P(clasificar) x % esperado.
  - Lo correcto: P(ganar) = P(clasificar) x P(ganar | clasificó). Esa multiplicación es exacta (para ganar hay que clasificar), no asume independencia.
  - Problema 1, el segundo factor no es P(ganar | clasificó) sino el % de puntos medio esperado. Datos: los ganadores 2004-2009 sacaron 10-16% de los puntos y el favorito ganó 6/6; con un 10% se gana a menudo. El modelo espera 10.1% para Azerbaiyán y su índice queda en 0.090, frente a 0.202 del mercado. El índice comprime las diferencias.
  - Problema 2, ignora a los rivales: las opciones de un país dependen de quién más llega a la final. El índice calcula cada país por separado y solo normaliza al final.
  - Problema 3, los errores del modelo de semifinal pasan al índice. Datos: Eslovaquia, P(clasificar) 0.93 con cuota 38, queda 7ª en el índice (0.047) frente a 0.017 en el mercado.
  - Lo correcto sería simular: sortear clasificados con P(clasificar), dar a cada finalista su % esperado + un error sacado de los errores de validación cruzada, ver quién gana y repetir miles de veces. No incluido por simplicidad y tiempo; además supone errores independientes entre países, que no se cumple (los puntos suman 100%).
  - No comprobado: si la simulación cambiaría el orden de los primeros.

### Decisión: el objetivo del modelo de final es `points_share` (Ridge), no "ganó sí/no"
- Por qué: solo hay 6 ganadores en 2004-2009. Una logística con 6 positivos y ~15 columnas tiene coeficientes inestables. `points_share` usa la información de los 146 finalistas (quedar 2º con 12% no es lo mismo que 20º con 1%).
- `points_share` y no `Points`: el total de puntos cambia por año (2088 en 2004, 2436 en 2008-2009) porque cambia el número de votantes. El máximo de 2009 fue 387.
- Alternativas descartadas:
  - Logística sobre ganador: 6 positivos. La versión con solo log(cuota) se sostiene pero equivale a ordenar por cuota (no aporta sobre el baseline).
  - Predecir `Place`: depende del número de finalistas (24-25), y tiene empates (2004: Bélgica e Irlanda ambos 22º).
  - Logística sobre top-3 o top-5: más positivos (18-30), opción razonable; descartada para mantener un solo enfoque.
  - Modelos de ranking (learning-to-rank, logit condicional por año): más adecuados en teoría, más difíciles de explicar.
- Débil: el % de puntos esperado no se traduce directamente en probabilidad de ganar.

### Decisión: métrica = posición predicha del ganador real + Spearman por año; en semifinal, clasificados acertados + AUC
- Por qué no accuracy: un ganador entre ~25 por año; predecir "nadie gana" da 96% de accuracy.
- Spearman (no Pearson): importa el orden, no la relación lineal.
- Semifinal: "de los N con mejor score, cuántos clasificaron de verdad" (N = 10 por semifinal). Es directamente interpretable. AUC como métrica complementaria sin umbral.
- Débil: con 5 años de validación, cada métrica de final se basa en 5 ganadores. Diferencias de 1 año son ruido.

### Decisión: "Índice de victoria" en vez de "probabilidad"
- Índice = P(clasificar) x % puntos esperado, normalizado a suma 1 sobre los 39 países.
- Por qué no es probabilidad: el % de puntos esperado de un favorito es ~10%, pero su probabilidad real de ganar es mucho mayor. El índice comprime las diferencias (favoritos 8-9% vs ~20% del mercado).
- Alternativa: calibrar con más años o simular. Fuera de alcance.

---

## 2. Carga y calidad de datos

### Observaciones
- Hojas: finales 291 filas (1998-2009), semifinales 173 filas (2004-2009), 2010 con 39 filas.
- Finales: 145 nulos en cuota, género y solista/grupo, todos de 1998-2003. 2004-2009 sin nulos.
- `Place` en finales es texto mezclado con números: 5 valores con BOM (`'﻿3'`) y espacios.
- Nombres de columna con espacios finales (`'Country   '`, `'Albania '`). Nombres de país también (`'Finland '` en 2010).
- `Host country` (semifinal) vs `Host Country` (final): se unifica.
- Columna de voto `Serbia & Montenegro` vs país `Serbia and Montenegro` (no afecta a v1 porque no usa votos).
- Semifinales: una por año 2004-2007, dos en 2008-2009. Entre 18 y 28 países por semifinal. Siempre clasifican 10.
- Corte por puesto en semifinal: en 2004-2007 siempre ≤ 10; en 2008-2009 algún clasificado quedó 12º-13º (comodín del jurado).
- `Home/Away Country` en semifinales es siempre "Away" (ningún anfitrión compite en semifinal).
- Hoja de semifinales no tiene género ni solista/grupo.
- Todos los países de 2010 aparecen en el histórico.
- Anomalías de la descripción no tratadas: jurados de respaldo en algunos países/años, Turquía 2004 sin poder votar a Eslovenia, empate a 14 puntos en 2008. Afectan a pocos votos individuales; v1 no usa votos individuales.

### Decisión: función `clean` única para las tres hojas
- `map` con `isinstance(v, str)` en lugar de `.str.strip()`: `.str` sobre una columna mezcla int/str convierte los int en NaN (me pasó al desarrollar: `Place` quedaba con NaN).

### Decisión: entrenar solo con 2004-2009
- Por qué: es donde hay cuotas, la variable más predictiva. Imputar 145 nulos (50% de las finales) en la variable principal la distorsionaría.
- Alternativas descartadas:
  - Usar 1998-2009 sin cuota: más filas, mucha menos señal (el control sin cuota muestra que sin ella casi no se predice).
  - Imputar cuota en 1998-2003 (por ejemplo, con puesto del año anterior): inventa la variable principal.
- Coste: se pierden 6 años de finales. Los votos de esos años sí servirían para variables de afinidad (ver exploración v2).

### Decisión: `qualified` = el país aparece en la final de ese año
- Alternativa descartada: `Place <= 10` en semifinal. Falla en 2008-2009 por el comodín del jurado.
- Comprobación: 10 clasificados en cada una de las 8 semifinales.
- Reglas de clasificación por periodo (descripción del dataset):
  - 2004-2007: top 10 de la semifinal por televoto. Además, el top 10 de la final del año anterior pasaba directo a la final.
  - 2008-2009: en cada semifinal, top 9 por televoto + 1 comodín del jurado (el mejor del jurado fuera de ese top 9). En los datos: clasificados en 12º (2008 SF2), 12º (2009 SF1) y 13º (2009 SF2).
  - 2010: top 10 de cada semifinal por 50% jurado + 50% televoto.
- Efecto 1, regla de 2010 distinta: el modelo aprende de semifinales decididas casi siempre por televoto, y 2010 usa 50% jurado. Los jurados pesan menos el voto de vecinos y diásporas, así que el efecto región aprendido (Europa Occidental -0.98, ex-Yugoslavia +0.86) probablemente es más débil en 2010. Refuerza la sospecha de que el modelo sobreestima a países como Eslovaquia. Está en limitaciones (sección 8).
- Efecto 2, nivel distinto en cada semifinal (verificado con datos). Hipótesis inicial descartada: "2004-2007 eran semifinales más débiles porque el top 10 del año anterior pasaba directo". Los datos no la apoyan:

| Semifinal | Países | Cuota mínima | Cuotas < 20 | Media 1/cuota |
|---|---|---|---|---|
| 2004 | 22 | 2.4 | 4 | 0.048 |
| 2005 | 25 | 7.6 | 4 | 0.025 |
| 2006 | 23 | 2.5 | 4 | 0.039 |
| 2007 | 28 | 2.7 | 6 | 0.034 |
| 2008 SF1 | 19 | 2.5 | 4 | 0.042 |
| 2008 SF2 | 19 | 5.3 | 2 | 0.021 |
| 2009 SF1 | 18 | 16.4 | 1 | 0.013 |
| 2009 SF2 | 19 | 1.7 | 3 | 0.046 |

  - Las cuotas mínimas de 2004, 2006 y 2007 (2.4, 2.5, 2.7) coinciden con las de los ganadores de esos años (Ucrania 2.40, Finlandia 2.55, Serbia 2.67): ganaron saliendo desde la semifinal.
  - La más débil es 2009 SF1 (mejor cuota 16.4); las más fuertes, 2004 y 2009 SF2.
  - Conclusión: el nivel varía mucho entre semifinales, sin patrón por periodo. Por eso `price_rank_pct` (posición de la cuota dentro de su semifinal) es útil: con cuota 16 eras el favorito de 2009 SF1, pero en 2007 había 6 países por debajo de 20. `log_price` es absoluto y no capta el nivel del grupo.

### Decisión: descartar las columnas de votos por país
- Justificación del notebook: "la hoja 2010 no las trae".
- **Matiz importante**: esa justificación es incompleta. Los votos de 2010 no se pueden usar como variable directa, pero sí se pueden derivar variables históricas (afinidad entre países) y aplicarlas a 2010, porque se sabe quién vota en cada semifinal y en la final. Lo probé (v2) y no aporta (ver exploración al final).
- Respuesta si preguntan: "Los votos de 2010 no existen, así que no son variable directa. Una variable de afinidad construida con años anteriores sí sería válida; la probé fuera del notebook y no mejoraba, así que la dejé fuera por simplicidad."

---

## 3. EDA

### Observaciones (outputs del notebook)
- Estadísticos 2004-2009 (146 finalistas): `Points` media 95 / mediana 73.5 / máx 387; `points_share` media 0.041 / mediana 0.031 / máx 0.159; cuota media 93 / mediana 61 / rango 1.73-607.
- Histograma de cuota: la de 2010 se parece a la histórica en el centro, pero sin la cola de favoritos claros. Ojo: el histograma 2010 incluye los 39 países y el histórico solo finalistas, no son poblaciones idénticas.
- `points_share`: concentrado por debajo del 4%, pocos países superan el 10%.
- Ganador = favorito de las apuestas los 6 años (Ucrania 2.40, Grecia 2.14, Finlandia 2.55, Serbia 2.67, Rusia 2.51, Noruega 1.73).
- Spearman cuota vs puesto: 0.70.
- Gráfico cuota vs % de puntos: relación decreciente y aproximadamente lineal en log. Los 6 ganadores son los 6 puntos más a la izquierda. Con cuotas 5-6 hay países que llegan al 12%. Mucha dispersión en cuotas intermedias.
- Boxplot semifinal: mediana de ranking de cuota ~6 en clasificados y ~16 en eliminados; solape (eliminados con ranking 3, clasificados con 21). Estas medianas son lectura visual.
- Tablas por categoría en la final (% medio de puntos, ganadores):
  - Región: ex-socialista 0.048 (2 ganadores), ex-Yugoslavia 0.047 (1), Escandinavia 0.043 (2), Independiente 0.037 (0), Europa Occidental 0.030 (1, Grecia).
  - Solista 0.044 (5 ganadores de 99) vs grupo 0.035 (1 de 47).
  - Género: mixto 0.026, femenino 0.043, masculino 0.044.
  - Anfitrión: 5 casos, 0.053, 0 ganadores.
  - Inglés: 112 casos con 5 ganadores vs 34 con 1.
- Semifinal: tasa de clasificación por región: Escandinavia 0.64, ex-Yugoslavia 0.56, ex-socialista 0.51, Independiente 0.50, Europa Occidental 0.22. Inglés 0.47 vs 0.43.

### Para qué sirven los dos histogramas (cuota y % de puntos)
- Se añadieron para cubrir el requisito (c) del enunciado; su relación con las decisiones es más débil que la de los otros gráficos.
- Qué se dibuja: las 146 filas país-año de finalistas 2004-2009 juntas (no un año, no un promedio) y las 39 filas de 2010. Cuota en densidad para comparar formas con tamaños distintos (146 vs 39). Juntar años es correcto para `points_share` (normalizado por año); para la cuota es una simplificación (la suma de 1/cuota de finalistas varía 1.16-1.36 entre años).
- **Gráfico de cuota** — pregunta: ¿las cuotas de 2010 se parecen a las de entrenamiento?
  - Muestra: ninguna cuota 2010 < 3; en el histórico, los 6 ganadores (y nadie más) estaban < 3. 2010 no tiene favorito claro.
  - No muestra el sesgo de la cuota (está en log10); el sesgo se ve en `describe()` (media 93 vs mediana 61, skew 2.06 → -0.49 con log).
  - Limitación: compara finalistas históricos con los 39 participantes de 2010. Comparación justa (datos, exploración):

| Percentil cuota | Finalistas 2004-09 | Todos participantes 2004-09 | 2010 (39) |
|---|---|---|---|
| Mínimo | 1.7 | 1.7 | 3.2 |
| 25% | 18.7 | 36.9 | 34.8 |
| Mediana | 61.3 | 102.7 | 72.3 |
| 75% | 125.4 | 256.1 | 125.9 |
| Máximo | 607.3 | 840.6 | 259.9 |

  - Frente a finalistas el centro se parece (lo que dice el notebook). Frente a todos los participantes, la cuota 2010 está mucho más comprimida (ni favoritos tan claros ni descartados tan extremos). Hipótesis no verificada: fuentes distintas (2010 puntual entre casas vs promedio de Betfair).
  - Defensa: "Compruebo que las cuotas de 2010 están en el rango de entrenamiento y veo que 2010 no tiene un favorito tan claro."
- **Gráfico de % de puntos** — pregunta: ¿cómo es la variable que se predice?
  - Muestra: 56% de finalistas bajo el 4%, solo 10 de 146 sobre el 10%; los ganadores están en esa cola (10.2-15.9%).
  - Implicación: un modelo lineal predice valores medios y le cuesta llegar a la cola; anticipa que subestimará a los ganadores (Finlandia 2006: 13% real vs 8% predicho).
  - No muestra por qué `points_share` en vez de `Points`: eso es porque el total de puntos cambia por año.
  - Defensa: "La variable objetivo está concentrada en valores bajos con una cola de ganadores; anticipa que el modelo lineal los subestimará, como confirma el análisis de errores."
- Origen de las frases del markdown de esa celda: "ganadores 1.7-2.7" sale de un output; "mínimo 2010 = 3.2" es correcto (3.22) pero no está impreso en el notebook; "parecido en el centro", "concentrado bajo el 4%" y "pocos superan el 10%" fueron lecturas visuales, verificadas después con los datos de arriba.

### Conclusiones que usa el resto del notebook
- La cuota es la variable principal → entra en log.
- Categóricas con poca señal en la final → entran, pero con regularización (Ridge).
- Región relevante en semifinal → entra en la logística.
- Baseline "gana el favorito" acierta 6/6 → el modelo no puede mejorarlo en top-1 histórico.

### Otros datos observados en exploración (no en notebook)
- Overround (suma de 1/cuota) por año en finales: 1.16-1.36. En semifinales varía mucho (0.24-1.06) porque la cuota es "ganar el festival", no "clasificar", y depende de qué países caen en cada semifinal. En 2010 la suma sobre 39 países es 1.54.
- La cuota de semifinal es idéntica a la de la final para los 80 países que aparecen en ambas hojas: una sola cuota por país y año.

---

## 4. Preprocesamiento

### `log_price`
- Por qué: cuota de 1.7 a 607 con cola larga; en log la relación con `points_share` es aproximadamente lineal (gráfico del EDA), que es lo que asume Ridge.
- Alternativas: cuota cruda (relación muy no lineal), probabilidad implícita normalizada por año (1/cuota dividido por la suma del año). Esta última corregiría la diferencia entre cuota histórica (promedio) y cuota 2010 (puntual); la descarté por simplicidad. Si preguntan, es la primera mejora que haría.

### `price_rank_pct` (solo semifinal)
- Por qué: clasifican los 10 mejores de cada grupo, importa la posición relativa. Percentil porque los grupos tienen 18-28 países.
- No es leakage: usa solo cuotas del mismo grupo, nunca el resultado. En 2010 se calcula igual (dentro de SF1 y SF2).
- No se usa en la final: allí los finalistas reales no se conocen en 2010.
- **Comprobado después de entregar: es redundante con `log_price`.**
  - Spearman `log_price` vs `price_rank_pct` en semifinales 2004-2008: 0.972.
  - CV 2004-2008: con las dos 48/60, AUC 0.859; solo `log_price` 48/60, AUC 0.857; solo `price_rank_pct` 46/60, AUC 0.863. Diferencias de ruido.
  - Por qué no aporta (hipótesis, no verificada): dentro de una semifinal, ordenar por cuota y por ranking de cuota da exactamente el mismo orden; las dos solo difieren en cómo comparan países de semifinales distintas, y con 6 semifinales de entrenamiento esa diferencia no llega a notarse.
  - Defensa: "La incluí porque clasificar depende de la posición dentro del grupo. Al comprobarlo, está correlacionada 0.97 con la cuota y no mejora el resultado; la quitaría y dejaría solo `log_price`, que rinde igual (48/60) y es más simple."

### `english`
- 1 si la canción contiene inglés, incluidas las mixtas ("English, Hebrew"). Alternativa: solo inglés puro; descartada porque las mixtas suelen ser mayoritariamente en inglés.

### Variables por modelo
- Semifinal: `log_price`, `price_rank_pct`, `english`, `Region`, `Home/Away Region`.
- Final: `log_price`, `english`, `Region`, `Home/Away Region`, `Home/Away Country`, `Group/Solo`, `Artist gender`.
- Género y solista/grupo no están en semifinales. `Home/Away Country` es constante en semifinales.
- Descartadas: artista, canción, traducción (texto libre, sin señal generalizable con 150 filas), país (one-hot de ~40 países con 3-6 apariciones cada uno: sobreajuste), anfitrión del año (constante por año).

### `Pipeline` + `ColumnTransformer`
- `StandardScaler` en numéricas y `OneHotEncoder(handle_unknown='ignore')` en categóricas, dentro del pipeline: se ajustan solo con los años de entrenamiento de cada fold (evita leakage).
- Estandarizar es necesario con Ridge y logística regularizadas: la penalización depende de la escala.
- One-hot sin `drop`: con regularización L2 es válido (no hay problema de colinealidad perfecta). Consecuencia en la interpretación: los coeficientes de una binaria salen simétricos (Home -0.167 / Away +0.167); lo interpretable es la diferencia (Home vs Away = -0.33).
- `english` se escala aunque sea binaria; no cambia nada relevante.

### Splits
- Desarrollo 2004-2008: 136 filas de semifinal, 121 de final. Test 2009: 37 y 25.

---

## 5. Modelado

### Logística (semifinal) y Ridge (final)
- Por qué: ~150 filas, una variable dominante y relación casi lineal en log. Modelos lineales regularizados, interpretables por coeficientes, sin sobreajuste.
- Hiperparámetros: `C=1` y `alpha=1` por defecto, sin búsqueda.
  - Por qué: con 5 folds y una variable dominante, una búsqueda anidada consume tiempo y apenas cambia el resultado.
  - Débil: no lo comprobé. Si preguntan: "Usaría RidgeCV / LogisticRegressionCV dentro de cada fold; no esperaría cambios grandes porque la cuota domina".
- Alternativas descartadas:
  - Random Forest / gradient boosting: con 146 filas sobreajustan y pierden interpretabilidad; la relación es casi lineal.
  - Modelo único sobre semifinales + finales: los objetivos son distintos (clasificar vs % de puntos).

### Baseline
- Ordenar por cuota (favorito primero), en semifinal y final. Es el "modelo del mercado".
- Alternativa: un baseline más tonto (aleatorio, o "repite resultado del año anterior"). El de cuota es el más exigente y el más honesto.

### Validación: leave-one-year-out sobre 2004-2008, test 2009
- Por qué por año: las filas de un mismo año comparten contexto (mismos votantes, mismo reparto de puntos, mismos rivales). Mezclar filas de un año entre train y validación filtraría información.
- Por qué 2009 como test: el año más reciente, el más parecido a 2010 (dos semifinales).
- Alternativas:
  - KFold aleatorio: mezcla años → leakage.
  - Ventana temporal creciente (entrenar con años < t, validar en t): más realista porque nunca usa el futuro. Con 5 años, los primeros folds entrenarían con 1-2 años. Leave-one-year-out usa años "futuros" para validar años pasados; aceptable si se asume que la relación cuota-resultado es estable entre años.
- Débil: 5 folds y 1 año de test. La varianza de las métricas es alta.

### Resultados CV (outputs)
- Semifinal, clasificados acertados por grupo (de 10):

| | 2004 | 2005 | 2006 | 2007 | 2008 SF1 | 2008 SF2 | Total |
|---|---|---|---|---|---|---|---|
| Modelo | 6 | 9 | 9 | 8 | 8 | 8 | 48 |
| Baseline | 7 | 8 | 9 | 5 | 9 | 8 | 46 |

- AUC semifinal: 0.859 modelo vs 0.862 baseline. AUC calculado sobre todas las filas juntas (mezcla años); alternativa más estricta: AUC por semifinal y promedio.
- Final:

| Año | Puesto ganador (modelo) | Spearman modelo | Puesto ganador (baseline) | Spearman baseline |
|---|---|---|---|---|
| 2004 | 1 | 0.61 | 1 | 0.62 |
| 2005 | 1 | 0.77 | 1 | 0.74 |
| 2006 | 3 (Finlandia) | 0.80 | 1 | 0.77 |
| 2007 | 1 | 0.73 | 1 | 0.65 |
| 2008 | 1 | 0.70 | 1 | 0.72 |

- Lectura: empate práctico. El modelo ordena algo mejor el resto (Spearman mayor en 3 de 5), falla al ganador de 2006.

---

## 6. Evaluación

### Test 2009 (outputs)
- Semifinal: modelo 7+7 = 14/20; baseline 7+8 = 15/20.
- Final: ambos ponen a Noruega 1ª. Spearman 0.75 modelo vs 0.67 baseline.
- Consistente con CV: ni mejor ni peor de forma clara.

### Análisis de errores (outputs CV)
- Subestimados: Turquía 2004 (9.3% real vs 3.1% predicho, cuota 31), Finlandia 2006 (13.2% vs 8.1%, ganadora), Noruega 2008, Rusia 2006, Lituania 2006.
- Sobreestimados: Letonia 2007 (2.2% vs 7.9%, cuota 18), Bélgica 2004 y Irlanda 2004 (0.3% vs ~6%), Suecia 2007, Reino Unido 2006.
- Patrón: Ridge en log-cuota es lineal y "aplana" los extremos. No capta lo concentrado del voto en el ganador (siempre 10-16%).
- Patrón posible (no verificado): Turquía, Rusia, Lituania subestimadas (países con diáspora o vecinos que votan); Bélgica, Irlanda, Reino Unido sobreestimados (Europa Occidental/Independiente). Encaja con un efecto de voto regional que el modelo no capta del todo.

### Qué se evalúa exactamente (y qué no)
- Final: el puesto predicho es el orden de `pred_share` (% de puntos de Ridge) entre los **finalistas reales** de cada año (`final_ranking_metrics(final_dev, 'pred_share')`). El modelo de semifinal no interviene.
- Semifinal: se evalúa por separado (48/60 en CV, 14/20 en test).
- **El índice de victoria (P(clasificar) x % esperado) no se validó de punta a punta.** Solo se calcula para 2010. Combina dos modelos validados por separado, pero nunca se comprobó en años pasados si el ganador real sale arriba en el índice partiendo de todos los participantes.
- Cómo se haría: en cada año de validación, calcular el índice para todos los participantes del año (semifinalistas + los que pasan directos), ordenar y ver en qué puesto queda el ganador real.
- Spearman en la evaluación se calcula sobre puestos, así que coincide exactamente con Pearson sobre puestos (2004: 0.61, 2005: 0.77, 2006: 0.80, 2007: 0.73, 2008: 0.70 en ambos). En el EDA (cuota vs puesto) sí difieren: Pearson 0.57, Spearman 0.70, Pearson con log(cuota) 0.71, porque la cuota tiene cola larga y la relación no es lineal.

### Control sin cuota (outputs)
- Mismos modelos solo con `english` + categóricas.
- Semifinal: 32/60 (azar: 27.0), AUC 0.605.
- Final: Spearman 0.03-0.49 por año; puesto predicho del ganador real 2.5, 19, 22, 12, 3 (el 2.5 es un empate).
- Conclusión: casi todo el poder predictivo es la cuota.
- Cálculo del azar: en una semifinal con n países, elegir 10 al azar acierta en media 10 x 10/n. Sumado sobre las 6 semifinales de desarrollo (22, 25, 23, 28, 19, 19 países) da 27.0.

### Posible leakage en la cuota histórica
- La descripción dice que la cuota histórica es un promedio en el tiempo de Betfair, sin especificar el periodo. Betfair permite apostar durante el festival; si el promedio incluye la noche de la votación, la cuota del ganador baja artificialmente y contiene parte del resultado.
- Indicios compatibles: favorito ganó 6/6; una sola cuota por país-año (semifinal = final).
- Indicio en contra: la cuota 2010 (3.22 Azerbaiyán, 3.25 Alemania) es coherente con cuotas previas publicadas (5/2 en William Hill 23 días antes; fuente externa ESCXtra). La 2010 es claramente previa.
- Consecuencia si existe: métricas históricas infladas, el modelo confía demasiado en la cuota, y en 2010 rendiría peor.
- Por qué se mantiene la cuota: es información legítima previa al festival y el enunciado la da para 2010. El riesgo afecta a cuánto confiar en las métricas, no a la validez de usarla. Está como limitación principal.

---

## 7. Interpretación y predicción 2010

### Reentrenamiento con 2004-2009 completo
- Por qué: tras validar, usar todos los datos disponibles para la predicción final. Práctica estándar.

### Coeficientes (outputs)
- Semifinal (log-odds por desviación estándar o por categoría): `price_rank_pct` -1.09, `log_price` -0.73, Europa Occidental -0.98, Independiente -0.29, Escandinavia -0.12, ex-socialista +0.52, ex-Yugoslavia +0.86, Home región -0.17 / Away +0.17, inglés -0.14.
- Final (% de puntos): `log_price` -0.025 por desviación estándar; resto entre 0.000 y 0.008 en valor absoluto.
- `price_rank_pct` (-1.09) y `log_price` (-0.73) no se pueden leer por separado: están correlacionadas 0.97 y el modelo reparte entre ellas el efecto de la cuota de forma arbitraria. Lo correcto es "la cuota, en conjunto, es lo que más pesa".
- Cuidado al comparar: numéricas están por desviación estándar y categóricas por categoría. Decir "la región pesa casi tanto como la cuota" es aproximado; por eso lo quité de las conclusiones.
- Inglés con coeficiente negativo pequeño: no significa que cantar en inglés perjudique; es ruido o colinealidad con la región.
- Jugar en casa (región) negativo en semifinal: contraintuitivo; pocos casos (28 en finales), no lo interpretaría.

### Predicción 2010
- Semifinales asignadas a mano desde la descripción (SF1, SF2, 5 directos con P(clasificar) = 1).
- Top del índice de victoria (outputs): Azerbaiyán 0.090, Alemania 0.078, Israel 0.071, Croacia 0.062, Armenia 0.061, Noruega 0.055, Eslovaquia 0.047, Grecia 0.041, Dinamarca 0.039, España 0.034.
- Mercado normalizado: Azerbaiyán 0.202, Alemania 0.200, Israel 0.099, Dinamarca 0.057, Armenia 0.049, Croacia 0.048, Noruega 0.045.
- Alemania por debajo de Azerbaiyán con cuota casi igual (3.25 vs 3.22): coeficiente de Europa Occidental en la final (-0.007) frente a ex-socialista (+0.008).
- `clip(min=0)`: Ridge puede predecir % negativos para países con cuota muy alta; un % negativo no tiene sentido.
- Normalización sobre los 39 (ponderados por P(clasificar)). Alternativa: normalizar solo sobre los 25 finalistas predichos; descartada porque convierte una predicción incierta (quién clasifica) en certeza.

### Casos raros que un evaluador puede señalar
- Eslovaquia 7ª con cuota 38 (mercado 0.017): P(clasificar) 0.93 porque SF1 es un grupo débil en cuotas (su ranking relativo es bueno) + región ex-socialista. Probablemente sobreestimada; ejemplo de que el efecto región puede estar inflado. Recuerdo (externo, verificar) que en 2010 no pasó la semifinal.
- España 10ª con cuota 53: solo porque tiene la final asegurada (P = 1), mientras que países con mejor cuota tienen P < 1.
- Finalistas predichos SF1: Eslovaquia, Bielorrusia, Grecia, Serbia, Albania, Bélgica, Moldavia, Bosnia, Rusia, Islandia. SF2: Azerbaiyán, Croacia, Israel, Armenia, Dinamarca, Suecia, Turquía, Ucrania, Rumanía, Irlanda.

---

## 8. Conclusiones

- Mensaje central: las cuotas son el mejor predictor; el modelo las iguala y añade matiz regional en semifinales.
- Predicción: Azerbaiyán con Alemania muy cerca; no concluyente.
- 2010 es un año más abierto que 2004-2009 (cuota mínima 3.2 vs 1.7-2.7).
- Limitaciones escritas: 6 años/6 ganadores, cambio a 50% jurado en semifinales 2010, posible leakage de la cuota histórica, índice no calibrado.
- Limitaciones no escritas que conviene conocer: hiperparámetros sin ajustar, validación con años "futuros", AUC agrupado, anomalías de jurado de respaldo no tratadas, cuota histórica (Betfair, promedio) y 2010 (puntual, fuente no especificada) no son del mismo tipo.

---

## Preguntas probables y respuestas

- **¿Por qué solo 2004-2009?** Son los años con cuotas, y la cuota explica casi todo. Imputarla en 1998-2003 inventaría la variable principal.
- **¿Por qué no accuracy?** Un ganador entre ~25 por año; "nadie gana" tiene 96% de accuracy. Uso puesto del ganador, Spearman y clasificados acertados.
- **El baseline acierta 6/6, ¿para qué el modelo?** Para tener una predicción completa (incluida la semifinal) y medir si los datos del festival añaden algo. Resultado: casi nada, lo cual es una conclusión útil.
- **¿Por qué validar por año?** Las filas de un año comparten votantes y rivales; mezclarlas filtra información.
- **¿Por qué no validación temporal?** Con 5 años, los primeros folds tendrían 1-2 años de entrenamiento. Asumo relación estable entre años. Es una debilidad aceptable.
- **¿Por qué Ridge y no árboles?** 146 filas, relación casi lineal en log-cuota; los árboles sobreajustarían.
- **¿Por qué alpha=1?** Valor por defecto, no ajustado. Con la cuota dominando no espero cambios grandes; lo ajustaría con RidgeCV dentro de cada fold.
- **¿Por qué `points_share` y no `Points`?** El total de puntos cambia con el número de votantes (2088 en 2004, 2436 en 2008-2009).
- **¿Es leakage usar la cuota?** La de 2010 es previa, válida. La histórica podría incluir apuestas durante la votación; no se puede verificar. Lo cuantifiqué: sin cuota el modelo casi no predice.
- **¿Por qué no usaste los votos?** No son variable directa en 2010. Se pueden derivar variables de afinidad histórica; las probé fuera del notebook y no mejoraban.
- **¿Por qué el índice no es una probabilidad?** El % de puntos esperado no es probabilidad de ganar y el modelo subestima la concentración en el ganador. Calibrar requiere más años.
- **¿Por qué Eslovaquia sale tan arriba?** Grupo de semifinal débil + región. Es probablemente una sobreestimación del efecto región.
- **¿Por qué Alemania por debajo de Azerbaiyán?** Cuotas casi iguales; la región Europa Occidental tiene coeficiente negativo en la final.
- **¿Qué harías con más tiempo?** (1) Cuota relativa por año en vez de log-cuota. (2) Ajuste de hiperparámetros. (3) Modelar la matriz de votos (quién da cuántos puntos a quién). (4) Cuotas actualizadas justo antes de la final.
- **¿Cómo se calcula el azar en semifinal (27)?** 10 x 10/n por semifinal, sumado.
- **¿Por qué no hay `random_state`?** Ridge y la logística (lbfgs) son deterministas y la validación por año no baraja filas.
- **¿Cómo sabes que el índice funciona?** No lo validé de punta a punta: validé el modelo de semifinal (48/60) y el de final (sobre finalistas reales) por separado. Para validarlo habría que recalcular el índice en cada año de validación con todos los participantes y ver en qué puesto queda el ganador.
- **¿Por qué Spearman y no Pearson?** Importa el orden, no la cantidad de puntos. En el EDA, la cuota tiene cola larga y Pearson subestima la relación (0.57 frente a 0.70). En la evaluación comparo puestos, y ahí Spearman y Pearson coinciden.
- **¿Por qué dos variables de cuota en semifinal?** Hipótesis de que la posición en el grupo importa más que la cuota absoluta. Comprobado después: correlación 0.97 y mismo rendimiento con solo `log_price` (48/60). Es redundante; la quitaría. Reconocerlo es mejor que defenderla.
- **¿Por qué los coeficientes Home/Away son simétricos?** One-hot sin `drop` + regularización L2; lo interpretable es la diferencia.

---

## Conceptos a repasar

- Spearman vs Pearson.
- Leakage en datos agrupados; GroupKFold / LeaveOneGroupOut; validación temporal.
- Regresión logística: log-odds, regularización (C), interpretación de coeficientes con variables estandarizadas.
- Ridge: penalización L2, por qué estandarizar, efecto en categorías con pocos casos.
- AUC: qué mide (probabilidad de que un positivo tenga score mayor que un negativo).
- Cuotas decimales, probabilidad implícita (1/cuota) y overround.
- One-hot con y sin `drop`; trampa de la variable ficticia.
- Baseline fuerte vs débil y por qué un buen baseline es parte del análisis.

---

## Contexto externo (no está en el notebook, no usar como resultado)

- Este dataset es el de la primera competición de Kaggle (2010), que pedía predecir las votaciones de Eurovisión 2010 explotando patrones de voto. El consenso de Kaggle puso a Azerbaiyán 1ª y Alemania 2ª (igual que este modelo). Acertaron 7 del top 10 final frente a 5 del mercado. Fuentes: KDnuggets, Club Troppo.
- Resultado real 2010: ganó Alemania (Lena, "Satellite"); Azerbaiyán 5ª. Alemania y Azerbaiyán eran cofavoritas a 5/2 (William Hill) 23 días antes. Fuente: ESCXtra.
- Literatura: modelos de redes de voto aportan poco sobre las tablas de apuestas (arXiv 2002.06609). Los modelos públicos conocidos (Kavanagh) combinan cuotas con voto de vecinos y diáspora.
- Si sale en la entrevista: decir que no se usó en el notebook porque en la prueba no se conoce el resultado.

---

## Exploración adicional no entregada (v2: `20261010_LuisDioses_v2.ipynb`)

Útil si preguntan "¿probaste usar los votos?".

- Variable de afinidad: bias(v, c) = puntos medios que v dio a c en años anteriores − puntos medios que c recibe de cualquiera. Para cada fila, media del bias sobre los votantes de ese concurso. Solo votos de años anteriores (para 2004 se usan 1998-2003). Pares sin historia = 0.
- Pares con más afinidad: Grecia→Chipre, Rumanía↔Moldavia, España↔Andorra (esperables); Georgia→Bélgica (ruido, pocas observaciones).
- Resultado: Spearman 0.18 con % de puntos, AUC 0.45 en semifinal, nube sin tendencia. Variantes (mínimo 2-4 observaciones por par, sin centrar) no superan AUC 0.65 / Spearman 0.2. Mantuve la definición original para no elegir por resultados.
- Tabla CV (2004-2008):

| | Semi /60 | Semi AUC | Ganador top-1 /5 | Spearman medio |
|---|---|---|---|---|
| Mercado | 46 | 0.862 | 5 | 0.701 |
| Sin cuota (afinidad + categóricas) | 35 | 0.631 | 0 | 0.253 |
| Cuota + categóricas (= v1) | 48 | 0.859 | 4 | 0.722 |
| Cuota + afinidad | 46 | 0.837 | 5 | 0.706 |
| Todo | 48 | 0.867 | 4 | 0.720 |

- Hipótesis de por qué no funciona: promediado sobre ~40 votantes, el favoritismo de 2-3 vecinos se diluye; y el mercado ya lo incorpora.
