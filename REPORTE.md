# Exploración de la Planificación de Procesos en xv6

**Autor:** Yorman Andrés Correa Castillo  
**Curso:** Sistemas Operativos, Facultad de Ingeniería, Universidad del Valle  
**Profesor:** **\[John Sanabria\]**  
**Fecha:** 17 de noviembre de 2026

> Los textos en negrita entre corchetes, como **\[Pendiente: …\]** o **\[Insertar captura: …\]**, marcan lo que falta completar antes de entregar.

Este informe documenta la exploración de la planificación de procesos en xv6, un sistema operativo creado con fines educativos que reimplementa Unix versión 6 para la arquitectura RISC-V. El trabajo se ejecutó en el emulador QEMU y tiene dos propósitos: investigar cómo planifica los procesos xv6 y analizar qué cambios serían necesarios para incorporar el algoritmo MLFQ (Multilevel Feedback Queue).

Repositorio (fork): **https://github.com/\[usuario\]/xv6-riscv**

## Repositorio y Entorno

### Versión de Linux y Commit de Partida

El código fuente se clonó en Google Cloud Shell. La versión del sistema operativo se obtuvo con los comandos cat /etc/os-release y uname -r (Figura 1):

- **Distribución:** Ubuntu 24.04.5 LTS (Noble Numbat).

- **Kernel de Linux:** 6.18.59+.

- **Entorno:** Google Cloud Shell.

**Figura 1**

*Versión de Linux del Entorno de Trabajo*

![Versión de Linux del Entorno de Trabajo](docs/img/fig01-version-linux.png)

*Nota.* Salida de los comandos cat /etc/os-release y uname -r en Google Cloud Shell.

El identificador del commit del código fuente desde el cual comencé a trabajar es fa2ddcb7ca87a77af579380a6eafd5aff163b44f, obtenido con el comando git log -1 (Figura 2).

**Figura 2**

*Commit de Partida del Repositorio*

![Commit de Partida del Repositorio](docs/img/fig02-commit-partida.png)

*Nota.* Salida del comando git log -1 dentro de la carpeta del repositorio xv6-riscv.

### Herramientas para Compilar y Ejecutar xv6

Según la sección “Building and running xv6” del archivo README del repositorio oficial (MIT PDOS, s.f.), las herramientas necesarias son:

> You will need a RISC-V “newlib” tool chain from https://github.com/riscv/riscv-gnu-toolchain, and qemu compiled for riscv64-softmmu. Once they are installed, and in your shell search path, you can run “make qemu”. (MIT PDOS, s.f.)

Es decir, se necesita el toolchain de RISC-V (compilador gcc y binutils), que traduce el código C a instrucciones para RISC-V; el emulador QEMU, que simula una computadora RISC-V donde arranca xv6; y make, que ejecuta el comando make qemu. Antes de la instalación, Cloud Shell solo tenía make (Figura 3). Las herramientas se instalaron con el comando:

    sudo apt update && sudo apt install -y qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu

**Figura 3**

*Herramientas Antes de la Instalación*

![Herramientas Antes de la Instalación](docs/img/fig03-herramientas-antes.png)

*Nota.* Antes de instalar los paquetes, el compilador, el enlazador y QEMU no estaban disponibles (command not found); solo make 4.3 estaba instalado.

La Tabla 1 resume las versiones de las herramientas utilizadas, tomadas de la Figura 4.

**Tabla 1**

*Versiones de las Herramientas Utilizadas*

| **Herramienta**            | **Comando**                     | **Versión** |
|----------------------------|---------------------------------|-------------|
| Compilador GCC para RISC-V | riscv64-linux-gnu-gcc --version | 13.3.0      |
| Enlazador (GNU Binutils)   | riscv64-linux-gnu-ld --version  | 2.42        |
| Emulador QEMU              | qemu-system-riscv64 --version   | 8.2.2       |
| GNU Make                   | make --version                  | 4.3         |

**Figura 4**

*Versiones de las Herramientas Instaladas*

![Versiones de las Herramientas Instaladas](docs/img/fig04-versiones-herramientas.png)

*Nota.* Salida de los comandos de versión después de la instalación.

## Compilación y Ejecución de xv6

Para compilar y ejecutar xv6 se usó el comando make qemu, que compila el kernel y los programas de usuario, construye el disco del sistema de archivos (fs.img) y arranca xv6 en QEMU. La Figura 5 muestra el final de la compilación y el arranque del sistema, y la Figura 6 muestra los comandos ls y cat README ejecutados dentro de xv6.

**Figura 5**

*Compilación y Arranque de xv6*

![Compilación y Arranque de xv6](docs/img/fig05-compilacion-arranque.png)

*Nota.* Se observa el enlace de los últimos programas de usuario, la creación de fs.img, el comando con el que se lanza QEMU, el mensaje xv6 kernel is booting y el prompt \$ de la shell de xv6.

**Figura 6**

*Comandos ls y cat Ejecutados en xv6*

![Comandos ls y cat Ejecutados en xv6](docs/img/fig06-ls-cat.png)

*Nota.* El comando ls lista los archivos y programas del sistema de archivos de xv6, y cat README muestra el contenido del archivo README, lo que demuestra que xv6 está operativo.

### Instrucciones Reproducibles y Ejecución con una Sola CPU Virtual

Para reproducir el entorno desde cero en Ubuntu 24.04 (o en Google Cloud Shell) se ejecutan los siguientes comandos. Para salir de QEMU se presiona Ctrl+A y luego X.

    sudo apt update && sudo apt install -y qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
    git clone https://github.com/[usuario]/xv6-riscv.git
    cd xv6-riscv
    git checkout fa2ddcb7ca87a77af579380a6eafd5aff163b44f
    make qemu CPUS=1

El número de CPU virtuales lo controla la variable CPUS del Makefile. Si no se indica otro valor al invocar make, vale 3, y se le entrega a QEMU con la opción -smp, que fija cuántos núcleos (harts, en la terminología de RISC-V) tiene la máquina emulada:

    ifndef CPUS
    CPUS := 3
    endif
     
    QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic

