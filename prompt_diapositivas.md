# Prompt para generar las diapositivas

Copiar todo el bloque de abajo y pegarlo en la herramienta de generación de
presentaciones.

---

## Rol

Eres un diseñador de presentaciones académicas para una sustentación de
analítica de datos en salud (materia Healtech, Universidad Externado de
Colombia, profesora Eliana González Artunduaga).

## Objetivo

Generar una presentación de **exactamente 9 diapositivas** en **español**, con
la estructura obligatoria que se detalla más abajo. Cada sección del pipeline se
defiende en el número de láminas indicado: ni una más, ni una menos.

## Reglas duras

1. **No inventes ningún dato.** Todas las cifras, nombres de genes, valores y
   conteos que puedas usar están en la sección "Datos verificados". Si algo no
   está ahí, no lo afirmes.
2. **Usa el texto literal** que aparece en cada diapositiva bajo "Texto de la
   lámina". Puedes reacomodar saltos de línea o acortar una viñeta si no cabe,
   pero no reescribas el contenido ni añadas afirmaciones nuevas.
3. **Construye el gráfico descrito** en cada diapositiva bajo "Gráfico". Los
   datos para dibujarlo están dados; respeta el tipo de gráfico indicado.
4. No agregues diapositivas de agradecimiento, agenda, índice ni preguntas.
5. La sustentación no tiene límite de tiempo, así que prioriza claridad
   conceptual sobre concisión extrema, pero mantén la lámina legible.
6. Usa notación técnica correcta: $P \gg N$, AOR, TPM, L1/L2.
7. Paleta sobria y consistente en todo el mazo: un color de acento para el
   tumor y otro para el tejido normal, grises para el resto. Nada de degradados
   ni iconos decorativos.

## Datos verificados

**Cohorte elegida (Paso 1)**
- Proyecto: `TCGA-KIRP` — Kidney Renal Papillary Cell Carcinoma.
- Sitio primario (GDC): Kidney. Programa: TCGA. Casos en el portal: **291**.
- Indexación: se consultaron las **33 cohortes TCGA** vía el endpoint
  `/projects` de la API del GDC, no navegando el portal a mano.
- Justificación clínica: el carcinoma renal papilar es el segundo subtipo
  histológico más frecuente del cáncer renal (~15–20 % de los casos), con dos
  variantes de biología distinta: tipo 1 asociado a alteraciones de *MET*, y
  tipo 2 ligado a *CDKN2A*, *SETD2* y al fenotipo metilador (*FH*).
- Razón analítica: la cohorte incluye tumor y tejido normal adyacente, lo que
  habilita un contraste supervisado tumor vs. normal.

**Herramienta y pipeline (Paso 2)**
- Entorno: Python 3.14 en un `.venv` local del repositorio + Jupyter Notebook.
- Librerías: `requests` (API REST), `pandas` (matriz), `scikit-learn` 1.9.1
  (modelado), `numpy`.
- Argumento de elección: la API del GDC es REST/JSON, así que `requests` y
  `pandas` bastan y no hace falta la pila Bioconductor de R. El `.venv` fija
  versiones y hace reproducible la ejecución; un solo cuaderno encadena
  descarga, consolidación y modelado sin archivos intermedios.
- Consulta: endpoint `/files` filtrando por `cases.project.project_id`
  = TCGA-KIRP, `data_type` = Gene Expression Quantification y
  `workflow_type` = STAR - Counts.
- Resultado de la consulta: **323 archivos** de **290 pacientes**, repartidos en
  290 `Primary Tumor`, 32 `Solid Tissue Normal` y 1 `Additional - New Primary`.
- Descarga: un único POST al endpoint `/data` devuelve los 323 archivos
  empaquetados en un `.tar.gz` de 331 MB.
- Detalle técnico defendible: el GDC responde con `Transfer-Encoding: chunked`
  y **sin `Content-Length`**, así que una conexión cortada termina el stream sin
  lanzar error y no hay tamaño esperado contra el cual compararlo. La solución
  fue usar la descompresión como verificación: el paquete conserva un nombre
  provisional (`.parte`) hasta que se extrae con éxito, y solo entonces pasa a
  `.tar.gz`. Así una descarga rota nunca se da por buena y no se repite trabajo
  ya hecho.
