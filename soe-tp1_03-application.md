# TP1 – Actividad 03 – 6to Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 02

## ¿Cómo usar el parámetro de Tarea ?
Se pasa un puntero `void *` como cuarto argumento de la funcion `xTaskCreate()`. La tarea lo recibe y lo castea para usarlo.

El parámetro es de tipo puntero a `void *` que se pasa a `xTaskCreate()`. Permite enviar cualquier tipo de dato (int, estructura, arreglo) a la tarea cuando esta se empieza a ejecutar. La tarea recibe ese puntero y los castea al tipo original para usarlo. Es útil para que múltiples instancias de una misma tarea se comporten de forma distinta según el dato que reciban.

int numero = 5;
xTaskCreate(tarea, "T1", 128, (void*)&numero, 1, NULL);

void tarea(void *parametro) {
    int *num = (int*)parametro;
    // usar *num
}

## ¿Cómo cambiar la prioridad de una Tarea ya creada?
Para cambiar la prioridad de una tarea se usa la funcion `Se usa vTaskPrioritySet()`, que recibe 2 parámetros: el handle (manejador) de la tarea y la nueva prioridad de la forma: 
    `vTaskPrioritySet(hadleTarea, nuevaPrioridad);` 
Si se le pasa NULL como manejador, la función cambia la prioridad de la tarea que llama. 
    `vTaskPrioritySet(NULL, nuevaPrioridad);`
Esto permite ajustar dinámicamente la importancia relativa de una tarea durante la ejecución, por ejemplo, elevando la prioridad de una tarea que tiene que responder rápidamente ante un evento.


# ------------------- # ------------------ # -------------- # ------------------ # -------------- # ------------------ # --------------
## Paso 03 - Gestión de dos botones usando parámetro de tarea

### Configuración implementada
**Modificaciones en el código:**
- Se declararon dos estructuras `task_btn_dta_t`: `task_btn_1` (para PC13) y `task_btn_2` (para PA0)
- Se modificó `task_btn()` para recibir un puntero a la estructura correspondiente como parámetro
- Se modificó `task_btn_statechart()` para recibir un puntero a `task_btn_dta_t` y operar sobre él
- En `app.c` se crearon dos instancias de `task_btn`:
  - `Task BTN 1` → parámetro `(void *)&task_btn_1` → botón interno PC13
  - `Task BTN 2` → parámetro `(void *)&task_btn_2` → botón externo PA0

**Conexión del botón externo:**
- Pin utilizado: **PA0**
- Conexión: un terminal a **GND**, el otro a **PA0**
- Configuración: pull-up interno activado (estado normal = HIGH, presionado = LOW)

### Resultado en la terminal
[info]  
[info] app_init is running - Tick [mS] =   0
[info]  RTOS - Event-Triggered Systems (ETS)
[info]  soe-tp0_03-application: Demo Code
[info]  
[info] Task LED is running - Tick [mS] =   0
[info]  
[info] Task BTN 1 is running - Tick [mS] =   0
[info]  
[info] Task BTN 2 is running - Tick [mS] =   1
[info]  Task BTN 1 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN 1 - BTN HOVER
[info]  Task LED - LED OFF
[info]  Task BTN 2 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN 2 - BTN HOVER
[info]  Task LED - LED OFF


**Observaciones:**
1. **Inicialización:** Ambas tareas `Task BTN 1` y `Task BTN 2` se inicializan correctamente, cada una con su propia estructura de datos.
2. **Detección independiente:** Al presionar el botón interno (PC13), se muestra `Task BTN 1 - BTN PRESSED`. Al presionar el botón externo (PA0), se muestra `Task BTN 2 - BTN PRESSED`. Ambos eventos son detectados por separado.
3. **Respuesta del LED:** El LED responde correctamente a ambos botones, ejecutando `LED BLINK` y `LED OFF` para cada pulsación.
4. **Parámetro de tarea:** El uso del parámetro `void *parameters` permite que una misma función `task_btn()` sirva para múltiples instancias, cada una con sus propios datos (puerto, pin, estado, etc.).


# ------------------- # ------------------ # -------------- # ------------------ # -------------- # ------------------ # --------------
## Paso 04

### Prioridad de `task_led` mayor que `task_btn`

**Configuración:**
- **Prioridad `task_led`:** `(tskIDLE_PRIORITY + 2ul)`
- **Prioridad `task_btn`:** `(tskIDLE_PRIORITY + 1ul)`

**Resultado en la terminal:**
[info]
[info] app_init is running - Tick [mS] = 0
[info] RTOS - Event-Triggered Systems (ETS)
[info] soe-tp0_03-application: Demo Code
[info]
[info] Task LED is running - Tick [mS] = 0

**Observaciones:**
1. La tarea `Task LED` se ejecuta primera y exclusivamente.
2. No aparece ningún mensaje de `Task BTN`.
3. Al presionar el botón físico, no hay reacción alguna.
4. El sistema queda "congelado" ejecutando solo `task_led`, que nunca cede el procesador (no tiene `vTaskDelay` ni espera eventos).


## Restauración de prioridades originales
Luego de las pruebas, se modificó nuevamente el archivo `app.c` para dejar ambas tareas con la misma prioridad.
