### ¿Cómo usar el parámetro de Tarea?
Se utiliza el puntero `void *argument` en la función de creación. Esto permite pasar estructuras o variables a la tarea al momento de su inicio, permitiendo que una misma función de tarea gestione diferentes recursos según el parámetro recibido.

Al crear la tarea con `osThreadNew()`, se puede pasar la dirección de memoria de una variable o estructura. 
Dentro de la función de la tarea, este puntero se castea al tipo de dato correspondiente. Luego, dentro de la tarea se hace un "cast" (conversión de tipo) para volver a usar el dato original.

### ¿Cómo cambiar la prioridad de una Tarea ya creada?
Se utiliza la función `osThreadSetPriority()` pasando el *Handle* de la tarea y el nuevo nivel de prioridad. Si le subís la prioridad a una tarea y esta está lista para ejecutarse, el Scheduler hará un cambio de contexto inmediatamente, deteniendo lo que esté haciendo para atender a la tarea que ahora es más importante. Esto permite que el sistema operativo se adapte dinámicamente a las necesidades de procesamiento en tiempo real.
