# Análisis de Airbnb Madrid — README

> **Resumen ejecutivo**  
> Este repositorio contiene un flujo completo (Quarto + R) para **segmentar barrios por tamaño de vivienda (m²)** y **predecir/imputar metros cuadrados** cuando faltan, a partir de listados de Airbnb. Integra:
> - **Limpieza y *feature engineering***
> - **Pruebas estadísticas** (normalidad, homogeneidad de varianzas, comparación de grupos)
> - **Clusterización jerárquica** de barrios con evaluación de **Silhouette**
> - **Modelo de regresión** para imputación/predicción de m²
>
> ¿Para qué? Para **pricing** y **benchmarking €/m²**

---

## Tabla de contenidos

- [Objetivo de negocio](#objetivo-de-negocio)
- [Lógica de negocio (clave)](#lógica-de-negocio-clave)
- [Datos](#datos)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Requisitos](#requisitos)
- [Instalación rápida](#instalación-rápida)
- [Ejecución](#ejecución)
- [Configuración](#configuración)
- [Flujo del notebook](#flujo-del-notebook)
  - [1) Ingesta, selección y filtrado](#1-ingesta-selección-y-filtrado)
  - [2) Feature engineering y calidad](#2-feature-engineering-y-calidad)
  - [3) Diagnóstico estadístico](#3-diagnóstico-estadístico)
  - [4) Comparación entre barrios](#4-comparación-entre-barrios)
  - [5) Clusterización de barrios](#5-clusterización-de-barrios)
  - [6) Regresión: imputación y predicción de m²](#6-regresión-imputación-y-predicción-de-m²)
  - [7) Evaluación del modelo](#7-evaluación-del-modelo)
  - [8) Imputación de valores faltantes](#8-imputación-de-valores-faltantes)
- [Resultados esperados / métricas](#resultados-esperados--métricas)
- [Buenas prácticas y mejoras](#buenas-prácticas-y-mejoras)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [FAQ](#faq)
- [Licencia](#licencia)

---

## Objetivo de negocio

1. **Entender** la variación de m² por barrio (caracterización del mercado local).
2. **Predecir/imputar m²** en anuncios incompletos para habilitar métricas robustas (**€/m²**, comparables).
3. **Agrupar barrios** en clusters operativos para reglas de precio, marketing local y expansión.

**Preguntas clave:**
- ¿Qué barrios tienen viviendas sistemáticamente más grandes/pequeñas?
- ¿Cuáles son comparables entre sí para construir *benchmarks*?
- ¿Podemos estimar m² faltantes con precisión útil para pricing y reporting?

---

## Lógica de negocio (clave)

- **Producto homogéneo**: nos centramos en *Entire home/apt* dentro de **Madrid** para evitar mezclar tipologías.
- **Métrica accionable**: m² es la base para **€/m²**, estándar en pricing/valuación.
- **Solidez inferencial**: tests acordes a la realidad de los datos (no normalidad, varianzas distintas).
- **Segmentación útil**: clusters de barrios ≈ “normas locales de tamaño” para reglas por zona.
- **Imputación informada**: completar m² preserva **consistencia** entre variables y mejora KPIs.

---

## Datos

- **Archivo**: `airbnb-listings.csv` (no se incluye por licencia/tamaño).
- **Ubicación recomendada**: `data/airbnb-listings.csv`.
- **Variables clave**:  
  `City`, `Room.Type`, `Neighbourhood`, `Accommodates`, `Bathrooms`, `Bedrooms`, `Beds`, `Price`, `Square.Feet`, `Guests.Included`, `Extra.People`, `Review.Scores.Rating`, `Latitude`, `Longitude`.
- **Ámbito analizado**: `City == "Madrid"`, `Room.Type == "Entire home/apt"`, `Neighbourhood != ""`.

> **Nota**: el notebook detecta automáticamente el separador del CSV (`;` o `,`).

---

## Requisitos

- **R** ≥ 4.2  
- **Quarto** ≥ 1.3 (para renderizar `.qmd`)
- Paquetes R:
  - `dplyr`, `ggplot2`, `forcats`, `scales`, `cluster`, `GGally`, `tidyr`

---


## Instalación rápida

```r
install.packages(c(
  "dplyr","ggplot2","forcats","scales","cluster","GGally","tidyr"
))
```

## 🔎 Flujo del notebook (Airbnb Madrid)

El notebook sigue una secuencia lógica de análisis y modelado orientada a resolver un problema de negocio:  
**entender la variación de m² por barrio, segmentar en clusters y predecir/imputar valores faltantes**.

---

### 1) Ingesta, selección y filtrado
- Carga del CSV (`airbnb-listings.csv`), detección automática del separador (`;` o `,`).
- Selección de variables clave: ciudad, tipo de habitación, barrio, capacidad, baños, dormitorios, camas, precio, superficie, etc.
- Filtrado a **Madrid** y tipo **Entire home/apt**.
- Eliminación de registros sin `Neighbourhood`.

**→ Objetivo:** trabajar sobre un **mercado homogéneo** y comparable.

---

### 2) Feature engineering y control de calidad
- Conversión de `Square.Feet` → `Square.Meters`.
- Reglas aplicadas:
  - Valores 0 → **NA**.
  - Valores \< 20 m² → **NA** (microviviendas poco interpretables).
- Eliminación de barrios donde todos los m² son NA.
- Histogramas y resúmenes para validar distribución y detectar outliers.

**→ Objetivo:** disponer de una **variable de tamaño consistente** y confiable.

---

### 3) Diagnóstico estadístico
- **Test de Shapiro–Wilk**: normalidad de m² por barrio (con ajuste FDR).
  
![Test Shapiro–Wilk](imagenes/1.png)
  
- **Test de Bartlett**: igualdad de varianzas entre barrios.
- QQ-plots y gráficos de varianza para confirmar supuestos.

**→ Hallazgos:** no normalidad y varianzas distintas → mejor aplicar **Kruskal–Wallis** en lugar de ANOVA clásico.

---

### 4) Comparación entre barrios
- **ANOVA** (exploratorio) → diferencias globales.
- **Kruskal–Wallis** (robusto) → confirma diferencias significativas.
- **Post-hoc**:
  - **Tukey HSD** tras ANOVA.
  - Recomendación: **Dunn** si se sigue coherentemente Kruskal–Wallis.

**→ Objetivo:** identificar barrios significativamente diferentes en tamaño medio de vivienda.

---

### 5) Clusterización de barrios
- Construcción de matriz de distancias a partir de p-valores (`d = 1 - p`).
- **Clustering jerárquico (hclust)** con enlace *complete*.
- Dendrograma de agrupaciones.
- Validación con **Silhouette** para evaluar estabilidad de los clusters.

**→ Objetivo:** agrupar barrios “similares” para reglas de negocio (pricing, marketing, operaciones).

---

### 6) Regresión para imputación/predicción de m²
- Variable objetivo: `Square.Meters`.
- Predictores: `Cluster`, `Accommodates`, `Bathrooms`, `Bedrooms`, `Beds`.
- División train/test (70/30).
- Ajuste de modelo con `lm()`.

**→ Objetivo:** predecir m² de nuevos anuncios y **imputar valores faltantes** de forma consistente.

---

### 7) Evaluación del modelo
- Métricas:
  - **MAE** (Error Medio Absoluto).
  - **RMSE** (Error Cuadrático Medio).
  - **R²** (bondad de ajuste).
- Análisis de predicciones vs valores reales.
- Discusión de posibles mejoras (GLM, regresión robusta).

**→ Resultado:** el modelo explica alrededor del 70% de la variabilidad en m².

---

### 8) Imputación de valores faltantes
- Identificación de NA en `Square.Meters`.
- Reemplazo por predicciones del modelo entrenado.
- Registro de qué filas fueron imputadas para trazabilidad.

**→ Objetivo:** asegurar **cobertura completa** del dataset y robustez de métricas como €/m².

---


---

