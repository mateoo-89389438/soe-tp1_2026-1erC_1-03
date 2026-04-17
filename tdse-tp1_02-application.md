# TP1 – Actividad 02 – 5to Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 02:

## ¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea en una aplicación?
FreeRTOS usa un mecanismo llamado `time slicing` basado en un "tick" del sistema (temporizador activo cada 1ms).
    - Si las tareas tienen la misma prioridad el procesador reparte el tiempo equitativamente entre ellas.
    - Si las tareas tienen distinta prioridad el tiempo se asigna siempre a la tarea de mayor prioridad hasta que esta se bloquee o termine.


## ¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?
Lo hace a través del `Scheduler`. La regla de decisión es: "siempre se ejecuta la tarea de mayor prioridad que esté en estado READY".
Si hay varias tareas con la misma prioridad máxima, el scheduler va alternando entre ellas en cada tick del sistema (`Round Robin`).


## ¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema?
Determina quién tiene el "poder" sobre el CPU:
*Prioridades iguales*: las tareas conviven amigablemente compartiendo el procesador.
*Prioridades distintas*: La tarea de mayor prioridad puede quedarse con todo el procesador. Si esta tarea nunca se bloquea, las tareas de menor prioridad nunca llegan a ejecutarse. generandose una `Starvation` (inanición).


## ¿Cuáles son los estados en los que puede encontrarse una Tarea ?
Hay cuatro estados:
1.  *Running:* La tarea está usando el CPU en este momento.
2.  *Ready:* La tarea quiere ejecutarse pero el Scheduler eligió otra (de igual o mayor prioridad).
3.  *Blocked:* La tarea está esperando algo (que pase un tiempo con `vTaskDelay` o que llegue un evento). No consume CPU.
4.  *Suspended:* La tarea está "dormida" explícitamente y no se va a despertar hasta que otra tarea la reanude.


## ¿Cómo implementar Tareas ?
Se implementan como cualquier funcion común de C, pero con dos reglas obligatorias:
1.  Firma específica: tiene que recibir como parámetro un puntero a `void` y el return tiene que ser de tipo `void`.
2.  Bucle infinito: adentro de la función se debe tener un `for(;;)` o `while(1)`. Una tarea nunca debe retornar (no debe llegar al final de la función ni usar el `return`); si la tarea debe terminar, tiene que ser eliminada explícitamente.


## ¿Cómo crear una o más instancias de una Tarea ?
Para hacer esto se utiliza la función `xTaskCreate()`.
    - Para crear *una instancia*: se llama a la función una vez pasando el nombre de la función de la tarea.
    - Para crear *múltiples instancias*: se lamas a la función varias veces usando la misma función de tarea. Cada llamada va a tener su propio Handle y su propia memoria (Stack), dentro del mismo código.


## ¿Cómo eliminar una Tarea ?
Para hacer esto se utiliza la función `vTaskDelete(handle)`.
    - Para eliminar *otra tarea*: se le pasa el `TaskHandle_t` de esa tarea.
    - Para hacer que una tarea se elimine a *sí misma*: la tarea debe llamar a `vTaskDelete(NULL)`. Una vez que fue eliminada, el scheduler libera la memoria RAM que estaba usando esa tarea.

# ------------------- # ------------------ # -------------- # ------------------ # -------------- # ------------------ # --------------
# Paso 03:

### Prueba A: prioridad de `task_btn` mayor a `task_led`
**Configuración:**
* **Prioridad `task_btn`:** `(tskIDLE_PRIORITY + 2ul)`
* **Prioridad `task_led`:** `(tskIDLE_PRIORITY + 1ul)`

*Resultado*'
    [info]  
    [info] app_init is running - Tick [mS] =   0
    [info]  RTOS - Event-Triggered Systems (ETS)
    [info]  soe-tp0_03-application: Demo Code
    [info]  
    [info] Task BTN is running - Tick [mS] =   0
    [info]  Task BTN - BTN PRESSED
    [info]  Task BTN - BTN HOVER
    [info]  Task BTN - BTN PRESSED
    [info]  Task BTN - BTN HOVER
    [info]  Task BTN - BTN PRESSED
    [info]  Task BTN - BTN HOVER
    [info]  Task BTN - BTN PRESSED
    [info]  Task BTN - BTN HOVER

