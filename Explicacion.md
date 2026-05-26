# Autonomous Semicircular Radar System (0°-180°) with TFT Display

Este proyecto consiste en el desarrollo de un sistema de radar digital bidimensional con un rango de escaneo semicircular de 180° que opera de manera 100% autónoma. A diferencia de los desarrollos convencionales que dependen de una computadora externa (procesamiento en Processing/Java), este dispositivo realiza el control de movimiento, la adquisición y filtrado de datos de proximidad mediante ultrasonido y el renderizado gráfico de alta velocidad en tiempo real directamente sobre una pantalla TFT integrada al microcontrolador.

## 🚀 Características Principales

* **Procesamiento y Visualización Embebida:** Operación completamente independiente de una PC utilizando una pantalla TFT controlada por hardware mediante bus paralelo.
* **Escaneo Semicircular Dinámico:** Movimiento oscilatorio de ida y vuelta (0° -> 180° -> 0°) controlado por un servomotor de alta precisión con pasos angulares parametrizables.
* **Filtrado Anti-Glitches por Doble Muestreo:** Algoritmo de verificación síncrona flash que detecta falsos positivos físicos (zonas de la señal ultrasónica con saturación cíclica en torno a los 15 cm) relanzando un pulso de comprobación inmediata para estabilizar la telemetría.
* **Eficiencia de Memoria Dinámica (SRAM):** Migración de cadenas de texto estáticas a la memoria Flash del programa mediante la macro `F()`, liberando recursos de memoria crítica en el microcontrolador.
* **Gráficos en Tiempo Real Avanzados:** Renderizado optimizado mediante el cálculo unificado de funciones trigonométricas y borrado selectivo de trazos anteriores, eliminando el parpadeo (*flicker*) y reparando la cuadrícula radial sin ralentizar el bucle principal.

---

## 🛠️ Arquitectura del Sistema (Hardware)

El sistema está diseñado bajo criterios de **seguridad de periféricos**, liberando los pines de comunicación serial de la placa (`0` y `1`) para permitir la depuración y la carga del código de forma directa y sin interferencias físicas.

* **Microcontrolador:** Arduino Uno, Mega o compatible.
* **Unidad de Visualización:** Pantalla TFT LCD Shield con controlador compatible con la librería `MCUFRIEND_kbv` (configurada en modo horizontal `setRotation(1)`).
* **Sensor de Distancia:** Sensor ultrasónico HC-SR04 (Conectado a pines de propósito general fuera del bus del Shield).
* **Actuador de Giro:** Servomotor estándar (p. ej., SG90 o MG90S).

### Mapeo de Conexiones Físicas

| Componente | Pin del Componente | Pin Digital Arduino | Notas |
| :--- | :--- | :--- | :--- |
| **TFT Shield** | Pines estándar del Shield | Pines 2 al 9, A0 al A4 | Ocupa bus de datos y control estándar |
| **HC-SR04** | Trig | `12` | Señal de disparo de pulso de ultrasonido |
| | Echo | `11` | Captura del tiempo de retorno del eco |
| | VCC / GND | 5V / GND Arduino | Alimentación de la lógica del sensor |
| **Servomotor** | PWM (Línea de datos) | `10` | Señal de control de posición servoasistida |
| | VCC (+) / GND (-) | 5V / GND Arduino | Alimentación del motor (*Recomendado fuente externa si el Shield TFT demanda alta corriente*) |

---

## 📐 Lógica y Modelado Matemático

### 1. Control Angular y Cinemática del Barrido
El servomotor realiza avances discretos definidos por la constante `PASO_ANGULO`. La inversión del sentido del barrido se calcula de manera algorítmica mediante dos ciclos independientes de control en el bucle principal (`loop`):

