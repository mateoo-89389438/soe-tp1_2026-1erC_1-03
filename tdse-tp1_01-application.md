# TP1 – Actividad 01 – 4to Proyecto p/placa NUCLEO-F103RB con FreeRTOS

## Paso 06: 

## 1. Respuesta de la IA Gemini con el prompt:

## Analizar y explicar (en español), el funcionamiento del código fuente contenido en los archivos adjuntos: startup_stm32f103rbtx.s , main.c , stm32f1xx_it.c , FreeRTOSConfig.h y freertos.c . Indicar la evolución de las variables SysTick y SystemCoreClock al ejecutar dicho código fuente desde su inicio ( Reset_Handler : de startup_stm32f103rbtx.s ) hasta el loop principal de la aplicación ( while (1) de main.c ). Indicar el comportamiento del programa al ejecutar dicho código fuente desde su inicio ( Reset_Handler : de startup_stm32f103rbtx.s ) hasta antes de llegar al loop principal de la aplicación ( while (1) de main.c ). Indicar cómo y para qué SysTick y Timer 1 ( TIM2 ) interactúan con FreeRTOS . Indicar cómo y para qué el Timer 4 ( TIM4 ) interactúa con la HAL del proyecto STM32 .

A continuación, presento un análisis detallado del código fuente proporcionado, explicando la función de cada archivo, el flujo de ejecución y la interacción de los temporizadores con el sistema operativo y la capa de abstracción de hardware (HAL).

### 1. Análisis y Explicación de los Archivos

* **`startup_stm32f103rbtx.s`**: Este es el archivo ensamblador de inicio (startup). Su función principal es definir la **tabla de vectores de interrupción** (donde se encuentran las direcciones de memoria de las rutinas de servicio de interrupción, como `Reset_Handler`, `SysTick_Handler`, etc.). Además, contiene el punto de entrada del programa tras un reinicio (`Reset_Handler`), encargado de inicializar la memoria copiando los datos inicializados (sección `.data`) de la memoria Flash a la SRAM, inicializando en cero las variables no inicializadas (sección `.bss`), y finalmente llamando a la función `main()` de C.
* **`main.c`**: Es el núcleo de la aplicación. Aquí se inicializa la HAL (Hardware Abstraction Layer), se configura el reloj del sistema (System Clock), se inicializan los periféricos (GPIO, USART2, TIM2), y se inicializa la aplicación del usuario (`app_init()`). Posteriormente, crea una tarea de FreeRTOS (`defaultTask`) e inicia el planificador del sistema operativo (`osKernelStart()`). También contiene las funciones de *callback* de los temporizadores (`HAL_TIM_PeriodElapsedCallback`).
* **`stm32f1xx_it.c`**: Contiene las Rutinas de Servicio de Interrupción (ISR - *Interrupt Service Routines*). Aquí se capturan las excepciones del procesador Cortex-M3 (como `HardFault_Handler` o `NMI_Handler`) y las interrupciones de los periféricos de hardware, específicamente `TIM2_IRQHandler`, `TIM4_IRQHandler` y `EXTI15_10_IRQHandler`. Su función principal es derivar estos eventos a las funciones manejadoras de la HAL (`HAL_TIM_IRQHandler` y `HAL_GPIO_EXTI_IRQHandler`).
* **`FreeRTOSConfig.h`**: Es el archivo de configuración de FreeRTOS. Define los parámetros operativos del sistema operativo en tiempo real, como la frecuencia del reloj (`configCPU_CLOCK_HZ`), la tasa de *ticks* del OS (`configTICK_RATE_HZ`, configurada a 1000 Hz o 1 ms), el tamaño del *heap*, los niveles de prioridad y la habilitación de estadísticas en tiempo de ejecución (`configGENERATE_RUN_TIME_STATS`). También mapea los *handlers* de FreeRTOS a las interrupciones estándar de ARM (ej. `#define xPortSysTickHandler SysTick_Handler`).
* **`freertos.c`**: Contiene funciones auxiliares requeridas por FreeRTOS. En este caso, implementa `vApplicationGetIdleTaskMemory`, que es necesaria porque FreeRTOS está configurado para soportar asignación estática de memoria (`configSUPPORT_STATIC_ALLOCATION = 1`). Esta función provee la memoria para la tarea "Idle" (inactiva) del OS. También define versiones "débiles" (*weak*) de las funciones para el conteo de estadísticas.

---

