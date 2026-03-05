# Práctica 2: Interrupciones en ESP32-S3

Este repositorio contiene la resolución de la Práctica 2 de la asignatura, enfocada en el uso y manejo de **Interrupciones** (Hardware y Timer) en un microcontrolador ESP32-S3. El proyecto está desarrollado utilizando **Visual Studio Code** y la extensión **PlatformIO** bajo el framework de Arduino.

## 🛠️ Hardware y Software Utilizado

* **Placa:** ESP32-S3 DevKitC-1
* **Componentes:** 1x Pulsador (Push button), Protoboard, Cables puente (Jumpers).
* **Entorno de Desarrollo:** Visual Studio Code + PlatformIO
* **Lenguaje:** C++ (Arduino Framework)

## ⚙️ Configuración del Entorno (`platformio.ini`)

Para garantizar una correcta compilación y evitar problemas con la visualización de caracteres en el Monitor Serie, el archivo `platformio.ini` está configurado de la siguiente manera:

```ini
[env:esp32-s3-devkitc-1]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200

Descripción

El objetivo es monitorizar las pulsaciones de un botón conectado al pin GPIO 18 utilizando interrupciones de hardware en lugar de polling.

Modificaciones y Correcciones respecto al código original:

El código proporcionado inicialmente en las instrucciones presentaba varios problemas de lógica y compatibilidad que fueron solucionados:
Inclusión de la librería principal: Al usar PlatformIO (a diferencia del IDE de Arduino clásico), es obligatorio añadir #include <Arduino.h> al principio del archivo.
Falta del calificador volatile (Error Crítico): En el código original, las variables del struct Button no tenían la palabra clave volatile.
Formato en Monitor Serie: El código original imprimía Serial.printf("Inicio de procesador");
Bucle infinito de desconexión: El código original evaluaba if (millis() - lastMillis > 60000) para desconectar la interrupción, pero como reiniciaba lastMillis, la condición se volvía a cumplir cada 60 segundos, imprimiendo "Interrupt Detached!" en un bucle infinito.
Codigo correcto:
#include <Arduino.h>

struct Button {
  const uint8_t PIN;
  volatile uint32_t numberKeyPresses; // Añadido volatile
  volatile bool pressed;              // Añadido volatile
};

Button button1 = {18, 0, false};
bool isDetached = false; // Bandera añadida

void IRAM_ATTR isr() {
  button1.numberKeyPresses += 1;
  button1.pressed = true;
}

void setup() {
  Serial.begin(115200);
  pinMode(button1.PIN, INPUT_PULLUP);
  attachInterrupt(button1.PIN, isr, FALLING);
  Serial.printf("Inicio de procesador\n"); // Añadido \n
}

void loop() {
  if (button1.pressed) {
    Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses);
    button1.pressed = false; 
  }

  static uint32_t lastMillis = 0;
  
  // Lógica corregida para ejecutar solo una vez
  if (!isDetached && (millis() - lastMillis > 60000)) {
    detachInterrupt(button1.PIN);
    Serial.println("Interrupt Detached!");
    isDetached = true; 
  }
}