- Consolidación: se leyó la columna `tpm_unstranded` de cada TSV (TPM, ya
  normalizado por longitud del gen y profundidad de secuenciación, por eso las
  muestras son comparables sin un paso extra de normalización).

**Reducción y genes (Paso 3)**
- Matriz ómica cruda: **60.660 genes × 323 pacientes**.
- Filtro previo: genes con TPM ≥ 1 en al menos el 20 % de las muestras →
  **19.676 genes**. Transformación: log2(TPM + 1), luego estandarizada.
- Variable objetivo `y`: indicador binario tumor (1) / normal (0), derivado del
  campo `cases.samples.sample_type` de los metadatos del GDC — no de la matriz
  de expresión, así que no hay fuga de información. Reparto: **291 tumor / 32
  normal**.
- Modelo: **regresión logística** (y es binaria), comparando dos penalizaciones
  con validación cruzada de 5 pliegues y `scoring="neg_log_loss"`.
- **Intento 1 — L1 (Lasso):** C = 2.7826; conserva **50 genes de 19.676**
  (anula 19.626); exactitud en entrenamiento 1.000; AOR entre 0.639 y 0.852.
- **Intento 2 — L2 (Ridge):** C = 21.5443; conserva **los 19.676 genes**;
  exactitud en entrenamiento 1.000; AOR entre 0.987 y 0.990.
- **Decisión: L1.** Ambas clasifican perfecto, así que el ajuste no las
  distingue: con 19.676 genes para 323 muestras separar los grupos es trivial y
  el riesgo real es el sobreajuste. Lo que las separa es la esparsidad: las AOR
  de L2 quedan todas a un paso de 1, de modo que cortar en 10 sería arbitrario,
  mientras que L1 hace la selección como parte del ajuste. 6 de los 10 genes
  coinciden entre ambos top-10.
- Reproducibilidad: entre genes correlacionados L1 conserva uno y anula el resto
  de forma casi intercambiable, así que el top-10 depende del barajado interno
  del solver. Se fijó `random_state=0`; esta lista es *una* firma válida entre
  varias casi equivalentes.

**Interpretación (AOR)**
- Se interpretan **AOR** (*adjusted odds ratio*), AOR = e^β. Como la matriz está
  estandarizada, es el factor por el que se multiplican las odds de que la
  muestra sea tumor por **cada desviación estándar de aumento en log2(TPM+1)**,
  ajustada por los demás genes del modelo.
- AOR > 1 acompaña al tumor; AOR < 1 acompaña al tejido normal.
- **Los 10 genes tienen AOR < 1**: la firma no recoge genes que el tumor
  encienda, sino el programa del nefrón diferenciado cuya pérdida acompaña al
  carcinoma papilar.

**Los 10 genes seleccionados**

| Gen | AOR | TPM medio | Varianza log2 | Función | Relación con el KIRP |
| --- | --- | --- | --- | --- | --- |
| DUSP9 | 0.639 | 19.32 | 4.82 | Fosfatasa de doble especificidad (MKP-4) que inactiva las MAPK | Freno de la vía MAPK: al perderse, esa señal proliferativa queda sin contrapeso |
| CASR | 0.734 | 5.11 | 2.58 | Receptor sensor de calcio (GPCR) del nefrón distal | Gobierna la reabsorción de Ca²⁺; se pierde con la identidad del túbulo distal |
| UMOD | 0.756 | 882.84 | 13.62 | Uromodulina (Tamm-Horsfall), exclusiva de la rama ascendente gruesa | Gen más expresado de la firma y el de mayor varianza: marcador clásico de nefrón funcional |
| AQP2 | 0.775 | 181.08 | 9.85 | Acuaporina 2, canal de agua del conducto colector regulado por vasopresina | Marcador de conducto colector diferenciado |
| ATP6V1C2 | 0.787 | 2.93 | 1.38 | Subunidad C2 de la V-ATPasa, células intercaladas | Acidificación urinaria; función propia del conducto colector |
| AC099684.2 | 0.805 | 1.39 | 0.81 | lncRNA sin función establecida | Muestra que la selección no se limita a genes codificantes |
| TMEM178A | 0.813 | 6.20 | 1.75 | Proteína transmembrana poco caracterizada, reguladora negativa de señalización de calcio | Papel en riñón aún no establecido |
| TMEM174 | 0.827 | 22.22 | 3.99 | Proteína transmembrana de expresión prácticamente restringida al riñón | Marcador de túbulo proximal |
| ALDOB | 0.844 | 484.35 | 11.20 | Aldolasa B, metabolismo de fructosa y gluconeogénesis en túbulo proximal | Su caída es hallazgo repetido en carcinoma renal: reprogramación metabólica |
| IRX2 | 0.852 | 3.65 | 2.16 | Factor de transcripción homeobox Iroquois-2 | Regulador del desarrollo y la segmentación del nefrón |

