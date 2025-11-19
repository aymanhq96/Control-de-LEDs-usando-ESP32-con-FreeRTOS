# Control de LEDs con ESP32 usando FreeRTOS e Interrupciones

Este proyecto implementa un sistema multitarea en un ESP32 para controlar dos botones y dos LEDs (rojo y verde). Se ha diseñado pensando en buenas prácticas de programación embebida, uso eficiente de FreeRTOS y respuesta rápida mediante interrupciones.

Se ha usado Wokwi para realizar la simulación, es una herramienta muy interesante para poder simular elementos básicos y microcontroladores.

A continuación se explica detalladamente el funcionamiento del sistema, la estructura del código y el propósito de cada sección.

## **Simulación del ESP32, con los leds y botones **
<img width="922" height="601" alt="image" src="https://github.com/user-attachments/assets/ae0c1e20-af9c-4d52-a21b-f3e289501482" />


##  **Objetivo del proyecto**

El propósito es demostrar la capacidad de un ESP32 para gestionar múltiples eventos simultáneos usando:

* **Interrupciones (ISR)** para detectar pulsaciones.
* **Tareas de FreeRTOS** para procesar el comportamiento de cada LED.
* **Estados y temporización** para permitir dos modos de funcionamiento:

  * **Pulsación corta:** enciende/apaga el LED.
  * **Pulsación larga (>500 ms):** activa un modo de parpadeo continuo.

Este tipo de lógica es muy utilizada en sistemas de interacción físico-digital, como paneles de control, automatización industrial o interfaces embebidas.



##  **Descripción del hardware**

El montaje incluye:

* 1 ESP32-WROOM.
* 2 pulsadores (rojo y verde).
* 2 LEDs (rojo y verde) con sus resistencias limitadoras.

### Conexiones principales:

* **Botón rojo → GPIO 12** (entrada con `INPUT_PULLUP`)
* **LED rojo → GPIO 13**
* **Botón verde → GPIO 17**
* **LED verde → GPIO 23**

Los botones están configurados en modo **pull-up interno**, por lo que su nivel es HIGH normalmente y pasan a LOW al presionar.

---

##  **Arquitectura del software**

El proyecto está dividido en tres grandes componentes:

### 1. **Interrupciones (ISR)**

Al detectar un flanco descendente (`FALLING`), cada ISR activa un **flag volatile** indicando que ese botón fue pulsado.

Esto permite capturar pulsaciones con mínima latencia y sin bloquear la CPU.

### 2. **Tareas FreeRTOS**

Se usan dos tareas independientes:

* `TaskRojo()` para el LED rojo.
* `TaskVerde()` para el LED verde.

Cada tarea:

* Procesa su propio botón.
* Gestiona temporización con `vTaskDelayUntil()`.
* Cambia estados del LED según el tipo de pulsación.

### 3. **Estados del sistema**

Se utilizan variables para controlar:

* Si el botón está presionado.
* Si el LED está encendido o apagado.
* Si está activo el modo parpadeo.
* Tiempos de medición con `millis()`.

Esto evita bloqueos y asegura un comportamiento fluido, incluso si ambos botones se usan simultáneamente.



##  **Lógica de funcionamiento de cada botón**

Cada botón sigue el mismo flujo:

1. **Pulsación detectada por interrupción** → `rojo_pulsado = true` o `verde_pulsado = true`.
2. La tarea correspondiente analiza:

   * Si el botón sigue presionado.
   * Si han pasado *más de 500 ms*.
3. Si se suelta antes de 500 ms → **toggle** del LED.
4. Si se supera el tiempo → entra en modo **parpadeo**.
5. Si se presiona de nuevo mientras parpadea → se **detiene el parpadeo**.



##  **Medición del uso de CPU con FreeRTOS**

En `loop()` se imprime cada 2 segundos:

```cpp
vTaskGetRunTimeStats(buffer);
```

Esto muestra el porcentaje de CPU utilizado por cada tarea, útil para depuración y optimización.



##  **Ventajas del diseño**

* Aísla la lógica de cada LED en tareas independientes.
* Evita bloqueos usando FreeRTOS.
* Usa interrupciones para respuestas rápidas.
* Código escalable y profesional.
* Manejo claro de estados y temporización.

Este enfoque es muy valorado en entornos industriales y de automatización, donde la responsividad y la separación de responsabilidades son fundamentales.



##  **Posibles mejoras futuras**

* Reemplazar variables compartidas por **colas de FreeRTOS** para mayor robustez.
* Añadir **debounce por software** dentro de las tareas.
* Implementar un modo configurable con un menú serial.
* Migrar la lógica a una **máquina de estados** formal.



##  **Conclusión personal**

Este proyecto me ha permitido profundizar en:

* Programación concurrente en sistemas embebidos.
* Uso correcto de interrupciones y FreeRTOS en ESP32.
* Diseño de sistemas interactivos basados en eventos.




