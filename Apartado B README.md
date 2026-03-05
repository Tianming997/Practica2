# Práctica 2 - Apartado B: Interrupciones por Timer (ESP32)

Este apartado de la Práctica 2 demuestra el uso de temporizadores de hardware (Timers) en el microcontrolador ESP32-S3 usando el entorno de PlatformIO.

## 📝 Descripción del Proyecto

El objetivo es generar una interrupción periódica exacta cada 1 segundo sin detener la ejecución del procesador con funciones bloqueantes como `delay()`. Para ello, se configura una alarma vinculada a un temporizador interno que incrementa un contador de forma segura.

## 🛠️ Correcciones realizadas al código original

El código base del documento PDF contenía diversos errores de sintaxis y tipográficos que impedían la compilación en PlatformIO. Se han aplicado los siguientes cambios:

1. **Inclusión de librería:** Se añadió `#include <Arduino.h>`, obligatorio en PlatformIO.
2. **Corrección de nombres de variables:** Se eliminaron los espacios erróneos en las variables (Ej: `interrupt Counter` corregido a `interruptCounter`).
3. **Corrección de operadores de asignación faltantes:** * `hw_timer_t* timer NULL;`  -> `hw_timer_t* timer = NULL;`
   * `portMUX_TYPE timerMux portMUX_INITIALIZER_UNLOCKED;` -> `portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;`
   * `timer timerBegin(0, 80, true);` -> `timer = timerBegin(0, 80, true);`
4. **Zonas Críticas (Mutex):** Se corrigió la sintaxis de las llamadas a `portENTER_CRITICAL_ISR` y `portENTER_CRITICAL` para manejar correctamente el acceso concurrente a la variable `interruptCounter`.

---

## 💻 Código Fuente Corregido (`main.cpp`)

```cpp
#include <Arduino.h>

// Variables volátiles porque se modifican dentro de una interrupción
volatile int interruptCounter = 0;
int totalInterruptCounter = 0;

// Puntero al temporizador de hardware
hw_timer_t* timer = NULL;

// Mutex para proteger la variable compartida (Zonas Críticas)
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;

/* * RUTINA DE SERVICIO DE INTERRUPCIÓN DEL TIMER (ISR)
 * Se ejecuta cada vez que el temporizador alcanza el valor de la alarma.
 */
void IRAM_ATTR onTimer() {
  // Bloqueamos la variable para que no se modifique desde otro lado simultáneamente
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}

void setup() {
  Serial.begin(115200);
  
  /* Configuración del Timer:
   * 0: Usamos el temporizador 0 (el ESP32 tiene 4).
   * 80: Preescalador. Como el ESP32 va a 80MHz, un preescalador de 80 
   * hace que el timer se incremente cada 1 microsegundo.
   * true: Cuenta hacia arriba.
   */
  timer = timerBegin(0, 80, true);
  
  // Vinculamos la interrupción a nuestra función 'onTimer'
  timerAttachInterrupt(timer, &onTimer, true);
  
  /* Configuración de la Alarma:
   * 1000000: Valor de la alarma en microsegundos (1 segundo).
   * true: Autorrecarga (repite la alarma cíclicamente).
   */
  timerAlarmWrite(timer, 1000000, true);
  
  // Activamos la alarma
  timerAlarmEnable(timer);
  
  Serial.println("\n--- Procesador Iniciado: Práctica B (Timer) ---");
}

void loop() {
  // Si la interrupción ha sumado al contador, procesamos el evento
  if (interruptCounter > 0) {
    
    // Entramos en zona crítica para restar el contador de forma segura
    portENTER_CRITICAL(&timerMux);
    interruptCounter--;
    portEXIT_CRITICAL(&timerMux);
    
    // Incrementamos el total y lo imprimimos
    totalInterruptCounter++;
    Serial.print("An interrupt has occurred. Total number: ");
    Serial.println(totalInterruptCounter);
  }
}
