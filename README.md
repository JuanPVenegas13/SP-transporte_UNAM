# transporte_UNAM — Limpieza y preparación del dataset ENMT

Encuesta Nacional de Movilidad y Transporte. Este repo cubre la **limpieza** y la **construcción de
la base analítica** del proyecto (a cargo de Juan Pablo Venegas). El modelado —las redes bayesianas
y la inferencia— se hace **en R**, a partir de `enmt_bn_final.csv`.

## Cómo correrlo

```bash
uv sync                              # instala dependencias en .venv
uv run jupyter lab scripts/          # abre los notebooks
```

O sin abrir Jupyter, en orden (cada uno depende del anterior):

```bash
uv run jupyter nbconvert --to notebook --execute --inplace scripts/01_limpieza_enmt.ipynb
uv run jupyter nbconvert --to notebook --execute --inplace scripts/02_seleccion_columnas.ipynb
uv run jupyter nbconvert --to notebook --execute --inplace scripts/03_base_final.ipynb
```

## Estructura

```
transporte_UNAM/
├── data/
│   ├── enmt_unam.csv               ← microdatos originales (versionado, no tocar)
│   ├── diccionario_enmt.xls        ← libro de códigos SPSS (versionado)
│   └── processed/                  ← generado por los notebooks (NO versionado)
│       ├── enmt_limpio.csv         ← 01: base limpia completa (1,191 × 556)
│       ├── enmt_analitico.csv      ← 01: subconjunto temático (108 columnas)
│       ├── diccionario.json        ← 01: etiquetas de variable y de valor
│       ├── reporte_calidad.csv     ← 01: cobertura por columna
│       ├── mapa_nodos.csv          ← 02: reporte de cobertura de los nodos candidatos
│       ├── enmt_bn_final.csv       ← 03: BASE FINAL, la que se lee en R
│       ├── enmt_bn_codebook.csv    ← 03: qué es cada nodo y en qué orden van sus niveles
│       └── enmt_bn_q1..q4.csv      ← 03: un archivo por query, filtrado y sin NA
├── scripts/
│   ├── 01_limpieza_enmt.ipynb      ← limpieza: centinelas, tipos, PII
│   ├── 02_seleccion_columnas.ipynb ← exploración de variables candidatas
│   └── 03_base_final.ipynb         ← nodos derivados, recodificación y base final
├── pyproject.toml
└── uv.lock
```

`data/processed/` está en `.gitignore` a propósito: esos archivos se **reproducen** corriendo
los notebooks. Lo que se versiona es el código y los datos originales.

## El problema principal: nulos invisibles

El CSV original **no tiene un solo `NaN`**, y eso engaña. Viene de SPSS, que codifica los
faltantes como números centinela. Si cargas el archivo tal cual y sacas un promedio, esos
códigos entran como si fueran respuestas reales.

Ejemplo concreto: en `p31` (escala 0–10) el valor `99` significa "no contestó". Sin limpiar,
el promedio de esa variable sale disparado.

Además, el archivo **no está en UTF-8** sino en `latin-1`. `pd.read_csv()` sin más truena.

## Qué hace el notebook

| Paso | Qué resuelve |
|---|---|
| 2 | Carga probando codificaciones hasta dar con `latin-1` |
| 4 | Parsea el libro de códigos SPSS → etiquetas de variable y de valores |
| 5 | Nombres de columna a `snake_case` (`Tam_loc` → `tam_loc`) |
| 6 | **Centinelas → `NaN`**, según 4 reglas (abajo) |
| 7 | Duplicados: filas, `con1`, `folio` |
| 8 | Elimina columnas vacías, constantes y **datos personales** |
| 9 | Tipos: `float64` → `Int64` nullable (no más `3.0` donde debe decir `3`) |
| 10 | Normaliza el texto libre de `p12` y lo agrupa en categorías |
| 11 | Valida rangos imposibles; reporta atípicos sin eliminarlos |
| 12 | Arma el subconjunto analítico |
| 13 | Exporta los 4 archivos |

### Las reglas de conversión a `NaN`

No se convierten los centinelas a ciegas: en `p4` el `4` significa "Nada" (respuesta real) y en
`p1a_1` el `8` significa "No sabe". Por eso todo se decide **variable por variable** con el
diccionario en la mano.

- **A — `-1` global.** Verificado contra el diccionario: `-1` nunca es un código válido en
  ninguna de las 607 variables con catálogo. Es el "no aplica" por salto de pregunta.
- **B — NS/NC por variable.** Solo los códigos que el catálogo de *esa* variable etiqueta como
  `NS`, `NC` o `No aplica`.
- **B2 — el `97` no documentado.** El codebook etiqueta `97: No aplica` en la familia `p15_*`
  pero omite esa fila en `p18_*` y `p1c_*`, donde el `97` aparece en escalas de 0 a 10. Es un
  hueco del diccionario, no un dato. Se aplica solo a variables que ya usan la convención
  `98: NS` / `99: NC`.
- **C — centinelas de texto.** Cadenas vacías y `999-9` en `ageb`.
- **D — fuera de rango.** Horas > 23, escalas 0–10 fuera de rango, etc. Tras aplicar B2 esta
  regla ya no encuentra nada, lo que confirma que la captura fue consistente.

## Resultados

| | Antes | Después |
|---|---|---|
| Filas | 1,191 | 1,191 |
| Columnas | 698 | 556 (analítico: 108) |
| Codificación | latin-1 | UTF-8 |
| Celdas `NaN` | 0 | 421,949 |
| % de faltantes | 0.0 % | 70.3 % |

**584,400 celdas** convertidas a `NaN` (548,939 por regla A, 19,061 por B2, 10,336 por C,
6,064 por B). De esas, 162,451 estaban en columnas que después se eliminaron.

