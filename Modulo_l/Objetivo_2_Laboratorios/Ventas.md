## 1. Laboratorio: Ventas

### Objetivo
Contabilizar la cantidad exacta de ventas ocultas de la competencia manipulando parámetros de la URL y generar un hash MD5 del total resultante.

### Herramientas utilizadas
Burp Suite (Módulos Proxy e Intruder), Generador MD5 local.

### Paso a paso
1. Al ingresar una venta inexistente (ej. `?id=9999`) el servidor devuelve un código `404 Not Found`, mientras que al ingresar un ID válido
pero sin autorización (ej. `?id=1`) devuelve un `403 Forbidden`.
2. Interceptamos la petición `GET /ventas/?id=1` y la enviamos al módulo Intruder. 
3. Limpiamos las variables automáticas y encerramos únicamente el valor numérico del parámetro como payload (`§1§`).
4. En la pestaña Payloads, configuramos un ataque secuencial de tipo *Numbers* del 1 al 3000.
5. Lanzamos el ataque y, al finalizar, ordenamos la columna *Status code* para agrupar los resultados.
6. Seleccionamos todas las peticiones que devolvieron el código `403 Forbidden`. El recuento total arrojó **1641** ventas existentes.
7. Convertimos el número 1641 a formato MD5 mediante la terminal (`echo -n 1641 | md5sum`) y obtuvimos el código final para validar el laboratorio.
