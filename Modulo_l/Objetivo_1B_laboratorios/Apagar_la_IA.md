# Laboratorio: Apagar la IA

## Objetivo
Encontrar un código oculto de 16 dígitos dentro de un archivo HTML cuyo nombre es un hash MD5, y generar el hash MD5 de dicho código para 
apagar la inteligencia artificial.

## Herramientas utilizadas
Script en Python/Bash (generación de diccionario), Burp Suite (Módulo Intruder), Generador MD5 local (Terminal).

## Paso a paso de la resolución

1. Al analizar los enlaces proporcionados en la página de inicio, se identifica que los nombres de los archivos corresponden a
hashes MD5 de números enteros (con valores cercanos al 10000). Como el algoritmo MD5 es propenso a ser revertido mediante tablas precalculadas si la
entrada es sencilla, el objetivo es realizar fuerza bruta iterando sobre una lista de hashes.
2. Se desarrolló un script para aplicar la función MD5 a un rango numérico acotado (del 9000 al 13000) y guardar las salidas en un archivo
de texto (`hashes.txt`). Este diccionario actúa como la lista base de payloads.
3. **Configuración del Ataque (Intruder):** 
   * Se interceptó la petición GET base dirigida al servidor y se envió al módulo **Intruder**.
   * Se configuró como única posición de payload el hash en la URL, asegurando que la ruta y las barras diagonales estuvieran correctas.
   * En la pestaña *Payloads*, se cargó el diccionario `hashes.txt` generado en el paso anterior.
4. Dado que los archivos falsos tienen un tamaño (*Content-Length*) muy similar al archivo válido,
se configuró una regla de extracción en la pestaña *Settings* utilizando la expresión regular `([0-9]{16})` o `(\d{16})`. Esto instruye a Burp Suite a capturar y
aislar cualquier secuencia exacta de 16 números en una columna separada.
5. Al iniciar el ataque y ordenar la columna de extracción, la herramienta filtró el ruido del servidor y
aisló el hash `cdl49l5251e7b3eb4l009483121e9b64`, el cual contenía el código real de 16 dígitos: `5524663362514956`.
6. Por último, se convirtió el código obtenido a su equivalente en MD5
(ej. ejecutando `echo -n "5524663362514956" | md5sum` en la terminal de Kali). El hash resultante, `a8e0e8ff02dde0f62fdf4de5142d7de0`,
fue validado exitosamente en la plataforma para superar el desafío.
