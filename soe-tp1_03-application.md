### ¿Cómo usar el parámetro de Tarea?
Se utiliza el puntero `void *argument` en la función de creación. Esto permite pasar estructuras o variables a la tarea al momento de su inicio, permitiendo que una misma función de tarea gestione diferentes recursos según el parámetro recibido.

Al crear la tarea con `osThreadNew()`, se puede pasar la dirección de memoria de una variable o estructura. 
Dentro de la función de la tarea, este puntero se castea al tipo de dato correspondiente. Esto es fundamental para:
* **Reutilización de código:** Permite que múltiples instancias de una misma función de tarea operen sobre diferentes datos o periféricos (por ejemplo, pasarle a una tarea de LED qué pin específico debe conmutar).

### ¿Cómo cambiar la prioridad de una Tarea ya creada?
Se utiliza la función `osThreadSetPriority()` pasando el *Handle* de la tarea y el nuevo nivel de prioridad. Esto permite que el sistema operativo se adapte dinámicamente a las necesidades de procesamiento en tiempo real.
