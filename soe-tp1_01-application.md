# Respuesta de Gemini - Explicación del contenido en: /Core

## 📁 1. Descripción de los Archivos Fuente

El proyecto se compone de varios archivos clave que manejan desde el arranque a bajo nivel hasta la gestión de tareas del sistema operativo:

* **`startup_stm32f103rbtx.s`**: Es el código de arranque en lenguaje ensamblador. Define la **Tabla de Vectores de Interrupción**. Al encender el equipo, ejecuta el `Reset_Handler`, inicializa el sistema de relojes básico, prepara la memoria RAM (copia `.data` y limpia `.bss`) y finalmente hace el salto (`bl main`) a la aplicación en C.
* **`main.c`**: Es el punto de entrada de la aplicación. Configura los relojes (`SystemClock_Config`), inicializa periféricos (GPIO, UART, TIM2) mediante la HAL, crea las tareas del sistema operativo y le cede el control a FreeRTOS llamando a `osKernelStart()`.
* **`stm32f1xx_it.c`**: Contiene las Rutinas de Servicio de Interrupción (ISR). Actúa como puente entre las interrupciones de hardware (como desbordamientos de timers) y los callbacks de la capa HAL o FreeRTOS.
* **`FreeRTOSConfig.h`**: Es el archivo de configuración global del RTOS. Define la frecuencia del *Tick* (`configTICK_RATE_HZ`), el uso de memoria (estática/dinámica), prioridades, y mapea las interrupciones del núcleo Cortex-M necesarias para el planificador (`SysTick_Handler`, `SVC_Handler`, `PendSV_Handler`). También habilita el perfilado de ejecución (`configGENERATE_RUN_TIME_STATS`).
* **`freertos.c`**: Contiene funciones "hook" y configuraciones específicas requeridas por FreeRTOS, como la asignación de memoria estática para la "Tarea Idle" (`vApplicationGetIdleTaskMemory`).

---

## 🔄 2. Comportamiento del Programa (Desde el Reset hasta el Main)

El flujo de ejecución desde que se energiza el microcontrolador hasta que el RTOS toma el control es el siguiente:

1. **Reinicio:** El microcontrolador lee la dirección base `0x00000000` y salta al `Reset_Handler` (en el archivo `startup`).
2. **Pre-inicialización:** Se llama a `SystemInit()` para un ajuste básico del reloj y se inicializa la memoria RAM.
3. **Punto de entrada C:** Salta a la función `main()`.
4. **Inicialización HAL:** Llama a `HAL_Init()` para resetear periféricos y configurar una base de tiempo inicial.
5. **Configuración de Relojes:** Ejecuta `SystemClock_Config()` para establecer la frecuencia final del sistema (usando PLL).
6. **Inicialización de Hardware:** Configura pines, UART y Timers (`MX_GPIO_Init`, `MX_USART2_UART_Init`, `MX_TIM2_Init`).
7. **Arranque de Periféricos:** Se inician interrupciones específicas, como el Timer 2 (`HAL_TIM_Base_Start_IT`).
8. **Creación de Tareas:** Se crean los hilos/tareas iniciales de FreeRTOS.
9. **Arranque del RTOS:** Se llama a `osKernelStart()`. A partir de este momento, el planificador de FreeRTOS toma el control total del procesador.
10. **Bucle Infinito (`while(1)`):** El programa **nunca** debería llegar al `while(1)` ubicado al final de `main.c`. Si llega allí, significa que FreeRTOS falló al iniciar (generalmente por falta de memoria RAM para el *Heap* o la tarea Idle).

---

## 📈 3. Evolución de Variables Globales del Sistema

### `SystemCoreClock`
Variable de CMSIS que almacena la frecuencia del núcleo de la CPU:
* **Al inicio:** `SystemInit()` la configura con el reloj interno básico (ej. 8 MHz HSI).
* **Durante el main:** Tras llamar a `SystemClock_Config()`, el sistema actualiza esta variable para reflejar la frecuencia máxima configurada (ej. 72 MHz). 
* **En FreeRTOS:** Se utiliza para definir la macro `configCPU_CLOCK_HZ`, permitiendo al RTOS calcular los tiempos con precisión.

### `SysTick`
Es el temporizador integrado del núcleo ARM Cortex-M:
* **Al inicio:** La función `HAL_Init()` lo configura para generar interrupciones cada 1 ms. En esta etapa, la HAL lo usa exclusivamente para `HAL_Delay()`.
* **Al iniciar FreeRTOS:** Cuando se llama a `osKernelStart()`, FreeRTOS "secuestra" el SysTick. A partir de aquí, se usa como el *Tick* del sistema operativo, permitiendo el cambio de contexto (*preemption*) y la medición de tiempos para las tareas.

---

## ⏱️ 4. Gestión de Temporizadores y RTOS

