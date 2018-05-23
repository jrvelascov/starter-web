>>>> _**Documento**: API Provisioning (api.md)_  
     _**Versión**: 0.5.0_

>>>> _**BASE-URL**: http://192.168.128.216:8080/mfsjava_

# Provisioning

Provisioning es una aplicación java EE7 cuya misión básica es intermediar e integrar los webservices Tigo Money para
la aplicación cliente desplegada en los dispositivos MP2.

Actualmente se encuentra instalada en un servidor de aplicaciones Glassfish 4.1, accesible para HTTP en el puerto 8080
y es configurable vía dos archivos de texto plano.
* **`tipo.properties`** donde se mantienen las constantes de operación ( endpoints internos, mensajes de error,
timeouts, white lists, etc )
* **`tigo.receipts.properties`** que mantiene las plantillas para los recibos de las diferentes transacciones.

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

En algunos casos especiales, cualquiera de los mensajes de texto podrá ser subdividido a su vez usando como separador una
de las secuencias:
* **<¿>**
* **\t** ( tab )

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

* `BASE_URL/Provisioning?funtion=getAppVersion  ->` **`1`**`|called no function`
<br>"funtion" no es "function"
* `BASE-URL/Provisioning?FUNCTION=getAppVersion ->` **`1`**`|called no function`
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

Método invocado|Llamar `nextId` previamente
------|:---------:
getSession|**NO**
login|**NO**
getTemplates|**NO**
changePin|SI
getAccount|SI
autoRegister|SI
getLastTransactions|SI
getReceiptCopy|SI
getBalance|SI
getAgentStatement|SI
viewBillers|SI
getTransactionStatus|**NO**
confirm|**NO**
automaticReversal|**NO**
cashin|SI
cashinothers|SI
cashinagents|SI
agentSendMoney|SI
creditDebit|SI
payBill|SI
merchantPayment|SI
cashout|SI

En la descripción detallada de los métodos, presentada a continuación, se indicará cuando debe llamarse previamente a `nextId`.

><br> *Los dos métodos siguientes,* **`getSession`** *y* **`login`** *son transacciones de* ***una etapa*** *y conforman el
protocolo de inicio de sesión.  
Eventualmente, dependiendo de la respuesta a* **`login`**, *podría ser necesario llamar a* **`getTemplates`** *( de una etapa )*  
><br>

#### getSession - obtiene el JSESSIONID
* request: `BASE_URL/Provisioning?funtion=`<strong>`getSession`</strong>
* response: `0` ( no hay mensaje )  
  Entre los encabezados HTTP de la respuesta vendrá el header "Set-Cookie". Deberá:
    1. Extraer y guardar literalmente el string "`JSESSIONID=hashquedefineeliddelasesion`"
    2. Agregarlo como valor del encabezado "Cookie" en todos los requests subsiguientes.
  
