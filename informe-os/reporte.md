# **Informe Sistemas Ops: Configuración, Análisis e Implementación de MLFQ en xv6**

**Nombre del estudiante:** JUAN MIGUEL SALCEDO GARAY

**Código:** 2538076

## **1\. Entorno de Desarrollo y Herramientas Base**

Para el desarrollo, compilación y depuración del sistema operativo **xv6** se trabaja sobre la siguiente configuración de entorno y control de versiones:

> * **Sistema operativo en el que estoy instalando xv6:** ubuntu-2204-arm64  
> * **Identificador del commit sobre el que trabajo de xv6:** eeb7b41

### **Herramientas con las que tengo que trabajar**

> * **Qemu:** es el emulador que uso para poder trabajar con el sistema operativo y otros mas los cuales no son compatibles con la arquitectura de mi sistema operativo (arm). Entonces este me ayuda a ejecutarlos sin ningun problema, en este caso xv6.  
> * **Risc-V:** esta herramienta me ayuda a traducir el lenguaje de mi computadora al lenguaje en el que trabaja xv6 ya que los 2 tienen arquitecturas diferentes (xv6 estaba hecho inicialmente para procesadores RISC-V y no procesadores comerciales por eso es que se necesita).  
> * **GDB:** esta herramienta me ayuda a saber exactamente donde ocurrio un error. Me permite como pausar y diagnosticar los que le paso al sistema operativo en cierto tiempo de ejecucion de algo.

## **2\. Conceptualización y Reglas Formales del Planificador MLFQ**

**MLFQ:** es una herramienta que se usa para poder planificar procesos y darle prioridad si se necesita. La forma de almacenamiento de procesos es por colas y pueden haber mas o 2 procesos que necesiten compararse para conocer cual necesita prioridad. MLFQ utiliza 2 formas de comparacion, si el proceso tiene a una misma prioridad que el proceso b entonces usa el algoritmo de RR (round-robin), pero si el proceso a es mayor que el proceso b entonces simplemente ejecuta a.

Hay tipos de demanda que hacen que la prioridad varie, entre esos esta cuanto esfuerzo va a hacer en la CPU y si utiliza todo su quantum de tiempo para hacer las tareas que necesita. Esto nos ayuda a determinar el nivel de prioridad del proceso.

### **Precisión de las 5 reglas y parámetros fundamentales de MLFQ**

Para garantizar un comportamiento determinista y evitar problemas como la inanición de procesos largos o la manipulación del planificador, el modelo teórico de MLFQ se formaliza mediante cinco reglas y tres parámetros clave:

> 1. **Regla 1 (Jerarquía estricta entre colas):** Si la prioridad del proceso A es mayor que la del proceso B, se ejecuta el proceso A y el proceso B permanece en espera mientras existan procesos listos en niveles superiores.  
> 2. **Regla 2 (Round-Robin dentro del mismo nivel):** Si dos o más procesos tienen el mismo nivel de prioridad, comparten el procesador mediante turno rotatorio circular (*Round-Robin*) usando el *quantum* definido para ese nivel de cola.  
> 3. **Regla 3 (Prioridad inicial al entrar al sistema):** Todo proceso nuevo inicia siempre en la cola de máxima prioridad (Q0). Dado que el sistema operativo desconoce si el proceso será corto o largo, asume inicialmente que es interactivo.  
> 4. **Regla 4 (Degradación por asignación de tiempo y prevención de manipulación):** Cada nivel de cola tiene una cuota máxima de tiempo (*time allotment*) y un tamaño de *quantum* (generalmente creciente en colas inferiores, por ejemplo: 1, 2, 4 y 8 *ticks* para Q0, Q1, Q2 y Q3). Una vez que un proceso consume su cuota acumulada de CPU en un nivel, desciende un nivel de prioridad, independientemente de cuántas veces haya cedido la CPU voluntariamente por operaciones de entrada/salida (E/S). Esto impide que un proceso haga llamadas de E/S justo antes de agotar su *quantum* para permanecer indefinidamente en Q0.  
> 5. **Regla 5 (Reinicio periódico de prioridades o *Priority Boost*):** Cada cierto intervalo global de tiempo S (*boost period*), todos los procesos del sistema regresan a la cola de máxima prioridad (Q0). Este mecanismo evita la inanición (*starvation*) de procesos intensivos en CPU cuando llegan muchos procesos cortos continuamente y permite que un proceso que cambia de fase (de intensivo en CPU a interactivo) recupere prioridad alta.