Para evitar conflictos de recursos entre la biblioteca HAL y FreeRTOS, el proyecto utiliza múltiples temporizadores con roles bien definidos:

### Interacción de `SysTick` y `TIM2` con FreeRTOS
* **`SysTick` (Tick del SO):** Es el marcapasos de FreeRTOS. Define la base de tiempo (1 ms) para bloqueos (`osDelay`) y decide cuándo cambiar la ejecución entre tareas de igual prioridad.
* **`Timer 2 (TIM2)` (Estadísticas de Ejecución):** Se utiliza para medir cuánto tiempo de CPU consume cada tarea (`configGENERATE_RUN_TIME_STATS`). Se configura a una frecuencia mucho más alta que el SysTick. En su interrupción, incrementa la variable `ulHighFrequencyTimerTicks`, la cual es leída por FreeRTOS para calcular porcentajes de uso de CPU precisos.

### Interacción de `TIM4` con la HAL de STM32
Dado que FreeRTOS se apropia del SysTick, la capa HAL se queda sin su temporizador base para medir timeouts. Para solucionar esto:
* **`Timer 4 (TIM4)`** se configura como la **Nueva Base de Tiempo (Timebase)** de la HAL.
* Interrumpe cada 1 ms y, a través de su ISR (`TIM4_IRQHandler`), llama a `HAL_TIM_PeriodElapsedCallback()`.
* Dentro de ese callback, se ejecuta `HAL_IncTick()`, lo que mantiene operativa a la HAL (y a funciones como `HAL_Delay`) de forma totalmente independiente del RTOS.

---
---

# Explicación del contenido en: /app

## 📁 1. Descripción General de los Archivos

El código de la aplicación se divide en los siguientes archivos principales:

* **`app.c`**: Es el punto de entrada de la aplicación en el entorno RTOS. Inicializa contadores de monitoreo y crea las tareas del sistema (`Task BTN` y `Task LED`).
* **`task_btn.c`**: Contiene la lógica de la tarea del botón. Implementa una máquina de estados para filtrar el rebote mecánico (debounce) y genera eventos cuando detecta pulsaciones válidas.
* **`task_led.c`**: Contiene la lógica de la tarea del LED. Implementa su propia máquina de estados para apagar o hacer parpadear el LED en respuesta a los eventos enviados por el botón.
* **`task_led_interface.c`**: Actúa como una API interna para acoplar el emisor (Botón) con el receptor (LED), manejando la transmisión de eventos.
* **`freertos.c`**: Contiene las implementaciones de los *Hooks* (funciones de enganche) de FreeRTOS, utilizados para métricas de CPU y detección de errores de memoria.

---

## ⚙️ 2. Inicialización y Tareas (`app.c`)

En la función `app_init()` se configuran los elementos base de la aplicación:
* **Contadores globales:** Se inicializan `g_app_tick_cnt`, `g_task_idle_cnt` y `g_app_stack_overflow_cnt` para monitoreo de rendimiento.
* **Creación de tareas:** Se instancian las dos tareas principales mediante `xTaskCreate`.
    * Ambas tienen el mismo tamaño de pila: `2 * configMINIMAL_STACK_SIZE`.
    * Ambas comparten la misma prioridad: `tskIDLE_PRIORITY + 1ul`.

---

## 🔄 3. Máquinas de Estados

El corazón de la aplicación funciona mediante máquinas de estados (Statecharts) que se ejecutan de forma continua en los bucles infinitos de cada tarea.

### Tarea del Botón (`task_btn_statechart`)
Evalúa el hardware y filtra ruidos eléctricos mediante 4 estados:
1.  **`ST_BTN_XX_UP` (Reposo):** Espera a que el botón sea presionado (`EV_BTN_XX_DOWN`). Al detectarlo, guarda el *tick* actual y avanza de estado.
2.  **`ST_BTN_XX_FALLING` (Debounce Presión):** Espera un retardo de `DEL_BTN_XX_MAX`. Si el botón sigue presionado, valida la acción, envía el evento `EV_LED_XX_BLINK` a la tarea del LED y avanza.
3.  **`ST_BTN_XX_DOWN` (Presionado):** Espera a que el botón sea soltado (`EV_BTN_XX_UP`).
4.  **`ST_BTN_XX_RISING` (Debounce Liberación):** Espera el retardo `DEL_BTN_XX_MAX`. Si el botón sigue suelto, valida la acción, envía el evento `EV_LED_XX_OFF` al LED y vuelve al estado inicial.

