# Provisioning

Provisioning es una aplicación java EE7 cuya misión básica es intermediar e integrar los webservices Tigo Money para
la aplicación cliente desplegada en los dispositivos MP2.

Actualmente se encuentra instalada en un servidor de aplicaciones Glassfish 4.1, accesible para HTTP en el puerto 8080
y es configurable vía dos archivos de texto plano.
* **`tipo.properties`** donde se mantienen las constantes de operación ( endpoints internos, mensajes de error,
timeouts, white lists, etc )
* **`tigo.receipts.properties`** que mantiene los modelos para los recibos de las diferentes transacciones.

Ambos archivos deben cumplir con el formato
[java property file](https://docs.oracle.com/cd/E23095_01/Platform.93/ATGProgGuide/html/s0204propertiesfileformat01.html).
La aplicación dispone de valores por defecto para todas las constantes para el caso de que cualquiera de los archivos
no esté disponible.

Para que las actualizaciones en los valores de las constantes sean tenidas en cuenta, se requiere reiniciar la aplicación.

# API [`/Provisioning`]

## Request [ GET | POST ]

La llamada siempre tendrá la forma **`http://BASE-URL/Provisioning`**.

* El parámetro **`function`** es obligatorio e identifica la llamada a realizar.
* Opcionalmente podrán agregarse N parámetros opcionales identificados por la cadena **`p[0-N]`**.
* Se espera que la llamada esté codificada en *UTF-8*

## Response 200 (`text/plain;charset=UTF-8`)

La respuesta estará conformada por un *código APP* numérico y hasta 3 mensajes de texto, separados por el caracter **`'|'`**
( barra vertical )  
En algunos casos especiales, cualquiera de los mensajes de texto podrá ser subdividido a su vez usando el separador **"<¿>"**
o el separador **"\t" ( tab )**.

El código HTTP retornado como respuesta a un request bien formado será siempre **`200`**.
La única excepción la constituye el código **`500`**, con el contenido **`Internal server error`** seguido de una o más
excepciones.

El número y contenido de los mensajes dependerá de la llamada específica y su resultado.

>Un *request / response* típico:  
--> `http://192.168.128.216:8080/mfsjava/Provisioning?function=getAppVersion`  
<-- **`0`**`|1.3.1`
