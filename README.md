# Auditoría técnica del sistema operativo de TechNova

## 1. Introducción
(pendiente)

## 2. Arquitectura del kernel

### Problemas del sistema actual

TechNova tiene dos problemas principales: cuellos de botella que impiden que el sistema siga escalando y cierres inesperados que afectan a su estabilidad. Los registros muestran que hay varios aspectos que revisar, pero no todos dependen de la arquitectura del kernel.

Los logs 1, 2 y 3 apuntan a errores del propio programa, relacionados con punteros, condiciones de carrera y posibles desbordamientos de pila. El log 4 muestra un agotamiento de memoria que ha obligado al sistema a terminar un proceso. Estos problemas requieren corregir el código y revisar la gestión de memoria; cambiar la arquitectura del kernel no los solucionaría por sí solo.

El log 5 sí está relacionado directamente con la arquitectura. El controlador de red `nova_nic` ha provocado un *kernel panic* al fallar mientras se ejecutaba en modo kernel. Esto demuestra el riesgo de que un componente defectuoso pueda afectar a todo el sistema.

Por tanto, el problema no es que los kernels monolíticos sean poco fiables por naturaleza. Linux utiliza una arquitectura monolítica modular y puede ofrecer un buen rendimiento y una gran estabilidad. En el caso de TechNova, hay que investigar tanto los cuellos de botella como la fiabilidad y compatibilidad de sus componentes.

### Comparación de las arquitecturas

Un **kernel monolítico**, como Linux, ejecuta en modo kernel gran parte de los servicios del sistema operativo, incluidos los controladores. La comunicación directa entre sus componentes permite un buen rendimiento, aunque un fallo grave en uno de ellos puede provocar la caída de todo el sistema.

Un **microkernel**, como QNX, mantiene en modo kernel solo las funciones esenciales y ejecuta otros servicios, como los controladores, en modo usuario. Esto permite aislar mejor los fallos y facilita que determinados servicios puedan reiniciarse sin detener todo el sistema. QNX se utiliza en sectores como la automoción y los sistemas médicos, donde la fiabilidad es especialmente importante. Su principal inconveniente es que la comunicación entre componentes puede añadir costes de rendimiento.

Una **arquitectura híbrida**, como la de Windows NT o macOS mediante XNU, combina características de ambos modelos. Mantiene en modo kernel los componentes que necesitan un acceso rápido y directo, mientras que otros pueden ejecutarse en modo usuario. Busca un equilibrio entre rendimiento y estabilidad, aunque el nivel de aislamiento depende de cómo esté diseñado el sistema y de dónde se ejecute cada componente.

### Recomendación para TechNova

Se recomienda una arquitectura híbrida, porque TechNova necesita procesar transacciones en tiempo real y reducir el riesgo de que un fallo provoque la caída completa del servidor.

La propuesta sería distribuir los componentes de la siguiente manera:

- **En modo kernel:** la gestión de memoria, la planificación de procesos y los controladores de red y disco que sean estables, estén verificados y ofrezcan las garantías necesarias de rendimiento.
- **En modo usuario:** los controladores de terceros que no estén verificados o cuya fiabilidad sea dudosa, además de los servicios no críticos que puedan aislarse y reiniciarse si fallan.

El controlador de red plantea una dificultad importante: es fundamental para el rendimiento porque por él pasan las transacciones, pero también es el responsable del fallo más grave detectado. Ejecutarlo en modo usuario podría mejorar el aislamiento, aunque añadiría costes de comunicación. Por eso, la mejor solución sería sustituir `nova_nic` por un controlador estable, compatible y suficientemente probado para ejecutarse en modo kernel. Mientras no se pueda garantizar su fiabilidad, convendría valorar su aislamiento en modo usuario, siempre que la plataforma lo permita sin comprometer el funcionamiento de la red.

Esta propuesta busca mantener la velocidad de los componentes esenciales y limitar el impacto de los fallos de aquellos que presenten más riesgos. Aun así, la arquitectura no resolverá los errores de programación ni el agotamiento de memoria identificados en los otros logs. También será necesario corregir esos problemas, actualizar el sistema y revisar los controladores para mejorar tanto el rendimiento como la estabilidad.

