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

# API [/Provisioning]

## Request [ GET | POST ]

La llamada siempre tendrá la forma **`http://BASE_URL/Provisioning`**.

* El parámetro **`function`** es obligatorio e identifica la llamada a realizar.
* Opcionalmente podrán agregarse N parámetros opcionales identificados por la cadena **`p[0-N]`**.
* Se espera que la llamada esté codificada en *UTF-8*

## Response 200 (text/plain;charset=UTF-8)

La respuesta estará conformada por un *código APP* numérico y un mensaje de texto, separados por el caracter **`'|'`**
( barra vertical )
<br>En algunos casos podrán ser añadidos mensajes adicionales usando el mismo separador.

El código HTTP retornado como respuesta a un request bien formado será siempre **`200`**.
La única excepción la constituye el código **`500`**, con el contenido **`Internal server error`** seguido de una o más
excepciones.

El número y contenido de los mensajes dependerá de la llamada específica y su resultado.

>Un *request / response* típico:
<br>`http://192.168.128.216:8080/mfsjava/Provisioning?function=getAppVersion`
<br> **`0`**`|1.3.1`

## LLamadas (métodos)

Las llamadas o métodos son identificados por el parámetro **function** del request. El sistema es *"case sensitive"* tanto
para los nombres como los valores de los parámetros.

Muestra de requests mal formados y sus respectivas respuestas:

* `BASE_URL/Provisioning?funtion=getAppVersion  ->` **`1`**`|null not recognized`
<br>"funtion" no es "function"
* `BASE_URL/Provisioning?FUNCTION=getAppVersion ->` **`1`**`|null not recognized`
<br>"FUNCTION" no es "function"
* `BASE_URL/Provisioning?function=getVersion    ->` **`1`**`|getVersion not recognized`
<br>"getVersion" no es método

El API está dividido en dos grandes grupos funcionales, diferenciados únicamente por la presencia del IMEI en el encabezado
del request.

### Grupo de llamadas TigoMoney

*Todos los requests de este grupo deben tener el encabezado HTTP `imei=valor_del_imei`*. En caso contrario serán tratados
como llamadas de mantenimiento e información, generando una excepción debidamente reportada en la respuesta.

Los posibles *códigos APP* en la respuesta se muestran en la tabla siguiente:

Código|Descripción||
:----:|-----------|-|
0|OK|Operación exitosa. Mensaje contiene data pertinente
1|Error|El mensaje indica lo sucedido y/o la acción a seguir
3|Intente más tarde|El servidor no puede procesar otra petición
4|Sesión inválida|La sesión expiró. Debe reloguear
5|Primer LOGIN|Se indica cambiar la clave del usuario
6|Mantenimiento|Sistema en ventana de mantenimiento

La funcionalidad de Tigo Money requiere soporte transaccional y para acceder a los métodos definidos en este grupo es
preciso establecer una sesión en el servidor.

#### getSession
* request: BASE_URL/Provisioning?funtion=<strong>getSession</strong>
* response: 0 ( no hay mensaje )
<br>En el encabezado HTTP de la respuesta vendrá el header "Set-Cookie". Deberá:
    1. Guardar el valor de JSESSIONID
    2. Agregarlo como valor del encabezado "Cookie" en cada uno de los requests subsiguientes.
  
![alt text](http://cirth.radiumtec.com/honnduras/getSessionRequest.png "getSession headers")  
En la imagen pueden observarse el header **imei** en el request y el header **Set-Cookie** en el response





### Grupo de llamadas de mantenimiento e información

Para acceder a los métodos declarados en este grupo se requiere hacerlo desde una IP listada en la propiedad **valid.ip.list**
del archivo *tigo.properties*.

Los requests de este grupo NO deben tener el encabezado HTTP **`imei=valor_del_imei`**

