# soe-tp1_2026-1erC_1-03
FIUBA - Electrónica - Sistemas Operativos Embebidos - Trabajo Práctico N°: 1 – Tareas de FreeRTOS
# FIUBA - Electrónica - Sistemas Operativos Embebidos
## Trabajo Práctico N°: 1 - Tareas de FreeRTOS
### Año-Cuatrimestre - Curso-Grupo
### Responsable de la entrega:
| Padrón | Apellidos, Nombres | Fecha | Deadline |
| :----- | :--------------------- | :------: | :-------: |
| XXXXXX | YYYY, ZZZ | | Semana 04 |


# Análisis de Proyecto STM32 con FreeRTOS

Este documento describe la arquitectura, el flujo de ejecución y la configuración de temporizadores de un proyecto basado en un microcontrolador **STM32F103** utilizando **FreeRTOS** y la biblioteca **HAL** de STMicroelectronics.

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
