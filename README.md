[README.md](https://github.com/user-attachments/files/32134894/README.md)
# Curva de Rendimientos y Valuación de Bonos Soberanos Argentinos

Proyecto de portfolio orientado a roles en ALM, trading de renta fija y tesorería. Construye la Estructura Temporal de Tasas de Interés (ETTI) del mercado soberano argentino mediante el modelo Nelson-Siegel, valuando 17 (2 exclusiones) instrumentos distribuidos en cuatro curvas: **hard dollar, dollar-linked, CER y tasa fija**.

---

## Objetivo

Calibrar curvas de rendimiento para el mercado soberano argentino, derivar tasas spot, forward y par, y calcular métricas de sensibilidad (duración y convexidad) descontando flujos con la curva spot ajustada — no con YTM flat.

---

## Universo de instrumentos

| Curva | Instrumentos |
| Hard dollar (ley arg.) | AL30, AL29, AL35, AL41, AE38 |
| Dollar-linked | TZV27, TZV28, TZVD8, D15E7, D31M7 |
| CER (Boncer / LECER) | TX26, TX28, TZX27, TZX28 |
| Tasa fija (LECAP / BONCAP) | S30S6, S30N6, T31Y7 |

Los bonos GD30 y GD35 (ley Nueva York) se utilizan en el cálculo de YTM pero se excluyen del ajuste Nelson-Siegel, que trabaja exclusivamente sobre la curva ley argentina.

---

## Metodología — fases del notebook

### Fase 1 — Datos
Carga y limpieza de precios históricos desde RAVA (CSV por instrumento). Se construye un DataFrame maestro con atributos estáticos de cada bono extraídos de los prospectos CNV: cupón vigente, frecuencia, amortización, valor técnico (VT) y fechas de vencimiento.

Se calculan **precio sucio** (precio de mercado) y **precio limpio** (precio sucio menos cupón corrido). El cupón corrido se calcula sobre el capital residual vigente (VT) y se convierte a ARS usando el tipo de cambio correspondiente a cada curva.

**Distinción de tipo de cambio:**
- Hard dollar → MEP (ARS/USD financiero)
- Dollar-linked → TC oficial BCRA
- CER y tasa fija → sin conversión (ARS directos)

### Fase 2 — TIR / YTM
Los flujos futuros se modelan bono por bono con:
- `amort_schedules`: calendario exacto de amortizaciones pendientes por instrumento
- `cupon_schedules`: schedule de cupones step-up para AL30, AL35 y AL41, derivado del Decreto 381/2020

La YTM se resuelve con el método de Brent (`scipy.optimize.brentq`) igualando el valor presente de los flujos al precio sucio de mercado. Para instrumentos cero cupón (LECAP, LELINK, LECER) se usa la fórmula directa de descuento.

**Nota sobre CER:** la YTM de TX26 y TX28 es una tasa nominal en "pesos de hoy" bajo supuesto de CER constante, no una tasa real sobre inflación.

### Fase 3 — Nelson-Siegel
Ajuste del modelo de cuatro parámetros (β₀, β₁, β₂, λ) por mínimos cuadrados no lineales (`scipy.optimize.curve_fit`):

$$y(\tau) = \beta_0 + \beta_1 \cdot \frac{1 - e^{-\tau/\lambda}}{\tau/\lambda} + \beta_2 \cdot \left(\frac{1 - e^{-\tau/\lambda}}{\tau/\lambda} - e^{-\tau/\lambda}\right)$$

**Decisiones metodológicas documentadas:**

| Curva | λ | Parámetros libres | Justificación |
| hard_dollar | 7.75 (fijo) | β₀, β₁, β₂ | λ libre produce betas inestables con 5 puntos en 3-15 años |
| dollar_linked | libre | β₀, β₁, β₂, λ | 5 puntos en 0.4-2.3 años, ajuste estable |
| CER | 0.50 (fijo) | β₀, β₁, β₂ | 4 puntos, rango corto |
| tasa_fija | 0.50 (fijo) | β₀, β₁ | 3 puntos < 1 año, β₂=0 |

La selección de λ para hard dollar se valida con una grilla SSE (incluida en el notebook).

**Particularidades del mercado argentino documentadas:**
- Curva hard dollar con forma de joroba: refleja prima de plazo positiva con compresión en el tramo largo
- Curvas dollar-linked y CER con forma de U (~plazo 0.8 años): consistente con segmentación de mercado y contexto electoral
- S30S6 por debajo de la curva tasa fija: efecto liquidez del tramo ultracorto (< 30 días), opera como cuasi-efectivo

### Fase 4 — Tasas derivadas
A partir de los parámetros NS ajustados se derivan analíticamente:

- **Tasa spot** — NS evaluada en cada plazo τ
- **Forward instantánea** — derivada analítica de [τ·y(τ)]: f(τ) = β₀ + β₁·e^{-τ/λ} + β₂·(τ/λ)·e^{-τ/λ}
- **Forward discreta** — tasa implícita entre dos plazos: f(τ₁,τ₂) = [(1+y(τ₂))^τ₂ / (1+y(τ₁))^τ₁]^{1/(τ₂-τ₁)} - 1
- **Tasa par** — cupón que hace que el bono cotice exactamente a la par, dado el descuento spot

El intervalo de forward discreto varía por curva (semestral para hard dollar y CER, trimestral para dollar-linked, mensual para tasa fija) acorde al horizonte de cada mercado.

### Fase 5 — Duración y Convexidad
Las métricas se calculan descontando cada flujo con **su propia tasa spot NS** (no con YTM flat), lo que produce valuaciones más precisas y permite el análisis rich/cheap:

- **Precio teórico** = sum (CF_t) / (1 + spot(t))^t
- **Duración de Macaulay** =  sum(t) · [CF_t / (1+spot(t))^t] / P
- **Duración modificada** = D_mac / (1 + ytm/m)
- **Convexidad** = sum t·(t + 1/m) · CF_t / (1+spot(t))^t / P

Se incluye una comparación explícita entre la aproximación de primer orden (solo duration) y la de segundo orden (duration + convexity) ante un shock de +100bps (1%), ilustrando el beneficio de convexidad positiva.

---

### Excel — Tablero de sensibilidad (`Sensibilidad_Bonos_Modulo2.xlsx`)
- Hoja **Métricas**: tabla resumen con YTM, duración y convexidad por instrumento
- Hojas **Sens_[tipo]**: variación de precio ante shocks de ±25 a ±300 bps para cada bono, con formato (verde = precio sube, rojo = precio baja)
- Hoja **Curvas_NS**: tasas derivadas con tablas dinamicas por curva

### PowerBI — Dashboard interactivo (`Dashboard.pbix`)
- **Página 1 — Curvas NS**: líneas spot + forward por curva, scatter de YTM observados con tamaño proporcional a duración modificada
- **Página 2 — Rich/Cheap**: scatter precio sucio vs precio teórico por instrumento, tabla con diferencias absolutas y porcentuales
- **Página 3 — Riesgo**: barras de duration modificada y convexity por instrumento, cards de promedio por curva

---

## Limitaciones

- **Curva CER** produce YTMs nominales bajo supuesto de CER constante, no comparables directamente con la curva hard dollar.
- La **curva dollar-linked** utiliza el TC oficial para la conversión de precios. Variaciones en el diferencial MEP/TC oficial afectan directamente las YTMs calculadas.
- Nelson-Siegel es un modelo de forma reducida sin arbitraje: no garantiza ausencia de arbitraje entre curvas. Recomendable el uso de Arbitrage-Free Nelson-Siegel (AFNS)
- La modificacion de Svensson que permite varias jorobas puede ser util para un analisis mas profundo

---

## Stack técnico

| Herramienta | Uso |
|---|---|
| Python | Pipeline de datos, calibración NS, métricas |
| pandas, numpy | Manipulación de datos y cálculo numérico |
| scipy.optimize | Ajuste NS , YTM |
| matplotlib | Visualizaciones de curvas y métricas |
| openpyxl | Generación del tablero Excel |
| Excel | Tablero de sensibilidad interactivo |
| PowerBI | Dashboard con slicers por curva |

---

## Fuente de datos

Precios históricos: [RAVA Bursátil](https://www.rava.com)  
Atributos estáticos (cupón, amortización, VT): prospectos CNV y Puente Net  
Tipo de cambio: cotización MEP y TC oficial BCRA a la fecha de análisis (agosto 2026)
