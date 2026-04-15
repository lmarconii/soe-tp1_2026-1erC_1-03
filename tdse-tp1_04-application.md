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
