# Contexto

Estoy resolviendo una prueba técnica para un puesto SENIOR en NTT Data. La entrega es un notebook de Jupyter.
Se permite usar IA, pero con criterio: el notebook debe reflejar decisiones que yo entiendo y puedo defender.
Lo que más se evalúa es cómo expongo cada paso y por qué tomé cada decisión.

# Forma de trabajo

1. Antes de escribir código, lee el enunciado y los datos, y propónme el enfoque en 5-6 puntos
   (qué harías en cada sección y por qué). Espera mi validación antes de desarrollar.
2. Desarrolla sección por sección, no todo de golpe. Al terminar cada sección, ejecuta las celdas
   y verifica que corren sin errores.
3. Si una decisión es ambigua o tiene alternativas igual de válidas, pregúntame en vez de elegir tú.
4. No inventes datos, resultados ni métricas. Toda conclusión debe salir de outputs reales del notebook.
5. Mantén las técnicas simples y estándar. No apliques métodos de nicho ni sofisticados (splines, bootstrap de
   intervalos, calibración, etc.) salvo que el problema lo exija de verdad. Si hay
   una opción más simple que rinde parecido, usa esa. Todo lo que entre al notebook debe poder explicarlo yo en
   una entrevista; ante la duda, pregúntame antes de incluirlo.
6. Tengo 2 horas para la entrega completa. Prioriza lo esencial del enunciado, evita análisis opcionales y
   mantén el notebook corto.

# Estilo del notebook (entrega)

Debe parecer escrito por un profesional senior: sobrio, directo, sin relleno.

Código:
- Comentarios solo cuando algo no es obvio (ej: `# drop: 98% nulos`). Nunca comentarios que repiten lo que hace el código.
- Nada de emojis, prints decorativos ("✅ Listo") ni separadores con ===.
- Nombres de variables y funciones descriptivos; funciones pequeñas y reutilizables cuando haya lógica repetida.
- Docstrings solo si la función no es evidente, y de una línea.
- Imports agrupados en la primera celda de código.
- random_state fijo para reproducibilidad.
- Gráficos limpios: título, ejes con nombre, sin estilos recargados.

Markdown:
- Frases cortas, tipo notas de trabajo. Ej: "Mediana en vez de media: `ingresos` tiene sesgo fuerte (skew 2.3)."
- Justifica solo las decisiones relevantes (imputación, encoding, elección de métrica, modelo, validación),
  no cada paso trivial.
- Debajo de cada gráfico importante, 1-2 líneas con lo que se concluye de él.
- Evita frases típicas de IA: "Es importante destacar", "En conclusión", "Cabe mencionar", "Vamos a",
  "A continuación procederemos", "robusto", "exhaustivo". Evita listas largas de viñetas.
- Nada de títulos grandilocuentes. Encabezados simples y numerados.

# Estructura sugerida (adáptala al enunciado)

0. Resumen: objetivo, enfoque, resultado clave y recomendación (máx. 5 líneas, se escribe al final)
1. Entendimiento del problema y métrica de éxito
2. Carga y calidad de datos
3. EDA (solo lo que aporta a decisiones posteriores)
4. Preprocesamiento
5. Modelado: baseline simple primero, luego mejoras justificadas
6. Evaluación: métrica adecuada al problema, validación cruzada, análisis de errores
7. Interpretación en lenguaje de negocio
8. Conclusiones, limitaciones y próximos pasos

Buenas prácticas senior que deben notarse sin explicarlas de más:
- Evitar data leakage (split antes de transformar, Pipeline/ColumnTransformer de sklearn).
- Comparar contra un baseline.
- Elegir la métrica según el problema (ej: desbalance → no usar accuracy sola).
- Mencionar limitaciones con honestidad.

# Archivo de notas (NO se entrega)

En paralelo, mantén un archivo `NOTAS_DEFENSA.md` (fuera del notebook) con:
- Cada decisión tomada, el porqué completo y las alternativas descartadas con su razón.
- Preguntas que un evaluador podría hacerme sobre cada sección y cómo responderlas.
- Conceptos que debería repasar antes de una posible entrevista de revisión.
Este archivo sí puede ser detallado.

# Idioma

Notebook y notas en español. Nombres de variables y funciones en inglés.
