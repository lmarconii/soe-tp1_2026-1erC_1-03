## ⏱️ ¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea?

Utilizando **SysTick**, en cada interrupción del Tick, el **Scheduler** evalúa el sistema. Si hay tareas con la misma prioridad, utiliza un mecanismo de programación **Round-Robin** (ronda de mate) para asignar equitativamente porciones de tiempo (**Time Slices**) a cada una de esas tareas.

---

## 🎯 ¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?

Mediante el **Scheduler**, el cual ejecuta la tarea de mayor prioridad primero, y y luego desciende hasta la última a medida que las de mayor prioridad van cediendo el control.

---

## ⚖️ ¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema?

Una tarea de mayor prioridad interrumpirá inmediatamente a una de menor prioridad en cuanto pase al estado **READY**. Esto significa que las tareas de baja prioridad solo obtendrán tiempo de CPU si todas las tareas de mayor prioridad están en estado **BLOCKED** o **SUSPENDED**.

---

## 🔄 ¿Cuáles son los estados en los que puede encontrarse una Tarea?

Una tarea en FreeRTOS siempre se encuentra en uno de estos cuatro estados:

1. **RUNNING** (Ejecutándose)
2. **READY** (Lista para ejecutarse)
3. **BLOCKED** (Bloqueada esperando un evento o tiempo)
4. **SUSPENDED** (Suspendida temporalmente)

---

## 🛠️ ¿Cómo implementar Tareas?

Las tareas se implementan como funciones en C que contienen un bucle infinito y nunca retornan:

```c
void Tarea(void const * argumento)
{
    // Código de inicialización de la tarea
    
    for(;;)
    {
        // Código de la tarea (bucle infinito)
        osDelay(100);
    }
}
```

---

## 🏗️ ¿Cómo crear una o más instancias de una Tarea?

Utilizando la API de CMSIS-OS, se hace en dos pasos: se define la estructura y luego se crean las instancias.

```c
// 1. Declaramos los "Handles" (identificadores) para cada instancia
TaskHandle_t instancia1;
TaskHandle_t instancia2;

// 2. Creación de las instancias
// Parámetros: Función, Nombre, Tamaño Pila, Argumento, Prioridad, Puntero al Handle
xTaskCreate(MiTarea, "Tarea 1", 128, (void*) 1, tskIDLE_PRIORITY + 1, &instancia1);
xTaskCreate(MiTarea, "Tarea 2", 128, (void*) 2, tskIDLE_PRIORITY + 1, &instancia2);
```

---

## 🗑️ ¿Cómo eliminar una Tarea?

Para destruir una tarea y liberar sus recursos:

```c
// Para eliminar una tarea específica indicando su Handle:
vTaskDelete(instancia1);

// Para que la tarea se elimine a sí misma (se llama dentro de su propio código):
vTaskDelete(NULL);
```