### 2. Evolución de las Variables `SysTick` y `SystemCoreClock`

Desde el inicio en `Reset_Handler` hasta el bucle de `main.c`, estas entidades sufren la siguiente evolución:

* **`SystemCoreClock`**:
    1.  **Inicio (`Reset_Handler`)**: Al llamarse a `SystemInit` (típicamente definido externamente), el microcontrolador arranca usando el oscilador interno (HSI) por defecto. `SystemCoreClock` toma el valor base (generalmente 8 MHz en los STM32F1).
    2.  **En `main.c`**: Se ejecuta `SystemClock_Config()`. Aquí se enciende el PLL utilizando el HSI dividido por 2 y multiplicado por 16. Esto configura el reloj del sistema a 64 MHz. Las funciones internas de la HAL actualizan la variable global `SystemCoreClock` para reflejar estos 64 MHz, de modo que FreeRTOS y otras librerías sepan a qué velocidad corre la CPU.
* **`SysTick` (Temporizador del sistema Cortex-M)**:
    1.  **En `HAL_Init()`**: Se inicializa el `SysTick` para generar una interrupción cada 1 ms. En esta etapa temprana, se utiliza puramente para que la HAL pueda manejar tiempos de espera (por ejemplo, en inicializaciones que requieren `HAL_Delay`).
    2.  **Al iniciar FreeRTOS (`osKernelStart()`)**: FreeRTOS toma el control total del `SysTick` reconfigurándolo. A partir de este momento, el `SysTick` deja de pertenecerle a la HAL y se convierte en el "latido" (*tick*) de FreeRTOS para planificar tareas y manejar los tiempos del OS.

---

### 3. Comportamiento del Programa desde `Reset_Handler` hasta antes del `while(1)`

El flujo de ejecución es estrictamente secuencial hasta que arranca el sistema operativo:

1.  **Reinicio**: El procesador salta a la dirección apuntada por el vector de reset y ejecuta `Reset_Handler`.
2.  **Inicialización de bajo nivel**: Se configura el reloj inicial (`SystemInit`) y se prepara la memoria RAM (copiando variables inicializadas y limpiando las no inicializadas).
3.  **Salto a C**: Se llama a la función `main()`.
4.  **Configuración de Hardware**: `HAL_Init()` resetea los periféricos, luego `SystemClock_Config()` ajusta el reloj principal a 64 MHz. Se configuran pines de GPIO, el puerto UART2 y el Timer 2.
5.  **Arranque de Periféricos**: Se arranca el Timer 2 en modo interrupción (`HAL_TIM_Base_Start_IT(&htim2)`) y se llama a la inicialización lógica del usuario (`app_init()`).
6.  **Creación de Tareas**: Se define y crea la tarea `defaultTask` que encapsula la función `StartDefaultTask`.
7.  **Arranque del Sistema Operativo**: Se llama a `osKernelStart()`. En este punto, **el planificador de FreeRTOS toma el control del procesador**.
8.  **Punto de no retorno**: La ejecución **nunca llega** al `while(1)` que se encuentra al final de `main()`. La función `osKernelStart()` solo retorna si no hay suficiente memoria para crear la tarea Idle (lo cual no ocurrirá aquí gracias a la asignación estática). El procesador ahora está ejecutando la tarea `defaultTask` (que tiene su propio bucle infinito) o la tarea Idle.

---

### 4. Interacción de `SysTick` y el Timer 1 (`TIM2`) con FreeRTOS

* **`SysTick`**: Es el motor del sistema operativo. FreeRTOS utiliza el `SysTick` para generar el `TICK_RATE` (configurado a 1000 Hz en `FreeRTOSConfig.h`). Cada vez que ocurre esta interrupción (1 ms), el planificador de FreeRTOS se despierta, evalúa si es momento de cambiar de tarea (cambio de contexto), gestiona los bloqueos (`osDelay(1)`) y actualiza los temporizadores de software.
* **Timer 2 (`TIM2`)**: Interactúa con FreeRTOS única y exclusivamente para generar **Estadísticas de Tiempo de Ejecución (*Run Time Stats*)**.
    * *Para qué sirve*: Permite a FreeRTOS perfilar exactamente cuánto tiempo de CPU consume cada tarea (útil para *debugging* y optimización). Para esto, FreeRTOS necesita un contador que sea mucho más rápido que su propio `SysTick`.
    * *Cómo funciona*: En `FreeRTOSConfig.h`, `configGENERATE_RUN_TIME_STATS` está activado (1). El macro `portGET_RUN_TIME_COUNTER_VALUE` mapea a `getRunTimeCounterValue`. En `main.c`, cada vez que el `TIM2` desborda e interrumpe, se incrementa la variable `ulHighFrequencyTimerTicks`. Esta variable de alta frecuencia es la que FreeRTOS lee para calcular con precisión los tiempos de ejecución.