*Nota.* Fragmento del Makefile, líneas 174 a 178, del commit fa2ddcb (MIT PDOS, s.f.). **\[Confirmar que el texto coincide con su Makefile.\]**

Como ifndef solo asigna el valor 3 cuando CPUS no está definida, el comando make qemu CPUS=1 define la variable desde la línea de comandos y QEMU arranca con -smp 1, sin modificar el Makefile. Con una sola CPU virtual solo la CPU 0 ejecuta scheduler(), así que en cada instante hay como máximo un proceso en estado RUNNING. Eso hace visible el turno entre procesos y es la configuración que se usa en las pruebas de la sección siguiente.

Durante el arranque, main() imprime el mensaje hart N starting por cada CPU distinta de la 0. Por eso, con 3 CPU aparecen los mensajes hart 1 starting y hart 2 starting (Figura 9), y con una sola CPU no aparece ninguno (Figura 10).

**Figura 9**

*Arranque de xv6 con 3 CPU Virtuales*

**\[Insertar captura: salida de make qemu, desde el comando de QEMU con -smp 3 hasta el prompt \$\]**

*Nota.* Con el valor por defecto CPUS := 3, QEMU recibe -smp 3 y aparecen los mensajes hart 1 starting y hart 2 starting.

**Figura 10**

*Arranque de xv6 con una Sola CPU Virtual*

**\[Insertar captura: salida de make qemu CPUS=1, desde el comando de QEMU con -smp 1 hasta el prompt \$\]**

*Nota.* Con CPUS=1, QEMU recibe -smp 1 y no aparece ningún mensaje hart N starting.

## Investigación del Planificador Original

### Algoritmo de Planificación que Utiliza xv6

Según el libro *xv6: A Simple, Unix-Like Teaching Operating System* (Cox et al., s.f.), en la sección 8.6 se explica el algoritmo de planificación usado por xv6, citando lo siguiente:

> The xv6 scheduler implements a simple scheduling policy that runs each process in turn. This policy is called round robin. Real operating systems implement more sophisticated policies that, for example, allow processes to have priorities. The idea is that a runnable high-priority process will be preferred by the scheduler over a runnable low-priority process. These policies can become complex because there are often competing goals: for example, the operating system might also want to guarantee fairness and high throughput. (Cox et al., s.f., p. 79)

Y como lo revisamos en clase, round robin es un algoritmo de planificación de procesos expropiativo, en el cual el planificador del sistema operativo le asigna un quantum de tiempo a cada proceso y, una vez el proceso consume todo el quantum, el planificador le quita la CPU y permite que otro proceso la utilice.

**Evidencia.** En mi caso, la herramienta de IA me mostró en el kernel del sistema operativo la función de planificación de procesos scheduler(), explicando su funcionamiento y cómo esta trabaja con el algoritmo de planificación round robin. A continuación adjunto el código original (Figura 7) y su documentación traducida (Figura 8), lo cual me permitió comprenderlo.

**Figura 7**

*Funciones scheduler(), sched() y yield() en kernel/proc.c (Código Original)*

    // Per-CPU process scheduler.
    // Each CPU calls scheduler() after setting itself up.
    // Scheduler never returns.  It loops, doing:
    //  - choose a process to run.
    //  - swtch to start running that process.
    //  - eventually that process transfers control
    //    via swtch back to the scheduler.
    void
    scheduler(void)
    {
      struct proc *p;
      struct cpu *c = mycpu();
     
      c->proc = 0;
      for (;;) {
        // The most recent process to run may have had interrupts
        // turned off; enable them to avoid a deadlock if all
        // processes are waiting. Then turn them back off
        // to avoid a possible race between an interrupt
        // and wfi.
        intr_on();
        intr_off();
     
        int found = 0;
        for (p = proc; p < &proc[NPROC]; p++) {
          acquire(&p->lock);
          if (p->state == RUNNABLE) {
            // Switch to chosen process.  It is the process's job
            // to release its lock and then reacquire it
            // before jumping back to us.
            p->state = RUNNING;
            c->proc = p;
            swtch(&c->context, &p->context);
     
            // Don't re-enable interrupts on release.
            mycpu()->intena = 0;
     
            // Process is done running for now.
            // It should have changed its p->state before coming back.
            c->proc = 0;
            found = 1;
          }
          release(&p->lock);
        }
        if (found == 0) {
          // nothing to run; stop running on this core until an interrupt.
          asm volatile("wfi");
        }
      }
    }
     
    // Switch to scheduler.  Must hold only p->lock
    // and have changed proc->state. Saves and restores
    // intena because intena is a property of this
    // kernel thread, not this CPU. It should
    // be proc->intena and proc->noff, but that would
    // break in the few places where a lock is held but
    // there's no process.
    void
    sched(void)
    {
      int intena;
      struct proc *p = myproc();
     
      if (!holding(&p->lock))
        panic("sched p->lock");
      if (mycpu()->noff != 1)
        panic("sched locks");
      if (p->state == RUNNING)
        panic("sched RUNNING");
      if (intr_get())
        panic("sched interruptible");
     
      intena = mycpu()->intena;
      swtch(&p->context, &mycpu()->context);
      mycpu()->intena = intena;
    }
     
    // Give up the CPU for one scheduling round.
    void
    yield(void)
    {
      struct proc *p = myproc();
      acquire(&p->lock);
      p->state = RUNNABLE;
      sched();
      release(&p->lock);
    }

*Nota.* Fragmento de kernel/proc.c, líneas 421 a 509, del commit fa2ddcb (MIT PDOS, s.f.).

**Figura 8**

