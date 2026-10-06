# Laboratorio Asistencia

Enunciado: Te solicitaron que averigües cuántos asistentes hay pero no
tienes la clave para consultar. Se sabe que los asistentes no superan
los 200.

Una vez que sepas la cantidad, generá el md5 de eso y ese es el código
ganador.

Encontré una vulnerabilidad que permitía ingresar el código y la clave.

<img src="media/image1.png"
style="width:6.5in;height:3.81042in" />

Sabiendo que el código era numérico y de 3 dígitos se puede comenzar un
ataque de fuerza bruta con un valor numérico del 000 al 999 en el código
y un listado de posibles claves.

<img src="media/image2.png"
style="width:6.97839in;height:4.14975in" />

Para una lista de posibles contraseñas de 124 (baja), se arma un ataque
de 124000 veces.

Como el enunciado a su vez indicaba que para resolver es necesario
conocer la cantidad de asistentes menor a 200, se me ocurrió realizar un
ataque sobre la resolución del laboratorio: para lo cual se creo un
alistado de MD5 del 0 al 200 y se incio el ataque (más rápido que el
anterior.

<img src="media/image3.png"
style="width:6.5in;height:3.81042in" />

<img src="media/image4.png"
style="width:6.5in;height:3.81042in" />

Como resultado obtenemos que el valor 160 en md5 nos devuelve un código
200, por lo que hemos resuelto el laboratorio.

<img src="media/image5.png"
style="width:6.5in;height:3.86528in" />
