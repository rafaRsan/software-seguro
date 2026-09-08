# Objetivo 3:
## que es el "Handshake" TLS
primero vamos a definir en rasgos generales que es el "handshake" despues ire punto por punto, el "Handshake" es el proceso que sucede la primera vez que se conecta a una web, 
el cliente(navegador) revisa y valida que los certificados del servidor sean reales y esten en vigencia para definir que el servidor es seguro, despues de esto crea 
una clave secreta con el servidor para asi cifrar toda la comunicacion que pase en la sesion.
## Ahora que es un certificado digital que rol cumple y por que se usan dos tipos de cifrados?
Un certificado digital es una especie de DNI para los servidores, sirven para que los navegadores sepan que el servidor es seguro y tiene sus certificados en vigencia, ya que 
dentro de estos certificados hay una **clave publica** la cual es importante para saber por que se usan dos tipos de cifrados, los **cifrados asimetricos y simetricos** 
pasan la primera vez que un cliente y un servidor se conectan en el Handshake, ya que durante este proceso se crea una clave secreta primero hay que confirmar que 
el canal es seguro, asi que el cliente usa la **clave publica** que viene en los certificados digitales para confirmar que es seguro y dentro de este mensaje 
pasar en secreto la clave secreta que se usara en la sesion, este proceso es el **cifrado asimetrico** y el **cifrado simetrico** es el que pasa una vez 
hecho el asimetrico y habiendo pasado con exito la clave secreta al servidor, ya que hacer el **cifrado asimetrico** constantemente en una sesion es pesado y 
lento para la conexion, asi que con el **cifrado simetrico** se reduce la carga al ya compartir una unica clave secreta entre cliente y servidor