**Parámetros fundamentales del modelo:** Para parametrizar MLFQ se requiere definir el número total de colas jerárquicas (por ejemplo, cuatro niveles desde Q0 hasta Q3), la duración del *quantum* asignado a cada cola (corto en niveles superiores para maximizar la interactividad y más amplio en niveles inferiores para reducir sobrecostos por cambios de contexto) y el periodo global S tras el cual se ejecuta el reinicio general de prioridades.

## 

## **3\. Relación entre MLFQ y el Funcionamiento Actual de xv6**

El planificador nativo de xv6 implementa únicamente **Round-Robin simple con expropiación por reloj**. A continuación se contrasta su funcionamiento interno actual frente al comportamiento esperado por MLFQ:

| Dimensión | Funcionamiento Actual en xv6 | Comportamiento Esperado en MLFQ   |
| :---- | :---- | :---- |
| **Estructura de datos** | Una única tabla estática y plana de procesos sin niveles de jerarquía. | Múltiples colas lógicas ordenadas de mayor a menor prioridad (Q0 a Qn-1). |
| **Criterio de selección** | Recorre la tabla linealmente y selecciona el primer proceso en estado listo (*Runnable*). | Explora desde Q0 hacia abajo; solo evalúa un nivel inferior si no existen procesos listos en colas superiores, aplicando Round-Robin dentro del nivel activo. |
| **Duración del *quantum*** | Fijo de **1 tick** para todos los procesos: cada interrupción del temporizador obliga a ceder la CPU. | Variable según la cola (por ejemplo, 1, 2, 4 y 8 *ticks*). Un proceso de cola baja conserva la CPU hasta agotar su *quantum*, salvo que aparezca un proceso listo en una cola superior. |
| **Historial del proceso** | El bloque de control de proceso no almacena métricas de consumo de CPU ni nivel de prioridad. | Cada proceso registra su nivel de cola actual y el tiempo acumulado consumido en dicho nivel para decidir degradaciones. |

A pesar de estas diferencias, el ciclo de vida de los estados de proceso en xv6 (listo, en ejecución, dormido por E/S y terminado) y el mecanismo de cambio de contexto entre el planificador y los procesos se conservan intactos: MLFQ modifica únicamente **qué proceso listo es elegido** y **en qué momento se le exige ceder la CPU**.

## **4\. Cómo Lograr la Implementación de MLFQ en xv6**

Desde una perspectiva conceptual y arquitectónica, para incorporar MLFQ en el núcleo de xv6 manteniendo la seguridad ante concurrencia en sistemas multinúcleo mediante bloqueos de exclusión mutua (*spinlocks*), se deben intervenir cuatro componentes principales del sistema operativo:

### **Paso 1: Definir los parámetros globales del planificador**

**Postulado y justificación:** Se debe establecer en las definiciones globales del núcleo la cantidad total de niveles de prioridad (por ejemplo, cuatro colas numeradas de la 0 a la 3\) y el intervalo global de *ticks* de reloj tras el cual se ejecutará el reinicio periódico de prioridades (Regla 5). Centralizar estos hiperparámetros permite ajustar las políticas de planificación desde un único punto del sistema sin alterar la lógica interna de selección.

### **Paso 2: Extender el Bloque de Control de Procesos (PCB)**

**Postulado y justificación:** La estructura interna que representa a cada proceso en el núcleo debe ampliarse con tres atributos: el nivel de prioridad o cola en la que reside actualmente, un contador de *ticks* acumulados dentro de ese nivel y un registro histórico de *ticks* ejecutados por cada cola. Esto se justifica porque el planificador requiere memoria histórica del comportamiento pasado de cada proceso entre múltiples cambios de contexto para aplicar las reglas de asignación inicial y degradación (Reglas 3 y 4).

### **Paso 3: Actualizar la inicialización, selección jerárquica y reinicio periódico de procesos**

**Postulado y justificación:** En el módulo encargado de administrar la tabla de procesos del núcleo se requieren tres ajustes conceptuales:

