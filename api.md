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

## LLamadas (métodos)

Las llamadas o métodos son identificados por el parámetro **function** del request. El sistema es *"case sensitive"* tanto
para los nombres como los valores de los parámetros.

Muestra de requests mal formados y sus respectivas respuestas:

* `BASE_URL/Provisioning?funtion=getAppVersion  ->` **`1`**`|null not recognized`
<br>"funtion" no es "function"
* `BASE-URL/Provisioning?FUNCTION=getAppVersion ->` **`1`**`|null not recognized`
<br>"FUNCTION" no es "function"
* `BASE-URL/Provisioning?function=getVersion    ->` **`1`**`|getVersion not recognized`
<br>"getVersion" no es método

El API está dividido en dos grandes grupos funcionales, diferenciados únicamente por la presencia del IMEI en el encabezado
del request.

### Grupo de llamadas TigoMoney

*Todos los requests de este grupo deben tener el encabezado HTTP `imei=valor_del_imei`*. En caso contrario serán tratados
como llamadas de mantenimiento e información, generando una excepción debidamente reportada en la respuesta.

Cuando se hable de *CLIENTE* se hará referencia al triplete *USUARIO / CLAVE / IMEI*

Los posibles *códigos APP* en la respuesta se muestran en la tabla siguiente:

Código|Descripción||
:----:|-----------|-|
0|OK|Operación exitosa. Mensaje contiene data pertinente
1|Error|El mensaje muestra lo sucedido y/o la acción a seguir
3|Intente más tarde|El servidor no puede procesar otra petición
4|Sesión inválida|La sesión expiró. Debe reloguear
5|Primer LOGIN|Se indica cambiar la clave del usuario
6|Mantenimiento|Sistema en ventana de mantenimiento
7|Re-confirmación|Transacción ya confirmada

La funcionalidad de Tigo Money requiere la seguridad de un soporte transaccional que ha sido implementada a través de dos
mecanismos complementarios, como se describe a continuación.

* **Sesión**. Para comenzar a operar es preciso establecer una sesión en el servidor por usuario y dispositivo.
Este proceso requiere dos llamadas al API, la primera `getSession` para inicializar una sesión y la siguiente `login` para registrar
el triplete *USUARIO / CLAVE / IMEI* enviando las credenciales adecuadas.  
La duración de la sesión es configurable en `tigo.properties:sesion.timeout`.
* **Transacción**. El cliente deberá iniciar y vincular las transacciones haciendo una llamada previa al método `nextId`. Estas
transacciones son mantenidas por el Provisioning y podrán abarcar más de una llamada al API Provisioning, así como una o más
consultas internas del servidor Provisioning a los webservices *core* de TigoMovil por cada uno de los métodos invocados.  
**Nunca** debe repetirse una llamada a un método transaccional; en caso de no recibir respuesta, debe consultarse por la vía
del método `getTransactionStatus`.  
Dependiendo del número mínimo de llamadas necesario para una culminación exitosa, las transacciones se clasifican como de:  
   1 ***una etapa***  
   2 ***dos etapas***  
   3 ***tres etapas***  
   4 ***tres etapas con verificación***  
La transacción permanece en vigencia hasta la siguiente llamada a `nextId`; debido a esto, se debe tener mucho cuidado de iniciar
la transacción únicamente como se muestra en la tabla siguiente:

En la descripción detallada de los métodos, presentada a continuación, se indicará cuando debe llamarse previamente a `nextId`.