**Observaciones:**
1.  **Terminal:** Se observa que la tarea del botón se inicializa correctamente y responde a las pulsaciones mostrando los mensajes `Task BTN - BTN PRESSED` y `Task BTN - BTN HOVER`.
2.  **Comportamiento del LED:** El LED permanece apagado y no realiza ningún parpadeo, a pesar de que la lógica de la tarea del botón indica que se están enviando los eventos para iniciar el parpadeo.
3.  **Logs del LED:** En la terminal no aparece el mensaje `Task LED is running`, lo que indica que la tarea ni siquiera llegó a su etapa de inicialización.


### Prueba B: Prioridad de `task_led` mayor a `task_btn`
**Configuración:**
* **Prioridad `task_led`:** `(tskIDLE_PRIORITY + 2ul)`
* **Prioridad `task_btn`:** `(tskIDLE_PRIORITY + 1ul)`

*Resultado*
    [info]  
    [info] app_init is running - Tick [mS] =   0
    [info]  RTOS - Event-Triggered Systems (ETS)
    [info]  soe-tp0_03-application: Demo Code
    [info]  
    [info] Task LED is running - Tick [mS] =   0

**Observaciones:**
1.  **Terminal Serial:** Se observa que el sistema inicia y llega a ejecutar la inicialización de la tarea del LED, mostrando el mensaje `Task LED is running - Tick [mS] = 0`,  pero no aparecen mensajes de la tarea del botón.
2.  **Respuesta del Hardware:** Al presionar el botón, el sistema no reacciona. El LED permanece apagado en todo momento.
3.  **Estado del Sistema:** El sistema parece estar "congelado" en la tarea del LED, ignorando cualquier entrada del usuario.


# ------------------- # ------------------ # -------------- # ------------------ # -------------- # ------------------ # --------------
# Paso 04:

**Configuración y Corrección Previas:**
* Se crearon tres tareas independientes (`BTN 1`, `BTN 2` y `BTN 3`) apuntando a la misma función base `task_btn`.
* Se amplió el tamaño del Heap (`TOTAL_HEAP_SIZE`) desde 3072 Bytes (valor default) a 8192 Bytes. Esto fue necesario para evitar que FreeRTOS se quedara sin memoria al intentar alojar el `stack` y el `Task Control Block (TCB)` de las nuevas tareas, lo cual previamente causaba una detención del sistema mediante `configASSERT`.
* Se agregó la instrucción `vTaskDelete(h_task_btn)` al principio de la inicialización de `task_led`.

*Resultado* 
    [info]  
    [info] app_init is running - Tick [mS] =   0
    [info]  RTOS - Event-Triggered Systems (ETS)
    [info]  soe-tp0_03-application: Demo Code
    [info]  
    [info] Task LED is running - Tick [mS] =   0
    [info]  Task LED - INSTANCIA BTN 1 ELIMINADA
    [info]  
    [info] Task BTN 2 is running - Tick [mS] =   1
    [info]  
    [info] Task BTN 3 is running - Tick [mS] =   2
    [info]  Task BTN 2 - BTN PRESSED
    [info]  Task LED - LED BLINK
    [info]  Task BTN 2 - BTN HOVER
    [info]  Task LED - LED OFF
    [info]  Task BTN 2 - BTN PRESSED
    [info]  Task BTN 3 - BTN PRESSED
    [info]  Task LED - LED BLINK
    [info]  Task BTN 2 - BTN HOVER
    [info]  Task LED - LED OFF


**Observaciones:**
1. **Eliminación Exitosa:** Al arrancar el sistema, la tarea del LED ejecuta la orden de borrado antes de entrar en su bucle infinito y reporta `Task LED - INSTANCIA BTN 1 ELIMINADA`. Como consecuencia, la instancia "BTN 1" nunca llega a ejecutarse ni a imprimir su log de inicialización.
2. **Ejecución Concurrente:** Las instancias "BTN 2" y "BTN 3" sobreviven, se inicializan correctamente y entran en ejecución (Ticks 1 y 2).
3. **Duplicación de Eventos:** Al presionar el pulsador físico una sola vez, ambas tareas ("BTN 2" y "BTN 3") detectan el cambio de estado en el hardware simultáneamente. Esto genera logs paralelos y casi inmediatos (ej. `Task BTN 2 - BTN PRESSED` seguido de `Task BTN 3 - BTN PRESSED`).
4. **Respuesta del Actuador:** El LED reacciona correctamente encendiéndose y apagándose (`LED BLINK` y `LED OFF`) a pesar de recibir las órdenes por duplicado desde dos tareas distintas.

