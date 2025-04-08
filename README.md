# PPS-Unidad3Actividad-ConfiguracionSeguraTLSCifradoDatosAES
Actividad de configuración segura de TLS

Tenemos como objetivo:

> - Ver cómo se pueden hacer ataques .
>
> - Analizar el código de la aplicación que permite ataques de .
>
> - Implementar diferentes modificaciones del codigo para aplicar mitigaciones o soluciones.

## ¿Qué es CSRF?
---

Consecuencias de :
- 
## ACTIVIDADES A REALIZAR
---
> Lee el siguiente [documento sobre Configuración Segura de TLS y Cifrado de Datos Sensibles con AES](./files/ConfiguracionTLSCifradoDatosAES.pdf)
> 
> También y como marco de referencia, tienes [ la sección de correspondiente de pruebas del **Proyecto Web Security Testing Guide** (WSTG) del proyecto **OWASP**.](https://owasp.org/www-project-web-security-testing-guide/v41/4-Web_Application_Security_Testing/09-Testing_for_Weak_Cryptography/01-Testing_for_Weak_SSL_TLS_Ciphers_Insufficient_Transport_Layer_Protection)
>


Vamos realizando operaciones:

### Iniciar entorno de pruebas

-Situáte en la carpeta de del entorno de pruebas de nuestro servidor LAMP e inicia el esce>

~~~
docker-compose up -d
~~~

~~~
docker exec -it lamp-php83 /bin/bash
~~~


~~~
apt update;apt install openssl
~~~


~~~
mkdir -p /etc/apache2/ssl 
openssl req -newkey rsa:2048 -nodes -keyout /etc/apache2/ssl/server.key \
-x509 -days 365 -out /etc/apache2/ssl/server.crt \
-subj "/C=US/ST=State/L=City/O=Organization/OU=Department/CN=localhost" \
-addext "basicConstraints=CA:FALSE" \
-addext "subjectAltName=DNS:localhost"
~~~
![](images/TLS1.png)

![](images/TLS2.png)

Vemos la versión de ssl:
~~~
openssl version
~~~
![](images/TLS3.png)

Y la versión de apache
~~~
dpkg -l |grep apache
~~~
![](images/TLS5.png)

Guardamos el archivo de configuración existente y creamos uno nuevo.
~~~
mv /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl-old
nano /etc/apache2/sites-available/default-ssl.conf
~~~

El nuevo tendrá el siguiente contenido:

~~~
VirtualHost *:443>
        ServerAdmin admin@localhost
        ServerName localhost
        DocumentRoot /var/www/html
        SSLEngine on
        SSLCertificateFile /etc/apache2/ssl/server.crt
        SSLCertificateKeyFile /etc/apache2/ssl/server.key
        SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
        SSLCipherSuite HIGH:!aNULL:!MD5
</VirtualHost>
~~~
![](images/TLS4.png)

Habilitar SSL y reiniciar Apache
Habilitar el módulo SSL y el sitio seguro:


~~~
a2enmod ssl
a2ensite default-ssl
~~~

Reiniciar Apache para aplicar los cambios:
~~~
service apache2 reload
~~~
Comprueba que tu servidor apache tiene el archivo index.html. Si no es así, [descárgalo de aquí.](files/index.html)
![](images/TLS6.png)

Abrimos navegador y lanzamos: `https://localhost/index.html`

Nos muestra advertencia de conexión no privada

![](images/TLS7.png)

Pulsamos en **Avanzado** y **Acceder a localhost (sitio no seguro)**

![](images/TLS8.png)

### Verificar la configuración con SSL Labs (con dominio)

Para asegurarse de que la configuración de TLS es segura, se puede comprobar el dominio en: SSL Labs Test

~~~
<https://www.ssllabs.com/ssltest/>
~~~

![](images/TLS8.png)

### Deshabilitar versiones inseguras de TLS

Para evitar vulnerabilidades, en default-ssl.conf configurar:

SSLProtocol TLSv1.2 TLSv1.3

~~~
nano /etc/apache2/sites-available/default-ssl.conf
~~~


~~~
SSLProtocol TLSv1.2 TLSv1.3
~~~

![](images/TLS10.png)

 y reiniciamos el servicio apache2

~~~
service apache2 reload
~~~

### * Forzar HTTPS con HSTS

Añadir dentro del bloque <VirtualHost *:443>:

~~~
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
~~~

Habilitar los encabezados HTTP y reiniciar Apache con:

~~~
a2enmod headers
service apache2 restart
~~~

![](images/TLS11.png)

 Se nos cerrará el contenedor docker, o sea que volvemos a iniciarlo.

~~~
docker-compose start
~~~

* Verificar la conexión HTTPS
Para comprobar si HTTPS funciona correctamente, acceder a:
https://localhost
Si se observa el candado en la barra de direcciones, TLS está activo.


### ¿Cómo eliminar la advertencia del candado? (Opcional)

Si solo trabajas en local, no hay problema en ignorar la advertencia. Pero si se quiere que el navegador lo reconozca como seguro sin advertencias, dado que Firefox solo permite importar certificados de CA en la pestaña "Authorities", se debe generar un certificado raíz de CA y luego firmar el certificado con él.

1. **Crear un Certificado de Autoridad (CA)**

Ejecutar estos comandos para generar una CA local:

~~~
mkdir -p /etc/apache2/ssl
cd /etc/apache2/ssl

# Generar la clave privada de la CA
openssl genrsa -out myCA.key 2048

# Generar el certificado raíz de la CA (válido por 10 años)
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 3650 -out myCA.pem \ -subj "/C=ES/ST=Extremadura/L=Plasencia/O=CiberseguridadJoseMI/OU=Soy Yo solo /CN=localhost"
~~~

![](images/TLS12.png)

2. **Firmar el Certificado SSL con la CA Local**

Generar una clave para el servidor Apache:

~~~
cd /etc/apache2/ssl
openssl genrsa -out MiServidor.key 2048
~~~

Crear una solicitud de firma (CSR Certificate Signing Request, es un archivo que contiene información sobre una entidad que solicita un certificado SSL/TLS):
~~~
openssl req -new -key MiServidor.key -out MiServidor.csr \ -subj "/C=ES/ST=Extremadura/L=Plasencia/O=CiberseguridadJoseMI/OU=Soy Yo solo /CN=localhost"
~~~

Firmar el certificado con nuestra CA local:

~~~
openssl x509 -req -in MiServidor.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out MiServidor.crt -days 365 -sha256
~~

Ahora disponemos de un certificado server.crt firmado por nuestra CA myCA.pem.

~~~
~~~


## ANEXO II

Se puede verificar de manera local que la configuración TLS está funcionando correctamente, especialmente útil
cuando:
•
•
•
No se dispone de un dominio público.
Se está trabajando en un entorno de desarrollo o laboratorio.
Se quiere confirmar que TLS 1.3 está habilitado y operativo antes de poner el servidor en producción.
openssl s_client -connect localhost:443 -tls1_3
Este comando:
•
 Intenta establecer una conexión TLS específicamente con la versión 1.3.
•
 Muestra un resumen de la negociación TLS, incluyendo:
o La versión del protocolo usada.
o El certificado presentado.
o El conjunto de cifrado negociado.
Si se obtiene en la salida:
Protocol
 : TLSv1.3
Cipher
 : TLS_AES_256_GCM_SHA384
... entonces TLS 1.3 está activo y funcionando.


![](images/TLS13.png)
![](images/TLS14.png)
![](images/TLS10.png)
![](images/TLS10.png)

## Código vulnerable
---



![](images/.png)
![](images/.png)
![](images/.png)
![](images/.png)


### **Código seguro**
---

Aquí está el código securizado:

🔒 Medidas de seguridad implementadas

- :

        - 

        - 



🚀 Resultado

✔ 

✔ 

✔ 

## ENTREGA

> __Realiza las operaciones indicadas__

> __Crea un repositorio  con nombre PPS-Unidad3Actividad6-Tu-Nombre donde documentes la realización de ellos.__

> No te olvides de documentarlo convenientemente con explicaciones, capturas de pantalla, etc.

> __Sube a la plataforma, tanto el repositorio comprimido como la dirección https a tu repositorio de Github.__

