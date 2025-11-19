# Control de LEDs con ESP32 usando FreeRTOS e Interrupciones

Este proyecto implementa un sistema multitarea en un ESP32 para controlar dos botones y dos LEDs (rojo y verde).

Se ha usado Wokwi para realizar la simulación, es una herramienta muy interesante para poder simular elementos básicos y microcontroladores.

A continuación se explica detalladamente el funcionamiento del sistema, la estructura del código y el propósito de cada sección.

## **Simulación del ESP32**
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

## **Código del proyecto**

```cpp
#define boton1 12
#define led1 13

#define boton2 17
#define led2 23

volatile bool rojo_pulsado = false;
volatile bool verde_pulsado = false;


bool ledRojoState= false;
bool ledVerdeState= false;

bool rojopresionado = false;
bool verdepresionado = false;

bool rojo_parpadeo = false;
bool verde_parpadeo = false;

unsigned long tiempoRojo = 0;
unsigned long tiempoVerde = 0;

void IRAM_ATTR ISRBotonRojo(){
  rojo_pulsado = true;
}

void IRAM_ATTR ISRBotonVerde(){
  verde_pulsado = true;
}


TaskHandle_t HandleRojo;
TaskHandle_t HandleVerde;



void setup() {
  Serial.begin(115200);

  xTaskCreatePinnedToCore(
    TaskRojo,
    "Rojo",
    2048/4,
    NULL,
    0,
    &HandleRojo,
    0
  );

  xTaskCreatePinnedToCore(
    TaskVerde,
    "Verde",
    2048/4,
    NULL,
    0,
    &HandleVerde,
    0
  );

  
  pinMode(boton1,INPUT_PULLUP);
  pinMode(led1,OUTPUT);

  pinMode(boton2,INPUT_PULLUP);
  pinMode(led2,OUTPUT);

  attachInterrupt(digitalPinToInterrupt(boton1),ISRBotonRojo,FALLING);
  attachInterrupt(digitalPinToInterrupt(boton2),ISRBotonVerde,FALLING);

 

}

void loop() {
    printStats();
    delay(2000);
}

void TaskRojo(void *parameters){

  const TickType_t xPeriod = pdMS_TO_TICKS(10);
  TickType_t xLastWakeTime = xTaskGetTickCount();

  while(1){
    if(rojo_pulsado) {
      rojo_pulsado = false;
      rojopresionado = true;
      tiempoRojo = millis();

      
    }
    if(rojopresionado){
      if(digitalRead(boton1)==LOW){
        //medir tiempo
        if(millis() - tiempoRojo > 500){
          rojo_parpadeo  = true;
          rojopresionado = false;
        }
      }
      else{
        if(!rojo_parpadeo ){
          ledRojoState = !ledRojoState;
          digitalWrite(led1,ledRojoState);
        }
        rojopresionado = false;
      }
    }

    if(rojo_parpadeo ){

      static unsigned long UltimoParpadeo = 0;

      if(millis()- UltimoParpadeo > 200){
        UltimoParpadeo = millis();
        ledRojoState = !ledRojoState;
        digitalWrite(led1,ledRojoState);
      }

      if(rojopresionado){
        rojopresionado = false;
        rojo_parpadeo = false;
        ledRojoState = false;
        digitalWrite(led1,LOW);
      }
    }
    vTaskDelayUntil(&xLastWakeTime, xPeriod); 


  }
}

void TaskVerde(void *parameters){

  const TickType_t xPeriod = pdMS_TO_TICKS(10);
  TickType_t xLastWakeTime = xTaskGetTickCount();

  while(1){
    while(1){
    if(verde_pulsado) {
      verde_pulsado = false;
      verdepresionado = true;
      tiempoVerde = millis();

      
    }
    if(verdepresionado){
      if(digitalRead(boton2)==LOW){
        //medir tiempo
        if(millis() - tiempoVerde > 500){
          verde_parpadeo  = true;
          verdepresionado = false;
        }
      }
      else{
        if(!verde_parpadeo ){
          ledVerdeState = !ledVerdeState;
          digitalWrite(led2,ledVerdeState);
        }
        verdepresionado = false;
      }
    }

    if(verde_parpadeo ){

      static unsigned long UltimoParpadeo = 0;

      if(millis()- UltimoParpadeo > 200){
        UltimoParpadeo = millis();
        ledVerdeState = !ledVerdeState;
        digitalWrite(led2,ledVerdeState);
      }

      if(verdepresionado){
        verdepresionado = false;
        verde_parpadeo = false;
        ledVerdeState = false;
        digitalWrite(led2,LOW);
      }
    }
    vTaskDelayUntil(&xLastWakeTime, xPeriod); 
  }
}
}

void printStats() {
    char buffer[400];
    vTaskGetRunTimeStats(buffer);
    Serial.println("=== Run Time Stats ===");
    Serial.println(buffer);
}


```

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




