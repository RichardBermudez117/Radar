# Autonomous Omnidirectional Radar System (360°) with TFT Display

Este proyecto consiste en el desarrollo de un sistema de radar digital bidimensional con un rango de escaneo completo de 360° que opera de manera 100% autónoma. A diferencia de los radares convencionales que dependen de una computadora externa, este dispositivo realiza el control de movimiento, la adquisición de datos de proximidad mediante ultrasonido con filtrado estadístico y el renderizado gráfico en tiempo real directamente sobre una pantalla TFT integrada al microcontrolador.

## 🚀 Características Principales

*   **Procesamiento y Visualización Embebida:** Operación completamente independiente de una PC utilizando una pantalla TFT controlada por hardware.
*   **Escaneo Circular de 360°:** Movimiento continuo e incremental utilizando un motor paso a paso con cálculo de posición angular dinámico.
*   **Filtrado Estadístico por Mediana:** Algoritmo de ordenamiento burbuja embebido para calcular la mediana de múltiples lecturas ultrasónicas, mitigando picos de falsos positivos y ruido ambiental.
*   **Manejo de Potencia Directo (Sin Optoacopladores):** Conexión directa al controlador de potencia Darlington ULN2003, optimizando la distribución de energía compartiendo una sola referencia (Tierra común) pero aislando las líneas de alimentación de corriente.
*   **Gráficos en Tiempo Real:** Renderizado adaptativo que borra la línea de barrido anterior y repara dinámicamente la cuadrícula radial para evitar la degradación de la interfaz visual.

---

## 🛠️ Arquitectura del Sistema (Hardware)

El sistema está diseñado para optimizar el uso de pines, ya que las pantallas TFT tipo *Shield* ocupan la gran mayoría de la interfaz del microcontrolador.

*   **Microcontrolador:** Arduino Uno o compatible.
*   **Unidad de Visualización:** Pantalla TFT LCD con controlador compatible con la librería `MCUFRIEND_kbv` (montada en configuración horizontal `Rotation(1)`).
*   **Sensor de Distancia:** Sensor ultrasónico HC-SR04 (Conectado a los pines de comunicación serial `0 (RX)` y `1 (TX)` para liberar el bus de la pantalla).
*   **Actuador de Giro:** Motor paso a paso de 4 fases 28BYJ-48 con controlador ULN2003.

### Mapeo de Conexiones Físicas

| Componente | Pin del Componente | Pin Digital Arduino | Notas |
| :--- | :--- | :--- | :--- |
| **TFT Shield** | Pines estándar del Shield | Pines 2 al 9, A0 al A4 | Ocupa bus de datos y control estándar |
| **HC-SR04** | Trig | `0 (RX)` | Señal de disparo de pulso |
| | Echo | `1 (TX)` | Captura del tiempo de eco |
| | VCC / GND | 5V / GND Arduino | Alimentación de la lógica del sensor |
| **ULN2003** | IN1, IN2, IN3, IN4 | `10, 11, 12, 13` | Control de fases del motor |
| | VCC (+) | **Fuente Externa (5V)** | Separada de la línea lógica para evitar ruidos |
| | GND (-) | GND Fuente y **GND Arduino** | Conexión de tierra común |

---

## 📐 Lógica y Modelado Matemático

### 1. Control Angular del Motor
El motor posee un reductor interno que requiere 2048 pasos por revolución completa. El firmware realiza avances discretos de 16 pasos por ciclo, calculando el ángulo de la siguiente manera:

$$\theta = \frac{p \times 360.0}{2048}$$

### 2. Conversión Espacial a Coordenadas Cartesianas
Dado que las funciones de dibujo de la pantalla (`drawLine`, `fillCircle`) operan en coordenadas rectangulares $(X, Y)$, pero el radar recopila datos en coordenadas polares $(\theta, \text{distancia})$, se aplica la transformación trigonométrica desfasando el origen en $-90^\circ$ para alinear el cero con el eje vertical:

$$X = X_{\text{centro}} + \cos\left(\theta - 90^\circ\right) \times R_{\text{máx}}$$
$$Y = Y_{\text{centro}} + \sin\left(\theta - 90^\circ\right) \times R_{\text{máx}}$$

Para el mapeo del punto de obstáculo dentro del rango máximo calibrado (400 cm), se utiliza una función de escalado lineal proporcional al radio de la pantalla (110 píxeles).

### 3. Filtro de Mediana Móvil
Para evitar perturbaciones por rebotes de onda, el sistema toma un arreglo de 3 lecturas consecutivas separadas por un retardo de estabilización de 3ms. Las lecturas se ordenan de menor a mayor mediante el método de la burbuja y se selecciona el valor central:

```cpp
// Ordenamiento e identificación del valor central limpio
if (lecturas[j] > lecturas[j+1]) {
    float temp = lecturas[j];
    lecturas[j] = lecturas[j+1];
    lecturas[j+1] = temp;
}
return lecturas[n/2];