> * **Asignación inicial al crear procesos:** Durante la reserva e inicialización de un nuevo proceso, su prioridad debe fijarse automáticamente en el nivel superior (Q0) con su contador de consumo en cero, garantizando el cumplimiento de la **Regla 3**.  
> * **Selección jerárquica con cursor rotatorio por nivel:** En la rutina principal del planificador, la búsqueda lineal simple debe sustituirse por un recorrido escalonado desde la cola de mayor prioridad hasta la de menor prioridad (**Regla 1**). Para asegurar un reparto justo mediante **Round-Robin** dentro de un mismo nivel (**Regla 2**), se debe mantener un puntero o cursor circular independiente por cada cola, de modo que tras cada expropiación la búsqueda continúe a partir del siguiente proceso de ese nivel y no reinicie siempre desde el principio de la tabla.  
> * **Rutina de reinicio periódico (*Priority Boost*):** Se debe incorporar una operación que recorra todos los procesos activos de forma sincronizada y restablezca su nivel de prioridad a Q0 junto con sus contadores de cuota, cumpliendo la **Regla 5** para evitar la inanición.

### **Paso 4: Controlar el *quantum* dinámico y la degradación en el manejo del temporizador**

**Postulado y justificación:** En el módulo de atención de interrupciones del núcleo, en lugar de forzar al proceso activo a ceder la CPU incondicionalmente en cada interrupción del reloj de hardware, el sistema debe aplicar una lógica de retroalimentación en cada *tick*: primero incrementa un contador global de tiempo y dispara el reinicio periódico de prioridades si se alcanza el umbral establecido (**Regla 5**); segundo, incrementa los *ticks* consumidos por el proceso que está ocupando la CPU y evalúa si agotó el *quantum* asignado a su cola actual. Si lo agotó, degrada su prioridad al nivel inmediatamente inferior y le ordena ceder el procesador (**Regla 4**); si aún conserva tiempo disponible pero ha ingresado un proceso listo en una cola de mayor prioridad, también le ordena ceder la CPU de inmediato para respetar la jerarquía estricta (**Regla 1**).

## **5\. Propuesta de Validación Experimental en xv6**

Para comprobar rigurosamente que una futura implementación en xv6 satisface todas las reglas de MLFQ, se propone incorporar un mecanismo de consulta de estado de procesos (que exponga el identificador del proceso, su cola actual y el tiempo acumulado en cada nivel) junto con cuatro pruebas experimentales controladas:

> 1. **Verificación de prioridad inicial y expropiación estricta (Reglas 1 y 3):**  
   * *Procedimiento:* Ejecutar un proceso intensivo en cálculo y esperar a que descienda hasta la cola de menor prioridad (Q3). En ese instante, lanzar un proceso nuevo de corta duración.  
   * *Resultado esperado:* Las métricas del sistema deben confirmar que el proceso nuevo inicia en Q0 y desplaza de inmediato al proceso intensivo en el siguiente *tick* de reloj, finalizando su ejecución antes de que el proceso de baja prioridad vuelva a recibir CPU.  
> 2. **Verificación de Round-Robin intra-cola (Regla 2):**  
   * *Procedimiento:* Lanzar simultáneamente tres procesos idénticos intensivos en CPU.  
   * *Resultado esperado:* Los tres procesos deben descender de manera sincronizada hasta Q3 y repartirse el tiempo de procesador equitativamente (aproximadamente un tercio del tiempo total para cada uno), intercalando sus turnos de ejecución según el *quantum* de dicha cola.  
> 3. **Verificación de diferenciación entre procesos CPU-bound e I/O-bound (Regla 4):**  
   * *Procedimiento:* Ejecutar en paralelo un proceso de cálculo puro y un proceso interactivo que realiza ráfagas cortas de trabajo seguidas de bloqueos voluntarios por espera de entrada/salida.  
   * *Resultado esperado:* Al consultar las estadísticas de ejecución por cola, el proceso de cálculo concentrará la mayor parte de su tiempo en Q3, mientras que el proceso interactivo permanecerá mayoritariamente en Q0 y Q1. Adicionalmente, al ejecutar un proceso que realiza pausas voluntarias repetidamente justo antes de agotar su \*quantum\*, su contador acumulado debe terminar degradándolo igualmente.  
> 4. **Verificación del *Priority Boost* contra inanición (Regla 5):**  
   * *Procedimiento:* Mantener un proceso intensivo en CPU ejecutándose en segundo plano en Q3 mientras se generan ráfagas continuas de procesos cortos que ocupan las colas superiores (Q0 y Q1).  
   * *Resultado esperado:* Monitoreando periódicamente los niveles de prioridad, se debe observar que cada vez que transcurre el intervalo global configurado, el nivel de prioridad del proceso de fondo regresa a Q0, garantizando que avance su ejecución y nunca quede bloqueado indefinidamente por inanición.

## **6\. Análisis del Planificador por Defecto de xv6 (Código Fuente y Compilación)**

