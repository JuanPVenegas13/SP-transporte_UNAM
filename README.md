# transporte_UNAM — Limpieza del dataset ENMT

Encuesta Nacional de Movilidad y Transporte. Este repo contiene la **etapa de limpieza**
del proyecto (a cargo de Juan Pablo Venegas); el análisis y la visualización van aparte.

## Cómo correrlo

```bash
uv sync                              # instala dependencias en .venv
uv run jupyter lab scripts/          # abre el notebook
```

O sin abrir Jupyter:

```bash
uv run jupyter nbconvert --to notebook --execute --inplace scripts/01_limpieza_enmt.ipynb
```

## Estructura

```
transporte_UNAM/
├── data/
│   ├── enmt_unam.csv           ← microdatos originales (versionado, no tocar)
│   ├── diccionario_enmt.xls    ← libro de códigos SPSS (versionado)
│   └── processed/              ← generado por el notebook (NO versionado)
│       ├── enmt_limpio.csv
│       ├── enmt_analitico.csv
│       ├── diccionario.json
│       └── reporte_calidad.csv
├── scripts/
│   └── 01_limpieza_enmt.ipynb  ← todo el proceso de limpieza
├── pyproject.toml
└── uv.lock
```

`data/processed/` está en `.gitignore` a propósito: esos archivos se **reproducen** corriendo
el notebook. Lo que se versiona es el código y los datos originales.

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

## Si algo no les cuadra

Los criterios están explícitos en las secciones 6 y 11 del notebook, como reglas que se pueden
modificar y volver a correr. Si el equipo quiere otro umbral de faltantes o incluir otros bloques
en el analítico, es cambiar una constante y re-ejecutar.
