### ¿Cómo implementar el procesamiento periódico mediante una Tarea?
Para que una tarea se ejecute de forma periódica, se utiliza la función `osDelay()` o `vTaskDelay()`. 

**El proceso funciona de la siguiente manera:**
1. La tarea ejecuta su lógica (leer un sensor, parpadear un LED).
2. La tarea llama a `osDelay(periodo)`.
3. El planificador (Scheduler) mueve la tarea del estado **Running** al estado **Blocked**.
4. Durante ese tiempo, otras tareas de menor prioridad pueden ejecutarse.
5. Una vez transcurrido el tiempo, el Scheduler devuelve la tarea al estado **Ready** para que vuelva a ejecutarse.



### ¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?
La tarea **IDLE** es creada automáticamente por el kernel de FreeRTOS cuando se inicia el planificador (`osKernelStart`).

* **¿Cuándo se ejecuta?**: Se ejecuta únicamente cuando **no hay ninguna otra tarea** en estado *Ready*. Es decir, cuando todas las tareas de la aplicación están en estado *Blocked* o *Suspended*. Tiene la prioridad más baja posible (Prioridad 0).
  
* **¿Cómo se puede utilizar?**:
  1. **Liberación de memoria:** Su función principal es liberar la memoria de tareas que han sido eliminadas.
  2. **Modo de bajo consumo (Low Power):** Se puede utilizar para poner al microcontrolador en modo "Sleep" o "Stop" mientras no hay trabajo, ahorrando energía.
  3. **Estadísticas:** Para medir la carga de la CPU (cuanto más tiempo pase el sistema en la tarea IDLE, menos cargado está el procesador).
  4. **Idle Hook:** Se puede configurar una función "Hook" (un callback) que se ejecute automáticamente cada vez que la tarea IDLE tome el control.

 ---
### Procesamiento periódico del Led

**Configuración realizada:**
* Se modificó la tarea `task_led` reemplazando el esquema de retardo relativo por uno de **tiempo absoluto**.
* Se definieron las variables locales `xLastWakeTime` (para almacenar el instante del último desbloqueo) y `xPeriod` (configurada en 500ms mediante la macro `pdMS_TO_TICKS(500)`).
* Se llamó a la función de usuario `task_led_statechart()` dentro del bucle infinito, seguida de la instrucción `vTaskDelayUntil(&xLastWakeTime, xPeriod)`.

**Comportamiento observado:**
* El LED parpadea con una frecuencia constante de 2Hz (500ms por estado). Al utilizar `vTaskDelayUntil`, el scheduler descuenta automáticamente el tiempo de ejecución de la lógica interna de la tarea, logrando un parpadeo preciso y sin derivas temporales. Se verificó que el sistema libera el CPU correctamente mientras el LED permanece en estado de espera.

---

### Procesamiento periódico del Btn

**Configuración realizada:**
* Se implementó el uso de la función `vTaskDelayUntil()` dentro del bucle de `task_btn`.
* Se estableció un período de muestreo de **10ms** mediante `pdMS_TO_TICKS(10)` para asegurar una respuesta reactiva del botón y evitar el fenómeno de *aliasing*.
* Se utilizó la variable `xLastWakeTime` para llevar el registro del tiempo de referencia, asegurando que cada llamado a `task_btn_statechart()` ocurra en intervalos exactos.

**Comportamiento observado:**
* La tarea se ejecuta de forma estrictamente periódica cada 10ms. A diferencia de `osDelay()` o `vTaskDelay()` (que generan retardos relativos), `vTaskDelayUntil()` garantiza un **procesamiento determinístico**. Esto es preferible en sistemas embebidos ya que evita que el tiempo de ejecución de la lógica del botón "empuje" el siguiente inicio de ciclo, manteniendo una base de tiempo real y estable.

---

### Notas de Implementación (Kernel)
Para que estas funciones estuvieran disponibles, fue necesario realizar ajustes en la configuración de FreeRTOS:
* **Habilitación de API:** Se localizó el archivo `FreeRTOSConfig.h` y se modificó `#define INCLUDE_vTaskDelayUntil 1`. Sin este cambio, el linker no puede resolver la referencia a la función, ya que viene desactivada por defecto para ahorrar memoria.