*Funciones scheduler(), sched() y yield() con los Comentarios Traducidos al Español*

    // Planificador de procesos, uno por CPU.
    // Cada CPU llama a scheduler() después de configurarse.
    // El planificador nunca retorna. Repite en ciclo:
    //  - elegir un proceso para ejecutar.
    //  - hacer swtch para empezar a ejecutar ese proceso.
    //  - en algún momento, ese proceso devuelve el control
    //    al planificador mediante swtch.
    void
    scheduler(void)
    {
      struct proc *p;
      struct cpu *c = mycpu();
     
      c->proc = 0;
      for (;;) {
        // El último proceso que se ejecutó pudo haber dejado las
        // interrupciones desactivadas; se activan para evitar un
        // bloqueo total (deadlock) si todos los procesos están esperando.
        // Luego se desactivan otra vez para evitar una posible
        // condición de carrera entre una interrupción y la instrucción wfi.
        intr_on();
        intr_off();
     
        int found = 0;
        for (p = proc; p < &proc[NPROC]; p++) {
          acquire(&p->lock);
          if (p->state == RUNNABLE) {
            // Cambiar al proceso elegido. Es responsabilidad del
            // proceso liberar su candado (lock) y luego volver a
            // tomarlo antes de regresar aquí.
            p->state = RUNNING;
            c->proc = p;
            swtch(&c->context, &p->context);
     
            // No volver a activar las interrupciones al liberar el candado.
            mycpu()->intena = 0;
     
            // El proceso terminó de ejecutarse por ahora.
            // Debió haber cambiado su estado (p->state) antes de regresar.
            c->proc = 0;
            found = 1;
          }
          release(&p->lock);
        }
        if (found == 0) {
          // No hay nada que ejecutar; detener este núcleo
          // hasta que llegue una interrupción.
          asm volatile("wfi");
        }
      }
    }
     
    // Cambiar al planificador. Se debe tener tomado únicamente
    // el candado p->lock y ya se debe haber cambiado proc->state.
    // Guarda y restaura intena porque intena es una propiedad de
    // este hilo del kernel, no de esta CPU. Debería ser
    // proc->intena y proc->noff, pero eso fallaría en los pocos
    // lugares donde se tiene un candado tomado pero no hay
    // ningún proceso.
    void
    sched(void)
    {
      int intena;
      struct proc *p = myproc();
     
      if (!holding(&p->lock))
        panic("sched p->lock");
      if (mycpu()->noff != 1)
        panic("sched locks");
      if (p->state == RUNNING)
        panic("sched RUNNING");
      if (intr_get())
        panic("sched interruptible");
     
      intena = mycpu()->intena;
      swtch(&p->context, &mycpu()->context);
      mycpu()->intena = intena;
    }
     
    // Ceder la CPU durante una ronda de planificación.
    void
    yield(void)
    {
      struct proc *p = myproc();
      acquire(&p->lock);
      p->state = RUNNABLE;
      sched();
      release(&p->lock);
    }

*Nota.* Traducción propia de los comentarios del código de la Figura 7. El código no fue modificado.

**Evidencia en el comportamiento.** El código muestra el mecanismo; las dos pruebas siguientes muestran su efecto. Ambas se ejecutan con una sola CPU virtual (make qemu CPUS=1), para que el reparto de la CPU no se confunda con paralelismo real entre núcleos.

En la primera prueba, usertests & lanza la batería de pruebas de xv6 en segundo plano y, sin esperar a que termine, se ejecuta ls en primer plano. Si el planificador no expropiara, ls tendría que esperar a que usertests terminara, lo que tarda varios minutos. Con round robin, cada interrupción del timer obliga al proceso en ejecución a ceder la CPU, ls recibe turnos y su salida aparece intercalada con la de usertests (Figura 11).

En la segunda prueba, usertests preempt ejecuta solo la prueba preempt de user/usertests.c. Esa prueba crea procesos hijos que quedan en un ciclo infinito sin hacer llamadas al sistema y comprueba que el padre sigue recibiendo CPU para terminarlos con kill. Como esos hijos nunca ceden la CPU por su cuenta, la prueba solo puede pasar si la interrupción del timer los expropia (Figura 12).

**Figura 11**

*Salida Intercalada de usertests y ls con una Sola CPU*

**\[Insertar captura: ejecución de usertests & seguida de ls con CPUS=1\]**

*Nota.* La salida de ls aparece mezclada con la de usertests, aunque solo hay una CPU. **\[Describir lo observado.\]**

**Figura 12**

*Resultado de la Prueba preempt con una Sola CPU*

**\[Insertar captura: ejecución de usertests preempt con CPUS=1\]**

*Nota.* La prueba termina con OK, lo que indica que el timer expropió a los procesos en ciclo infinito. **\[Ajustar si el resultado fue distinto y analizar la causa.\]**

### Archivo y Función que Selecciona el Siguiente Proceso

El archivo y la función que implementan la operación del algoritmo de planificación, donde se hace la selección del siguiente proceso a entrar a la CPU, se encuentran en el kernel: el archivo es kernel/proc.c y la función es scheduler(). Esta función y la documentación de su código se anexaron en la respuesta anterior (Figuras 7 y 8).

### Relación con los Estados de los Procesos y los Eventos

Los estados posibles de un proceso están definidos en kernel/proc.h (línea 79):

    enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

