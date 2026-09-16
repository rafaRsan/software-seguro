# Puertas de Entrada: Puertos y Servicios Fundamentales

Al interactuar con un servidor, es importante saber que **puertos** hay abiertos. 
Estos son los servicios que corren por defecto en los puertos fundamentales:

* **Puerto 20 y 21 (FTP-File Transfer Protocol):** Utilizados para la transferencia de archivos entre un cliente y un servidor.
*  El puerto 21 se encarga de la conexión de control, mientras que el puerto 20 transfiere los datos.
* **Puerto 22 (SSH-Secure Shell):** Servicio utilizado para el acceso y administración remota de servidores de forma segura y cifrada.
* **Puerto 23 (Telnet):** Protocolo de acceso remoto por línea de comandos. Al igual que FTP antiguo, es inseguro porque transmite los datos en texto plano.
* **Puerto 25 (SMTP-Simple Mail Transfer Protocol):** Es el protocolo estándar para el envío y enrutamiento de correos electrónicos a través de internet.
* **Puerto 53 (DNS-Domain Name System):** Servicio fundamental de resolución de nombres que traduce los dominios legibles por humanos (ej. softwareseguro.com.ar)
* en direcciones IP.
* **Puerto 80 (HTTP-Hypertext Transfer Protocol):** Puerto estándar para el tráfico web sin cifrar. La información viaja en texto plano.
* **Puerto 110 (POP3-Post Office Protocol version 3):** Protocolo utilizado por los clientes locales de correo electrónico para recibir y descargar los mensajes
*  desde un servidor.
* **Puerto 443 (HTTPS-HTTP Secure):** Puerto estándar para el tráfico web seguro. Utiliza protocolos de cifrado (como TLS) para proteger la información que viaja
*  entre el cliente y el servidor.
* **Puerto 3306 (MySQL/MariaDB):** Es el puerto por defecto que utilizan estos populares motores de bases de datos relacionales para escuchar y recibir consultas.