A partir de la inspección directa del código fuente y la documentación del sistema operativo xv6, se responden las interrogantes sobre el funcionamiento interno del planificador nativo junto con el respaldo exacto de rutas, funciones y fragmentos de código (referenciados sobre la estructura estándar kernel/ de xv6-riscv; en la variante clásica xv6-public de x86 estos archivos residen directamente en la raíz del proyecto con el mismo nombre):

### **i. Algoritmo de planificación por defecto en xv6 y evidencia en su comportamiento**

**Política utilizada:** xv6 implementa por defecto un algoritmo **Round-Robin (turno rotatorio circular) con expropiación (*preemptive*) basada en tiempo**. No existen niveles de prioridad, colas múltiples ni envejecimiento: todos los procesos son almacenados en una única tabla estática (struct proc proc\[NPROC\]) y reciben exactamente la misma jerarquía y el mismo tamaño de *quantum* (1 *tick* de reloj).

**Cómo evidenciarlo en el comportamiento del sistema operativo:**

> * **Evidencia observable en ejecución concurrente:** Si desde la *shell* de xv6 se lanzan simultáneamente en segundo plano dos o más programas intensivos en CPU (por ejemplo, un programa de usuario con un ciclo infinito de cálculo que imprima periódicamente su identificador de proceso PID), se observa en la consola que **ningún proceso bloquea al otro ni monopoliza el procesador**. Las salidas de ambos procesos aparecen intercaladas de manera alternada y equitativa. Si xv6 utilizara una política FIFO (*First-In, First-Out*) o cooperativa sin expropiación, el primer proceso retendría la CPU indefinidamente y el segundo nunca llegaría a ejecutarse.  
> * **Evidencia estructural en el código fuente:** Este comportamiento es consecuencia directa de dos mecanismos acoplados en el núcleo: (1) un bucle infinito en scheduler() (kernel/proc.c) que recorre circularmente el arreglo proc\[\] tomando el siguiente proceso en estado RUNNABLE sin comparar prioridades, y (2) la invocación obligatoria a yield() dentro del manejador de interrupciones del temporizador (kernel/trap.c) cada vez que ocurre un *tick* de reloj, forzando al proceso en ejecución a ceder la CPU.

// Evidencia 1: Recorrido circular sin prioridades en kernel/proc.c (función scheduler)  
for(p \= proc; p \< \&proc\[NPROC\]; p++) {  
  acquire(\&p-\>lock);  
  if(p-\>state \== RUNNABLE) {  
    // Selecciona inmediatamente el siguiente proceso listo en la tabla  
    p-\>state \= RUNNING;  
    c-\>proc \= p;  
    swtch(\&c-\>context, \&p-\>context);  
    c-\>proc \= 0;  
  }  
  release(\&p-\>lock);  
}

// Evidencia 2: Expropiación periódica por reloj en kernel/trap.c (usertrap / kerneltrap)  
// Si la interrupción proviene del temporizador (which\_dev \== 2), se cede la CPU:  
if(which\_dev \== 2\)  
  yield();

### **ii. Archivo y función donde se selecciona el siguiente proceso a entrar a la CPU**

> * **Ruta del archivo:** kernel/proc.c *(en xv6-public: proc.c)*.  
> * **Función:** void scheduler(void)

**Explicación de su operación:** Cada núcleo físico de CPU ejecuta de forma independiente su propio ciclo infinito dentro de scheduler() tras concluir la inicialización del sistema. En cada iteración habilita las interrupciones de hardware (intr\_on()) para evitar bloqueos cuando no hay procesos listos, recorre secuencialmente la tabla global proc\[NPROC\] y selecciona el primer proceso cuyo estado sea RUNNABLE, transfiriéndole el control de ejecución mediante la rutina de cambio de contexto swtch().

// Ruta: kernel/proc.c  
// Función: scheduler(void)  
void  
scheduler(void)  
{  
  struct proc \*p;  
  struct cpu \*c \= mycpu();

  c-\>proc \= 0;  
  for(;;){  
    // Habilita interrupciones para evitar deadlocks si ningún proceso está listo  
    intr\_on();

    for(p \= proc; p \< \&proc\[NPROC\]; p++) {  
      acquire(\&p-\>lock);  
      if(p-\>state \== RUNNABLE) {  
        // Cambia el estado del proceso elegido y lo asigna a la CPU actual  
        p-\>state \= RUNNING;  
        c-\>proc \= p;  
        swtch(\&c-\>context, \&p-\>context);

        // El proceso cedió la CPU y devolvió el control al planificador  
        c-\>proc \= 0;  
      }  
      release(\&p-\>lock);  
    }  
  }  
}