En conclusión, el objetivo no es cambiar de arquitectura simplemente porque el kernel sea monolítico, sino elegir una configuración que permita a TechNova seguir escalando, procesar las transacciones con rapidez y evitar que un fallo aislado termine afectando a todo el servidor.


## 3. Interfaz de system calls

### Cómo funciona una llamada al sistema

Una llamada al sistema es la forma que tiene un programa de solicitar servicios al kernel. Un programa que se ejecuta en modo usuario no puede acceder directamente al hardware ni realizar determinadas operaciones protegidas. Para leer un archivo, enviar datos por la red o solicitar recursos, necesita recurrir al sistema operativo.

El proceso funciona de la siguiente manera:

1. El programa solicita una operación mediante una función, como `read()`, para leer datos.
2. La biblioteca correspondiente prepara el número de la llamada al sistema y sus argumentos.
3. El procesador ejecuta una instrucción especial, como `syscall` en x86-64, y pasa de modo usuario a modo kernel.
4. El kernel comprueba los argumentos y los permisos, realiza la operación solicitada y prepara el resultado.
5. El procesador vuelve a modo usuario y el programa continúa su ejecución.

Este mecanismo permite que los programas utilicen los recursos del sistema sin tener acceso directo e ilimitado a ellos.

### Impacto en el rendimiento y la estabilidad

Las llamadas al sistema tienen un coste de ejecución. Cada entrada al kernel y vuelta al modo usuario implica un **cambio de modo**, además de guardar y recuperar parte del estado del procesador. Esto no significa que siempre se produzca un cambio de contexto, ya que el programa puede continuar ejecutándose después de la llamada sin ceder la CPU a otro proceso. Las medidas de seguridad del procesador y del sistema operativo también pueden añadir costes.

En TechNova, esto podría ser relevante si el servicio procesa muchas transacciones y realiza operaciones de entrada y salida de forma repetida. Agrupar varias operaciones en una sola llamada o reducir las copias de datos puede ayudar a disminuir ese coste y mejorar el rendimiento. No obstante, los logs disponibles no demuestran que las llamadas al sistema sean la causa directa de los cuellos de botella; sería necesario medirlas para confirmarlo.

Las llamadas al sistema también sirven como frontera de protección entre los programas y el kernel. Hay dos situaciones diferentes:

- **Dirección inválida en una llamada al sistema:** si el programa pasa una dirección de memoria inválida, el kernel puede detectarlo al comprobar los argumentos y devolver un error, como `EFAULT`, sin que el proceso tenga que terminar.
- **Acceso directo a una dirección inválida:** si el propio programa intenta acceder a una dirección que no tiene asignada correctamente, la MMU genera un *page fault*. Si el acceso es ilegal, el kernel envía `SIGSEGV` al proceso afectado.

Esto encaja con los logs 1, 2 y 3: los errores de memoria afectan al proceso o al hilo correspondiente, pero no provocan por sí mismos la caída completa del servidor. El log 4 muestra otra situación: el OOM killer termina un proceso para intentar proteger al resto del sistema. En cambio, el log 5 es diferente, porque el fallo se produce dentro de un controlador que se ejecuta en modo kernel y termina provocando un *kernel panic*.

### Propuesta de mejora

Para mejorar el rendimiento de TechNova, se recomienda utilizar interfaces compatibles con POSIX y reducir las llamadas al sistema innecesarias mediante varias técnicas:

- **Agrupar operaciones:** `readv()` y `writev()` permiten leer o escribir varios bloques de datos mediante una sola llamada al sistema, reduciendo el número de operaciones necesarias.
- **Reducir las copias de datos:** `mmap()` permite mapear archivos en el espacio de direcciones del proceso, mientras que `sendfile()` puede transferir datos de un archivo a un socket sin copiarlos primero a un búfer del programa.
- **Evitar entradas innecesarias al kernel:** en Linux, el mecanismo vDSO permite consultar determinados datos, como la hora, sin realizar una llamada al sistema tradicional en cada consulta.
- **Usar operaciones asíncronas:** `io_uring`, disponible en Linux, permite gestionar muchas operaciones de entrada y salida mediante colas compartidas, reduciendo la necesidad de realizar llamadas al sistema repetidas.

