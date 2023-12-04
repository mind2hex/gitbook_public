---
description: Can you see me?
---

# You cant c me

### Análisis básico

Lo primero que hice fue analizar de forma básica el binario para saber que tipo de archivo es y de esta forma saber que hacer posteriormente. Esto se puede realizar con el comando **file**.

```
$ file auth                                                  
auth: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, stripped
```

Del output anterior, lo que más nos interesa son los siguientes campos:

* **dynamically linked**: Esto significa que el binario utiliza bibliotecas dinámicas en lugar de contener todo dentro del propio archivo ejecutable. En este caso podemos utilizar la herramienta **ltrace** para rastrear las llamadas a las librerías dinámicas que realice el binario.
* **stripped:** Un binario "stripped" ha tenido toda o la mayoría de su información de depuración y símbolos eliminada. Esto hace que el análisis sea más difícil porque cosas como los nombres de las funciones no estarán disponibles.

#### Rastreando las llamadas a las librerías con ltrace

**ltrace** ejecuta un comando dado e intercepta y registra las llamadas a las librerías dinámicas así que para esta tarea va a ser de bastante utilidad.

```
$ ltrace ./auth 
printf("Welcome!\n"Welcome!
)                                                                                                                               = 9
malloc(21)                                                                                                                                         = 0x11ba6b0
fgets(hola
"hola\n", 21, 0x7f0c35484aa0)                                                                                                                = 0x11ba6b0
strcmp("wh00ps!_y0u_d1d_c_m3", "hola\n")                                                                                                           = 15
printf("I said, you can't c me!\n"I said, you can't c me!
)                                                                                                                = 24
+++ exited (status 0) +++
```

La salida de **ltrace** nos muestra información muy útil (incluyendo la bandera) con la cuál podemos entender el funcionamiento del binario casi en su totalidad. El programa consiste en una comparación básica de strings, y a partir de esta comparación se muestra un mensaje para cada caso.

Ahora vamos a ver que pasa si le damos el string que encontramos en el output del comando anterior.

```
$ ltrace ./auth                        
printf("Welcome!\n"Welcome!
)                                                    = 9
malloc(21)                                                              = 0x17cd6b0
fgets(wh00ps!_y0u_d1d_c_m3
"wh00ps!_y0u_d1d_c_m3", 21, 0x7f3083c58aa0)                       = 0x17cd6b0
strcmp("wh00ps!_y0u_d1d_c_m3", "wh00ps!_y0u_d1d_c_m3")                  = 0
printf("HTB{%s}\n", "wh00ps!_y0u_d1d_c_m3"HTB{wh00ps!_y0u_d1d_c_m3}
)                             = 26
+++ exited (status 0) +++
```

### Creando el código fuente a partir del binario

Con la información anterior ya tenemos idea de como funciona el binario y que funciones utiliza así que podemos crear un script en lenguaje C que almenos se parezca en funcionamiento al binario que acabamos de analizar.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char argv[]){
  char *flag = "wh00ps!_y0u_d1d_c_m3";
  char *user_input_data;

  user_input_data = (char*) malloc(21  * sizeof(char));

  printf("Welcome!\n");

  fgets(user_input_data, 21, stdin);

  if (strcmp(flag, user_input_data) != 0){
    // NO MATCH                                                                                                                       
    printf("I said, you can't c me!\n");

  }else{
    // MATCH                                                                                                                          
    printf("HTB{%s}", flag);

  }

  return 0;
}
```

Ahora compilamos el código fuente y lo ejecutamos con **ltrace** para comparar el funcionamiento.

```
$ gcc main.c -o auth_copy && ltrace ./auth_copy  
malloc(21)                                                              = 0x556fbfbd52a0
puts("Welcome!"Welcome!
)                                                        = 9
fgets(wh00ps!_y0u_d1d_c_m3
"wh00ps!_y0u_d1d_c_m3", 21, 0x7ff8f0aaaaa0)                       = 0x556fbfbd52a0
strcmp("wh00ps!_y0u_d1d_c_m3", "wh00ps!_y0u_d1d_c_m3")                  = 0
printf("HTB{%s}", "wh00ps!_y0u_d1d_c_m3")                               = 25
HTB{wh00ps!_y0u_d1d_c_m3}+++ exited (status 0) +++

```

El comportamiento del programa es el mismo.
