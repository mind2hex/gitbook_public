---
description: >-
  La biblioteca de curses proporciona una función de keyboard-handling y
  screen-painting independiente del terminal para terminales basados ​​en texto.
---

# ncurses

## Iniciando y terminando aplicaciones curse

```python
import curses

""" Inicializando curses
Esto debe realizarse con la funcion initscr(), la cual se encarga de determinar
el tipo de terminal ademas de crear varias estructuras de datos internas para
el correcto funcionamiento de curses. Si todo se ejecuta sin ningun problema, la
funcion va a retornar un objeto window el cual representa el screen de la terminal. 
"""
stdscr = curses.initscr()  # stdscr = screen

""" cbreak()
Esta funcion permite que la aplicacion reaccione a las teclas sin necesidad 
de oprimir ENTER.
"""
curses.cbreak()

""" keypad()
La función keypad en curses se usa para habilitar el reconocimiento de 
teclas especiales por parte de la terminal. Cuando lo habilitas en una 
ventana con keypad(True), puedes empezar a capturar eventos de teclas 
especiales como las teclas de flecha, la tecla F1, Enter, etc., de una 
manera más sencilla.
"""
stdscr.keypad(True)

""" Terminando curses
Antes de terminar curses, primero se deben reestablecer los valores 
para que la terminal vuelva a la normalidad.
"""
curses.nocbreak()
stdscr.keypad(False)
curses.echo()

""" Por ultimo se llama a la funcion endwin()
"""
curses.endwin()
```
