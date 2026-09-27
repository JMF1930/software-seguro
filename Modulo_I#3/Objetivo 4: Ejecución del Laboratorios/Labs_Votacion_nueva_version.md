# Labs Votación nueva versión

Enunciado: Un compañero me acaba de enviar un link de una página que
realiza una extraña votación, en la cual participa nuestra facultad.
Estaría bueno que votes, ¿podrá ganar la UTN?

Pista 1: Obtendrás el código HASH cuando la cantidad de votos de la UTN
supere a Harvard.

Pista 2: Como desean que cada persona vote solo una única vez han
reforzado las defensas en el código para conseguir esto. ¿Pero habrá
sido suficiente?

Abro el laboratorio:

<img
src="media/image1.png"
style="width:6.5in;height:3.85972in" />

Antes de votar abro el Burp y empiezo a interceptar:

Me indica los valores actuales de votación:

<img
src="media/image2.png"
style="width:6.5in;height:3.85972in" />

Busco el get de votación y lo envío a Repeter:

<img
src="media/image3.png"
style="width:6.5in;height:3.81389in" />

Me doy cuenta que cuando envío otra votación me da un mensaje de error
indicando que la IP indicada ya voto.

<img
src="media/image4.png"
style="width:6.5in;height:3.81389in" />

Para lo cual intento engañar el control de IP indicando una IP diferente
con el comando:

X-Forwarded-For: 192.168.1.1

Ahora veo que si funciona por lo que la envío a Intruder para el ataque:

<img
src="media/image5.png"
style="width:6.5in;height:3.81389in" />

Configuro el payload del último número de la IP secuencial del 2 al 255
y avanzó con ataque:

<img
src="media/image6.png"
style="width:6.5in;height:3.81389in" />

Me fijo que en cada línea de ataque cambia la dirección IP y el ataque
es exitoso:

<img
src="media/image7.png"
style="width:6.5in;height:3.86528in" />

Cuando los valores de UTN superan a Harvard, pauso el ataque, actualizo
la pagina del explorador y obtengo el dato buscado:

<img
src="media/image8.png"
style="width:6.5in;height:3.85972in" />

Cargo el dato:

**143885b3abc1012375b3846f84c39203**

<img
src="media/image9.png"
style="width:6.5in;height:3.85972in" />