POSIX define una interfaz común para muchas funciones y operaciones de los sistemas operativos, lo que facilita desarrollar software compatible con diferentes sistemas. Sin embargo, es importante distinguir entre las funciones estándar y las específicas de cada plataforma. `readv()`, `writev()` y `mmap()` forman parte de POSIX, mientras que `sendfile()`, el vDSO e `io_uring` son tecnologías específicas de Linux o de su entorno. Por tanto, utilizar estas últimas puede requerir alternativas si TechNova decide migrar a otro sistema operativo.

Existe un compromiso entre rendimiento y portabilidad. Las interfaces POSIX facilitan el desarrollo de software multiplataforma, mientras que las funciones específicas de Linux permiten aprovechar mecanismos propios de ese sistema. Una opción sería mantener una interfaz común y utilizar optimizaciones específicas cuando realmente aporten una mejora.

Antes de realizar cambios, se recomienda medir el comportamiento del servicio. La herramienta `strace -c` permite resumir las llamadas al sistema realizadas por un proceso y el tiempo empleado en ellas, mientras que `perf` ayuda a identificar dónde se consume el tiempo de CPU. Con estos datos, TechNova podrá localizar los cuellos de botella y decidir qué optimizaciones son necesarias.

En conclusión, se recomienda mantener una interfaz POSIX para facilitar la compatibilidad entre sistemas y aplicar técnicas de optimización en las operaciones que realmente lo necesiten. Así se podrá mejorar el rendimiento sin comprometer la estabilidad ni limitar innecesariamente las futuras opciones de migración.


## 4. Diagnóstico de excepciones
## Log 1 — 09:00:47

**Hipótesis:**

El proceso ha intentado leer información desde la dirección `0x0`, que corresponde a un puntero nulo (`NULL`). Esto puede ocurrir si intenta utilizar un dato que no se ha inicializado correctamente o si una reserva de memoria ha fallado y no se ha comprobado el resultado. Durante los picos de carga, esta situación podría ser más frecuente si el programa necesita reservar más memoria de la habitual.

**Recomendación:** comprobar el resultado de las reservas de memoria y asegurarse de que los punteros sean válidos antes de utilizarlos.

## Log 2 — 09:01:22

**Hipótesis:**

El hilo `tx-worker-12` ha intentado escribir en la dirección `0x18`, situada a 24 bytes de la dirección cero. Esto podría indicar que intenta modificar un campo de una estructura mediante un puntero nulo. Otra posibilidad es una condición de carrera: al trabajar varios hilos al mismo tiempo, uno podría acceder a un dato que otro acaba de liberar o cuyo puntero ha puesto a `NULL`. Los picos de carga aumentan las posibilidades de que se produzcan estos conflictos si no existe una sincronización adecuada.

**Recomendación:** comprobar los punteros antes de utilizarlos y proteger los datos compartidos entre hilos mediante mecanismos de sincronización, como los *mutex*.

## Log 3 — 09:02:05

**Hipótesis:**

El hilo ha intentado escribir en una dirección situada justo debajo del límite de su pila. Cada hilo tiene una pila de tamaño limitado y una página de guarda debajo para detectar si se sale de ese espacio. El fallo podría deberse a un desbordamiento de pila (*stack overflow*), provocado por llamadas recursivas que no terminan o por funciones que utilizan demasiada memoria local.

**Recomendación:** revisar las llamadas recursivas y reducir el uso de la pila. Si el tamaño actual resulta insuficiente, se puede valorar aumentarlo para los hilos de trabajo.

## Log 4 — 09:03:31

**Hipótesis:**

El proceso ha sido terminado por el OOM killer del kernel porque el sistema se ha quedado sin memoria disponible para atender nuevas peticiones. El proceso tenía unos 30 GiB de memoria virtual y alrededor de 15 GiB de memoria anónima residente, una cantidad considerable. No es un *segmentation fault*, sino una decisión del sistema operativo para evitar que el agotamiento de memoria afecte al resto del sistema. Entre las posibles causas están un consumo excesivo de memoria o una fuga que haga que el proceso acumule memoria sin liberarla. También podría existir una relación con los fallos anteriores si alguna reserva de memoria no se gestiona correctamente, aunque los logs no permiten confirmarlo.