### **iii. Relación con los estados de los procesos y los eventos de cesión o recuperación de la CPU**

> * **Rutas de los archivos:** kernel/proc.h (definición de estados) y kernel/proc.c (transiciones de estado y cambio de contexto).  
> * **Funciones involucradas:** scheduler(), sched(), yield(), sleep(), wakeup(), fork() y exit().

**Relación con los estados en el momento de la selección:**

> * **Proceso que va a entrar a la CPU:** Debe encontrarse obligatoriamente en estado RUNNABLE (listo para ejecutarse). Al ser elegido por scheduler(), su estado cambia inmediatamente a RUNNING bajo la protección de su *spinlock* (p-\>lock), lo que garantiza que ningún otro núcleo de CPU pueda seleccionarlo simultáneamente.  
> * **Proceso que estaba en la CPU:** Antes de que scheduler() pueda evaluar y elegir otro proceso en esa CPU, el proceso que estaba en estado RUNNING tuvo que haber cambiado su propio estado (a RUNNABLE, SLEEPING o ZOMBIE) e invocado a la función puente sched(), la cual guarda los registros del proceso en p-\>context y restaura el contexto del planificador (c-\>context).

**Eventos que permiten ceder la CPU (salida de RUNNING hacia sched()):**

> 1. **Expropiación por tiempo (yield() en kernel/proc.c):** Disparada por la interrupción del temporizador en kernel/trap.c. Cambia el estado del proceso activo de RUNNING a RUNNABLE y llama a sched() para ceder el turno al siguiente proceso listo.  
> 2. **Bloqueo voluntario por espera o Entrada/Salida (sleep() en kernel/proc.c):** Ocurre cuando un proceso solicita leer del disco, espera entrada por consola o aguarda la terminación de un hijo (wait()). Registra un canal de espera (p-\>chan), cambia su estado de RUNNING a SLEEPING e invoca sched().  
> 3. **Finalización de ejecución (exit() en kernel/proc.c):** Cambia el estado del proceso de RUNNING a ZOMBIE, notifica a su proceso padre y llama a sched() por última vez para liberar la CPU de manera definitiva.

**Eventos que permiten recuperar la elegibilidad para la CPU (transición hacia RUNNABLE):**

> 1. **Despertar por interrupción o liberación de recurso (wakeup() en kernel/proc.c):** Cuando el controlador de disco o UART concluye una operación de E/S, invoca wakeup(chan), que recorre la tabla proc\[\] y pasa de SLEEPING a RUNNABLE a los procesos que esperaban en ese canal.  
> 2. **Creación de un nuevo proceso (fork() en kernel/proc.c):** Al duplicar un proceso padre, fork() finaliza asignando np-\>state \= RUNNABLE al proceso hijo, incorporándolo al conjunto de candidatos de scheduler().

// 1\. Definición de estados en kernel/proc.h:  
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

// 2\. Funciones de cesión y recuperación en kernel/proc.c:  
void  
sched(void)  
{  
  int intena;  
  struct proc \*p \= myproc();  
  // ... validaciones de lock e interrupciones ...  
  intena \= mycpu()-\>intena;  
  swtch(\&p-\>context, \&mycpu()-\>context); // Devuelve el control a scheduler()  
  mycpu()-\>intena \= intena;  
}

void  
yield(void)  
{  
  struct proc \*p \= myproc();  
  acquire(\&p-\>lock);  
  p-\>state \= RUNNABLE; // Pasa de RUNNING a RUNNABLE  
  sched();             // Cede la CPU  
  release(\&p-\>lock);  
}

void  
sleep(void \*chan, struct spinlock \*lk)  
{  
  struct proc \*p \= myproc();  
  acquire(\&p-\>lock);  
  release(lk);  
  p-\>chan \= chan;  
  p-\>state \= SLEEPING; // Pasa de RUNNING a SLEEPING  
  sched();             // Cede la CPU hasta que ocurra wakeup(chan)  
  p-\>chan \= 0;  
  release(\&p-\>lock);  
  acquire(lk);  
}

void  
wakeup(void \*chan)  
{  
  struct proc \*p;  
  for(p \= proc; p \< \&proc\[NPROC\]; p++) {  
    if(p \!= myproc()){  
      acquire(\&p-\>lock);  
      if(p-\>state \== SLEEPING && p-\>chan \== chan) {  
        p-\>state \= RUNNABLE; // Recupera elegibilidad para scheduler()  
      }  
      release(\&p-\>lock);  
    }  
  }  
}