![alt text](http://cirth.radiumtec.com/honnduras/getSessionRequest.png "getSession headers")  
En la imagen pueden verse tanto el header **imei** en el request como el header **Set-Cookie** en el response.

><br> *Para mantener la sesión, a partir de la siguiente llamada,* **`login`**, *TODOS los requests deberán llevar el encabezado*
**_Cookie_** *con el valor "JSESSIONID=hashquedefineeliddelasesion" además del antes mencionado encabezado* **_`imei`_**.  
><br>

#### login - autentica y registra al *CLIENTE* en la sesión
* request: `BASE_URL/Provisioning?funtion=`<strong>`login`</strong>`&p0=msisdn&p1=pin`  
  Parámetros extra:
    1. **`p0`** : msisdn del agente
    2. **`p1`** : pin del agente
* response: una cualquiera de
    * `0|comentario|opciones.menu<¿>cashinagents.fields<¿>receipt.version<¿>graphic.print|agente`  
    * `5|comentario|opciones.menu<¿>cashinagents.fields<¿>receipt.version<¿>graphic.print|agente`  
	  Donde las respuesta **`0`** y **`5`** indican login exitoso, pero en el caso de **`5`** debe hacerse un cambio de clave
	  usando el método `changePin` antes de continuar.  
	  El resto de los campos se describe a continuación:
		 1. **`comentario`** : mensaje proveniente del webservice `ValidatePin`
		 2. **`opciones.menu`** : listado de las opciones de menú disponibles para el usuario, separadas por espacios  
		 en blanco. Corresponden al campo `opcion.id` de la base de datos y deben usarse para armar el menú de opciones.
		 Ej. `1 2 4 6 7 8 10 13 17`
		 3. **`cashinagents.fields`** : son 4 dígitos binarios ( Ej. `1011` ) usados para armar la pantalla de toma de datos
       correspondiente al servicio *CASHINAGNT*. En orden estricto determinan la presencia ( `1` ) o no ( `0` ) de los campos:
			  1. Teléfono del beneficiario
			  2. Identificación del beneficiario
			  3. Teléfono del remitente
			  4. Identificación del remitente
		 4. **`receipt.version`** : número de versión de las plantillas para impresión de recibos. En caso de que el dispositivo  
		 no tenga almacenadas las plantillas, o que las versiones no coincidan, deberá descargar la nueva versión usando el método
		 `getTemplates`, en "background" para evitar retrasos innecesarios en la IU.
		 5. **`graphic.print`** : `true / false` - estrictamente para los dispositivos MP2. Determina el modo de impresión
		 gráfico en caso de `true`.
		 6. **`agente`** : nombre completo del agente
    * `1|comentario`  
	  En este caso la respuesta **`1`** indica que no fue exitoso el `login`. El campo `comentario` describe la circunstancia
	  (credenciales erróneas, dispositivo no autorizado, etc) y las posibles acciones a seguir.
    * `3`  
	  Sistema no dispone de ejecutores. Debe reintentarlo luego.
    * `6|comentario`  
	  El sistema está en mantenimiento. El mensaje es configurable en `tigo.properties:maintain.window.mess`.

#### getTemplates - obtiene las plantillas para impresión.
* request: `BASE_URL/Provisioning?funtion=`<strong>`getTemplates`</strong>
* response: `0|header<¿>`  
    `cashin-0<¿>cashin-1<¿>cashin-2<¿>cashin-3<¿>`  
    `cashinothn-0<¿>cashinothn-1<¿>cashinothn-2<¿>cashinothn-3<¿>`  
    `cashinagnt-0<¿>cashinagnt-1<¿>cashinagnt-2<¿>cashinagnt-3<¿>`  
    `cashout-0<¿>cashout-1<¿>cashout-2<¿>cashout-3<¿>`  
    `uonlinebp-0<¿>uonlinebp-1<¿>uonlinebp-2<¿>uonlinebp-3<¿>`  
    `merchpay-0<¿>merchpay-1<¿>merchpay-2<¿>merchpay-3<¿>`  
    `p2p-0<¿>p2p-1<¿>p2p-2<¿>p2p-3<¿>`  
    `transcomision-0<¿>transcomision-1<¿>transcomision-2<¿>transcomision-3`  
	 Donde aparecen, en orden y separadas por `<¿>`, las plantillas para el encabezado y las 8 operaciones de TigoMoney
    implementadas.  
	 Se recomienda hacer esta llamada en **background**, inmediatamente después de recibir la aprobación de `login`, mientras
	 se dibuja la IU.
    Los sufijos identifican, para cada operación:
    * `-0` : Pantalla de confirmación.
    * `-1` : Pantalla de operación realizada.
    * `-2` : Recibo impreso.
    * `-3` : Copia del recibo impreso.

><br> *En este punto ya queda formalizada la sesión en el servidor.*  
><br>
><br> *Los ocho métodos siguientes,* **`changePin`**, **`getAccount`**, **`autoRegister`**, **`getLastTransactions`**, **`getReceiptCopy`**,
 **`getBalance`**, **`getAgentStatement`** y **`viewBillers`** *son transacciones de* ***dos etapas*** *y tienen el flujograma:*  
>> `nextId`  
	`changePin` / `getAccount` / `autoRegister` / `getLastTransactions` / `getReceiptCopy`, / `getBalance` / `getAgentStatement` / `viewBillers`  
><br>

#### changePin - cambia clave para un *CLIENTE* (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`changePin`</strong>`&p0=msisdn&p1=oldPin&p2=newPin`  
  Parámetros extra:
    1. **`p0`** : msisdn del agente
    2. **`p1`** : pin del agente
    3. **`p2`** : nuevo pin
* response: `0|mensaje`  
  Si el código de respuesta es distinto de **`0`**, no se pudo cambiar la clave. El mensaje indicará la acción a seguir.

#### getAccount - obtiene los datos de cuenta de un *CLIENTE* (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`getAccount`</strong>
* response: `0|Nombre:\nnombreAgente,apellidoAgente\nTeléfono:\nmsisdn\nFecha de registro:\n%s`  
  La aplicación MP2 interpreta el literal `"\n"` como un salto de línea.  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará la acción a seguir.

#### autoRegister - registra usuarios (`nextId`)
* request: uno cualquiera de  
  `?funtion=`<strong>`autoRegister`</strong>`&p0=nombre&p1=apellido&p2=pasaporte&p3=msisdn&p4=fechaNac`  
  `?funtion=`<strong>`autoRegister`</strong>`&p0=&p1=&p2=identidad&p3=msisdn&p4=fechaNac`  
  Parámetros extra:  Todos los parámetros listados deben estar presentes. Cuando se indique, deberán dejarse en blanco.  
  Hay dos formas de registrar al usuario. El tipo de registro queda definido por la presencia o no de valor para el  
  parámetro `p0`:
    * Por pasaporte
        1. **`p0`** : Nombre
        2. **`p1`** : Apellido
        3. **`p2`** : Número de pasaporte
        4. **`p3`** : Teléfono
        5. **`p4`** : Fecha de nacimiento
    * Por identidad
        1. **`p0`** : 
        2. **`p1`** : 
        3. **`p2`** : Número de identidad
        4. **`p3`** : Teléfono
        5. **`p4`** : Fecha de nacimiento
* response: cualquiera entre  
  `0|"Nombre completo:\nNombre\nNúmero de teléfono:\nmsisdn\nNúmero de pasaporte:\npasaporte\nFecha de nacimiento:\nfechaNac`  
  `0|"Nombre completo:\nNombre\nNúmero de teléfono:\nmsisdn\nNúmero de identidad:\nidentidad\nFecha de nacimiento:\nfechaNac`  
  Si el código de respuesta es distinto de **`0`**, no se pudo registrar al usuario. El mensaje indicará la acción a seguir.

#### getLastTransactions - obtiene las últimas transacciones de un *CLIENTE* (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`getLastTransactions`</strong>
* response: `0|listado de transacciones`  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará la acción a seguir.  
  El listado de transacciones, que puede estar en blanco, consiste en una lista separada por líneas en
  blanco, donde cada `TRANSACCION` es:

```
       serviceDescription              ( cashin, cashout, etc )
       Fecha: dd-MM-yy hh:mm:ss
       Referencia:
       código transacción TigoMoney
       __Si es pago de servicio_
       	Servicio: billCompany
       	Cuenta: msisdn
       __Si NO es pago de servicio_
       	Cliente: msisdn
       Monto: L 9.99
       Cargo: L 9.99
       Comisión: L 9.99
```

#### getReceiptCopy - obtiene el recibo de impresión para una transacción (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`getReceiptCopy`</strong>`&p0=tigo.transId`  
  Parámetros extra:
    1. **`p0`** : código **TigoMoney** de la transacción
* response: `0|recibo.pantalla<¿>recibo.impresión<¿>recibo.impresión.copia|statusTigoMoney`  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  Este método NO puede ser usado para imprimir la transacción actual, ya que requiere `nextId` previo, la ejecución tendría
  lugar dentro de una nueva transacción.  
  Se usa exclusivamente para imprimir recibos de transacciones anteriores cuando sea necesario y, generalmente, de la
  respuesta sólo interesa tomar el elemento `recibo.impresión.copia`.

#### getBalance - obtiene el saldo de las billeteras de un *CLIENTE* (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`getBalance`</strong>
* response: `0|saldo.principal|saldo.comisiones`  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  

#### getAgentStatement - obtiene el reporte de transacciones de un *CLIENTE* (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`getAgentStatement`</strong>`&p0=tipo`  
  Parámetros extra:
    1. **`p0`** : entero que define el tipo de reporte de acuerdo a:
        * `0` : Cierre
        * `1` : Día actual
        * `2` : Día anterior
        * `3` : Semana actual
        * `4` : Semana anterior
        * `5` : Mes actual
        * `6` : Mes anterior
* response: `0|reporte`  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  Los diferentes reportes vienen formateados para su impresión por el dispositivo.

#### viewBillers - obtiene el listado (XML) de proveedores de servicio (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`viewBillers`</strong>
  response: `0|reporte`  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  El listado es un XML bien formado del cual se ofrece un ejemplo reducido:

```
        <?xml version="1.0" encoding="UTF-8"?>
        <sch1:billers>
           <sch1:biller>
              <sch1:id>MR1011181336001</sch1:id>
              <sch1:code>00</sch1:code>
              <sch1:shortName>POSTPAID</sch1:shortName>
              <sch1:fullName>Tigo Mobile</sch1:fullName>
              <sch1:categoryCode>1</sch1:categoryCode>
              <sch1:categoryName>Pagos Tigo</sch1:categoryName>
              <sch1:partialPayment>Y</sch1:partialPayment>
              <sch1:viewBill>Y</sch1:viewBill>
           </sch1:biller>
           <sch1:biller>
              <sch1:id>MR1201232211002</sch1:id>
              <sch1:code>27</sch1:code>
              <sch1:shortName>WSPS</sch1:shortName>
              <sch1:fullName>Aguas de San Pedro Sula</sch1:fullName>
              <sch1:categoryCode>2</sch1:categoryCode>
              <sch1:categoryName>Servicios Publicos</sch1:categoryName>
              <sch1:partialPayment>N</sch1:partialPayment>
              <sch1:viewBill>Y</sch1:viewBill>
           </sch1:biller>
        </sch1:billers>
```

><br> *Los siguientes cuatro métodos,* **`nextId`**, **`getTransactionStatus`**, **`confirm`** *y* **`automaticReversal`**
 *son transacciones de* ***una etapa*** *y conforman el protocolo de control transaccional.  
Los tres últimos actúan de manera implícita sobre la transacción en proceso, iniciada previamente por* **`nextId`**.  
><br>

#### nextId - *inicia* y vincula una transacción de Provisioning
* request: `BASE_URL/Provisioning?funtion=`<strong>`nextId`</strong>
* response: `0|transID`  
  Donde **`transID`** es la identidad de la transacción ( Provisioning )  
  Sólo si se recibe el código de respuesta **`0`** puede continuar con la llamada al método transaccional. En caso conrtario,
  el mensaje indicará la acción a seguir ( reiniciar sesión, sistema en mantenimiento, etc )

#### getTransactionStatus - obtiene el *estado* de la transacción en proceso
* request: `BASE_URL/Provisioning?funtion=`<strong>`getTransactionStatus`</strong>
* response: `0|mensaje|estado`  
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  Se usa para verificar el estado de transacciones de tres etapas que requieren verificación, en concreto _MERCHPAY_ y _CASHOUT_.  
  **`estado`** : puede ser uno cualquiera de los estados TigoMovil: `TI`, `TF`, `TS` o `TC`.  
  **`mensaje`** : el contenido dependerá del `estado` de la transacción como se especifica en la tabla siguiente.

Estado|Transacción:|mensaje
:----:|:----------:|:---------:
`TI`|iniciada| En blanco
`TF`|fallida|Texto con el motivo
`TS`|exitosa|`tigo.transId\tyy-MM-dd\tHH:mm:ss` [ separado por \t ( tab ) ]
`TC`|cancelada|No ocurre en operación normal

```
Y donde:
   tigo.transId : código TigoMovil
   yy-MM-dd     : fecha TigoMovil
   HH:mm:ss     : hora TigoMovil
de la transacción exitosa
```

#### confirm - *confirma* la transacción en proceso
* request: `BASE_URL/Provisioning?funtion=`<strong>`confirm`</strong>
* response: `0|tigo.transId\tyy-MM-dd\tHH:mm:ss`  
  Donde los campos, separados por "`\t`" son los descritos en el método `getTransactionStatus` para una transacción exitosa,
  de TigoMovil el código, la fecha y la hora de la transacción ya confirmada.  
  En el caso de transacciones que requieren verificación, como *CASHIN* y *MERCHPAY*, este valor de la hora no es relevante,
  puesto que hay que esperar por la verificación ( incluso la fecha podría variar )
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  

#### automaticReversal - *revierte* la transacción en proceso
* request: `BASE_URL/Provisioning?funtion=`<strong>`automaticReversal`</strong>
* response: `0|estado`  
  Donde **`estado`** es el estado TigoMovil de la transacción y puede tomar cualquiera de los valores descritos en el
  método `getTransactionStatus`.  
  Se garantiza que la transacción ha sido ya revertida previamente al retorno de la llamada, en cuyo caso `estado` será "`TC`",
  o bien encolada para su reversión por el *core* TigoMovil, siendo su `estado` cualquiera de los válidos.

><br> *Los siguientes seis métodos,* **`cashin`**, **`cashinothers`**, **`cashinagents`**, **`agentSendMoney`**, **`creditDebit`**,
 **`payBill`** *son transacciones de* ***tres etapas*** *y tienen el flujograma:*
>> `nextId`  
	 `cashin` / `cashinothers` / `cashinagents` / `agentSendMoney` / `creditDebit` / `payBill`  
	 `confirm`  
><br>

#### cashin - *deposita* dinero a suscriptores registrados (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`cashin`</strong>`&p0=teléfono&p1=monto`  
  Parámetros extra:
    1. **`p0`** : número de teléfono del receptor
    2. **`p1`** : monto de la transacción
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\t\t\t`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 * **`nombre.cliente`** : nombre del receptor  
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

#### cashinothers - *deposita* dinero a suscriptores registrados o no (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`cashinothers`</strong>`&p0=beneficiario&p1=remitente&p2=remitente.id&p3=monto`  
  Parámetros extra:
    1. **`p0`** : número de teléfono del beneficiario
    2. **`p1`** : número de teléfono del remitente
    3. **`p2`** : cédula de identidad del remitente
    4. **`p3`** : monto de la transacción
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\t\t\t`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene un mensaje separado por "`\t`" ( tab )
	 * **`nombre.cliente`** : nombre del beneficiario ( podría no estar registrado )  
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

#### cashinagents - *deposita* dinero a suscriptores registrados o no (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`cashinagents`</strong>`&p0=remitente&p1=remitente.id&p2=beneficiario&
  p3=beneficiario.id&p4=monto&p5=método`  
  Dependiendo de la configuración ( `tigo.properties:cashin.agents.fields` ) puede actuar como `cashin` o como `cashinothn`.  
  Los seis parámetros deben estar referenciados en la llamada; no son opcionales, pero pueden o no estar en blanco dependiendo
  del contenido del campo `cashinagents.fields` en la respuesta a `login`, que define también el valor del parámetro `p6`  
  Parámetros extra:
    1. **`p0`** : número de teléfono del remitente
    2. **`p1`** : cédula de identidad del remitente
    3. **`p2`** : número de teléfono del beneficiario
    4. **`p3`** : cédula de identidad del beneficiario
    5. **`p4`** : monto de la transacción
    6. **`p5`** : tipo. Respetando el *case*, uno cualquiera de los literales
         * `CASHIN` - si los campos del remitente están en blanco
         * `CASHINOTHN` - si aparece cualquiera de los datos del remitente
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\t\t\t`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene un mensaje separado por "`\t`" ( tab )
	 * **`nombre.cliente`** : nombre del beneficiario ( podría no estar registrado )  
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

#### agentSendMoney - *envía* dinero a otro agente (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`agentSendMoney`</strong>`&p0=agente.receptor&p1=monto`  
  Parámetros extra:
    1. **`p0`** : número de teléfono del agente receptor
    2. **`p1`** : monto de la transacción
* response: `0|nombre.agente\tmonto\tfee\tmonto.total\t\t\t`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene un mensaje separado por "`\t`" ( tab )
	 * **`nombre.agente`** : nombre del agente receptor
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

#### creditDebit - *transfiere* dinero entre billeteras de un agente (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`creditDebit`</strong>`&p0=monto`  
  Estrictamente, de la billetera principal a la de comisiones.
  Parámetros extra:
    1. **`p0`** : monto de la transacción
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\t\t\t`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene un mensaje separado por "`\t`" ( tab )
	 * **`nombre.agente`** : no se envía. Ya fue enviado durante el `login`.
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

#### payBill - *paga* servicios públicos, Tigo y colecturías (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`payBill`</strong>`&p0=código&p1=corto&p2=ref&
  p3=monto&p4=teléfono&p5=largo&p6=factura&p7=parcial`  
  Se hace referencia al contenido de los campos de cualquier elemento `<biller>` ( de la respuesta a `viewBillers` )  
  Parámetros extra:
    1. **`p0`** : `<code>`
    2. **`p1`** : `<shortName>`
    3. **`p2`** : cuenta a pagar
    4. **`p3`** : monto del pago ( puede estar en blanco )
    5. **`p4`** : teléfono a notificar
    6. **`p5`** : `<fullName>`
    7. **`p6`** : `<viewBill>`
    8. **`p7`** : `<partialPayment>`
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\t\t\t|monto.total.sin.formato`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 Hay un caso particular para el código de respuesta `1`; cuando es posible ver la factura, se permiten pagos parciales,
	 y el monto está en blanco, si el cliente no tiene deuda pendiente se envía la respuesta:  
	 ***`1|Servicio: nombre.compañía\nCuenta: referencia\nCliente: nombre.cliente\n\nNo tiene deuda pendiente, puede abonar.|0.00`***  
	 La respuesta **`0`** tiene dos mensajes separados por "`|`". El primero de ellos está separado por "`\t`" ( tab )
	 * **`nombre.cliente`** : nombre del cliente
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )
	 * **`monto.total.sin.formato`** : monto total no necesariamente formateado como moneda.

