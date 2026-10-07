# Laboratorio: El mejor secreto

## Objetivo
Descifrar la contraseña de 12 dígitos de un archivo comprimido (`secreto.zip`) utilizando técnicas de fuerza bruta offline y análisis de patrones visuales.

## Paso a paso

1- Al observar las repeticiones de las pulsaciones físicas, se logró aislar el siguiente patrón matemático de 12 posiciones compuesto por 5 variables únicas (A, B, C, D, E):
**Patrón:** `ABCCDADDDDEC`
Al reducir el universo de posibilidades de un ataque de fuerza bruta convencional (10^12 combinaciones) a una permutación de 10 dígitos tomados de a 5 variables únicas, el espacio de búsqueda bajó drásticamente a **30.240 combinaciones** ($10 \times 9 \times 8 \times 7 \times 6$).

2- Se desarrolló un script en Python para generar el diccionario exacto basado en el patrón:

```python
import itertools

with open("diccionario_zip.txt", "w") as f:
    for p in itertools.permutations("0123456789", 5):
        a, b, c, d, e = p
        password = f"{a}{b}{c}{c}{d}{a}{d}{d}{d}{d}{e}{c}"
        f.write(password + "\n")
```
3- Se utilizó la suite **John the Ripper** para el ataque offline. Al no interactuar con un servidor, este ataque no sufre penalizaciones por *Rate Limit*.

4- Se extrajo el hash criptográfico del contenedor ZIP para que la herramienta pudiera procesarlo:
`zip2john secreto.zip > hash.txt`

5- Se inyectó el diccionario generado contra el hash extraído:
`john --wordlist=diccionario_zip.txt hash.txt`

6- El ataque fue exitoso y se completó en menos de un segundo dada la reducida carga de procesamiento.
*   **Contraseña descubierta:** `547795999937`
*   **Estado:** Archivo descomprimido exitosamente y flag obtenida.