**Salidas de consola a citar textualmente en la diapositiva 8**

```
Matriz ómica cruda:      60660 genes x 323 pacientes
Matriz analítica final:  10 genes x 323 pacientes
```

**Salvedades que deben aparecer (honestidad metodológica)**
- Las AOR vienen de un ajuste penalizado: están encogidas hacia 1, son
  conservadoras, y esta estimación no produce valores p ni intervalos de
  confianza.
- El contraste es tumor contra tejido normal adyacente: habla del desarrollo del
  tumor y de su diagnóstico, **no** de agresividad ni supervivencia, que
  exigirían modelar estadio o tiempo de seguimiento.

## Las 9 diapositivas: texto literal y gráficos

### Sección 0 — Identificación

#### Diapositiva 1 — Portada

**Título:** KIRP Health tech 1

**Subtítulo:** Analítica traslacional sobre TCGA-KIRP: del portal GDC a una
firma de 10 genes

**Texto de la lámina:**
- Juan Pablo Carrillo Hortua · Mariana Uribe Vanegas · Sara Gabriela Chizaba
  Cardenas · Juana Gomez · Martin Ricardo Santos
- Healtech / Analítica de Datos en Salud — Universidad Externado de Colombia
- 21 de septiembre de 2026

**Gráfico:** sin gráfico de datos. Portada tipográfica: título grande alineado a
la izquierda, subtítulo en gris, y los integrantes en una sola línea o en dos
columnas. Si se quiere un motivo visual, una silueta discreta de riñón o una
retícula de puntos al margen derecho, en gris claro, nunca detrás del texto.

---

### Sección 1 — ¿Por qué eligió ese cáncer?

#### Diapositiva 2 — Contexto clínico

**Título:** ¿Por qué el carcinoma renal papilar?

**Texto de la lámina:**
- El carcinoma renal papilar es el segundo subtipo histológico más frecuente del
  cáncer renal: cerca del 15–20 % de los casos.
- No es una entidad única. El tipo 1 se asocia a alteraciones de *MET*; el tipo
  2, a *CDKN2A*, *SETD2* y al fenotipo metilador ligado a *FH*.
- Esa heterogeneidad molecular lo vuelve un buen candidato para buscar una firma
  de expresión, no un biomarcador aislado.
- Razón analítica: la cohorte trae tumor y tejido normal adyacente del mismo
  órgano, lo que habilita un contraste supervisado tumor vs. normal.

**Gráfico:** lámina partida en dos mitades.
- *Izquierda:* gráfico de dona con la participación del subtipo papilar dentro
  del cáncer renal. Un solo segmento resaltado en el color de acento cubriendo
  ~17 % (etiquetado "15–20 %, rango aproximado"), el resto en gris claro
  rotulado "otros subtipos de cáncer renal". En el centro de la dona, el texto
  "2.º subtipo más frecuente".
- *Derecha:* comparación de los dos subtipos en dos tarjetas apiladas. Tarjeta
  "Tipo 1" con el gen *MET*; tarjeta "Tipo 2" con *CDKN2A*, *SETD2* y *FH*.
  Cada tarjeta con un borde de color distinto y los genes en cursiva.

#### Diapositiva 3 — Metadata GDC

**Título:** Ficha técnica de la cohorte (portal GDC)

