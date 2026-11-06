# Monitor-biomedico-esp32
Monitor biomédico basado en ESP32 para la medición de presión arterial, frecuencia cardiaca, oximetría y detección de caídas.

BIOTWATCH es un proyecto de Aplicación de Internet de las Cosas enfocado en el desarrollo de un monitor biomédico basado en un ESP32. El sistema busca obtener y transmitir información relacionada con variables fisiológicas del usuario para su posterior procesamiento y visualización.

**Equipo de trabajo**
Internet of Thinkers
**Integrante	                  Rol**
Alexandra Yireh Chávez Rivas	  Hardware
Karen Guadalupe Ortiz Mendoza	  Conectividad / Datos y visualización

**Variables biomédicas**

El sistema contempla la medición y detección de las siguientes variables:
- Oximetría
- Frecuencia cardíaca
- Temperatura corporal
- Detección de caídas

  **Boceto de arquitectura:**
Fuente → Sensores: DS18B20, MAX30102, MPU6050
Información →  ESP32: lee, empaqueta (JSON) y transmite 
Procesamiento → Broker MQTT + base de datos 
Visualización → Dashboard con indicadores y alarmas

**Render**

<img width="742" height="406" alt="image" src="https://github.com/user-attachments/assets/267c6e2e-a50d-4474-9c58-2b50474831e0" />


**PARTE 2**
**Montaje y primera lectura Código del sketch de prueba usado**
#include <Wire.h>
#include <Adafruit_MLX90614.h>
#include "MAX30105.h"
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ArduinoJson.h> // Necesaria para armar el JSON

// --- Crear objetos de los sensores ---
Adafruit_MLX90614 mlx = Adafruit_MLX90614();
MAX30105 particleSensor;
Adafruit_MPU6050 mpu;

// Variables globales
float tempPiel = 0.0;
long irValue = 0;
long redValue = 0;
bool caidaDetectada = false;

// Umbrales para el MPU6050 (Detección de caídas)
const float UMBRAL_CAIDA = 2.5; // Aceleración en G (2.5g indica un impacto fuerte)

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22); // Pines I2C del ESP32
  
  Serial.println("\n--- Iniciando BIOTWATCH ---");

  // 1. Iniciar MLX90614 (Temperatura)
  if (!mlx.begin()) {
    Serial.println("Error: MLX90614 no encontrado. Revisa conexiones.");
    // No usamos while(1) para que el sistema siga intentando con los otros
  } else {
    Serial.println("MLX90614 (Temperatura) OK.");
  }

  // 2. Iniciar MAX30102 (Pulso/Oximetría)
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("Error: MAX30102 no encontrado. Revisa conexiones.");
  } else {
    Serial.println("MAX30102 (Pulso) OK.");
    // Configuración básica
    byte ledBrightness = 60;
    byte sampleAverage = 4;
    byte ledMode = 2; // Rojo e IR
    int sampleRate = 100;
    int pulseWidth = 411;
    int adcRange = 4096;
    particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange);
  }

  // 3. Iniciar MPU6050 (Acelerómetro para caídas)
  if (!mpu.begin()) {
    Serial.println("Error: MPU6050 no encontrado. Revisa conexiones.");
  } else {
    Serial.println("MPU6050 (Acelerómetro) OK.");
    mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
    mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
  }
  
  Serial.println("--- Sistema Listo ---\n");
}