### **iv. Incorporación del código del planificador durante la compilación y construcción de xv6**

> * **Rutas de los archivos:** Makefile (raíz del repositorio), kernel/kernel.ld, kernel/swtch.S y kernel/main.c.  
> * **Funciones y reglas clave:** Reglas OBJS y \$K/kernel en Makefile, función swtch en kernel/swtch.S y función main() en kernel/main.c.

**Proceso de construcción e integración en el binario del sistema operativo:**

> 1. **Compilación cruzada a archivos objeto (.o):** El archivo Makefile declara la variable OBJS con todos los módulos que conforman el núcleo, incluyendo explícitamente \$K/proc.o (que contiene scheduler(), sched() y yield()) y \$K/swtch.o (rutina en ensamblador que intercambia los registros del procesador). El compilador cruzado (riscv64-linux-gnu-gcc) compila kernel/proc.c con opciones de entorno independiente (-ffreestanding \-nostdlib \-fno-builtin), sin librerías estándar externas, generando el objeto reubicable kernel/proc.o.  
> 2. **Enlazado estático mediante el linker script (kernel/kernel.ld):** El enlazador (ld) combina todos los archivos objeto de OBJS guiándose por las reglas de memoria definidas en kernel/kernel.ld. Este script sitúa el código ejecutabe (.text), los datos (.data) y las estructuras globales como la tabla proc\[NPROC\] (.bss) a partir de la dirección física de arranque 0x80000000, produciendo el ejecutable ELF final del kernel: kernel/kernel.  
> 3. **Arranque en QEMU e invocación desde main():** Al ejecutar make qemu, el emulador carga el binario kernel/kernel junto con la imagen del sistema de archivos fs.img. Tras la inicialización temprana de la arquitectura, cada procesador ejecuta la función main() en kernel/main.c, la cual inicializa la tabla de procesos (procinit()), crea el primer proceso de usuario (userinit()) e invoca directamente a scheduler() para poner en marcha el algoritmo de planificación.

\# 1\. Fragmento de Makefile (definición de objetos y enlazado del kernel):  
OBJS \= \\  
  \$K/entry.o \\  
  \$K/start.o \\  
  \$K/console.o \\  
  \$K/printf.o \\  
  \$K/uart.o \\  
  \$K/kalloc.o \\  
  \$K/spinlock.o \\  
  \$K/string.o \\  
  \$K/main.o \\  
  \$K/vm.o \\  
  \$K/proc.o \\  
  \$K/swtch.o \\  
  \$K/trampoline.o \\  
  \$K/trap.o \\  
  \$K/syscall.o \\  
  \$K/sysproc.o \\  
  \$K/bio.o \\  
  \$K/fs.o \\  
  \$K/log.o \\  
  \$K/sleeplock.o \\  
  \$K/file.o \\  
  \$K/pipe.o \\  
  \$K/exec.o \\  
  \$K/sysfile.o \\  
  \$K/kernelvec.o \\  
  \$K/plic.o \\  
  \$K/virtio\_disk.o

\$K/kernel: \$(OBJS) \$K/kernel.ld \$U/initcode  
	\$(LD) \$(LDFLAGS) \-T \$K/kernel.ld \-o \$K/kernel \$(OBJS)

// 2\. Fragmento de kernel/main.c (función main que activa el planificador compilado):  
void  
main()  
{  
  if(cpuid() \== 0){  
    // ... inicialización de consola, memoria, interrupciones y disco ...  
    procinit();      // Inicializa los locks y pilas de la tabla proc\[\]  
    // ...  
    userinit();      // Crea el primer proceso de usuario (initcode) en estado RUNNABLE  
    \_\_sync\_synchronize();  
    started \= 1;  
  } else {  
    while(started \== 0\)  
      ;  
    \_\_sync\_synchronize();  
  }

  scheduler();       // Entra al bucle infinito del planificador en kernel/proc.c  
}

## **7\. Reflexion del reporte**

Entendí como funciona el sistema de planificación de procesos de xv6 el algoritmo de RR (round-robin) y cómo funciona el sistema de prioridad MLFQ. Pero todavía no entiendo muy bien cómo manejar el sistema de procesos, como programarlos y cómo revisar el código cuál va a tener más prioridad. De forma teórica lo entiendo todo, pero al aplicarlo si me quedo un poco corto, tendré que estudiar más eso.  