**Texto de la lámina:**
- Sigla TCGA: **TCGA-KIRP** — Kidney Renal Papillary Cell Carcinoma.
- Órgano de origen: riñón (`primary_site: Kidney`). Programa: TCGA.
- Volumen muestral en el portal: **291 cases**.
- La consulta transcriptómica (RNA-Seq, *STAR - Counts*) devolvió **323
  archivos de 290 pacientes**: 290 Primary Tumor, 32 Solid Tissue Normal y 1
  Additional - New Primary.
- Las 33 cohortes TCGA se indexaron por código, vía el endpoint `/projects`, no
  navegando el portal a mano.

**Gráfico:** dos elementos, uno encima del otro.
- *Arriba:* barras horizontales con el volumen muestral de las cohortes TCGA,
  para situar KIRP entre sus pares. Usar estos valores: BRCA 1098, GBM 617, OV
  608, LUAD 585, UCEC 560, KIRC 537, HNSC 528, LGG 516, THCA 507, LUSC 504 y
  **KIRP 291**. Todas las barras en gris salvo KIRP, en el color de acento, con
  su valor rotulado. Título del gráfico: "KIRP entre las 33 cohortes TCGA".
- *Abajo:* una única barra apilada horizontal al 100 % con la composición de las
  323 muestras: 290 Primary Tumor, 32 Solid Tissue Normal, 1 Additional - New
  Primary. Colores: tumor en el acento "tumor", normal en el acento "normal",
  el caso adicional en gris. Etiquetar cada segmento con su conteo absoluto.

---

### Sección 2 — ¿Por qué eligió esa herramienta?

#### Diapositiva 4 — Arquitectura computacional

**Título:** ¿Por qué Python + Jupyter y no R o Colab?

**Texto de la lámina:**
- La API del GDC es REST/JSON: `requests` y `pandas` bastan, sin la pila
  Bioconductor que exige el camino en R.
- El entorno es un `.venv` local del repositorio: fija las versiones de
  `pandas`, `numpy` y `scikit-learn` 1.9.1, y hace reproducible la ejecución.
- Se descartó Colab porque el paquete descomprimido ocupa 1,6 GB y la sesión no
  es persistente: cada reinicio obligaría a repetir la descarga.
- Un solo cuaderno encadena descarga, consolidación y modelado sin archivos
  intermedios.

**Gráfico:** matriz de comparación de 3 columnas × 4 filas.
- Columnas: "Python + Jupyter (.venv)", "R + RStudio", "Google Colab".
- Filas: "Acceso a la API", "Reproducibilidad de versiones", "Almacenamiento de
  1,6 GB", "Dependencias".
- Celdas con un símbolo de valoración (✓ / ~ / ✗) más tres o cuatro palabras.
  La columna elegida, resaltada con fondo tenue del color de acento y un borde;
  las otras dos en gris. Ejemplos de celda: Python/API "REST nativo con
  requests"; R/Dependencias "requiere Bioconductor"; Colab/Almacenamiento
  "sesión no persistente".

#### Diapositiva 5 — Pipeline de interoperabilidad

**Título:** Cómo conversa el código con la API del GDC

**Texto de la lámina:**
- `/projects` indexa las 33 cohortes TCGA y extrae la ficha técnica.
- `/files` construye el manifiesto: `file_id` para descargar y `sample_type`
  para etiquetar cada muestra.
- `/data` devuelve, en un único POST, los 323 archivos empaquetados en un
  `.tar.gz` de 331 MB.
- Problema de integridad: el GDC responde con `Transfer-Encoding: chunked` y
  **sin `Content-Length`**, así que una conexión cortada termina el stream sin
  lanzar error y no hay tamaño contra el cual compararlo.
- Solución: la descompresión *es* la verificación. El paquete queda como
  `.parte` hasta que se extrae bien y solo entonces pasa a `.tar.gz`, de modo
  que una descarga rota nunca se da por buena ni se repite trabajo ya hecho.

**Gráfico:** diagrama de flujo horizontal de cinco cajas conectadas por flechas,
ocupando el ancho de la lámina:

`/projects` → `/files` → `/data` → descompresión = verificación → matriz

