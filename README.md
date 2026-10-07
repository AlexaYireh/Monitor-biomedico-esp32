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