><br> *Los siguientes métodos,* **`merchantPayment`** *y* **`cashout`** *son transacciones de tres etapas* ***con verificación*** *y tienen el flujograma:*
>> `nextId`  
	 `merchantPayment` / `cashout`  
	 `confirm`  
	 ***loop*** `getTransactionStatus`  
><br>

#### merchantPayment - envía *solicitud de pago a comercio* a un suscriptor (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`merchantPayment`</strong>`&p0=teléfono&p1=monto`  
  Parámetros extra:
    1. **`p0`** : número de teléfono del remitente
    2. **`p1`** : monto de la transacción
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\t\t\t`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene un mensaje separado por "`\t`" ( tab )
	 * **`nombre.cliente`** : nombre del receptor  
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

#### cashout - envía *solicitud de retiro* a un suscriptor (`nextId`)
* request: `BASE_URL/Provisioning?funtion=`<strong>`cashout`</strong>`&p0=suscriptor&p1=monto&p2=monto.exacto`  
  Parámetros extra:
    1. **`p0`** : teléfono del suscriptor
    2. **`p1`** : monto del retiro
    3. **`p2`** : tipo. Uno cualquiera de los literales
         * `true` - aplicar cálculo exacto.
         * `false` - no aplicarlo
* response: `0|nombre.cliente\tmonto\tfee\tmonto.total\tmonto.total\tmonto`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene un mensaje separado por "`\t`" ( tab )
	 * **`nombre.cliente`** : nombre del cliente
	 * **`monto`** : monto de la transación  
	 * **`fee`** : cargos  
	 * **`monto.total`** : monto total ( `monto` + `fee` )

### Grupo de llamadas de mantenimiento e información

Para acceder a los métodos declarados en este grupo se requiere hacerlo desde una IP listada en la propiedad **valid.ip.list**
del archivo *tigo.properties*.

Los requests de este grupo NO deben llevar el encabezado HTTP **`imei`**. En ese caso la respuesta,   
configurable en `tigo.properties:mess.session.expired`, será siempre:

>
> **`4`**`|mensaje de expiración`   
>

Los posibles *códigos APP* en la respuesta se muestran en la tabla siguiente:

Código|Descripción||
:----:|-----------|-|
0|OK|Operación exitosa. Mensaje contiene data pertinente
1|Error en request|El mensaje muestra lo sucedido y/o la acción a seguir
2|Base de datos|Violación de `constraints` ( duplicación, inexistencia, etc )
3|Error de sistema|El mensaje contiene la excepción
4|Sesión inválida|Acceso con encabezado `imei`

#### getAppVersion - obtiene la versión de Provisioning
* request: `BASE_URL/Provisioning?funtion=`<strong>`getAppVersion`</strong>
* response: `0|9.9.9`  
    Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
	 La respuesta `0` tiene el número de versión ( mayor . menor . menor )

#### setMaintenance - inicia / termina una ventana de mantenimiento
* request: `BASE_URL/Provisioning?funtion=`<strong>`setMaintenance`</strong>`&set=valor`  
  Parámetros extra:
    1. **`set`** : Puede tomar cualquiera de los literales ( case insensitive )
         * `Y` - iniciar ventana de mantenimiento
         * `N` - terminar la ventana
* response:
    * `0|Mantenimiento on)` - si el parámetro `set=Y`
    * `0|Mantenimiento off)` - si `set=N`
    * Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.   
	 Ej. `1|unauthorized IP`

#### addUser - da de alta un usario [ msisdn-imei ]
* request: `BASE_URL/Provisioning?funtion=`<strong>`addUser`</strong>`&msisdn=MSISDN&imei=IMEI&id_perfil=PERFIL`  
  Parámetros extra:
    1. **`msisdn`** : teléfono del usuario ( entre 8 y 15 dígitos )
    2. **`imei`** : IMEI del dispositivo ( exactamente 15 dígitos )
    3. **`id_perfil`** : ID del perfil asignado ( existente en la BD `'honduras'`.`'perfil'.'id'` )
* response: `0|OK`   
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  Ej:
    * req. **->** `BASE_URL/Provisioning?funtion=addUser&msisdn=94609426&imei=000AA01&id_perfil=1`   
      res. **<-** `1|wrong parameters - bad imei!` - ( 000AA01 no tiene 15 dígitos)
    * req. **->** `BASE_URL/Provisioning?funtion=addUser&msisdn=94609426&imei=000000000000000`   
      res. **<-** `1|addUser-id_perfil` ( ausencia del parámetro id_perfil )
    * req. **->** `BASE_URL/Provisioning?funtion=addUser&msisdn=94609426&imei=354648020000251&id_perfil=1`   
      res. **<-** `2|ConstraintViolationException: Duplicate entry '94609426-354648020000251' for key 'usuario_msisdn_imei_idx'`   
      ( El usuario existe )

#### deleteUser - da de baja un usario [ msisdn-imei ]
* request: `BASE_URL/Provisioning?funtion=`<strong>`deleteUser`</strong>`&msisdn=MSISDN&imei=IMEI`  
  Parámetros extra:
    1. **`msisdn`** : teléfono del usuario ( entre 8 y 15 dígitos )
    2. **`imei`** : IMEI del dispositivo ( exactamente 15 dígitos )
* response: `0|OK`   
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  Ej:
    * req. **->** `BASE_URL/Provisioning?funtion=deleteUser&msisdn=94609426&imei=000000000000000`   
      res. **<-** `2|ConstraintViolationException: No entry '94609426-000000000000000' for key 'usuario_msisdn_imei_idx'`   
      ( El usuario no existe )

#### updateUser - modifica el perfil de un usario  [ msisdn-imei ]
* request: `BASE_URL/Provisioning?funtion=`<strong>`updateUser`</strong>`&msisdn=MSISDN&imei=IMEI&id_perfil=PERFIL`  
  Parámetros extra:
    1. **`msisdn`** : teléfono del usuario ( entre 8 y 15 dígitos )
    2. **`imei`** : IMEI del dispositivo ( exactamente 15 dígitos )
    3. **`id_perfil`** : ID del nuevo perfil ( existente en la BD `'honduras'`.`'perfil'.'id'` )
* response: `0|OK`   
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  Ej:
    * req. **->** `BASE_URL/Provisioning?funtion=updateUser&msisdn=94609426&imei=000000000000000&id_perfil=90`   
      res. **<-** `2|no such user: 94609426-000000000000000` ( El usuario no existe )
    * req. **->** `BASE_URL/Provisioning?funtion=updateUser&msisdn=94609426&imei=354648020000251&id_perfil=90`   
      res. **<-** `2|ConstraintViolationException: Cannot add or update a child row: a foreign key constraint fails
      ( ``honduras ``. ``usuario ``, CONSTRAINT  ``fk_usuario_perfil `` FOREIGN KEY ( ``id_perfil ``)
      REFERENCES  ``perfil `` ( ``id ``))`   
      ( En la tabla `honduras.perfil` no hay un `id=90` )

#### findMsisdn - obtiene los usarios asociados a un *msisd*
* request: `BASE_URL/Provisioning?funtion=`<strong>`findMsisdn`</strong>`&msisdn=MSISDN`  
  Parámetros extra:
    1. **`msisdn`** : teléfono del usuario ( entre 8 y 15 dígitos )
* response: `0|usuarios.lista`   
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  **`usuarios.lista`** : lista de usuarios separados por "`,`". Una lista está delimitada por corchetes **`[ ... ]`** y una
  lista vacía es `[]`   
  **`usuario`** : `Usuario[ id=ID msisdn=MSISDN imei=IMEI id_perfil=ID ]`   
  Ej:
    * **->** `BASE_URL/Provisioning?funtion=findMsisdn&msisdn=94609426`
    * **<-** `0|[Usuario[ id=4 msisdn=94609426 imei=0000000000 id_perfil=1 ], Usuario[ id=91 msisdn=94609426 imei=000000000000000 id_perfil=2 ],
       Usuario[ id=5 msisdn=94609426 imei=1111111111 id_perfil=1 ], Usuario[ id=22 msisdn=94609426 imei=354648020000251 id_perfil=1 ],
       Usuario[ id=50 msisdn=94609426 imei=359128041339579 id_perfil=1 ]]`
    * **->** `BASE_URL/Provisioning?funtion=findMsisdn&msisdn=99999999`   
      **<-** `0|[]` ( no hay usuarios cuyo `msisdn=99999999`)

#### findImei - obtiene los usarios asociados a un *imei*
* request: `BASE_URL/Provisioning?funtion=`<strong>`findImei`</strong>`&imei=IMEI`  
  Parámetros extra:
    1. **`imei`** : IMEI del dispositivo ( exactamente 15 dígitos )
* response: `0|usuarios.lista`   
  Si el código de respuesta es distinto de **`0`**, el mensaje indicará lo ocurrido y la acción a seguir.  
  **`usuarios.lista`** : lista de usuarios separados por "`,`". Una lista está delimitada por corchetes **`[ ... ]`** y una
  lista vacía es `[]`   
  **`usuario`** : `Usuario[ id=ID msisdn=MSISDN imei=IMEI id_perfil=ID ]`   
  Ej:
    * **->** `BASE_URL/Provisioning?funtion=findImei&imei=000000000000000`
    * **<-** `0|[Usuario[ id=91 msisdn=94609426 imei=000000000000000 id_perfil=2 ]]`