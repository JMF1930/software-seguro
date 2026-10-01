# Laboratorio El mejor secreto

## Enunciado

Lograste grabar un video de un jefe de estado tecleando la clave que
protege uno de los archivos más importantes: secreto.zip.

Por seguridad, el jefe usa un teclado numérico modificado: las
posiciones de las teclas son visibles en el video, pero el dígito que
corresponde a cada tecla no coincide con la etiqueta física y no conocés
la correspondencia. Sí sabés que cada tecla corresponde a un dígito
distinto.

Objetivo: ¿Podrás descifrar el archivo zip?

Lo primero que hago es ver el video y observo que se pulsan 12 digitos,
de los cuales muchos se repiten con la siguiente secuencia:

6a-2-6b-6b-9-6a-9-9-9-9-5-6b

Para ello genero un archivo txt con todas las combinaciones posibles de
12 digitos con esa secuencia y me da un archivo con 30240 posibles
combinaciones.

Lógica aplicada (posición → grupo): 1 2 3 4 5 6 7 8 9 10 11 12 → A B C C
D A D D D D E C

- **A** (posiciones 1, 6)

- **B** (posición 2, dígito único)

- **C** (posiciones 3, 4, 12)

- **D** (posiciones 5, 7, 8, 9, 10)

- **E** (posición 11, dígito único)

