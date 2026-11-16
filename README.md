# Noise2CleanVision

> **Procesador interactivo de imágenes con filtros espaciales y frecuenciales**  
> Implementado desde cero en Python puro con una interfaz gráfica Tkinter.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)
![GUI](https://img.shields.io/badge/GUI-Tkinter-orange)
![NumPy](https://img.shields.io/badge/NumPy-FFT%20%26%20Arrays-lightblue)

---

## Descripción

**Noise2CleanVision** es una aplicación de escritorio para el procesamiento digital de imágenes que combina dos dominios de trabajo:

- **Dominio Espacial** — Filtros por convolución (máscaras personalizables), mediana y moda sobre cada canal RGB por separado.
- **Dominio Frecuencial** — Transformada de Fourier 2D (FFT2), visualización del espectro y filtrado pasa-bajos / pasa-altos con máscara ideal circular.

El proyecto está construido sin frameworks de procesamiento de alto nivel: las estructuras de datos son listas de `bytearray` y las operaciones del dominio espacial son implementadas manualmente con bucles explícitos, exponiendo la lógica subyacente de forma pedagógica.

---

## Algoritmos implementados

### Filtrado espacial por convolución

Dada una imagen $f(x,y)$ y una máscara $h(s,t)$ de tamaño $m \times n$, la salida filtrada es:

$$g(x, y) = \frac{\sum_{s=-a}^{a} \sum_{t=-b}^{b} h(s,t) \cdot f(x+s,\; y+t)}{\sum_{s,t} h(s,t)}$$

donde $a = \lfloor m/2 \rfloor$, $b = \lfloor n/2 \rfloor$.  
Cuando $\sum h(s,t) = 0$ (filtros de detección de bordes), se aplica valor absoluto al acumulador.

### Filtro de Mediana

Ordena los $m \times n$ vecinos del píxel central mediante Bubble Sort y selecciona el elemento central de la lista ordenada. Efectivo contra ruido sal-y-pimienta impulsivo sin suavizar bordes.

### Filtro de Moda

Selecciona el valor con mayor frecuencia de aparición dentro de la ventana. En caso de empate, se elige el más cercano al valor del píxel central original.

### Ruido Sal-y-Pimienta

Para densidad $\rho \leq 0.5$, se muestrea sin reemplazo:

$$n_{afectados} = \text{round}(H \cdot W \cdot \rho)$$

La mitad de los píxeles seleccionados se ponen a $0$ (pimienta) y la otra mitad a $255$ (sal), manteniendo la correlación espacial entre canales R, G, B.

### Máscara Gaussiana (generada dinámicamente)

$$h(x, y) = \exp\!\left(-\frac{x^2 + y^2}{2\sigma^2}\right), \quad \sigma = \frac{N}{4}$$

La máscara se normaliza dividiendo cada peso por la suma total, garantizando conservación de energía promedio.

### Máscara Laplaciana / Énfasis de Alta Frecuencia

Máscara Laplaciana fija $3\times3$:

$$L = \begin{bmatrix} -1 & -1 & -1 \\ -1 & 8 & -1 \\ -1 & -1 & -1 \end{bmatrix}$$

Para el modo **énfasis** (*High-Frequency Emphasis*):

$$g(x,y) = f(x,y) + \alpha \cdot \frac{L * f(x,y)}{8}$$

donde $\alpha \in [0, 5]$ controla la intensidad del realce.

### FFT2 y Espectro

Se aplica la Transformada Discreta de Fourier 2D con centrado:

$$F(u,v) = \mathcal{F}\{f(x,y)\} = \sum_{x=0}^{M-1}\sum_{y=0}^{N-1} f(x,y)\, e^{-j2\pi\!\left(\frac{ux}{M}+\frac{vy}{N}\right)}$$

El espectro de magnitud se visualiza en escala logarítmica:

$$S(u,v) = \log(1 + |F(u,v)|)$$

normalizado al rango $[0, 255]$.

### Filtrado Frecuencial Ideal

Se genera una máscara binaria $H(u,v)$ basada en la distancia al centro del espectro:

$$D(u,v) = \sqrt{\left(u - \frac{M}{2}\right)^2 + \left(v - \frac{N}{2}\right)^2}$$

**Pasa-bajos ideal:** $H_{LP}(u,v) = \begin{cases}1 & D(u,v) \leq D_0 \\ 0 & \text{en otro caso}\end{cases}$

**Pasa-altos ideal:** $H_{HP}(u,v) = 1 - H_{LP}(u,v)$

La imagen filtrada se recupera mediante la IFFT2:

$$g(x,y) = \mathcal{F}^{-1}\{F(u,v) \cdot H(u,v)\}$$

---

## Arquitectura del proyecto

```
Noise2CleanVision/
├── interfaz.py        # Módulo de presentación — GUI Tkinter (1 500+ líneas)
│                      #   · Pestaña Espacial: carga de imagen, ruido, filtrado y exportación
│                      #   · Pestaña Frecuencial: FFT2, espectro, filtro ideal e IFFT
│                      #   · Canvas de resultado con zoom (×0.25 – ×15) y paneo con drag
├── reduccion.py       # Módulo de dominio — Núcleo de procesamiento (351 líneas)
│                      #   · Primitivas: formato_uint8, crear_matriz, agregar_padding
│                      #   · Filtros espaciales: procesar_canal (Convolución / Mediana / Moda)
│                      #   · Ruido: aplicar_ruido_sal_pimienta
│                      #   · Frecuencial: calcular_fft2_imagen, generar_mascara_filtro,
│                      #                 filtrar_frecuencia_rgb, generar_espectro_visual
│                      #   · Máscaras fijas: MASCARA_GAUSSIANA_3X3, MASCARA_QUEMADA_3X3
├── LICENSE            # MIT License
├── .gitignore         # Reglas completas de exclusión
└── README.md          # Este archivo
```

---

## Funcionalidades de la GUI

### Pestaña — Dominio Espacial

| Control | Descripción |
|---|---|
| Cargar imagen | Abre PNG/JPG/BMP; extrae canales R, G, B como listas de `bytearray` |
| Selector de ruido | Deslizador de densidad $\rho \in [0.0, 1.0]$ para ruido sal-y-pimienta |
| Tipo de filtro | Convolución, Mediana o Moda |
| Tamaño de máscara | Entero impar $\geq 3$; valida en tiempo real |
| Saltos f / c | Stride vertical y horizontal (subsampling opcional) |
| Padding | Activa/desactiva relleno con ceros para conservar dimensiones |
| Cuadrícula editable | Entrada manual de pesos de la máscara personalizada |
| Máscaras preestablecidas | Gaussiana 3×3 calculada / Laplaciana 3×3 |
| Ver máscara activa | Popup con la matriz de pesos formateada |
| Canvas resultado | Zoom con rueda del ratón; paneo con arrastrar; doble clic para reset |
| Exportar resultado | Guarda la imagen procesada en PNG/JPG de resolución nativa |

### Pestaña — Dominio Frecuencial

| Control | Descripción |
|---|---|
| Cargar imagen | Reutiliza la imagen del dominio espacial o carga una independiente |
| Calcular FFT2 | Computa $F_R, F_G, F_B, F_{gray}$; muestra espectro |
| Modo espectro | Magnitud (log), Fase, Parte Real, Parte Imaginaria |
| Inspector de frecuencias | Clic sobre el espectro → muestra $(u,v)$, radio $D$, magnitud, fase |
| Tipo de filtro | Pasa-bajos ideal / Pasa-altos ideal |
| Corte $D_0$ | Deslizador + entrada numérica sincronizados |
| Aplicar filtro | Genera imagen filtrada + máscara $H(u,v)$ visualizada |
| Máscara quemada | Aplica realce Laplaciano con factor $\alpha$ ajustable |
| Modo Laplaciano | Muestra solo bordes (valor absoluto de la convolución Laplaciana) |
| Exportar resultado | Guarda la imagen recuperada (IFFT) en resolución nativa |

---

## Tabla comparativa de filtros

| Filtro | Dominio | Elimina ruido gaussiano | Elimina ruido S&P | Preserva bordes | Complejidad |
|---|---|:---:|:---:|:---:|---|
| Convolución Gaussiana | Espacial | ✅ Alto | ⚠️ Parcial | ❌ Los suaviza | $O(M \cdot N \cdot k^2)$ |
| Mediana | Espacial | ⚠️ Parcial | ✅ Alto | ✅ Sí | $O(M \cdot N \cdot k^2 \log k)$ |
| Moda | Espacial | ❌ Bajo | ✅ Alto | ✅ Sí | $O(M \cdot N \cdot k^4)$ |
| Pasa-bajos FFT ideal | Frecuencial | ✅ Alto | ⚠️ Parcial | ❌ Ringing | $O(M \cdot N \cdot \log(MN))$ |
| Pasa-altos FFT ideal | Frecuencial | ❌ No aplica | ❌ No aplica | ✅ Realza | $O(M \cdot N \cdot \log(MN))$ |
| Énfasis alta frecuencia | Frecuencial | ⚠️ Parcial | ❌ No aplica | ✅ Muy alto | $O(M \cdot N)$ |

> **Nota:** $k$ = tamaño de la máscara, $M \times N$ = dimensiones de la imagen.

---

## Instalación y ejecución

### Requisitos

- Python 3.10 o superior
- pip

### Instalación de dependencias

```bash
pip install pillow numpy
```

### Ejecutar la aplicación

```bash
python interfaz.py
```

La ventana principal se abrirá con las dos pestañas disponibles: **Espacial** y **Frecuencial**.

---

## Licencia

Distribuido bajo la licencia MIT. Ver [`LICENSE`](LICENSE) para más detalles.

Copyright (c) 2024-2026 Angel Ayuquina
