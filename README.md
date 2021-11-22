# Cómo la Tesorería General de la Republica de Chile usa Amazon Cognito, Lambda y DynamoDB para abstraer la autenticación de sus aplicaciones

La Tesorería General de la República (TGR) de Chile es la institución pública dependiente del Miniserio de Hacienda encargada de recolectar, custodiar y distribuir fondos y valores fiscales de los diferentes servicios públicos del país. Es responsable de darle a la ciudanía mecanismos para efectuar el pago de obligaciones al fisco y de pagar las obligaciones del gobierno a otras instituciones.

## El desafío

En el 2010 el Gobierno de Chile lanzó ClaveÚnica, una contraseña unificada para autenticar de manera digital la identidad de los ciudadanos chilenos y residentes mayores de catroce años. CláveÚnica sirve como puerta de entrada para todos los servicios virtuales que el Estado de Chile provee y tiene como objetivo facilitar el acceso a los trámites de manera digital,  aumentar las medidas de seguridad  y reducir los tiempos de espera. La TGR al ser uno de esos proveedores de servicios digitales se vió obligada a integrar sus aplicaciones usando ClaveÚnica y deprecar lentamente su sistema de autenticación legado.

## OpenID Connect

Tras bambalinas ClaveÚnica es un proveedor de identidad que implementa el protocolo OpenID Connect (OIDC). OIDC es una capa de identidad sobre OAuth 2.0 que le permite a las aplicaciones de terceros verificar la identidad del usuario final y obtener información básica del mismo. OIDC utiliza JSON Web Tokens (JWT) que obtiene mediante flujos que cumplen con las especificaciones de OAuth 2.0. El siguiente diagrama de sequencia resume el flujo de autenticación OIDC:

<p align="center">
  <br/>
  <img src="https://raw.githubusercontent.com/jorsj/tgr/master/oidc.svg" alt="Flujo OIDC"/>
  <br/>
</p>

1. El *Cliente* inicia el flujo con la redirección del usuario a la URL del *Servidor de Autorización*, junto con ello envía los *Scopes*;  permisos granulares que el *Cliente* requiere que el usuario le otorgue. En este caso, para indicar que es un flujo OIDC, el *Cliente* incluye el *Scope* "oidc".
2. El usuario se autentica en el formulario de login del *Servidor de Autorización* y confirma que quiere otorgar los *Scopes* que el *Cliente* solicitó en el paso anterior.
3. El *Servidor de Autorización* confirma la identidad del usuario y luego redirecciona de vuelta al *Cliente*. Como parte de la redirección el *Cliente* envía un *Código de Autorización*.
4. El *Código de Autorización* es un código temporal y de corta duracción que el *Cliente* puede intercambiar por un *Token de Identidad*.
5. El *Cliente* recibe un *Token de Identidad* y un *Token de Acceso*. El *Token de Identidad* es un JWT del cual el *Cliente* puede extraer información del usuario como: id, nombre, fecha de incio de sesión, fecha de expiración del *Token de Identificación*, etc. El *Token de Acceso* es una llave que el *Cliente* puede usar para obtener información adicional de identificación como el email del usuario.

En el diagrama anterior las aplicaciones de la TGR cumplen el rol de *Cliente* mientras que ClaveÚnica cumple el rol del *Servidor de Autorización*. 

## Abstrayendo la autenticación de las aplicaciones

Ahora, implementar el flujo OIDC tiene varios desafíos: 1. Todas las aplicaciones que necesitan autenticar al usuario final deben implementarlo, 2. Mantener actualizados los componentes que ejecutan el flujo OIDC para que sea seguro y complacente no es trivial, y 3. Cada vez que queremos integrar un nuevo proveedor de identidad se vuelve necesario intervenir las aplicaciones.

Para solucionar estos temas, la TGR utiliza Amazon Cognito. Cognito es un servico que permite incorporar de manera rápida y sencilla el registro, inicio de sesión y control de acceso a aplicaciones web y móviles. Cognito puede escalar a millones de usuarios y soporta el inicio de sesión de proveedores de identidad como OIDC, SAML y redes sociales.

Para el inicio de sesión con terceros, [Cognito puede alojar una interfaz de usuario (IU)](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-integration.html) compatible con OAuth 2.0 que implementa el rol de *Cliente* y le permite al usuario seleccionar el proveedor de identidad con el que desea autenticarse.