void loop() {
  // --- 1. Leer Temperatura (MLX90614) ---
  // Solo leemos si el sensor respondió en el setup (podemos intentar leer siempre)
  tempPiel = mlx.readObjectTempC();
  
  // --- 2. Leer Pulso y Oximetría (MAX30102) ---
  irValue = particleSensor.getIR();
  redValue = particleSensor.getRed();

  // --- 3. Leer Acelerómetro y detectar caídas (MPU6050) ---
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  
  // Calcular la magnitud del vector de aceleración (Regla de la suma de cuadrados)
  float magnitud = sqrt((a.acceleration.x * a.acceleration.x) + 
                        (a.acceleration.y * a.acceleration.y) + 
                        (a.acceleration.z * a.acceleration.z));
  
  // Si la magnitud supera el umbral, es probable una caída
  if (magnitud > UMBRAL_CAIDA) {
    caidaDetectada = true;
  } else {
    caidaDetectada = false;
  }

  // --- 4. Mostrar en Monitor Serie ---
  Serial.print("Temp: "); Serial.print(tempPiel, 2); Serial.print(" C | ");
  Serial.print("IR: "); Serial.print(irValue); Serial.print(" | ");
  Serial.print("Red: "); Serial.print(redValue); Serial.print(" | ");
  
  if (caidaDetectada) {
    Serial.println("¡ALERTA DE CAÍDA!");
  } else {
    Serial.println("Estado: Normal");
  }

  // --- 5. Armar JSON para IoT (WiFi/MQTT) ---
  // Aquí es donde empaquetarías los datos para enviarlos a tu broker
  StaticJsonDocument<200> doc;
  doc["dispositivo"] = "BIOTWATCH-01";
  doc["temperatura"] = tempPiel;
  doc["infrarrojo"] = irValue;
  doc["rojo"] = redValue;
  doc["caida"] = caidaDetectada;
  
  // String jsonString;
  // serializeJson(doc, jsonString);
  // Serial.println("JSON: " + jsonString); // Descomenta si quieres ver el JSON impreso

  delay(1000); // Actualizar cada 1 segundo
}


**Monitor serial mostrando valores de IR/Red y temperatura y nivel de ruido observado en la señal del MAX30102**
Temp: 36.47 °C  |  IR: 1051  |  Red: 1388  -> [RUIDO]: Sin dedo detectado.
Temp: 36.45 °C  |  IR: 1046  |  Red: 1392  -> [RUIDO]: Sin dedo detectado.
Temp: 36.49 °C  |  IR: 219511  |  Red: 198694  -> [RUIDO]: Movimiento/Presión excesiva.
Temp: 36.45 °C  |  IR: 222638  |  Red: 196312  -> [RUIDO]: Movimiento/Presión excesiva.
Temp: 36.42 °C  |  IR: 211191  |  Red: 189836  -> [RUIDO]: Movimiento/Presión excesiva.
Temp: 36.36 °C  |  IR: 162947  |  Red: 176710  -> [RUIDO]: Movimiento/Presión excesiva.
Temp: 36.35 °C  |  IR: 1501  |  Red: 1728  -> [RUIDO]: Sin dedo detectado.
Temp: 36.32 °C  |  IR: 1206  |  Red: 1703  -> [RUIDO]: Sin dedo detectado.

**Fotografía del cableado físico**


**Preguntas de reflexión**
¿Por qué el MAX30102 necesita dos longitudes de onda de luz (rojo e infrarrojo) y no le bastaría con una sola para medir SpO2?
Debido a que la hemoglobina oxigenada y la desoxigenada absorben la luz de manera diferente, el rojo es absorbido mucho más por la hemoglobina sin oxígeno, mientras que el infrarrojo es absorbido por la hemoglobina con oxígeno. Al emitir ambas luces y medir cuánta luz rebota, el sensor puede calcular la proporción entre ambos tipos, con una sola, no se podría saber si la sangre está oxigenada o no sin un punto de comparación.Porque la oximetría (medir el oxígeno en la sangre) se basa en el color de la hemoglobina.

Si tuvieran que elegir entre el DS18B20 y el MLX90614 para un wearable de muñeca que se usa todo el día, ¿cuál elegirían y por qué?
El DS18B20 en una versión pequeña ya que el MLX90614 mide la radiación infrarroja a distancia, lo que hace que la temperatura ambiente y aspectos externos influyan en la medición, por el otro lado, el DS18B20 es un sensor de contacto, por lo que mide la temperatura directamente sin ser afectado por factores externos.

¿Qué otro actuador, además de un LED o un motor de vibración, podría ser útil para notificar a una persona con discapacidad visual o auditiva?
Buzzer
Electroimán para generar vibraciones
Actuador de voz

**Borrador general de diagrama de conexiones (Sensores →ESP32)**


**Justificaciones y Evidencias de ESP-32 conectado a WiFi**
¿Por qué el equipo usará WiFi 802.11 b/g/n/ac en su prototipo?
 	El WiFi 802.11 b/g/n es el estándar que mejor se adapta a un prototipo de monitor biomédico que necesita enviar datos a un dashboard en la nube, su alta velocidad y su pila TCP/IP integrada permiten que se comunique directamente con el broker MQTT, sin necesidad de hardware intermediario. Aunque el consumo es mayor, para un prototipo de laboratorio conectado a la red eléctrica o a una batería recargable, es la opción más directa y compatible con tu arquitectura de software. 


