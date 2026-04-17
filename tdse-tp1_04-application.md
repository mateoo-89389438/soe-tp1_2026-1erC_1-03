# TP1 – Actividad 04 – 7mo Proyecto p/placa NUCLEO-F103RB con FreeRTOS

# Paso 02 

# ¿Cómo implementar el procesamiento periódico mediante una Tarea ?
Para esto se usa la función `vTaskDelay()` dentro del bucle infinito de la tarea. Esta función suspende la tarea durante una cantidad fija de *ticks*. cuando termina ese tiempo, el *scheduler* la vuelve a poner en estado *READY* y se ejecuta de nuevo. De esta manera se logra que la tarea se ejecute cada cierto tiempo, sin consumir CPU innecesariamente entre ejecuciones.

# ¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?
La tarea `IDLE` se ejecuta automáticamente cuando no hay ninguna otra tarea en estado *READY* para ejecutarse. Tiene la prioridad más baja (0). Se puede utilizar para: medir el tiempo libre del CPU, poner el microcontrolador en modo de bajo consumo (sleep), o ejecutar tareas de mantenimiento no críticas mediante un *hook* (callback) llamado `vApplicationIdleHook()`.

# ------------------- # ------------------ # -------------- # ------------------ # -------------- # ------------------ # --------------
# Paso 03 - Procesamiento periódico de `task_led`

### Implementación

**Modificaciones en `task_led.c`:**
- Se agregó la constante `#define PERIOD_LED_MS 500ul`
- Se agregó `vTaskDelay(pdMS_TO_TICKS(PERIOD_LED_MS))` dentro del bucle `for(;;)`
- Se simplificó `task_led_statechart()` para que realice un `toggle` del LED cada vez que se ejecuta

**Comportamiento esperado:**
- La tarea `task_led` se ejecuta cada 500 ms
- El LED cambia de estado (enciende/apaga) en cada ejecución
- La tarea se bloquea entre ejecuciones, liberando el CPU para otras tareas

**Resultado en la terminal:**
[info]  
[info] app_init is running - Tick [mS] =   0
[info]  RTOS - Event-Triggered Systems (ETS)
[info]  soe-tp0_03-application: Demo Code
[info]  
[info] Task LED is running - Tick [mS] =   0
[info]  
[info] Task BTN is running - Tick [mS] =   0


**Observaciones:**
1. **LED:** Parpadea continuamente cada 500 ms de forma autónoma, sin necesidad de intervención del botón.
2. **Terminal:** Solo se muestran los mensajes de inicialización de las tareas. No aparecen mensajes de "LED BLINK" ni "LED OFF" porque el statechart fue reemplazado por un toggle simple.
3. **Botón:** Aunque `task_btn` sigue ejecutándose y detectando pulsaciones, el LED ya no responde a sus eventos porque su comportamiento ahora es estrictamente periódico.




# ------------------- # ------------------ # -------------- # ------------------ # -------------- # ------------------ # --------------
## Paso 04 - Procesamiento periódico de `task_btn`


### Implementación
**Modificaciones en `task_btn.c`:**
- Se agregó la constante `#define PERIOD_BTN_MS 50ul`
- Se agregó `vTaskDelay(pdMS_TO_TICKS(PERIOD_BTN_MS))` dentro del bucle `for(;;)`
- El statechart de detección de flancos (PRESSED/HOVER) no se modificó.

**Comportamiento esperado:**
- La tarea `task_btn` se ejecuta cada 50 ms.
- En cada ejecución, se lee el estado del botón y se actualiza.
- La tarea se bloquea entre ejecuciones, liberando el CPU para otras tareas
- El debounce ya estaba configurado en `DEL_BTN_XX_MAX = 50`.

---

**Resultado en la terminal:**
[info]  
[info] app_init is running - Tick [mS] =   0
[info]  RTOS - Event-Triggered Systems (ETS)
[info]  soe-tp0_03-application: Demo Code
[info]  
[info] Task LED is running - Tick [mS] =   0
[info]  
[info] Task BTN is running - Tick [mS] =   0
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER


**Observaciones:**
1. **Detección:** El botón sigue respondiendo correctamente. Cada vez que se presiona y suelta, aparecen los mensajes `BTN PRESSED` y `BTN HOVER` en la terminal.
2. **Latencia de respuesta:** La detección tiene una latencia máxima de 50 ms (el tiempo entre muestreos), que es imperceptible para el usuario.
3. **Consumo de CPU:** Al usar `vTaskDelay(50 ms)`, `task_btn` se ejecuta solo 20 veces por segundo, liberando el CPU entre muestreos. Mucho más eficiente que la versión original, donde la tarea se ejecutaba continuamente.
4. **Interacción con el LED:** Aunque `task_btn` sigue enviando los eventos (`put_event_task_led(EV_LED_XX_BLINK)` y `put_event_task_led(EV_LED_XX_OFF)`), el LED **no reacciona** porque `task_led` se modificó en el Paso 03 para tener comportamiento periódico autónomo (toggle cada 500 ms), ignorando los eventos (en el enunciado del paso 03 no se dice que haya que volver a la version original). 