### Tarea del LED (`task_led_statechart`)
Reacciona a los eventos del botón para controlar el pin del microcontrolador:
1.  **`ST_LED_XX_OFF` (Apagado):** Mantiene el LED apagado. Si recibe el evento `EV_LED_XX_BLINK`, enciende el LED, "baja" la bandera de nuevo evento y avanza.
2.  **`ST_LED_XX_BLINK` (Parpadeo):** * Si recibe `EV_LED_XX_OFF`, apaga el pin y vuelve al estado `ST_LED_XX_OFF`.
    * Mientras no reciba la orden de apagado, invierte el estado del pin (`HAL_GPIO_TogglePin`) cada vez que transcurre el tiempo `DEL_LED_XX_MAX`, creando el efecto de parpadeo continuo.

---

## 📡 4. Comunicación entre Tareas (`task_led_interface.c`)

Para que la tarea del botón se comunique con la tarea del LED, se utiliza una interfaz dedicada:
* La función **`put_event_task_led(task_led_ev_t event)`** recibe el evento a transmitir.
* Sobrescribe una estructura global compartida asignando el nuevo evento a `task_led_dta.event`.
* Levanta un aviso asignando `task_led_dta.flag = true`.
> **Nota de Diseño:** Al tener ambas tareas la misma prioridad y operar de forma cooperativa, este método de variables globales funciona correctamente. Para escalar a sistemas más complejos o con distintas prioridades, se recomienda migrar a *Queues* (Colas) de FreeRTOS.

---

## 🛠️ 5. Monitoreo y Hooks (`freertos.c`)

El sistema aprovecha los *callbacks* automáticos de FreeRTOS para perfilar la aplicación:

* **`vApplicationIdleHook()`:** Se ejecuta cuando el procesador está inactivo. Incrementa `g_task_idle_cnt`. Permite calcular el porcentaje de uso de la CPU.
* **`vApplicationTickHook()`:** Se ejecuta en cada interrupción del sistema (ej. cada 1 ms). Incrementa `g_app_tick_cnt`.
* **`vApplicationStackOverflowHook()`:** Mecanismo de seguridad crítica. Si una tarea excede su memoria RAM asignada, esta función detiene el sistema (`configASSERT( 0 )`) e incrementa `g_app_stack_overflow_cnt` para facilitar la depuración.

---

## 🎯 6. Conclusión: ¿Qué hace esta aplicación?

En resumen, basándonos en el análisis de todo el código fuente (desde el arranque por hardware hasta la capa de aplicación), este proyecto implementa un **sistema embebido interactivo, robusto y guiado por eventos**, diseñado para un microcontrolador STM32F103 ejecutando el sistema operativo FreeRTOS.

A nivel funcional, el sistema realiza lo siguiente:
1. **Monitoreo de Usuario:** Escucha continuamente el estado de un pulsador físico (Botón).
2. **Filtrado Inteligente:** Aplica un algoritmo de *debounce* (anti-rebote) por software, mediante una máquina de estados, para ignorar el ruido eléctrico y detectar pulsaciones y liberaciones limpias y reales.
3. **Respuesta Visual Controlada:** * Al confirmar que el botón fue **presionado**, el sistema activa el LED y lo hace **parpadear** de forma continua.
   * Al confirmar que el botón fue **soltado**, el sistema interrumpe el parpadeo, **apaga** el LED y vuelve a su estado de reposo.

A nivel arquitectónico, el proyecto demuestra excelentes prácticas de diseño de software para sistemas RTOS:
* **Desacoplamiento:** Separa claramente la configuración del hardware de bajo nivel (HAL y relojes), la gestión del sistema operativo (temporizadores SysTick, TIM2 y TIM4) y la lógica de negocio (aplicación de usuario).
* **Concurrencia No Bloqueante:** Divide las responsabilidades en hilos independientes (`Task BTN` y `Task LED`) que operan de forma concurrente y se comunican a través de una interfaz definida. Utiliza el tiempo del sistema (`xTaskGetTickCount`) para medir retardos, evitando el uso de funciones bloqueantes (como `HAL_Delay`) que desperdiciarían ciclos de CPU.
* **Robustez y Perfilado:** Aprovecha las herramientas de FreeRTOS (*Hooks* y contadores) para medir el uso real del procesador, registrar el rendimiento del sistema y vigilar fallos críticos como el desbordamiento de memoria RAM (*Stack Overflow*), garantizando la estabilidad y facilitando la depuración a largo plazo.

---

# Observaciones Propias

- Al depurar el proyecto, la ejecución de las tareas queda asentada en la consola (`Task LED is running` y `Task BTN is running`).

- El comportamiento es tal que el LED (el incluido en la propia placa) permanece apagado mientras el botón azul no es presionado. Cuando este se presiona (`Task BTN - BTN PRESSED`), el LED comienza a parpadear (`Task LED - LED BLINK`) invirtiendo su estado cada segundo, hasta que se suelte. Ambas tareas tienen la misma prioridad, por lo que pueden desempeñar sus funciones correctamente y "en simultáneo" (se le asigna tiempo de CPU a ambas).

- Lo observado, por lo tanto, coincide con las respuestas de Gemini.