**144 columnas eliminadas:** 67 vacías, 69 constantes, 9 con datos personales.

Ese 70 % de faltantes **no es un error**. Viene casi todo de saltos de pregunta del cuestionario:
a quien no tiene coche no se le pregunta cuánto gasta en gasolina, y esa celda queda como "no
aplica". El dataset analítico, que filtra las columnas con más de 70 % de faltantes, se queda
en **7.9 % de faltantes** — ahí sí se puede trabajar.

## Datos personales

Las columnas `h8_1` … `h8_15` traían **nombres de pila reales** de los integrantes de cada hogar
(JOSE, ANTONIO, PEDRO…). Se eliminaron. El conteo de integrantes se conserva en `n_ind`, que es
lo único que sirve para análisis.

## Cosas que hay que saber antes de usar el dataset

- **Los datos están en códigos numéricos, no en etiquetas.** Para pasarlos a texto legible usa
  `data/processed/diccionario.json` o la función `etiquetar()` de la última celda del notebook.
- **No se imputó nada.** Si tu análisis lo necesita, hazlo en tu propio notebook y documenta el
  método. Así no arrastramos supuestos de otros.
- **No se eliminó ninguna fila.** Están los 1,191 encuestados. Para cualquier estimación
  poblacional usa los factores de expansión: `pondi2` (individual), `pondi_v` (vivienda),
  `pondi_h` (hogar).
- **`folio` no es un ID.** Solo tiene 25 valores distintos y se repite entre estados: es un
  consecutivo dentro del conglomerado muestral. El ID de encuestado es **`con1`**. La llave de
  vivienda es `edo + muni + loca + ageb + folio`.
- **`ing_ind` e `ing_fam` usan `0` sin etiqueta** en el diccionario. En contexto significa "sin
  ingreso declarado", no un faltante, así que se dejaron tal cual. Si tu análisis los trata
  distinto, déjalo escrito.
- **Los atípicos se reportan pero no se eliminan.** En una encuesta con factores de expansión,
  borrar respuestas raras pero posibles sesga las estimaciones.

## Dataset analítico

`enmt_analitico.csv` son 108 columnas a nivel individuo, agrupadas por bloque temático:
identificación, geografía, ponderadores, sociodemográficos, vivienda, uso de modos de transporte,
caminata, automóvil, seguridad vial, transporte público y discapacidad.

Quedó fuera el *roster* del hogar (`h9_*` … `h26_*`, repetido hasta 15 veces) y todo lo que tenía
más de 70 % de faltantes.

## Base final para las redes bayesianas (notebook 03)

`enmt_bn_final.csv` son los 1,191 encuestados con los nodos ya recodificados **como texto**, para que
en R salgan como factores sin traducir códigos:

```r
d  <- read.csv("data/processed/enmt_bn_final.csv", stringsAsFactors = TRUE)
q4 <- read.csv("data/processed/enmt_bn_q4.csv",    stringsAsFactors = TRUE)
```

Los faltantes van escritos como `NA` y los indicadores de auditoría como `TRUE`/`FALSE`, así que
`read.csv` los lee bien sin argumentos extra. El **orden de los niveles** no sobrevive a un CSV:
está en `enmt_bn_codebook.csv`, columna `niveles_en_orden`.

Los archivos `enmt_bn_q1..q4.csv` ya vienen filtrados a su universo y **sin `NA`**, que es lo que
`bnlearn` necesita para ajustar.

| Query | Nodos | n |
|---|---|---|
| Q1 — metro vs. resto de TP → seguridad y eficiencia | `grupo_tp` → `perc_seguridad`, `perc_eficiencia` | 1,017 |
| Q2 — patrones/autoempleados vs. profesionistas → costo | `ocupacion` → `perc_costo` | 137 |
| Q3 — escolaridad → modo principal | `escolaridad` → `modo_principal` | 1,184 |
| Q4 — asalto en TP según ingreso (usuarios de TP) | `ingreso_gpo` → `asalto_tp` | 1,014 |

Tres nodos **no existían como columna** y hubo que derivarlos:

- **`ocupacion`** vive en el *roster* del hogar (`h21_*`, una columna por integrante), no a nivel del
  informante. Se identifica su renglón cruzando `sexo` + `sd2` contra el sexo (`h10_*`) y la edad
  (`h11_*`) de cada integrante: funciona en **1,185 de 1,191 casos (99.5 %)**.
- **`modo_principal`** se deduce de la batería `p1a_*` (22 modos).
- **`grupo_tp` / `usa_tp` / `usa_metro`** definen los universos de Q1 y Q4 por **uso** del transporte
  público, no por modo principal.

Dos advertencias antes de modelar, ambas detalladas en las notas finales del notebook 03:

- **Q4 es el query frágil.** Solo 70 personas del universo de TP reportaron un asalto. Por eso el
  ingreso se exporta en dos versiones (`ingreso_gpo` de 3 niveles e `ingreso_bin` de 2): si la de
  3 niveles deja celdas demasiado delgadas, la binaria está lista.
- **`modo_principal` está dominado por `tp_concesionado`.** El 38 % de las personas declara la misma
  frecuencia máxima en varios modos, y el desempate favorece al grupo más usado. Para cualquier
  pregunta sobre el metro hay que usar `grupo_tp` (210 usuarios), no `modo_principal` (10).

## Si algo no les cuadra

Los criterios están explícitos como reglas que se pueden modificar y volver a correr: las reglas
A/B/C/D de las secciones 6 y 11 del notebook 01, y los mapas de recodificación de la sección 7 del
notebook 03. Si el equipo quiere otro umbral de faltantes, otro corte de ingreso o incluir otros
bloques en el analítico, es cambiar un diccionario y re-ejecutar.