**Recomendación:** investigar posibles fugas de memoria, controlar el consumo del servicio y establecer límites para evitar que llegue a agotar los recursos del sistema.

## Log 5 — 09:04:58

**Hipótesis:**

El sistema ha sufrido un *kernel panic* debido a un fallo en el controlador de red `nova_nic`. El error se ha producido mientras el kernel procesaba una interrupción relacionada con la recepción de paquetes de red. Por tanto, no parece un fallo directo del programa `procesador_tx`, sino un problema en el controlador, que es un módulo externo al kernel oficial. El acceso a una dirección de memoria no válida ha provocado una excepción en modo kernel y, como el fallo se ha producido en un contexto crítico, el sistema no ha podido continuar con seguridad.

**Recomendación:** revisar la compatibilidad del controlador `nova_nic` con el kernel e intentar actualizarlo o sustituirlo por una versión estable y compatible.

---

## 5. Protección de memoria y modo dual

### Aclaración sobre el enunciado

El enunciado solicita diseñar el aislamiento entre el modo real y el modo protegido. Sin embargo, estos conceptos corresponden a modos de funcionamiento del procesador y no deben confundirse con la separación de seguridad que utilizan los sistemas operativos actuales.

El modo real pertenece a la arquitectura de los primeros procesadores x86 y carece de los mecanismos de protección de memoria propios del modo protegido. En este modo no existe el aislamiento habitual entre procesos: el software puede acceder a direcciones de memoria sin las restricciones que imponen los sistemas operativos modernos.

Por tanto, el planteamiento del enunciado hace referencia a una arquitectura de hace décadas. Para abordar el problema de seguridad de TechNova, hay que centrarse en el aislamiento entre el **modo usuario y el modo kernel**, que es el mecanismo utilizado por los sistemas operativos actuales. En procesadores x86 modernos, esta separación se establece principalmente entre el anillo 3, donde se ejecutan las aplicaciones, y el anillo 0, donde funciona el kernel.

### Funcionamiento del modo dual

Los procesadores x86 disponen de cuatro niveles de privilegio, llamados anillos, del 0 al 3. En los sistemas operativos actuales se utilizan principalmente dos: el anillo 0 para el kernel y el anillo 3 para las aplicaciones.

Las aplicaciones se ejecutan en modo usuario, con permisos limitados. Cuando necesitan realizar una operación que requiere privilegios, como leer un archivo, solicitan el servicio mediante una llamada al sistema. El procesador pasa temporalmente al modo kernel, donde el sistema operativo comprueba la petición y la ejecuta si está permitida. Después, devuelve el control a la aplicación.

Esta separación impide que un programa pueda ejecutar directamente instrucciones privilegiadas o acceder sin restricciones a la memoria del sistema operativo y de otros procesos.

### Mecanismos de protección de memoria

Para proteger los sistemas de TechNova, se pueden utilizar los siguientes mecanismos:

- **Instrucciones privilegiadas:** determinadas operaciones, como modificar las tablas de páginas o desactivar interrupciones, solo pueden ejecutarse con los privilegios adecuados. Si una aplicación intenta ejecutarlas desde modo usuario, el procesador genera una excepción.

- **Paginación y permisos de memoria:** cada proceso dispone de un espacio de direcciones virtuales y de permisos que controlan qué memoria puede leer, escribir o ejecutar. Esto permite aislar los procesos entre sí y proteger la memoria del kernel. El bit NX, por ejemplo, permite marcar determinadas páginas como no ejecutables.

- **Protecciones SMEP y SMAP:** disponibles en determinados procesadores x86, ayudan a impedir que el kernel ejecute código situado en páginas de usuario o acceda indebidamente a datos de usuario. De esta forma, reducen algunos riesgos cuando se produce un fallo en el código del kernel.

### Aplicación al caso de TechNova

A partir de los logs, se proponen tres medidas para reforzar la seguridad y la estabilidad del servidor.

**1. Mantener el servicio con un usuario sin privilegios de administrador.**

