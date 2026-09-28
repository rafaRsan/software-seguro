# Resolución de Laboratorio: Ventas

## Objetivo
Descubrir cuántas ventas realizó la competencia de Fernando en la ruta `/ventas` y generar un hash MD5 de ese total para obtener la flag de resolución.

## Análisis Inicial y Detección de Vulnerabilidad
Al explorar la ruta objetivo, descubrimos que el servidor filtra información a través de los códigos de estado HTTP (Fuga de información / Information Disclosure):
* Si consultamos un ID de venta válido al que no tenemos acceso (ej. `?id=1`), el servidor responde con un código **403 Forbidden**.
* Si consultamos un ID inexistente, responde con un código **404 Not Found**.

*Nota técnica:* Se detectó que el servidor realiza una redirección estricta (**301 Moved Permanently**) si no se incluye la barra diagonal al final del directorio, por lo que la ruta exacta a auditar debe ser `/ventas/?id=X`.

## Ejecución del Ataque (Paso a Paso)

1. **Intercepción:** Activamos el *Intercept* en la pestaña Proxy de Burp Suite y capturamos una petición GET a la ruta objetivo.
2. **Configuración de Intruder:** Enviamos la petición al módulo Intruder (`Ctrl + I`). En la pestaña *Positions*, configuramos la petición exacta limpiando las variables por defecto y marcando únicamente el número del ID:
   ```http
   GET /ventas/?id=§1§ HTTP/2
   Host: chl-[tu-dominio]-ventas.softwareseguro.com.ar
   ```
3. **Configuración de Payloads:** En la pestaña *Payloads*, seleccionamos el tipo `Numbers` y configuramos un rango secuencial del **1 al 2000** con un paso (step) de 1.
4. **Fuerza Bruta:** Iniciamos el ataque (*Start attack*).
5. **Análisis de Resultados:** Al finalizar, ordenamos la tabla de resultados haciendo clic en la columna **Status**.
6. **Conteo:** Seleccionamos todas las peticiones que devolvieron el código **403 Forbidden**. El conteo total de Burp Suite arrojó exactamente **1641** ventas existentes.

## Obtención de la Flag
Para finalizar el desafío, convertimos el número total de ventas (1641) a formato MD5 utilizando la terminal de Kali Linux:

```bash
echo -n 1641 | md5sum
```

El hash alfanumérico resultante es la flag final ingresada en la plataforma para dar por completado el laboratorio.