Bajo cada caja, una línea de detalle: "33 cohortes", "manifiesto: 323 archivos",
".tar.gz de 331 MB", "323 TSV extraídos", "60.660 × 323". Debajo de la cuarta
caja, colgando con una flecha curva, un recuadro de alerta en color distinto con
el texto: "chunked sin Content-Length → el corte no lanza error". Y a su lado,
un recuadro en verde o en el acento con: "`.parte` → `.tar.gz` solo si extrae
bien". Las tres primeras cajas llevan el icono o la etiqueta "POST"/"GET" según
corresponda: GET en `/projects`, POST en `/files` y `/data`.

---

### Sección 3 — ¿Por qué eligió esos genes?

#### Diapositiva 6 — Matemática de la reducción

**Título:** $P \gg N$: por qué hay que regularizar

**Texto de la lámina:**
- 19.676 genes para 323 muestras: sin penalización el ajuste tiene infinitas
  soluciones y memoriza el ruido.
- Como `y` es binaria (tumor/normal) se ajusta una **regresión logística**, con
  `C` elegido por validación cruzada de 5 pliegues.
- **L2 (Ridge):** C = 21,5443 — conserva los 19.676 genes, AOR entre 0,987 y
  0,990.
- **L1 (Lasso):** C = 2,7826 — conserva 50 genes y anula 19.626, AOR entre
  0,639 y 0,852.
- Ambas clasifican perfecto (exactitud 1,000), así que el ajuste no decide:
  decide la esparsidad. Con L2 todas las AOR quedan a un paso de 1 y cortar en
  10 sería arbitrario; L1 hace la selección como parte del ajuste.

**Gráfico:** dos paneles lado a lado con **el mismo eje horizontal** (AOR, de
0,60 a 1,05), para que la comparación sea visual e inmediata.
- *Panel izquierdo, "L2 — Ridge (19.676 genes)":* nube de puntos o histograma
  muy estrecho apelotonado entre 0,987 y 0,990, todo pegado a la línea de
  referencia AOR = 1.
- *Panel derecho, "L1 — Lasso (50 genes)":* puntos claramente dispersos entre
  0,639 y 0,852, todos a la izquierda de AOR = 1.
- En ambos paneles, una línea vertical punteada en AOR = 1 rotulada
  "sin efecto".
- Al pie, una franja con tres cifras comparadas: "genes conservados: 19.676 vs
  50", "rango de AOR: 0,987–0,990 vs 0,639–0,852", "exactitud: 1,000 vs 1,000".

#### Diapositiva 7 — Mapeo de los 10 genes

**Título:** Los 10 genes y su AOR

**Texto de la lámina:**
- AOR = e^β. Como la matriz está estandarizada, es el factor por el que se
  multiplican las *odds* de que la muestra sea tumor por **cada desviación
  estándar de aumento en log2(TPM+1)**, ajustada por los demás genes.
- AOR > 1 acompaña al tumor; AOR < 1 acompaña al tejido normal.
- Los 10 seleccionados tienen AOR < 1, de 0,639 (*DUSP9*) a 0,852 (*IRX2*).
- Lectura de ejemplo: *DUSP9* con AOR 0,639 significa que una desviación
  estándar más de expresión reduce ~36 % las odds de que la muestra sea tumor.
- *UMOD* es el más expresado de la firma (TPM medio 883) y el de mayor varianza
  (13,62).

**Gráfico:** barras horizontales con los 10 genes ordenados de menor a mayor
AOR (*DUSP9* arriba, *IRX2* abajo). Valores exactos: DUSP9 0,639 · CASR 0,734 ·
UMOD 0,756 · AQP2 0,775 · ATP6V1C2 0,787 · AC099684.2 0,805 · TMEM178A 0,813 ·
TMEM174 0,827 · ALDOB 0,844 · IRX2 0,852.
- Eje X = AOR, arrancando en 0,6 y llegando a 1,05.
- **Línea vertical punteada en AOR = 1**, rotulada "sin efecto": todas las
  barras quedan a su izquierda, que es el mensaje de la lámina.