El log 4 muestra que el proceso `procesador_tx` tiene el UID 1001. Como en Linux el UID 0 corresponde a `root`, este dato confirma que el servicio no se ejecuta con esa identidad. Es una buena práctica de seguridad que TechNova ya está aplicando.

La recomendación es mantener esta configuración y limitar los permisos de la cuenta a lo imprescindible para realizar su trabajo. También conviene revisar que no tenga capacidades adicionales o permisos que le permitan realizar operaciones administrativas innecesarias.

**2. Revisar los controladores del sistema.**

En el log 5 aparece `Tainted: O`, lo que indica que el kernel ha cargado un módulo externo al árbol oficial de Linux. Este dato no demuestra que el controlador sea defectuoso ni que carezca de firma, pero sí justifica revisar su procedencia, compatibilidad y fiabilidad.

El fallo del controlador `nova_nic` termina provocando un *kernel panic*, por lo que es importante comprobar si existe una versión estable y compatible. También se puede valorar la firma de módulos del kernel, si la plataforma lo permite, para controlar qué módulos se pueden cargar. La firma ayuda a verificar su autenticidad e integridad, aunque no garantiza que el código esté libre de errores.

**3. Limitar el consumo de memoria.**

El log 4 muestra que el proceso `procesador_tx` alcanza aproximadamente 15 GiB de memoria residente antes de que intervenga el mecanismo OOM killer. Esto puede deberse a un consumo excesivo o a una fuga de memoria, aunque los logs no permiten determinar la causa exacta.

Se recomienda utilizar grupos de control de Linux (*cgroups*) para establecer límites de memoria al servicio. De esta forma, se reduce el riesgo de que un proceso consuma tantos recursos que perjudique al resto del sistema. Estos límites deben configurarse de acuerdo con las necesidades reales de la aplicación y no sustituyen la investigación de la causa del consumo.

### Conclusión

El modo real no proporciona los mecanismos de aislamiento de memoria que necesita un sistema operativo moderno. Por eso, aunque el enunciado plantea la comparación entre modo real y modo protegido, la solución relevante para TechNova consiste en reforzar la protección dentro del modo protegido o del modo largo de los procesadores actuales.

En la práctica, esto implica mantener la separación entre el anillo 3, donde se ejecutan las aplicaciones, y el anillo 0, donde funciona el kernel; limitar los privilegios de los servicios; proteger los espacios de memoria; revisar los controladores y controlar el consumo de recursos.

Estas medidas reducen el impacto de posibles fallos, pero no sustituyen la corrección de los errores de programación detectados en los logs.

## Conclusión general

Los logs muestran varios fallos que pueden tener causas diferentes. Los logs 1, 2 y 3 son *segmentation faults* provocados por excepciones de software. En los tres casos, una instrucción del programa intenta acceder a una dirección de memoria no válida, la MMU genera un *page fault* y el kernel termina enviando la señal `SIGSEGV` al proceso afectado. Por tanto, estos fallos no están provocados por interrupciones de hardware.

El log 4 tampoco es un *segmentation fault*: el OOM killer ha terminado el proceso mediante `SIGKILL` porque el sistema se estaba quedando sin memoria. Por su parte, el log 5 es el único relacionado con una interrupción de hardware, ya que el fallo se produce en el controlador de red mientras se procesa la recepción de paquetes. En este caso no se envía una señal a un proceso, sino que se produce un *kernel panic* que provoca la caída del sistema.

El log 5 también puede considerarse una pista falsa, porque ocurre durante la misma franja horaria y es el fallo más grave. Sin embargo, no hay motivos suficientes para afirmar que haya provocado los segfaults anteriores. Estos se producen en el código de `libtx.so` y en el propio programa, mientras que el *kernel panic* tiene su origen en el controlador `nova_nic`. La coincidencia temporal no demuestra que exista una relación causal entre ellos.

En conclusión, la auditoría apunta a problemas en la gestión de punteros y memoria, un posible desbordamiento de pila, un agotamiento de memoria y un fallo en el controlador de red. La prioridad es investigar el *kernel panic*, ya que afecta a todo el servidor, sin dejar de corregir los errores del programa y revisar su consumo de memoria.