* **Barrido Ida:** $\theta_{k+1} = \theta_k + \Delta\theta \quad \text{para } \theta \in [0^\circ, 180^\circ]$
* **Barrido Vuelta:** $\theta_{k+1} = \theta_k - \Delta\theta \quad \text{para } \theta \in [180^\circ, 0^\circ]$

### 2. Conversión Espacial a Coordenadas Cartesianas
Dado que las funciones de la pantalla (`tft.drawLine`, `tft.fillCircle`) operan sobre una matriz de píxeles en coordenadas rectangulares $(X, Y)$, pero el radar adquiere la información en formato polar $(\theta, \text{distancia})$, se aplica una transformación trigonométrica. 

Para alinear visualmente el ángulo de $0^\circ$ a la izquierda de la pantalla, $90^\circ$ en la vertical y $180^\circ$ a la derecha, se invierte el sentido angular de la pantalla restando el ángulo del sistema a la referencia horizontal ($180^\circ - \theta$):

$$X = X_{\text{centro}} + \cos\left(\frac{(180^\circ - \theta) \cdot \pi}{180^\circ}\right) \times R_{\text{máx}}$$

$$Y = Y_{\text{centro}} - \sin\left(\frac{(180^\circ - \theta) \cdot \pi}{180^\circ}\right) \times R_{\text{máx}}$$

El punto físico del obstáculo dentro de la matriz gráfica se determina escalando linealmente la distancia medida mediante la función `map()`, proyectando el límite físico de 400 cm de manera proporcional sobre el radio máximo de la pantalla ($R_{\text{máx}} = 130$ píxeles).

### 3. Optimización Computacional del Tiempo de Ejecución
Para mitigar la carga de procesamiento flotante en arquitecturas AVR de 8 bits (las cuales carecen de unidad de punto flotante por hardware - FPU), el firmware realiza una aproximación por **Caché Local Trigonométrica**. Las funciones complejas $\cos(\text{rad})$ y $\sin(\text{rad})$ se computan una única vez por paso angular, almacenándose en las variables locales `cosRad` y `sinRad`. Esto reduce las operaciones costosas de 3 llamadas por ciclo a solo 1, optimizando los ciclos de reloj asignados al refresco de pantalla.

---

## 💻 Justificación del Diseño del Código (Buenas Prácticas)

El firmware fue refactorizado siguiendo los más altos estándares de desarrollo embebido:

1. **Uso de Tipos de Datos Tipificados (`stdint.h`):** Se sustituyó el tipo de dato primitivo `int` (que consume 2 bytes en arquitecturas AVR) por `uint8_t` (1 byte) para variables de rango limitado ($0-255$) como pines, contadores y ángulos, y por `uint16_t` para coordenadas espaciales. Esto previene el desbordamiento (*overflow*) y optimiza la memoria estática (SRAM).
2. **Definiciones Inmutables con `constexpr`:** Se eliminaron las directivas de preprocesador `#define` en favor de `constexpr`. A diferencia de las macros ordinarias, `constexpr` obliga al compilador a aplicar verificación de tipos fuertemente tipados, previniendo errores de casteo colaterales durante la compilación.
3. **Erradicación de Números Mágicos:** Coordenadas de borrado de telemetría, frecuencias de refresco y dimensiones físicas de los círculos concéntricos se parametrizaron bajo constantes semánticas globales, garantizando la mantenibilidad y portabilidad del código a otras pantallas.
4. **Persistencia Visual Selectiva:** En lugar de limpiar toda la pantalla con `fillScreen()` en cada paso —lo que causaría un parpadeo insoportable— el firmware dibuja la línea de barrido actual en el color activo y, en el siguiente paso angular, borra la línea inmediatamente anterior redibujándola en color negro utilizando las coordenadas guardadas de retorno (`oldLx`, `oldLy`). La cuadrícula de referencia se refuerza de manera asíncrona mediante un contador divisor de frecuencia (`FREC_REFRESCO = 4`), logrando un equilibrio óptimo entre fluidez de animación y persistencia de datos.
