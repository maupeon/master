# Dataset — Vivienda en Madrid 2018

Dataset base para el análisis de precios de vivienda y el modelo predictivo del TFM (Máster en Big Data, Data Science y AI — UCM).

**Fichero:** `habitia_madrid_2018.parquet`
**Generado por:** [`data_model_habitia_v2.ipynb`](../data_model_habitia_v2.ipynb)
**Unidad de observación:** un anuncio de venta de vivienda en Madrid capital, en un trimestre de 2018.

---

## 1. Origen y alcance

| | |
|---|---|
| Cobertura geográfica | Madrid capital (municipio 28079) |
| Periodo | 2018, cuatro trimestres (`PERIOD`: 201803, 201806, 201809, 201812) |
| Unidad de observación | Anuncio de venta (`ASSETID` + `PERIOD`) |
| Fuente principal | [`idealista18`](https://github.com/paezha/idealista18) — Rey-Blanco et al., *Environment and Planning B: Urban Analytics and City Science* |

**Un mismo inmueble puede aparecer varias veces** si estuvo en venta en más de un trimestre. La llave de fila única es (`ASSETID`, `PERIOD`), no `ASSETID` solo.

### Aviso de vintage

Todo el dataset es de 2018: precios, secciones censales y alquiler. Si en fases posteriores del proyecto se compara contra datos actuales (API de idealista en vivo, seccionado censal reciente), la diferencia mezcla *concept drift* real del mercado con cambios de geografía censal entre años.

### Aviso de anonimización

Los precios y las coordenadas de `idealista18` llevan ruido aleatorio introducido por los autores del dataset original. 
Se estimó empíricamente (usando anuncios repetidos entre trimestres) un error posicional de σ ≈ 38 m por eje, p95 ≈ 94 m. Esto limita la resolución espacial fiable a ~100 m; es la razón por la que el enriquecimiento catastral posterior usa agregados de vecindad y no un *join* directo a edificio.

---

## 2. Diccionario de columnas

### 2.1 Identificación

| Columna | Tipo | Descripción |
|---|---|---|
| `ASSETID` | string | Identificador del inmueble. No único por fila — un mismo activo puede repetirse en varios `PERIOD` |
| `PERIOD` | int | Trimestre del anuncio, formato `YYYYMM` (03/06/09/12) |

### 2.2 Precio y características del inmueble

| Columna | Tipo | Descripción |
|---|---|---|
| `PRICE` | float | Precio de venta anunciado (€) |
| `UNITPRICE` | float | Precio por m² (€/m²) |
| `CONSTRUCTEDAREA` | int | Superficie construida (m²) |
| `ROOMNUMBER` | int | Número de habitaciones |
| `BATHNUMBER` | int | Número de baños |
| `FLOORCLEAN` | int | Planta del inmueble |
| `ISDUPLEX`, `ISSTUDIO`, `ISINTOPFLOOR` | 0/1 | Tipología |
| `HASTERRACE`, `HASLIFT`, `HASAIRCONDITIONING`, `HASPARKINGSPACE`, `HASBOXROOM`, `HASWARDROBE`, `HASSWIMMINGPOOL`, `HASDOORMAN`, `HASGARDEN` | 0/1 | Amenities |
| `HASNORTH/SOUTH/EAST/WESTORIENTATION` | 0/1 | Orientación (pueden ser varias a la vez) |
| `ISPARKINGSPACEINCLUDEDINPRICE`, `PARKINGSPACEPRICE` | 0/1, float | Detalle de plaza de garaje |
| `AMENITYID` | int | Código de servicio adicional del anuncio (idealista) |
| `CONSTRUCTIONYEAR` | int | Año de construcción declarado por el anunciante. **Solo el 41% de las filas lo tiene** — usar `CADCONSTRUCTIONYEAR` como alternativa más completa |

### 2.3 Catastro (idealista18, no reconstruido en esta fase)

| Columna | Tipo | Descripción |
|---|---|---|
| `CADCONSTRUCTIONYEAR` | int | Año de construcción según Catastro |
| `CADMAXBUILDINGFLOOR` | int | Número de plantas del edificio |
| `CADDWELLINGCOUNT` | int | Nº de viviendas en el edificio |
| `CADASTRALQUALITYID` | int | Índice de calidad catastral |
| `BUILTTYPEID_1/2/3` | 0/1 | Tipología constructiva (mutuamente excluyentes) |

Estos campos vienen precalculados por los autores de `idealista18` a partir de un cruce catastral con la dirección real (sin ruido de anonimización).
**Sirven como verdad de referencia** para validar cualquier pipeline propio de enriquecimiento catastral que se aplique después a datos sin este cruce (p. ej. los que llegarían de la API de idealista en 2026).

### 2.4 Distancias

| Columna | Tipo | Descripción |
|---|---|---|
| `DISTANCE_TO_CITY_CENTER` | float | Distancia a Puerta del Sol (m) |
| `DISTANCE_TO_METRO` | float | Distancia a la estación de metro más cercana (m) |
| `DISTANCE_TO_CASTELLANA` | float | Distancia al eje Castellana (m) |

### 2.5 Geografía

| Columna | Tipo | Descripción |
|---|---|---|
| `LONGITUDE`, `LATITUDE` | float | Coordenadas WGS84 (con ruido de anonimización, ver aviso arriba) |
| `geometry` | geometry | Punto shapely, mismo CRS que lon/lat |
| `codigo_censal` | string (10) | CUSEC del INE — sección censal del **seccionado de 2018**, asignada por *join* espacial (no por el `LOCATIONID` de idealista, ver nota) |
| `barrio` | string | Barrio (división administrativa del Ayuntamiento) |
| `distrito` | string | Distrito |
| `sec_dist_borde_m` | float | Distancia del punto al borde de su sección censal (m) |
| `sec_dudosa` | bool | `True` si `sec_dist_borde_m` < 2σ (≈77 m) — la asignación de sección podría corresponder a la sección contigua |
| `seccion_rescatada` | bool | `True` si el punto cayó fuera de toda sección (ruido de anonimización cerca del límite municipal) y se asignó por proximidad, no por contención |

> **Nota metodológica:** una versión anterior de este pipeline derivaba el
> código de sección a partir del `LOCATIONID` de idealista (formato
> `0-EU-ES-28-07-001-079-16-002` → `2807916002`). Se detectó que ese código
> **no es una sección censal real** — el segmento final de idealista
> identifica una *zona* interna del distrito (1–10), no una sección del
> INE (1–222). El cruce coincidía en 119 de 135 casos por azar de formato,
> pero unía barrios con datos de secciones equivocadas de forma silenciosa.
> Se sustituyó por *join* espacial contra la cartografía real del INE.

### 2.6 Alquiler (Ayuntamiento de Madrid, por sección censal, 2018)

| Columna | Tipo | Descripción |
|---|---|---|
| `n_viviendas_alquiler` | float | Nº de viviendas en alquiler registradas en la sección |
| `alq_mediana_m2`, `alq_p25_m2`, `alq_p75_m2` | float | Superficie mediana/p25/p75 de las viviendas en alquiler de la sección |
| `alq_mediana_eur_m2`, `alq_p25_eur_m2`, `alq_p75_eur_m2` | float | Precio de alquiler por m² — mediana/p25/p75 |
| `alq_mediana_eur`, `alq_p25_eur`, `alq_p75_eur` | float | Precio de alquiler mensual total — mediana/p25/p75 |
| `alq_nivel_imputacion` | string | `original`, `codigo_censal`, `distrito`, o `<NA>` si no se pudo imputar en ningún nivel (ver §3) |
| `alq_imputado` | bool | `True` si el valor no es el original de la fuente |

### 2.7 Variables derivadas (construidas en el notebook, no de fuente externa)

| Columna | Tipo | Descripción | Fórmula |
|---|---|---|---|
| `alq_estimado_eur` | float | Alquiler mensual estimado del inmueble | `alq_mediana_eur_m2 × CONSTRUCTEDAREA` |
| `rentabilidad_pct` | float | Rentabilidad bruta anual estimada (%) | `alq_estimado_eur × 12 / PRICE × 100` |

⚠️ **`rentabilidad_pct` contiene `PRICE` en el denominador.** No debe usarse como *feature* de entrada en ningún modelo cuyo target sea `PRICE` o `UNITPRICE` introduciría fuga de información (*leakage*). Es una variable de análisis exploratorio, no de modelado. 
El lado del alquiler (`alq_mediana_eur_m2` y percentiles) sí es seguro como *feature*, al ser externo al precio del propio anuncio.

#### Convención de nombres de las variables de alquiler

Todas las variables relacionadas con alquiler comparten el prefijo `alq_`, sea cual sea su origen. 
La distinción de procedencia se comunica con el **sufijo**, no con la presencia o ausencia del prefijo:

| Sufijo | Naturaleza | Ejemplo |
|---|---|---|
| `_mediana_*`, `_p25_*`, `_p75_*` | Percentil observado, tal cual publica la fuente | `alq_mediana_eur_m2` |
| `_estimado_*` | Calculado en este notebook a partir de otra variable | `alq_estimado_eur` |

Esto permite filtrar toda la familia con `df.filter(like="alq_")` sin perder la distinción entre dato de fuente y dato derivado, que queda explícita en el propio nombre de columna en vez de depender de esta documentación para saber cuál es cuál. `rentabilidad_pct` queda fuera de esta familia a propósito: no es una medida de alquiler, es un ratio entre dos dominios (alquiler y precio de venta).

Ambas variables son constantes para todos los anuncios de la misma `codigo_censal` salvo por el factor `CONSTRUCTEDAREA`; no aportan señal más allá de zona × superficie, que un modelo con dummies de sección y superficie ya captura. Su utilidad principal es el EDA de rentabilidad por zona, no como *feature* adicional.

---

## 3. Imputación del alquiler

Las columnas de la sección 2.6 tienen huecos porque no todas las secciones censales tienen dato publicado por el Ayuntamiento. Se rellenan con una imputación jerárquica: mediana de `codigo_censal` → mediana de `distrito` → sin rellenar (`NaN` deliberado, no se usa la media de toda la ciudad).

`alq_nivel_imputacion` indica qué nivel se usó en cada fila. En la práctica, el missing de esta fuente suele afectar a la sección censal completa (no a anuncios individuales dentro de ella), por lo que el nivel `distrito` es frecuentemente el que resuelve la mayoría de los huecos — comprobarlo con `datos.alq_nivel_imputacion.value_counts()` antes de asumirlo.

---

## 4. Validaciones aplicadas antes de exportar

El notebook ejecuta, antes de guardar el parquet:

- Sin filas duplicadas por (`ASSETID`, `PERIOD`)
- `PRICE` y `CONSTRUCTEDAREA` positivos
- `codigo_censal` de 10 dígitos y existente en la cartografía del INE
- `rentabilidad_pct` en rango plausible (0–15%) para >95% de las filas
- Coordenadas dentro del rango geográfico de Madrid

Un diccionario de cobertura por columna se genera junto al dataset en
`diccionario_datos.csv`.

---

## 5. Limitaciones conocidas

- **Coherencia temporal:** dataset íntegro de 2018. No mezclar con fuentes de otro año sin resolver el vintage del seccionado censal primero.
- **Ruido de anonimización:** ver aviso en §1. Limita la resolución espacial fiable a ~100 m.
- **Cobertura desigual de alquiler:** algunas secciones (y en menor medida distritos enteros) carecen de dato incluso tras la imputación jerárquica; esas filas quedan con `NaN` intencionadamente.
- **`rentabilidad_pct` no es apta como *feature* de modelado** — ver §2.7.
- **`CADCONSTRUCTIONYEAR` y demás campos catastrales** provienen del cruce original de idealista, no de un pipeline propio — no están disponibles para datos que lleguen por otras vías (p. ej. la API de idealista) sin reconstruirlos.

---