En el caso de la TGR, Cognito crea una capa de abstracción al actuar como intermediario entre ClaveÚnica y las aplicaciones, esto le permite a la TGR desacoplar la autenticación de la lógica de negocio. Esta capa de abstracción, además de separar las responsabilidades de los componentes, le da mayor agilidad a los desarolladores de la TGR -porque ya no deben preocuparse por mantener el flujo OIDC- y le permite agregar nuevos proveedores de identidad sin tener que modificar las aplicaciones. El siguiente diagrama de arquitectura muestra el flujo OIDC con Cognito:

<p align="center">
  <br/>
  <img src="https://raw.githubusercontent.com/jorsj/tgr/master/cognito.png" alt="Amazon Cognito"/>
  <br/>
</p>

1. La aplicación de la TGR redirige al usuario a la IU de Cognito. El usuario selecciona ClaveÚnica como proveedor de identidad. 
2. Cognito realiza el flujo OIDC para obtener el *Token de Identidad* y el *Token de Acceso*.
3. Cognito devuelve los tokens a la aplicación de la TGR, la cual le informa al usuario que el inicio de sesión fue exitoso.

## Interceptando el flujo OIDC

Como pueden ver, utilizar Cognito reduce la complejidad de la arquitectura y permite solucionar los tres incovenientes de implementar OIDC. 

Sin embargo, cuando la TGR comenzó a utilizar Cognito se encontró con dos dificultades adicionales:

1. [A pesar de que la IU puede personalizarse con logotipos y estilos del cliente](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-ui-customization.html), el formulario de inicio de sesión se presenta en Inglés y era necesario mostarlo en Español.

2. El largo del *Código de Autorización* que responde ClaveÚnica no cumplía con las especificaciones mínimas para poder ser utilizado por Cognito.

Para el primer tema, la TGR creó un frontend en Español que le permite al usuario seleccionar el proveedor de identidad que desea utilizar. Este frontend está escrito en Angular y es alojado en Amazon Simple Storage Service (S3); el servicio de almacenamiento de objetos de AWS que permite [alojar sitios web estáticos](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html) sin tener que aprovisionar servidores web.

Para el segundo tema, la TGR [inteceptó el flujo OIDC con Amazon Lambda](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html); Lambda es un servicio de cómputo que permite ejecutar código sin aprovisionar ni administrar servidores y que se integra nativamente con otros servicios AWS como Cognito.

El siguiente diagrama de arquitectura muestra la solución completa luego de incorporar estos dos puntos:

<p align="center">
  <br/>
  <img src="https://raw.githubusercontent.com/jorsj/tgr/master/final.png" alt="Arquitectura final"/>
  <br/>
</p>

1. El usuario carga el frontend de autenticación de la TGR desde S3.

2. Antes de cargar la página, el frontend consulta a Cognito por los proveedores de identidad habilitados y usa la respuesta para dibujar las opciones.

3. El usuario selecciona ClaveÚnica como proveedor de identidad y la aplicación invoca a Cognito para comenzar el flujo de autenticación.

4. Cognito inicia el flujo OIDC contra ClaveÚnica y ClaveÚnica devuelve el *Código de Autorización*.

5. Como el *Código de Autorización* que responde ClaveÚnica no cumple con el largo mínimo requerido por Cognito, la TGR interviene el flujo OIDC para reemplazar el *Código de Autorización* original por uno nuevo. De esta forma, Cognito cree que el *Código de Autorización* es complaciente y permite que el flujo OIDC continue.

6.  Una función Lambda autogenera un código con las especificaciones requeridas por Cognito y produce un JSON con el mapeo del *Código de Autorización* original al nuevo. La función envía este documento a Amazon DynamoDB para ser almacenado. DynamoDB es un servicio de base de datos NoSQL rápido y flexible con tiempos de respuesta menores a diez milisegundos a cualquier escala.

7.  La función Lambda retorna el nuevo *Código de Autorización*.

8.  Cognito continua el flujo OIDC con el nuevo *Código de Autorización*, y ClaveÚnica responde el *Token de Identidad* y el *Token de Acceso*.

9.  Cognito devuelve los tokens a la aplicación, junto con el resultado del proceso de autenticación.

10. La aplicación le informa al usuario que el inicio de sesión fue realizado correctamente.

## Conclusiones

Utilizando una arquitectura completamente serverless, la TGR consiguió desacoplar sus aplicaciones del mecanismo de autenticación lo que les da mayor flexibilidad a la hora de implementar nuevos proveedores de identidad, como mayor agilidad al momento de desarollar.

Junto a la TGR seguimos trabajando en brindar más servicios a la ciudadanía, con arquitecturas seguras, que reducen la complejidad, costos operativos y que permiten escalar a medida que la demanda lo requiera.

Si te interesa aprendar más de Cognito y arquitecturas serverless te recomiendo el siguiente [enlace](https://aws.amazon.com/getting-started/projects/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-2/).
