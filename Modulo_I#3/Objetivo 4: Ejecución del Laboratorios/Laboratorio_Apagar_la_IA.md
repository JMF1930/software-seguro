# Laboratorio Apagar la IA

Enunciado: Una Inteligencia Artificial se ha descontrolado y debemos
apagarla. Existe un código que permite apagarla.

Se sabe que ese código es de 16 dígitos y que se encuentra en un archivo
HTML del sitio de BlackHackers. Te consiguieron acceso a 2 archivos pero
ahí no está el código de 16 dígitos. ¿Podrías ayudar a la humanidad a
encontrar ese código?

Una vez que encuentres el código, generá el hash MD5 del mismo y eso
envialo al juez.

Cuando ejecuto el laboratorio me muestra estos dos números que al
clickearlos me envía a esta pagina
<https://chl-60478528-1906-4ab3-9879-241ab62f0597-apagar-ia.softwareseguro.com.ar/codes/0e1422ea79781ee046484893ce0010c4/>
con la siguiente información:

<img src="media/image1.png"
style="width:6.5in;height:3.85972in" />

<img src="media/image2.png"
style="width:6.5in;height:3.85972in" />

En ninguno de los casos código de 16 digitos.

Convierto a estos dos valores en MD5 y obtengo:

Para
[0e1422ea79781ee046484893ce0010c4](https://chl-60478528-1906-4ab3-9879-241ab62f0597-apagar-ia.softwareseguro.com.ar/codes/0e1422ea79781ee046484893ce0010c4/)
el valor es 9912

Para
[0602940f23884f782058efac46f64b0f](https://chl-60478528-1906-4ab3-9879-241ab62f0597-apagar-ia.softwareseguro.com.ar/codes/0602940f23884f782058efac46f64b0f/)
el valor es 9995

Es decir ambos valores altos.

Ahora busco en forma secuencial los posibles valores máximos y mínimos y
determino un piso de 9000 y un techo de 13000, es decir por debajo y
encima de esos valores no encuentro datos.

Ahora genero un listado de MD5 desde el número 9000 al 15000

Capturo la sentencia y la envío a intruder

Luego agrego en Burp un Grep – Match de 16 dígitos con la sentencia
\b\d{16}\b, donde:

- \b → límite de palabra (evita coincidencias dentro de números más
  largos).

- \d{16} → exactamente 16 dígitos.

- \b → cierre del límite.

Burp resaltará cualquier coincidencia de 16 dígitos en las respuestas

<img src="media/image3.png"
style="width:6.5in;height:3.81389in" />

<img src="media/image4.png"
style="width:6.5in;height:3.81389in" />

Hasta que encuentro el número buscado en
cdf49f5251e7b3eb4f009483121e9b64 (11520) y el dato es: 5524663362514956

<img src="media/image5.png"
style="width:6.5in;height:3.86528in" />

<img src="media/image6.png"
style="width:6.5in;height:3.86528in" />

Luego lo convierto en MD5

<img src="media/image7.png"
style="width:6.5in;height:3.85972in" />

Y obtengo el dato buscado: a8e0e8ff02dde0f62fdf4de5142d7de0
