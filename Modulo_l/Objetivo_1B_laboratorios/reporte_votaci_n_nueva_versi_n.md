# Laboratorio: Votación - Nueva Versión

## Objetivo

Manipular el sistema de votación para que la "Universidad Tecnológica Nacional" (UTN) supere en votos a la competencia, evadiendo el mecanismo de restricción basado en direcciones IP para obtener la *flag* (código HASH).

## Herramientas utilizadas

* Burp Suite (Módulos Proxy e Intruder).

## Análisis de Vulnerabilidad

Durante la etapa de reconocimiento, se identificó que el sistema bloqueaba los intentos de votos repetidos arrojando el error: *"No se puede votar más de una vez desde la misma ip"*. No obstante, la lógica de validación del servidor presenta una vulnerabilidad de confianza: en lugar de verificar exclusivamente la IP real de la conexión de red (el socket TCP), el *backend* prioriza la lectura de cabeceras HTTP que pueden ser inyectadas o modificadas por el cliente.

## Paso a paso de la resolución (Header Attack)

1. **Captura del tráfico:** Se interceptó una petición `POST` legítima de votación dirigida al *endpoint* `/src/ctl/votacion.ctl.php` utilizando el Proxy de Burp Suite, asegurándose de que la solicitud no contuviera cookies de bloqueo previas.

2. **Inyección de la cabecera:** Se envió la petición al módulo **Intruder**. Para explotar la confianza del servidor, se inyectó manualmente la cabecera `X-Forwarded-For: 192.168.1.§1§` debajo de los encabezados estándar. Se seleccionó el último octeto de la dirección IP como la variable del *payload*.

3. **Automatización:** En la configuración de *Payloads*, se estableció un ataque de tipo *Numbers* con un rango numérico amplio (ej. 1 a 2000). Esto permitió iterar la variable para simular miles de conexiones provenientes de IPs distintas de manera instantánea.

4. **Explotación:** Al iniciar el ataque, el servidor procesó la cabecera inyectada como el origen real de la conexión, registrando un voto válido por cada número iterado y devolviendo códigos de estado `200 OK`.

5. **Captura de la flag:** Una vez que la herramienta envió suficientes peticiones para superar el contador de la otra universidad, se detuvo el ataque y se recargó la interfaz web. El sistema validó el nuevo estado de la base de datos y reveló el código HASH de resolución.