*(Nota: En la pregunta mencionas "Timer 1 (TIM2)", asumo que te refieres al periférico `TIM2` que está explícitamente inicializado y utilizado en el código para este propósito).*

---

### 5. Interacción del Timer 4 (`TIM4`) con la HAL del proyecto STM32

* **Para qué sirve**: La HAL de STM32 requiere una base de tiempo de 1 ms obligatoria para manejar los tiempos muertos (*timeouts*) en funciones de comunicación (ej. esperar a que el UART transmita) y para la función `HAL_Delay()`. Normalmente, la HAL usa el `SysTick`. Como el `SysTick` fue "secuestrado" por FreeRTOS para funcionar como el reloj del sistema operativo, usarlo para la HAL en un entorno multihilo puede generar colisiones e imprecisiones. Por ello, se destina un temporizador de hardware dedicado exclusivo para la HAL.
* **Cómo interactúa**: Se utiliza el `TIM4` como esta base de tiempo alterna para la HAL. Cuando el `TIM4` genera una interrupción por desbordamiento, el flujo va a `stm32f1xx_it.c`, ejecuta `TIM4_IRQHandler`, pasa por la HAL y finalmente aterriza en el *callback* `HAL_TIM_PeriodElapsedCallback` (ubicado en `main.c`). Allí, el código detecta que la interrupción provino del `TIM4` e invoca `HAL_IncTick()`, la cual incrementa la variable interna `uwTick` de la HAL, manteniendo así las funciones nativas de ST operativas y sincronizadas sin interferir con el núcleo de FreeRTOS.





# -------------------------- # ------------------------- # -------------------------- # ------------------------- # -------------------------- # ------------------------- #
## 2. Respuesta de la IA Gemini con el prompt:

## Analizar y explicar (en español), el funcionamiento del código fuente contenido en los archivos adjuntos: app.c , task_btn.c , task_led.c , task_led_interface.c , y freertos.c .

El código fuente que has compartido corresponde a una aplicación de sistemas embebidos basada en el sistema operativo en tiempo real **FreeRTOS** (probablemente ejecutándose en un microcontrolador STM32, dada la nomenclatura de librerías como `HAL_GPIO_ReadPin`). 

En resumen, el código implementa un **sistema disparado por eventos (Event-Triggered System)** con dos tareas principales: una que monitorea el estado de un botón (aplicando técnicas para evitar rebotes físicos) y otra que hace parpadear un LED mientras el botón se mantenga presionado.

Aquí tienes el análisis y la explicación detallada de cada archivo y su función dentro del sistema:

---

### 1. `app.c` (Inicialización de la Aplicación)
Este archivo actúa como el punto de entrada para la configuración de la aplicación a nivel de sistema operativo.

* **Función `app_init()`:** Es la responsable de preparar el entorno antes de que el planificador (scheduler) de FreeRTOS comience a ejecutarse.
* **Contadores Globales:** Inicializa contadores para monitorear el rendimiento y estado del sistema (`g_app_tick_cnt`, `g_task_idle_cnt`, etc.).
* **Creación de Tareas:** Utiliza la API `xTaskCreate` para instanciar las dos tareas principales del sistema:
    * **Task BTN (`task_btn`):** Tarea del botón.
    * **Task LED (`task_led`):** Tarea del LED.
    * Ambas tareas se crean con la misma prioridad (`tskIDLE_PRIORITY + 1`), lo que significa que FreeRTOS alternará entre ellas de forma equitativa (time-slicing) si ambas están listas para ejecutarse.

### 2. `freertos.c` (Ganchos / Hooks del Sistema Operativo)
Contiene funciones de retrollamada (hooks) que el kernel de FreeRTOS ejecuta automáticamente cuando ocurren eventos específicos a nivel interno. Se usan principalmente para métricas y depuración.