La función scheduler() solo elige procesos que estén en estado RUNNABLE (listos) y los pasa a RUNNING (en ejecución) antes de entregarles la CPU (kernel/proc.c, línea 451):

    if (p->state == RUNNABLE) {
      p->state = RUNNING;
      c->proc = p;
      swtch(&c->context, &p->context);

El proceso que está en la CPU (estado RUNNING) debe cambiar su estado antes de devolver la CPU. Esto ocurre por tres eventos, y en los tres casos se llama a sched(), que hace swtch de vuelta al planificador (Tabla 2).

**Tabla 2**

*Eventos por los que el Proceso en Ejecución Cede la CPU*

| **Evento**                                                                     | **Función** | **Cambio de estado** | **Código**                                                      |
|--------------------------------------------------------------------------------|-------------|----------------------|-----------------------------------------------------------------|
| Se acaba el quantum (interrupción del timer)                                   | yield()     | RUNNING → RUNNABLE   | kernel/trap.c, líneas 85-86 y 157-158; kernel/proc.c, línea 505 |
| El proceso tiene que esperar algo (por ejemplo, leer del teclado o de un pipe) | sleep()     | RUNNING → SLEEPING   | kernel/proc.c, línea 568                                        |
| El proceso termina                                                             | kexit()     | RUNNING → ZOMBIE     | kernel/proc.c, línea 358                                        |

El proceso que entra a la CPU tiene que estar en estado RUNNABLE. Un proceso llega a ese estado por los cinco caminos de la Tabla 3, todos en kernel/proc.c. Ninguno lo coloca en una cola: solo cambia el campo p-\>state, y scheduler() lo encontrará la próxima vez que recorra el arreglo proc\[\].

**Tabla 3**

*Eventos por los que un Proceso Llega a RUNNABLE*

| **Evento**                                                                             | **Función** | **Cambio de estado** | **Archivo**   |
|----------------------------------------------------------------------------------------|-------------|----------------------|---------------|
| Se crea el primer proceso (init) durante el arranque                                   | userinit()  | USED → RUNNABLE      | kernel/proc.c |
| Un proceso crea un hijo                                                                | kfork()     | USED → RUNNABLE      | kernel/proc.c |
| Se acaba el quantum (interrupción del timer)                                           | yield()     | RUNNING → RUNNABLE   | kernel/proc.c |
| Ocurre lo que el proceso esperaba (dato disponible, hijo que termina, paso de un tick) | wakeup()    | SLEEPING → RUNNABLE  | kernel/proc.c |
| Se mata un proceso que estaba dormido                                                  | kkill()     | SLEEPING → RUNNABLE  | kernel/proc.c |

*Nota.* En el commit fa2ddcb, las funciones del kernel que implementan fork y kill se llaman kfork() y kkill(), igual que kexit() en la Tabla 2. **\[Verificar los nombres y agregar los números de línea, como en la Tabla 2.\]**

Así se cierra el ciclo de la Tabla 2: un proceso sale de RUNNING por yield(), sleep() o kexit(), y vuelve a ser candidato cuando otra parte del kernel lo pone en RUNNABLE. En el caso de kkill(), el proceso despierta solo para notar que fue marcado (p-\>killed) y terminar con kexit().

### Incorporación del Planificador en la Compilación de xv6

El planificador no es un programa aparte: scheduler() es una función del kernel y se compila con el resto. El Makefile incluye \$K/proc.o en la lista OBJS, junto con los demás módulos del kernel (entry.o, start.o, main.o, trap.o, entre otros). Al ejecutar make, el compilador cruzado riscv64-linux-gnu-gcc compila kernel/proc.c en kernel/proc.o, y luego el enlazador une todos los objetos de OBJS, siguiendo el guion kernel/kernel.ld, en un único ejecutable: kernel/kernel.

QEMU carga ese ejecutable con la opción -kernel \$K/kernel. La ejecución empieza en kernel/entry.S, pasa por start() en modo máquina y llega a main() en kernel/main.c. Allí la CPU 0 inicializa el sistema (memoria, tabla de procesos con procinit(), interrupciones y sistema de archivos) y crea el primer proceso con userinit(); las demás CPU esperan a que termine y luego hacen su propia inicialización. Al final de main(), cada CPU llama a scheduler(), que nunca retorna.

Esto tiene dos consecuencias. Primero, hay un ciclo de scheduler() por cada CPU virtual, y su número lo fija la variable CPUS. Segundo, cualquier cambio al planificador, como el que se propone en la sección siguiente, exige editar el código fuente en kernel/proc.c y volver a ejecutar make, porque el planificador forma parte del binario del kernel.

## Descripción de MLFQ y Exploración de su Incorporación

Esta sección es una propuesta: no se implementa MLFQ ni se construye un mecanismo para elegir entre planificadores. MLFQ reemplazaría a round robin como único planificador de xv6.

### Descripción Inicial

Descripción escrita antes de la conversación con la IA:

> MLFQ (Multi-Level Feedback Queue) es un algoritmo de planificación de procesos que organiza los procesos en varias colas con distinta prioridad. Existe para evitar los problemas de otros algoritmos: SJF necesita saber cuánto va a durar cada proceso, algo que el sistema normalmente no sabe, y Round Robin trata igual a todos los procesos, lo que puede afectar a los procesos interactivos.

### Prompt Inicial

La conversación con la IA empezó con el siguiente prompt, tomado del enunciado de la tarea. Por eso habla de una pareja, aunque el trabajo lo realicé solo:

> Actúa como tutor socrático de una pareja de estudiantes de cuarto semestre de Ingeniería de Sistemas. Conocemos C++ y hemos comenzado a explorar XV6 para RISC-V. Queremos identificar y justificar qué se necesitaría para incorporar MLFQ a XV6. En esta entrega no vamos a implementarlo.

### Descripción Revisada

MLFQ organiza los procesos listos en varias colas de prioridad y mueve a cada proceso entre colas según el uso de CPU que se le observa; esa observación es el feedback del nombre. Así resuelve los dos problemas de la descripción inicial: aproxima la duración que SJF necesita conocer con el comportamiento reciente del proceso (quien agota su quantum probablemente es de cómputo largo) y, a diferencia de round robin, deja arriba a los procesos que usan poca CPU antes de bloquearse, que suelen ser los interactivos. La variante elegida usa los parámetros de la Tabla 4, medidos en ticks del timer porque es la única medida de tiempo que el kernel ya registra.

**Tabla 4**

*Parámetros de la Variante de MLFQ Propuesta*

| **Parámetro**        | **Valor**                  | **Justificación**                                                                         |
|----------------------|----------------------------|-------------------------------------------------------------------------------------------|
| Número de colas      | 3 (0 = máxima, 2 = mínima) | Separa procesos interactivos, intermedios y de cómputo, y es fácil de observar en pruebas |
| Quantum de la cola 0 | 1 tick                     | Respuesta rápida para procesos interactivos                                               |
| Quantum de la cola 1 | 2 ticks                    | Nivel intermedio                                                                          |
| Quantum de la cola 2 | 4 ticks                    | Menos cambios de contexto para procesos de cómputo                                        |
| Periodo de boost (S) | 50 ticks                   | Unos 5 s en QEMU (1 tick ≈ 0,1 s); evita la inanición sin anular la diferenciación        |

La variante se rige por seis reglas:

- **Regla 1. Prioridad entre colas.** Si la prioridad de A es mayor que la de B, corre A.

- **Regla 2. Round robin dentro de una cola.** Los procesos de un mismo nivel se turnan en orden de llegada a la cola (FIFO), cada uno con el quantum de ese nivel.

- **Regla 3. Entrada.** Todo proceso nuevo, incluido el hijo de un fork, entra en la cola 0 con su contador de uso en cero.

- **Regla 4. Descenso por consumo acumulado.** Cada proceso acumula los ticks que consume en su nivel. Cuando el acumulado llega al quantum del nivel, baja un nivel (si ya está en la cola 2, se queda ahí) y el contador vuelve a cero. Bloquearse por I/O o por sleep no reinicia el contador.

- **Regla 5. Boost periódico.** Cada S = 50 ticks, todos los procesos vuelven a la cola 0 con el contador en cero.

- **Regla 6. Expropiación por prioridad.** Si mientras corre un proceso de nivel k aparece un proceso listo de nivel menor que k, el que corre cede la CPU en el siguiente tick, aunque no haya agotado su quantum. No baja de nivel por eso.

La regla 6 es necesaria porque, con quantums de 2 y 4 ticks, un proceso interactivo podría esperar hasta 4 ticks detrás de uno de cómputo. La expropiación ocurre con granularidad de tick y no al instante, porque xv6 solo vuelve a decidir en la interrupción del timer o cuando un proceso se bloquea.

### Lo Aprendido

La descripción inicial explicaba por qué existe MLFQ, pero no cómo funciona: decía que hay colas con distinta prioridad sin explicar cómo llega un proceso a una cola ni cuándo cambia de cola. El feedback es justamente ese mecanismo: el sistema no conoce de antemano la duración de un proceso, pero sí puede observar cuánta CPU consume. Quien agota su quantum baja de prioridad; quien se bloquea antes, se queda arriba.

La revisión también mostró que una regla ingenua se puede engañar. Si dormir reiniciara el contador, un proceso podría ceder la CPU justo antes de agotar su quantum y quedarse siempre en la cola más alta; por eso el contador se acumula (regla 4). Si la cola más alta nunca se vacía, los procesos de abajo nunca corren; por eso existe el boost (regla 5). Por último, la comprobación con un caso concreto mostró que el orden entre cobrar el tick y aplicar el boost cambia el resultado.

### Propuesta Justificada de Cambios

El análisis del planificador original muestra que xv6 no tiene colas, prioridades ni registro del uso de CPU por proceso. Para soportar la variante propuesta se necesitan los nueve cambios de la Tabla 5; cada uno se justifica con una regla y con el código actual.

**Tabla 5**

*Cambios Necesarios en xv6 para Soportar MLFQ*

| **\#** | **Cambio**                                                                          | **Archivo**                                                               | **Reglas** |
|--------|-------------------------------------------------------------------------------------|---------------------------------------------------------------------------|------------|
| B1     | Constantes del algoritmo: NMLFQ 3, BOOST_TICKS 50 y quantums 1, 2 y 4               | kernel/param.h                                                            | 1, 4, 5    |
| B2     | Campos nuevos en struct proc: level, ticks_used, enq_seq y contadores para pruebas  | kernel/proc.h                                                             | 1, 2, 4    |
| B3     | Inicializar esos campos al crear el proceso                                         | kernel/proc.c (allocproc)                                                 | 3          |
| B4     | Función única make_runnable() para pasar a RUNNABLE y registrar el orden de llegada | kernel/proc.c                                                             | 2          |
| B5     | Cobrar cada tick y decidir si el proceso cede la CPU (mlfq_tick())                  | kernel/trap.c, kernel/proc.c                                              | 4, 6       |
| B6     | No reiniciar el contador al dormir                                                  | kernel/proc.c (sleep, wakeup)                                             | 4          |
| B7     | Nuevo criterio de selección: menor nivel y, en empate, el más antiguo en la cola    | kernel/proc.c (scheduler)                                                 | 1, 2       |
| B8     | Boost periódico después del cobro del tick (mlfq_maybe_boost())                     | kernel/trap.c, kernel/proc.c                                              | 5          |
| B9     | Observabilidad: procdump() ampliado y una syscall getmlfqinfo()                     | kernel/proc.c, syscall.h, syscall.c, sysproc.c, user/user.h, user/usys.pl | todas      |

**B1. Constantes.** kernel/param.h solo define límites de recursos (NPROC, NCPU, NOFILE, entre otros) y ningún parámetro de planificación. Se agregan el número de colas, el periodo de boost y los quantums por nivel, centralizados para poder cambiar la variante sin tocar la lógica.

**B2. Estado por proceso.** struct proc en kernel/proc.h guarda estado, canal de espera, PID, padre, memoria, contexto y archivos, pero no prioridad ni uso de CPU, así que las reglas 1, 2 y 4 no se pueden evaluar. Se agregan level (nivel actual), ticks_used (ticks consumidos en ese nivel), enq_seq (número de llegada a la cola, para el orden FIFO) y, para las pruebas, ticks_total y ticks_level\[NMLFQ\]. Las colas se representan con level más el orden de llegada, no con listas enlazadas: con NPROC = 64 recorrer el arreglo cuesta poco y se evita mantener listas bajo los candados de varias CPU.

**B3. Inicialización.** allocproc() reutiliza una ranura UNUSED de proc\[\] y hoy no inicializa campos de planificación. Como las ranuras se reciclan, un proceso nuevo heredaría el nivel del anterior. Se inicializan level y ticks_used en 0; como kfork() y userinit() usan allocproc(), la regla 3 se cumple para todos los procesos.

**B4. Orden de llegada.** El kernel asigna RUNNABLE en los cinco lugares de la Tabla 3, y ninguno registra cuándo entró el proceso a la cola. Se reemplazan por make_runnable(p), que cambia el estado y asigna enq_seq desde un contador global atómico. Así, un proceso que agota su quantum o despierta queda al final de su cola.

**B5. Cobro del tick.** En usertrap() y kerneltrap() (kernel/trap.c, líneas 85-86 y 157-158; Tabla 2), toda interrupción del timer provoca un yield() sin condición. Ese if es el que fija el quantum en 1 tick para todos los procesos.

    // give up the CPU if this is a timer interrupt.
    if(which_dev == 2)
      yield();

*Nota.* Fragmento de usertrap() en kernel/trap.c; en kerneltrap() la condición incluye además myproc() != 0.

Se reemplaza por un bloque que primero cobra el tick, después aplica el boost si corresponde (B8) y al final cede la CPU si hace falta. mlfq_tick() incrementa los contadores del proceso en ejecución; si ticks_used llega al quantum del nivel, lo baja un nivel (si no está en el último), reinicia el contador y devuelve 1 (regla 4); si no, devuelve 1 solo si existe un proceso RUNNABLE de nivel menor (regla 6). Se cobra también en kerneltrap(), porque el tiempo dentro de una llamada al sistema es CPU consumida por el proceso.

**B6. Dormir no reinicia.** sleep() solo guarda el canal, pone SLEEPING y llama a sched(). No se le agrega nada: wakeup() usa make_runnable() y conserva level y ticks_used. Si dormir reiniciara el contador, un proceso que se bloquea justo antes de agotar su quantum nunca bajaría de la cola 0.

**B7. Selección.** El ciclo de scheduler() (Figura 7) ejecuta cada proceso RUNNABLE en orden de índice y, al volver de swtch(), sigue con p + 1 aunque haya un proceso más prioritario al principio del arreglo; ese recorrido circular es el round robin. Se cambia por: recorrer proc\[\] y elegir el RUNNABLE de menor level y, en empate, menor enq_seq; tomar su candado y volver a comprobar que sigue RUNNABLE, porque con varias CPU otra pudo tomarlo entre el recorrido y el acquire(); ejecutar swtch() y, al volver, empezar la búsqueda desde el principio. Si no hay procesos listos se conserva la espera con wfi.

**B8. Boost.** El único reloj global es la variable ticks, que clockintr() incrementa en la CPU 0. Se agrega mlfq_maybe_boost(), que actúa solo en la CPU 0 cuando han pasado BOOST_TICKS desde el último boost y pone level = 0 y ticks_used = 0 en todos los procesos. Se llama después de mlfq_tick() y no dentro de clockintr(), que corre antes del cobro: de lo contrario, el proceso en ejecución subiría a la cola 0 y bajaría a la 1 en el mismo tick sin haber corrido en la cola 0 (ver la comprobación con un caso concreto).

**B9. Observabilidad.** procdump() (Ctrl+P) solo imprime PID, estado y nombre. Se amplía para mostrar level, ticks_used y ticks_total, y se agrega una syscall getmlfqinfo(pid, info) que copia esos datos a espacio de usuario con copyout(). No es parte del algoritmo, pero sin ella los escenarios de comprobación no tendrían nada que medir.

### Escenarios de Comprobación Futura

Cada escenario de la Tabla 6 prueba al menos una regla y tiene un resultado medible con getmlfqinfo(), uptime() o Ctrl+P. Se ejecutarían primero con make qemu CPUS=1, porque con varias CPU los procesos corren en paralelo y el orden deja de ser comparable con las reglas; después se repetirían C4 y C9 con CPUS=3 para buscar condiciones de carrera. Se necesitan tres programas de usuario: cpubound (un ciclo de cómputo sin llamadas al sistema), iobound (muy poco trabajo y sleep(1) en un ciclo) y tramposo (espera activa hasta que uptime() avanza un tick y luego sleep(1)).

**Tabla 6**

*Escenarios para Comprobar una Futura Implementación de MLFQ*

| **\#** | **Escenario**                                                                                    | **Reglas** | **Resultado esperado**                                                                                                   |
|--------|--------------------------------------------------------------------------------------------------|------------|--------------------------------------------------------------------------------------------------------------------------|
| C1     | Un proceso en la cola 2 hace fork(); el hijo consulta su nivel de inmediato                      | 3          | El hijo está en el nivel 0 con ticks_used = 0                                                                            |
| C2     | Un solo cpubound registra su nivel cada vez que cambia uptime()                                  | 4          | Secuencia 0 → 1 → 2: baja tras 1 tick en la cola 0 y tras 2 más en la cola 1; se queda en 2 hasta el boost               |
| C3     | Dos cpubound y un iobound durante 200 ticks                                                      | 1, 4       | iobound pasa casi todo el tiempo en los niveles 0 y 1; los cpubound, en el nivel 2, y acumulan muchos más ticks          |
| C4     | Tres cpubound idénticos durante 300 ticks                                                        | 2          | ticks_total parecido en los tres (diferencia de no más del 10 %)                                                         |
| C5     | Un tramposo junto a dos cpubound                                                                 | 4          | tramposo llega al nivel 2 pocos ciclos después de cada boost; si el contador se reiniciara al dormir, se quedaría arriba |
| C6     | Un cpubound víctima en el nivel 2 y un generador que crea sin parar hijos cortos                 | 5          | El ticks_total de la víctima aumenta en cada intervalo de 50 ticks; sin boost quedaría constante                         |
| C7     | Un proceso de cómputo durante 100 ticks que después se comporta como iobound                     | 5          | Tras el siguiente boost vuelve al nivel 0 y se mantiene en los niveles altos                                             |
| C8     | Un cpubound en el nivel 2 y un proceso que hace sleep(10) y mide uptime() antes y después        | 6          | El proceso que duerme vuelve a correr como máximo 1 tick después de despertar                                            |
| C9     | Regresión: usertests y forktest con CPUS=1 y CPUS=3; kill en cada nivel; Ctrl+P durante la carga | Todas      | Sin panic; ningún proceso en RUNNING en dos CPU a la vez; kill funciona en cualquier nivel                               |

Los umbrales de C3 y C4 son orientativos: lo importante es una diferencia clara entre grupos de procesos, porque la planificación por ticks y las interrupciones introducen variación.

### Cuestiones Pendientes de Investigación

Las cuestiones de la Tabla 7 no invalidan la propuesta, pero deben resolverse antes de implementarla o durante la calibración.

**Tabla 7**

*Cuestiones Pendientes de Investigación*

| **\#** | **Cuestión**                                                                                       | **Por qué importa**                                                                           | **Cómo investigarla**                                                                                 |
|--------|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| P1     | ¿Medir el uso de CPU por debajo del tick (por ejemplo, con r_time() al entrar y salir de swtch())? | Un proceso que se bloquea siempre antes del tick nunca acumula uso y la regla 4 no lo detecta | Revisar cómo se programa el timer y el costo de leer el contador de tiempo en cada cambio de contexto |
| P2     | ¿El hijo de fork() debe entrar en la cola 0 o heredar el nivel del padre?                          | Un proceso podría repartir su trabajo en hijos nuevos para correr siempre en la cola 0        | Comparar con Arpaci-Dusseau y Arpaci-Dusseau (2018, cap. 8) y observar el generador de C6             |
| P3     | ¿Recorrer proc\[\] o mantener una lista por nivel?                                                 | Afecta el costo de cada decisión y el diseño de candados                                      | Contar operaciones con NPROC = 64 y analizar qué candado global exigirían las listas                  |
| P4     | ¿Son adecuados los quantums 1, 2 y 4 y S = 50?                                                     | Con S pequeño todo se parece a round robin; con S grande hay inanición prolongada             | Repetir C3, C5 y C6 con varios valores                                                                |
| P5     | ¿Cuánto dura realmente un tick en el commit fa2ddcb?                                               | Los parámetros están en ticks; los 0,1 s salen de un comentario del código                    | Ubicar el intervalo del timer y medir con uptime() contra un reloj externo                            |
| P6     | Con varias CPU, ¿la regla 6 provoca expropiaciones innecesarias?                                   | Si hay otra CPU libre, el proceso prioritario correría allí sin expropiar a nadie             | Analizar el caso con CPUS=3                                                                           |
| P7     | Con varias CPU, ¿qué significa hacer boost a un proceso que corre en otra CPU?                     | El boost lo hace la CPU 0 sobre procesos en ejecución en otras                                | Verificar que p-\>lock serializa el boost con mlfq_tick()                                             |
| P8     | ¿Deben init y sh tener un trato especial?                                                          | Casi siempre están dormidos                                                                   | Observarlos con Ctrl+P durante los escenarios                                                         |

### Conclusiones

**xv6 es hoy round robin con quantum de 1 tick y sin estado de planificación.** Lo muestran el yield() sin condición ante cada interrupción del timer en kernel/trap.c y el recorrido en orden de índice de scheduler(); struct proc no guarda prioridad ni uso de CPU.

**MLFQ necesita cuatro capacidades que xv6 no tiene:** estado de planificación por proceso (B1 a B4), medición del uso de CPU (B5 y B6), selección por prioridad (B7) y una acción periódica sobre todos los procesos (B8). B9 no es parte del algoritmo, pero sin ella no se puede verificar.

**Los cambios se concentran en cuatro archivos del kernel:** param.h, proc.h, proc.c y trap.c. Solo la syscall de observación toca la interfaz de usuario; no hace falta modificar swtch(), la memoria virtual ni el sistema de archivos.

**La parte delicada es la contabilidad, no la selección:** cuándo se cobra un tick, por qué dormir no reinicia el contador y en qué orden se aplican el cobro y el boost. Ahí están las trampas posibles y la inconsistencia que encontró la comprobación con un caso concreto.

**Con varias CPU aparecen condiciones de carrera** que el algoritmo en abstracto no tiene; la principal es que dos CPU elijan el mismo proceso, y por eso B7 vuelve a comprobar el estado bajo el candado.

**Cada regla tiene al menos un escenario de comprobación** que distingue una implementación correcta de una incorrecta. Las cuestiones P1 a P8 afectan sobre todo los parámetros y los casos con varias CPU, no la estructura de la propuesta.

### Comprobación de Consistencia con un Caso Concreto

**Pregunta.** Con CPUS=1, en el tick 0 se crean dos procesos, primero A y luego B. A es de cómputo puro; B trabaja menos de un tick y llama a sleep(1), en un ciclo. ¿Qué proceso corre después de cada tick, en qué nivel queda cada uno y qué regla lo explica?

La Tabla 8 muestra la traza. «Nivel / uso» es level / ticks_used después de procesar la interrupción; sleep(1) despierta a B en el tick siguiente, dentro de clockintr(), antes de que se evalúe mlfq_tick().

**Tabla 8**

*Traza Tick a Tick de un Proceso de Cómputo (A) y uno Interactivo (B)*

| **Tick** | **Qué pasa**                                                              | **A**     | **B** | **Corre después**       | **Reglas** |
|----------|---------------------------------------------------------------------------|-----------|-------|-------------------------|------------|
| 0        | Se crean A y B en la cola 0, A primero                                    | 0 / 0     | 0 / 0 | A                       | 3, 2       |
| 1        | Se cobra el tick a A, que agota el quantum de la cola 0                   | 1 / 0     | 0 / 0 | B; cuando B duerme, A   | 4, 1       |
| 2        | B despierta; se cobra a A (1 de 2) y hay un proceso más prioritario listo | 1 / 1     | 0 / 0 | B, luego A              | 6          |
| 3        | B despierta; A agota el quantum de la cola 1                              | 2 / 0     | 0 / 0 | B, luego A              | 4, 6       |
| 4 a 6    | B despierta en cada tick; A acumula 1, 2 y 3 de 4                         | 2 / 1 a 3 | 0 / 0 | B, luego A              | 6          |
| 7        | A agota el quantum de la cola 2 y se queda en ella                        | 2 / 0     | 0 / 0 | B, luego A              | 4          |
| 50       | Se cobra el tick a A en el nivel 2 y después se aplica el boost           | 0 / 0     | 0 / 0 | B, luego A en la cola 0 | 5, 2       |
| 51       | A agota el quantum de la cola 0                                           | 1 / 0     | 0 / 0 | B, luego A              | 4          |

**Respuesta.** A baja de la cola 0 a la 2 en los ticks 1 y 3 y se queda en la 2 hasta el boost. B nunca baja, y cada vez que despierta corre en el mismo tick. Cada paso se explica con una regla y un cambio de la propuesta, y la secuencia de A coincide con lo que espera el escenario C2.

**Hallazgo 1: inconsistencia corregida.** En el primer borrador, el boost se aplicaba dentro de clockintr(), que corre antes de mlfq_tick(). En el tick 50, A habría subido al nivel 0 y enseguida se le habría cobrado ese mismo tick, así que pasaría al nivel 1 sin haber corrido en la cola 0, lo que contradice la regla 5. Por eso B8 aplica el boost después del cobro.

**Hallazgo 2: límite de la propuesta.** B nunca acumula uso porque no está corriendo cuando llega el tick. Aquí es lo deseado, porque B es interactivo, pero un proceso que hiciera casi un tick de cómputo antes de dormir recibiría el mismo trato. Esa es la cuestión P1.

## Evidencia del Aprendizaje con IA

**Servicio y modelo.** Se usó Claude, de Anthropic, en claude.ai, con el modelo Claude Opus 5.5 (Anthropic, 2026), configurado mediante el prompt inicial como tutor socrático. **\[Si la conversación en la que la IA explicó scheduler() fue con otro servicio o modelo, indicarlo aquí.\]**

**Declaración de uso.** El diálogo socrático sobre MLFQ duró dos intercambios. Después le pedí a la IA que dejara de actuar como tutor, y la IA redactó un borrador de la descripción revisada, la propuesta de cambios, los escenarios, las cuestiones pendientes, las conclusiones y la comprobación con un caso concreto. Mi trabajo sobre ese borrador fue: **\[completar: qué revisé, qué verifiqué contra el commit fa2ddcb, qué corregí o descarté\].**

### Intercambio 1: Hipótesis y Corrección sobre el Planificador Original

**IA:** ¿Cómo decide hoy XV6 qué proceso corre a continuación? ¿En qué momento pierde un proceso la CPU?

**Estudiante (hipótesis):** «el planificador original esta en un archivo llamado proc.c en el sistema compilado este algoritmo funciona bajo round robin».

**IA (corrección):** El planificador no está en el sistema compilado sino en el código fuente del kernel, que se vuelve a compilar con make. Para justificar que es round robin hay que ubicar el quantum: ¿quién llama a yield() y en qué situación? Revisar kernel/trap.c.

**Comprobación:** en kernel/trap.c toda interrupción del timer llama a yield() sin condición, y scheduler() recorre proc\[\] en orden (Figura 7 y Tabla 2). Eso confirma la hipótesis de round robin con quantum de 1 tick y corrige la idea del «sistema compilado»: proc.c se compila dentro de kernel/kernel (sección de compilación).

### Intercambio 2: Corrección de la Descripción Inicial de MLFQ

**Estudiante:** la descripción inicial de MLFQ (ver Descripción Inicial).

**IA (corrección):** La descripción explica por qué existe MLFQ, pero no cómo funciona. Si el sistema no sabe cuánto va a durar un proceso, ¿qué podría observar mientras se ejecuta para «adivinarlo»? ¿Cuándo debería bajar de prioridad un proceso, y qué pasa entre dos procesos de la misma cola?

**Resultado:** la descripción revisada define el feedback como la observación del uso de CPU (regla 4) y agrega el orden dentro de una cola, la protección contra trampas y el boost (reglas 2, 4 y 5).

### Intercambio 3

**\[Pendiente: agregar un tercer intercambio con la pregunta de la IA y la respuesta propia, que muestre una hipótesis, una comprobación o una corrección. Una fuente posible es la conversación en la que la IA explicó scheduler() (ver Evidencia en la sección del planificador original).\]**

### Resultado Inesperado

La comprobación con un caso concreto encontró una inconsistencia en el primer borrador de la propuesta: aplicar el boost antes de cobrar el tick hacía que el proceso en ejecución nunca corriera en la cola 0 después del boost. Se corrigió en el cambio B8. La misma traza mostró un límite que sigue abierto (P1): un proceso que se bloquea siempre antes de que llegue el tick nunca acumula uso de CPU, de modo que la regla 4 no lo detecta.

## Referencias

Anthropic. (2026). *Claude* (Opus 5.5) \[Modelo de lenguaje de gran tamaño\]. https://claude.ai

Arpaci-Dusseau, R. H., & Arpaci-Dusseau, A. C. (2018). *Operating systems: Three easy pieces* (Versión 1.00). Arpaci-Dusseau Books. https://pages.cs.wisc.edu/~remzi/OSTEP/

Cox, R., Kaashoek, F., & Morris, R. (s.f.). *xv6: A simple, Unix-like teaching operating system* (Rev. 5). Massachusetts Institute of Technology. https://pdos.csail.mit.edu/6.828/2025/xv6/book-riscv-rev5.pdf

MIT PDOS. (s.f.). *xv6-riscv* (Commit fa2ddcb) \[Código fuente\]. GitHub. https://github.com/mit-pdos/xv6-riscv
