### ¿Cómo usar el parámetro de Tarea?
Se utiliza el puntero `void *argument` en la función de creación. Esto permite pasar estructuras o variables a la tarea al momento de su inicio, permitiendo que una misma función de tarea gestione diferentes recursos según el parámetro recibido.

Al crear la tarea con `xTaskCreate()`, se puede pasar la dirección de memoria de una variable o estructura. 
Dentro de la función de la tarea, este puntero se castea al tipo de dato correspondiente. Luego, dentro de la tarea se hace un "cast" (conversión de tipo) para volver a usar el dato original.

### ¿Cómo cambiar la prioridad de una Tarea ya creada?
Se utiliza la función `vTaskPrioritySet()` pasando el *Handle* de la tarea y el nuevo nivel de prioridad. Si se le sube la prioridad a una tarea y esta está lista para ejecutarse, el Scheduler hará un cambio de contexto inmediatamente, deteniendo lo que esté haciendo para atender a la tarea que ahora es más importante. Esto permite que el sistema operativo se adapte dinámicamente a las necesidades de procesamiento en tiempo real.

---

## Gestión de Múltiples Botones Mediante el Parámetro de Tarea

**Configuración realizada:**

* Definir los puertos correspondientes en el archivo `.ioc`.
* Declarar un `enum` de IDs de botones en `task_btn_attribute.h` e incluir este archivo en `task_btn.h`, para que `app.c` tenga acceso a ambos y al `enum`.
* Enviar por parámetro de cada task desde `app.c` el valor de ID correspondiente al botón a utilizar, haciendo uso del `enum` y con un casteo a `(void *)`.
* Modificar la variable `task_btn_dta` a `task_btn_dta_list[]` para poder pasar de tener una estructura a tener una lista de estructuras.
* Adaptar la función `task_btn_statechart()` para que funcione con la lista de estructuras, pasándole un `index` para que sepa a cuál botón manipular.

---

## Modificación Dinámica de Prioridades en `task_led`

**Configuración realizada:**

* Elevar la prioridad inicial de ambas instancias de los leds. Luego una vez inicializadas, incluir la linea de codigo: `vTaskPrioritySet(NULL, 1)` asi cada una se autoasigna la prioridad original.

Para adaptar el proyecto para incluir multiples leds:

* Definir los puertos correspondientes en el archivo `.ioc`.
* Declarar un `enum` de IDs de leds en `task_led_attribute.h` e incluir este archivo en `task_led.h`, para que `app.c` tenga acceso a ambos y al `enum`.
* Enviar por parámetro de cada task desde `app.c` el valor de ID correspondiente al led a utilizar, haciendo uso del `enum` y con un casteo a `(void *)`.
* Modificar la variable `task_btn_dta` a `task_btn_dta_list[]` para poder pasar de tener una estructura a tener una lista de estructuras.
* Adaptar la función `task_btn_statechart()` para que funcione con la lista de estructuras, pasándole un `index` para que sepa a cuál botón manipular.

**Comportamiento observado:**

* Las tareas correspondientes a los LEDs se ejecutan antes que la de los botones debido a que comienzan la ejecución con una prioridad `2`. Luego de su primera ejecución, las prioridades de las tareas de los LEDs se degradan a `1`, teniendo así, la misma prioridad que el resto de tareas (exceptuando la tarea IDLE).
