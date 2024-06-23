---
title: 'Pinned vector: la solución perfecta a un problema muy concreto'
date: 2021-8-9
tags:
  informática
  programación
---
Existen muchas estructuras de datos que representan secuencias. Desde arrays dinámicos y listas a estructuras más esotéricas como listas de arrays de tamaño constante o el “chained group allocation pattern” muy usado por la biblioteca PLF. La selección de una u otra estructura depende mayormente de los requerimientos de complejidad asintótica en las operaciones, estabilidad de referencias y contigüidad de memoria. En el hardware actual, tanto para secuencias pequeñas como para secuencias grandes en las que el tamaño es conocido de antemano, el array dinámico es con frecuencia la estructura de datos preferida. Sin embargo, para colecciones muy grandes en las que el tamaño es desconocido, usar un array dinámico plantea un problema importante. Por lo general los arrays dinámicos reservan más memoria de la que necesitan, y presentan una distinción entre tamaño, la cantidad de elementos que contienen, y capacidad, la máxima cantidad de elementos que pueden contener con la memoria de la que disponen actualmente. Cuando se intenta insertar en un array en el que no hay capacidad para más, se reserva un array nuevo, frecuentemente del doble de tamaño, se mueven todos los elementos del antiguo al nuevo y se libera el bloque de memoria antiguo. Como por lo general mover elementos es una operación barata, reubicar arrays de decenas o cientos de elementos no suele ser un problema. Sin embargo, esto no es tan cierto cuando hablamos de cientos de miles o de millones de elementos. Para estas situaciones es preferible usar una estructura de datos distinta que no requiera de reubicar cientos de miles de elementos.

Por otro lado, un array es una estructura de datos con propiedades muy deseables. Al estar compuesto por un solo bloque de memoria contigua iterarlo es muy rápido ya que se maximiza el uso de la memoria caché y el prefetcher. Además, se puede acceder a cualquier elemento por índice en tiempo constante y en una sola instrucción de procesador. Es también muy fácil poblar o consumir un array de forma concurrente requiriendo como único mecanismo de sincronización un índice atómico. Por todas estas ventajas, sería muy deseable poder usar un array también para colecciones grandes de tamaño desconocido.

Un “pinned vector” es una estructura que explota la memoria virtual del sistema operativo para poder crear arrays que pueden crecer indefinidamente hasta un límite muy alto sin tener que reubicar los elementos nunca. Para entender cómo funciona es importante entender la diferencia entre memoria física y memoria virtual. La memoria física son los bytes en la tarjeta de memoria RAM del ordenador. En los ordenadores de hoy en día, el único programa que tiene acceso a la memoria física es el sistema operativo. El resto de programas acceden a memoria virtual. Es la función del sistema operativo asignar páginas de memoria física al espacio de direcciones de memoria virtual de los programas. Esto permite que un programa no pueda acceder a la memoria de los otros programas, ya que la memoria física que está usando el otro programa no es visible en el espacio de memoria virtual del primero. También permite tratar memoria discontigua como si fuera contigua, mapeando páginas de memoria física discontinuas a un intervalo continuo de memoria virtual. Por último, permite mapear al espacio de memoria de un programa cosas que no son memoria física, como archivos, la memoria de la tarjeta gráfica o la red.

![Memoria virtual](/images/virtual-memory.png)

Es importante también que un programa tiene muchas más direcciones de memoria virtual que bytes reales de memoria física. En una arquitectura de 64 bits, un proceso cuenta con un espacio de memoria virtual de 2^48 bytes (262144 GiB), mientras que por lo general un ordenador de escritorio tiene 4 u 8 gigas de memoria física. Y este es el hecho que explota un pinned vector. Se puede reservar intervalos de memoria virtual que no están apoyados por páginas de memoria física, siempre y cuando esa memoria no se use, e ir rellenándolos con páginas de memoria física conforma haga falta. Esto permite a un array garantizar que las direcciones de memoria directamente contiguas a su bloque de memoria están reservadas y nunca van a ser usadas por otras partes del programa, de forma que cada vez que tiene que crecer puede ir solicitando reservar memoria en esas direcciones en concreto. Se puede, al momento de crear el array, asignarle uno o varios gigas de memoria virtual, sin usar para ello ni un solo byte de memoria física, e ir ocupando esa memoria conforme el array va creciendo.

Pinned vector tiene todas las ventajas de un array, con memoria contigua y acceso a cualquier elemento por índice en una instrucción, pero además puede crecer en tiempo constante (en vez de constante amortizado), y como nunca tiene que reubicar sus elementos las referencias a ellos son estables siempre. Es una estructura de datos muy original que resuelve muy bien un problema difícil y para el que no hay ninguna otra solución realmente satisfactoria.

Por otro lado, pinned vector no es la panacea. Tiene una serie de inconvenientes que es importante entender. Para empezar, requiere que el sistema tenga memoria virtual. Esto está garantizado en ordenadores, móviles y consolas de videojuegos, pero posiblemente no sea el caso en sistemas integrados. Por otro lado, su granularidad a la hora de reservar memoria es muy grande. Las reservas al sistema operativo suceden por páginas, frecuentemente de 4096 bytes, por lo que no es una buena herramienta para colecciones pequeñas. Si es frecuente tener 4 o 6 elementos, no uses un pinned vector. Usa un vector normal o incluso uno con optimización de vector pequeño, como llvm::SmallVector. De la misma forma, si el tamaño del array es conocido de antemano, se puede hacer un solo malloc de ese tamaño y ya. Incluso usando la cantidad de páginas exacta para el tamaño deseado, un pinned vector puede desperdiciar una cantidad sustancial de memoria si necesita solamente una pequeña parte de la última página. Por ejemplo, un pinned vector de 13 KiB va a reservar 16KiB de memoria, desperdiciando 3 KiB. Una buena implementación de malloc detecta estas situaciones y reusa esos 3 KiB para otros objetos. Por último, aunque es cierto que un pinned vector nunca reubica sus elementos para poder crecer, eliminar un elemento del array sí que requiere mover todos los demás hacia atrás. Esto no es en absoluto deseable en arrays grandes, en los que puede implicar mover miles o millones de elementos. Si eliminar elementos en posiciones arbitrarias es una operación necesaria, es mejor usar una estructura de datos distinta.

Pinned vector no es una estructura de datos para todos los casos. Tiene un sobrecoste alto debido a que reserva memoria en páginas que no comparte con ninguna otra estructura, pero puede ser la elección perfecta para programas que necesitan construir un array muy grande cuyo tamaño no conocen de antemano, así que es una herramienta muy valiosa que conocer y tener a mano para cuando surge la necesidad de usarla.

## Referencias:

- Chained group allocation pattern [https://plflib.org/chained_group_allocation_pattern.htm](https://plflib.org/chained_group_allocation_pattern.htm)