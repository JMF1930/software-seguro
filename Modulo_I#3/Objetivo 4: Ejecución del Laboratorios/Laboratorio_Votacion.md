# Labs: Votación

Enunciado: Un compañero me acaba de enviar un link de una página que
realiza una extraña votación, en la cual participa nuestra facultad.
Estaría bueno que votes, ¿podrá ganar la UTN?

Pista: Obtendrás el hash cuando la cantidad de votos de la UTN supere a
Harvard.

<img src="media/image1.png"
style="width:6.87148in;height:4.08031in" />

Luego votar encuentro esta información.

Capturo la sentencia y la envío a intruder:

<img src="media/image2.png"
style="width:6.5in;height:3.81389in" />

Ahora configuro el payload null para que no cambie la sentencia

<img src="media/image3.png"
style="width:6.5in;height:3.81389in" />

Luego configuro el pool en 999 el máximo que me permite:

<img src="media/image4.png"
style="width:6.5in;height:3.81389in" />

y lanzo el ataque:

<img src="media/image5.png"
style="width:6.5in;height:3.88333in" />

Al finalizar cuando supera la UTN a Harvard tengo el dato que estoy
buscando:

<img src="media/image6.png"
style="width:6.5in;height:3.86528in" />

<img src="media/image7.png"
style="width:6.5in;height:3.86528in" />

Actualizo el explorador

<img src="media/image8.png"
style="width:6.5in;height:3.85972in" />

Cargo el algoritmo **sdf7fdj843if3jhg3**:

<img src="media/image9.png"
style="width:6.5in;height:3.85972in" />