Las 5 variables son mutuamente distintas entre sí (todas "diferentes a
los demás").

Cálculo: 10×9×8×7×6 = **30.240** combinaciones.

Luego generamos una aplicación en Python para que por fuerza bruta logre
la clave para abrir el archivo zip:

```python
#!/usr/bin/env python3
"""
zipcracker.py — Recuperación de clave de archivos ZIP por diccionario (fuerza bruta).

Uso legítimo: recuperar la contraseña de un ZIP propio o en auditorías/pentests
con autorización explícita. El uso indebido es responsabilidad de quien lo ejecuta.

Soporta:
- ZIP con cifrado clásico (ZipCrypto), vía la stdlib `zipfile`.
- ZIP con cifrado AES (WinZip), si está instalado `pyzipper` (pip install pyzipper).

Ejemplos:
    python zipcracker.py secreto.zip claves.txt
    python zipcracker.py secreto.zip rockyou.txt -w 8 -v
"""
import argparse
import sys
import time
import zipfile
from concurrent.futures import ThreadPoolExecutor, as_completed
from threading import Event

try:
    import pyzipper  # opcional, para ZIP con AES
    HAS_PYZIPPER = True
except ImportError:
    HAS_PYZIPPER = False


def detectar_aes(ruta_zip: str) -> bool:
    """Devuelve True si algún miembro del ZIP usa cifrado AES."""
    try:
        with zipfile.ZipFile(ruta_zip) as zf:
            for info in zf.infolist():
                # extra field 0x9901 = AES; heurística sobre compress_type
                if info.compress_type == 99:
                    return True
    except zipfile.BadZipFile:
        pass
    return False


def abrir_zip(ruta_zip: str, usar_aes: bool):
    """Abre el ZIP con la librería adecuada."""
    if usar_aes:
        if not HAS_PYZIPPER:
            sys.exit("El ZIP usa AES pero 'pyzipper' no está instalado. "
                      "Instalalo con: pip install pyzipper")
        return pyzipper.AESZipFile(ruta_zip)
    return zipfile.ZipFile(ruta_zip)


def probar_clave(ruta_zip: str, usar_aes: bool, nombre_test: str, clave: str) -> bool:
    """Intenta abrir y leer un miembro del ZIP con la clave dada."""
    try:
        with abrir_zip(ruta_zip, usar_aes) as zf:
            # Leer realmente el archivo fuerza la verificación del cifrado.
            zf.read(nombre_test, pwd=clave.encode("utf-8", errors="ignore"))
        return True
    except (RuntimeError, zipfile.BadZipFile, OSError, Exception):
        # Clave incorrecta -> excepción de descifrado/CRC. Se ignora.
        return False


def cargar_claves(ruta_txt: str):
    """Genera claves desde el diccionario, línea por línea (streaming)."""
    with open(ruta_txt, "r", encoding="utf-8", errors="ignore") as f:
        for linea in f:
            clave = linea.rstrip("\r\n")
            if clave:
                yield clave


def contar_lineas(ruta_txt: str) -> int:
    with open(ruta_txt, "rb") as f:
        return sum(1 for _ in f)


def crackear(ruta_zip: str, ruta_txt: str, workers: int, verbose: bool):
    usar_aes = detectar_aes(ruta_zip)
    if usar_aes:
        print("[i] Cifrado detectado: AES (usando pyzipper)")
    else:
        print("[i] Cifrado detectado: ZipCrypto / clásico")

    # Elegir un miembro cifrado para testear
    with abrir_zip(ruta_zip, usar_aes) as zf:
        miembros = [i.filename for i in zf.infolist() if not i.is_dir()]
    if not miembros:
        sys.exit("El ZIP no contiene archivos para probar.")
    nombre_test = miembros[0]

    total = contar_lineas(ruta_txt)
    print(f"[i] Diccionario: {total} claves | Hilos: {workers}")
    print(f"[i] Probando contra el miembro: {nombre_test}\n")

    encontrada = Event()
    resultado = {"clave": None}
    inicio = time.time()
    probadas = 0

    def worker(clave: str):
        if encontrada.is_set():
            return None
        if probar_clave(ruta_zip, usar_aes, nombre_test, clave):
            return clave
        return None

    with ThreadPoolExecutor(max_workers=workers) as ex:
        futuros = {}
        it = cargar_claves(ruta_txt)

        # Alimentar el pool en lotes para no cargar todo en memoria
        def enviar_siguiente():
            try:
                clave = next(it)
            except StopIteration:
                return False
            futuros[ex.submit(worker, clave)] = clave
            return True

        # Cebar el pool
        for _ in range(workers * 4):
            if not enviar_siguiente():
                break

        while futuros and not encontrada.is_set():
            for fut in as_completed(list(futuros)):
                clave = futuros.pop(fut)
                probadas += 1
                res = fut.result()
                if verbose:
                    print(f"  probando: {clave}")
                elif probadas % 500 == 0:
                    vel = probadas / (time.time() - inicio + 1e-9)
                    print(f"[.] {probadas}/{total} ({vel:.0f} claves/s)")
                if res is not None:
                    resultado["clave"] = res
                    encontrada.set()
                    break
                enviar_siguiente()
            break  # re-evaluar as_completed con el set actualizado

    dur = time.time() - inicio
    print()
    if resultado["clave"] is not None:
        print(f"[+] CLAVE ENCONTRADA: {resultado['clave']!r}")
        print(f"[+] Probadas {probadas} claves en {dur:.1f}s")
        return resultado["clave"]
    else:
        print(f"[-] No se encontró la clave. Probadas {probadas} claves en {dur:.1f}s")
        return None


def main():
    p = argparse.ArgumentParser(description="Recuperación de clave de ZIP por diccionario.")
    p.add_argument("zip", help="Ruta al archivo .zip protegido")
    p.add_argument("wordlist", help="Ruta al .txt con claves candidatas (una por línea)")
    p.add_argument("-w", "--workers", type=int, default=8, help="Cantidad de hilos (default: 8)")
    p.add_argument("-v", "--verbose", action="store_true", help="Mostrar cada clave probada")
    args = p.parse_args()

    try:
        clave = crackear(args.zip, args.wordlist, args.workers, args.verbose)
    except FileNotFoundError as e:
        sys.exit(f"Archivo no encontrado: {e.filename}")
    except KeyboardInterrupt:
        sys.exit("\n[!] Interrumpido por el usuario.")
    sys.exit(0 if clave else 1)


if __name__ == "__main__":
    main()
```

Se utiliza de la siguiente manera:

```bash
python zipcracker.py archivo.zip claves.txt
python zipcracker.py archivo.zip rockyou.txt -w 12 -v
```

- -w cantidad de hilos (más hilos = más rápido, hasta cierto punto).

- -v muestra cada clave que va probando.

En nuestro caso:

```bash
python zipcracker.py secreto.zip numeros_12digitos_v4.txt -w 12 -v
```

Asi encontramos la clave:

<img
src="media/image1.png"
style="width:6.5in;height:3.33958in" />

Clave encontrada: 547795999937

Abro el archivo y encuentro la clave:

<img
src="media/image2.png"
style="width:6.5in;height:3.75139in" />

Clave encontrada: 696026dd5bf583f34530a657d896ebea
