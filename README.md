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

## Conclusión general

Los logs muestran varios fallos que pueden tener causas diferentes. Los logs 1, 2 y 3 son *segmentation faults* provocados por excepciones de software. En los tres casos, una instrucción del programa intenta acceder a una dirección de memoria no válida, la MMU genera un *page fault* y el kernel termina enviando la señal `SIGSEGV` al proceso afectado. Por tanto, estos fallos no están provocados por interrupciones de hardware.

El log 4 tampoco es un *segmentation fault*: el OOM killer ha terminado el proceso mediante `SIGKILL` porque el sistema se estaba quedando sin memoria. Por su parte, el log 5 es el único relacionado con una interrupción de hardware, ya que el fallo se produce en el controlador de red mientras se procesa la recepción de paquetes. En este caso no se envía una señal a un proceso, sino que se produce un *kernel panic* que provoca la caída del sistema.

El log 5 también puede considerarse una pista falsa, porque ocurre durante la misma franja horaria y es el fallo más grave. Sin embargo, no hay motivos suficientes para afirmar que haya provocado los segfaults anteriores. Estos se producen en el código de `libtx.so` y en el propio programa, mientras que el *kernel panic* tiene su origen en el controlador `nova_nic`. La coincidencia temporal no demuestra que exista una relación causal entre ellos.

En conclusión, la auditoría apunta a problemas en la gestión de punteros y memoria, un posible desbordamiento de pila, un agotamiento de memoria y un fallo en el controlador de red. La prioridad es investigar el *kernel panic*, ya que afecta a todo el servidor, sin dejar de corregir los errores del programa y revisar su consumo de memoria.