- Barras en el color de acento "normal", con el valor de AOR rotulado al final
  de cada una. Nombres de gen en cursiva.
- Marcar *AC099684.2* con un matiz distinto o un asterisco y una nota al pie:
  "único lncRNA de la firma".

---

### Sección 4 — ¿Qué le dio de resultado?

#### Diapositiva 8 — Evidencia de la matriz ómica

**Título:** La matriz final, lista para modelar

**Texto de la lámina:**
- Matriz ómica cruda: **60.660 genes × 323 pacientes**.
- Filtro de expresión (TPM ≥ 1 en al menos el 20 % de las muestras): 19.676
  genes.
- Matriz analítica final: **10 genes × 323 pacientes**.
- El número de pacientes nunca cambia: la reducción actúa solo sobre la
  dimensión génica.

**Gráfico:** dos elementos.
- *Izquierda:* embudo o secuencia de tres rectángulos de anchura decreciente y
  la misma altura, que representan la dimensión génica: "60.660 genes" (ancho
  completo), "19.676 genes — filtro de expresión" (un tercio), "10 genes —
  logística L1" (una franja mínima). Bajo cada uno, "× 323 pacientes" repetido,
  para dejar claro que esa dimensión se conserva.
- *Derecha:* **espacio reservado para la captura de pantalla de la consola**,
  con marco fino y la leyenda "salida del cuaderno". La captura debe mostrar
  este texto, que también puede transcribirse en monoespaciada si la imagen no
  está lista:

```
Matriz ómica cruda:      60660 genes x 323 pacientes
Matriz analítica final:  10 genes x 323 pacientes
```

#### Diapositiva 9 — Curaduría y patología

**Título:** Qué dice la firma: el nefrón que se pierde

**Texto de la lámina:**
- Los 10 genes tienen AOR < 1: la firma no recoge genes que el tumor encienda,
  sino el programa del **nefrón diferenciado** cuya pérdida acompaña al tumor.
- Marcadores de segmento: *ALDOB* y *TMEM174* (túbulo proximal), *UMOD* (rama
  ascendente gruesa), *CASR* (túbulo distal), *AQP2* y *ATP6V1C2* (conducto
  colector).
- Reguladores de esa identidad: *DUSP9*, freno de la vía MAPK, e *IRX2*, factor
  de transcripción del desarrollo del nefrón.
- La caída de *ALDOB* es un hallazgo repetido en carcinoma renal: reprogramación
  metabólica con pérdida de la gluconeogénesis del túbulo proximal.
- Salvedades: las AOR vienen de un ajuste penalizado, están encogidas hacia 1 y
  no traen valores p ni intervalos de confianza. El contraste es tumor contra
  tejido normal adyacente, así que habla del desarrollo del tumor y de su
  diagnóstico, no de agresividad ni supervivencia.

**Gráfico:** esquema anatómico de un nefrón —glomérulo, túbulo proximal, asa de
Henle, túbulo distal y conducto colector— dibujado de forma esquemática y
limpia, con los genes anclados por líneas guía al segmento que les corresponde:
- Túbulo proximal → *ALDOB*, *TMEM174*
- Rama ascendente gruesa del asa de Henle → *UMOD*
- Túbulo distal → *CASR*
- Conducto colector → *AQP2*, *ATP6V1C2*

A un costado, en un recuadro aparte rotulado "reguladores de la identidad
tubular", los dos genes que no mapean a un segmento: *DUSP9* e *IRX2*. Y en un
segundo recuadro, "sin segmento asignado": *AC099684.2* (lncRNA) y *TMEM178A*.
Cada etiqueta de gen lleva su AOR entre paréntesis. Usar una flecha descendente
o un tono desvaído sobre el nefrón para transmitir que lo que se mide es la
**pérdida** de ese programa, no su activación.

## Identificación (contenido de la diapositiva 1)

- **Título del proyecto:** KIRP Health tech 1
- **Integrantes:**
  - Juan Pablo Carrillo Hortua
  - Mariana Uribe Vanegas
  - Sara Gabriela Chizaba Cardenas
  - Juana Gomez
  - Martin Ricardo Santos
- **Fecha:** 21 de septiembre de 2026