* **`vApplicationIdleHook()`:** Se ejecuta cuando ninguna otra tarea del usuario necesita ejecutarse. Aquí se incrementa un contador (`g_task_idle_cnt`) que sirve para medir cuánto tiempo el procesador está "ocioso".
* **`vApplicationTickHook()`:** Se ejecuta en cada *tick* (interrupción periódica) del sistema operativo. Incrementa el tiempo de la aplicación.
* **`vApplicationStackOverflowHook()`:** Un mecanismo de seguridad fundamental. Si FreeRTOS detecta que alguna tarea ha consumido toda la memoria RAM asignada a su pila (Stack Overflow), llama a esta función. Aquí ejecuta `configASSERT( 0 )`, lo que congela el sistema de inmediato para permitir al desarrollador conectar un depurador y ver qué falló.

### 3. `task_btn.c` (Manejo del Botón y Antirrebote)
Este archivo contiene la lógica de la tarea encargada de leer el botón físico. Su principal desafío es el **"debouncing" (antirrebote)**, ya que los botones mecánicos generan ruido eléctrico rápido al pulsarse.

* Implementa una **Máquina de Estados Finitos (FSM)** en la función `task_btn_statechart()`:
    * **`ST_BTN_XX_UP` (Reposo):** Espera a que el botón baje. Si detecta presión, pasa a *Falling*.
    * **`ST_BTN_XX_FALLING` (Validación de presión):** Espera un tiempo `DEL_BTN_XX_MAX` (50 ms). Si después de ese tiempo el botón sigue presionado, asume que es una presión real y no ruido. Es en este momento donde envía el evento `EV_LED_XX_BLINK` a la tarea del LED y avanza al estado *Down*.
    * **`ST_BTN_XX_DOWN` (Mantenido):** Espera a que el usuario suelte el botón. Si detecta que se soltó, pasa a *Rising*.
    * **`ST_BTN_XX_RISING` (Validación de liberación):** Aplica la misma técnica de espera de 50 ms para validar que realmente se soltó. Una vez confirmado, envía el evento `EV_LED_XX_OFF` para apagar el LED y regresa al inicio.

### 4. `task_led_interface.c` (Comunicación entre Tareas)
Sirve como puente de comunicación para encapsular los datos. En lugar de que la tarea del botón acceda directamente a las variables del LED, utiliza este archivo.

* **`put_event_task_led(event)`:** Es una función pública. Cuando el botón determina que el LED debe encenderse o apagarse, llama a esta función. Ésta modifica la estructura compartida `task_led_dta`, guardando el evento (Blink u Off) y levantando una bandera (`flag = true`) para avisarle al LED que tiene "mensajes no leídos".

### 5. `task_led.c` (Control Visual / Actuador)
Contiene la lógica de la tarea que maneja el hardware del LED (encender, apagar o parpadear), separando esta acción completamente de la lectura física del botón.

* Implementa su propia **Máquina de Estados Finitos (FSM)** en la función `task_led_statechart()`:
    * **`ST_LED_XX_OFF` (Apagado):** Mantiene el pin en estado bajo. Se queda en este estado hasta que la interfaz le avise que la bandera (`flag`) es verdadera y el evento es *Blink*.
    * **`ST_LED_XX_BLINK` (Parpadeo):** Mientras esté en este estado, evalúa el tiempo del sistema (`xTaskGetTickCount()`). Si ha pasado el tiempo estipulado (`DEL_LED_XX_MAX`, 500 ms), utiliza `HAL_GPIO_TogglePin` para invertir el estado del pin (si estaba encendido se apaga, y viceversa), logrando un parpadeo constante y **no bloqueante**.
    * Si en medio del parpadeo recibe el evento *Off* desde la interfaz (porque se soltó el botón), apaga el pin inmediatamente y regresa al estado inicial de reposo.

---

### Resumen del Flujo de Ejecución Global
1. El sistema arranca y ambas tareas (`task_btn` y `task_led`) corren en paralelo en bucles infinitos.
2. Al presionar físicamente el botón, `task_btn` filtra mecánicamente la señal esperando 50 ms.
3. Una vez filtrada, `task_btn` llama a `put_event_task_led()` para ordenar el parpadeo.
4. `task_led` recibe la bandera, pasa al estado de parpadeo y comienza a encender y apagar el LED cada 500 ms sin frenar el sistema (sin usar la típica y mala práctica de `delay()` bloqueantes).
5. Al soltar el botón, `task_btn` vuelve a filtrar por 50 ms, manda la señal de apagado, y `task_led` detiene el ciclo de encendido inmediatamente.
