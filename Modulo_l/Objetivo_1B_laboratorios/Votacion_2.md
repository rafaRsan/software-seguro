# Laboratorio: Votación - Nueva Versión

## Objetivo

Manipular el sistema de votación para que la UTN supere en votos a la competencia, evadiendo el mecanismo de restricción basado en direcciones IP para obtener la *flag* (código HASH).

## Herramientas utilizadas

* Burp Suite (Módulos Proxy e Intruder).

## Paso a paso de la resolución (Header Attack)

1. Se interceptó una petición `POST` legítima de votación dirigida al *endpoint* `/src/ctl/votacion.ctl.php` utilizando el Proxy de Burp Suite, asegurándose de que la solicitud no contuviera cookies de bloqueo previas.

2. Se envió la petición al módulo **Intruder**. Para explotar la confianza del servidor, se inyectó manualmente la cabecera `X-Forwarded-For: 192.168.1.§1§` debajo de los encabezados estándar. Se seleccionó el último octeto de la dirección IP como la variable del *payload*.

3. En la configuración de *Payloads*, se estableció un ataque de tipo *Numbers* con un rango numérico amplio (ej. 1 a 2000). Esto permitió iterar la variable para simular miles de conexiones provenientes de IPs distintas de manera instantánea.

4. Al iniciar el ataque, el servidor procesó la cabecera inyectada como el origen real de la conexión, registrando un voto válido por cada número iterado y devolviendo códigos de estado `200 OK`.

5. Una vez que la herramienta envió suficientes peticiones para superar el contador de la otra universidad, se detuvo el ataque y se recargó la interfaz web. El sistema validó el nuevo estado de la base de datos y reveló el código HASH de resolución.
