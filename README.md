# Cómo la Tesorería General de la República de Chile usa Amazon Cognito, AWS Lambda y Amazon DynamoDB para abstraer la autenticación de sus aplicaciones

La Tesorería General de la República (TGR) de Chile es la institución pública dependiente del Ministerio de Hacienda encargada de recolectar, custodiar y distribuir fondos y valores fiscales de los diferentes servicios públicos del país. Es responsable de darle a la ciudadania mecanismos para efectuar el pago de obligaciones al fisco y de pagar las obligaciones del gobierno a otras instituciones.

## El desafío

En el 2010 el Gobierno de Chile lanzó ClaveÚnica, una contraseña unificada para autenticar de manera digital la identidad de los ciudadanos chilenos y residentes mayores de catorce años. CláveÚnica sirve como puerta de entrada para todos los servicios virtuales que el Estado de Chile provee y tiene como objetivo facilitar el acceso a los trámites de manera digital,  aumentar las medidas de seguridad  y reducir los tiempos de espera. La TGR al ser uno de esos proveedores de servicios digitales, se vio obligada a integrar sus aplicaciones usando ClaveÚnica y deprecar lentamente su sistema de autenticación legado.

## OpenID Connect

Tras bambalinas, ClaveÚnica es un proveedor de identidad que implementa el protocolo OpenID Connect (OIDC). OIDC es una capa de identidad sobre OAuth 2.0 que le permite a las aplicaciones de terceros verificar la identidad del usuario final y obtener información básica del mismo. OIDC utiliza JSON Web Tokens (JWT) que obtiene mediante flujos que cumplen con las especificaciones de OAuth 2.0. El siguiente diagrama de secuencia resume el flujo de autenticación OIDC:

<p align="center">
  <br/>
  <img src="oidc.svg" alt="Flujo OIDC"/>
  <br/>
</p>


1. El *Cliente* inicia el flujo con la redirección del usuario a la URL del *Servidor de Autorización*, junto con ello envía los *Scopes*;  permisos granulares que el *Cliente* requiere que el usuario le otorgue. En este caso, para indicar que es un flujo OIDC, el *Cliente* incluye el *Scope* “openid”.
2. El usuario se autentica en el formulario de login del *Servidor de Autorización* y confirma que quiere otorgar los *Scopes* que el *Cliente* solicitó en el paso anterior.
3. El *Servidor de Autorización* confirma la identidad del usuario y luego redirecciona de vuelta al *Cliente*. Como parte de la redirección el *Cliente* envía un *Código de Autorización*.
4. El *Código de Autorización* es un código temporal y de corta duración que el *Cliente* puede intercambiar por un *Token de Identidad*.
5. El *Cliente* recibe un *Token de Identidad* y un *Token de Acceso*. El *Token de Identidad* es un JWT del cual el *Cliente* puede extraer información del usuario como: id, nombre, fecha de inicio de sesión, fecha de expiración del *Token de Identificación*, etc. El *Token de Acceso* es una llave que el *Cliente* puede usar para obtener información adicional de identificación como el email del usuario.

En el diagrama anterior las aplicaciones de la TGR cumplen el rol de *Cliente* mientras que ClaveÚnica cumple el rol del *Servidor de Autorización*.

## Abstrayendo la autenticación de las aplicaciones

Ahora, implementar el flujo OIDC tiene varios desafíos:
1. Todas las aplicaciones que necesitan autenticar al usuario final deben implementarlo.
2. Mantener actualizados los componentes que ejecutan el flujo OIDC para que sea seguro y complaciente no es trivial.
3. Cada vez que queremos integrar un nuevo proveedor de identidad se vuelve necesario intervenir las aplicaciones.

Para solucionar estos temas, la TGR utiliza Amazon Cognito. Amazon Cognito es un servicio que permite incorporar de manera rápida y sencilla el registro, inicio de sesión y control de acceso a aplicaciones web y móviles. Amazon Cognito puede escalar a millones de usuarios y soporta el inicio de sesión de proveedores de identidad como OIDC, Security Assertion Markup Language (SAML) y redes sociales como Facebook, Twitter, Amazon, etc. Para estos últimos, [Amazon Cognito puede alojar una interfaz de usuario (IU)](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-integration.html) compatible con OAuth 2.0 que implementa el rol de *Cliente* y le permite al usuario seleccionar el proveedor de identidad con el que desea autenticarse.

En el caso de la TGR,  Amazon Cognito crea una capa de abstracción al actuar como intermediario entre ClaveÚnica y las aplicaciones, esto le permite desacoplar la autenticación de la lógica de negocio. Esta capa de abstracción, además de separar las responsabilidades de los componentes, le da mayor agilidad a los desarrolladores porque ya no deben preocuparse por mantener el flujo OIDC, y le permite agregar nuevos proveedores de identidad sin tener que modificar código. El siguiente diagrama de arquitectura muestra el flujo OIDC con Amazon Cognito:

<p align="center">
  <br/>
  <img src="cognito.png" alt="Amazon Cognito" width="440" height="62"/>
  <br/>
</p>



1. La aplicación de la TGR redirige al usuario a la IU de Amazon Cognito. El usuario selecciona ClaveÚnica como proveedor de identidad.
2. Amazon Cognito realiza el flujo OIDC para obtener el *Token de Identidad* y el *Token de Acceso*.
3. Amazon Cognito devuelve los tokens a la aplicación de la TGR, la cual le informa al usuario que el inicio de sesión fue exitoso.

Como pueden ver, utilizar Amazon Cognito reduce la complejidad de la arquitectura y permite solucionar los tres inconvenientes de implementar OIDC.

## Traduciendo la UI a Español

Siendo el Español el idioma de facto en Chile, y [a pesar de que la IU de Amazon Cognito puede personalizarse con logotipos y estilos externos](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-ui-customization.html), el formulario de inicio de sesión sólo está disponible en Ingles.

Por esta razón la TGR creó un frontend en Español que le permite al usuario seleccionar el proveedor de identidad con el que desea auteticarse. Este frontend está escrito en Angular y es alojado en Amazon Simple Storage Service (S3); el servicio de almacenamiento de objetos de AWS que permite [alojar sitios web estáticos](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html) sin tener que aprovisionar servidores web.

Desde Amazon S3  la TGR disponibliza un HTML que usa el SDK de Amazon Cognito para consultar por los provedores de identidad disponibles para después dibujar las opciones en la página.

## Clave Tributaria

Además de proveer servicios a la ciudadanía en general, la TGR también entrega servicios a empresas. El proceso de autenticación para las empresas utiliza Clave Tributaria, en vez de ClaveÚnica, otro proveedor de identidad que implementa el flujo OIDC y que es disponibilizado por el Servicio de Impuesto Internos (SII).

En principio, implementar un nuevo proveedor de identidad con Amazon Cognito es trivial. Sin embargo cuando la TGR agregó Clave Tributaria se dio cuenta que el largo del *Código de Autorización* no cumplía con las especificaciones mínimas para poder ser utilizado por Amazon Cognito.

Para amendar esta situación, la TGR utiliza la [integración nativa de Amazon Cognito con AWS Lambda](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html) para interceptar el flujo OIDC y ajustar el *Código de Autorización*; AWS Lambda es un servicio de cómputo que permite ejecutar código sin aprovisionar ni administrar servidores y que Amazon Cognito puede invocar durante las operaciones de registro, confirmación e inicio de sesión. 

El rol que cumple la función de AWS Lambda es el de generar un nuevo *Código de Autorización* —complaciente con los requerimientos de Amazon Cognito—, mapear el código original al nuevo, y almacenar el mapeo en Amazon DynamoDB para ser utilizado después.

El siguiente diagrama de arquitectura muestra la implementación completa luego de incorporar la UI en Español y la interceptación del flujo OIDC:

<p align="center">
  <br/>
  <img src="final.png" alt="Arquitectura final" width="495" height="310"/>
  <br/>
</p>



1. El usuario carga el frontend de autenticación de la TGR desde Amazon S3.
2. Antes de mostrar la página, el frontend consulta a Amazon Cognito por los proveedores de identidad habilitados y usa la respuesta para dibujar las opciones.
3. El usuario selecciona Clave Tributaria como proveedor de identidad y la aplicación invoca a Amazon Cognito para comenzar el flujo de autenticación.
4. Amazon Cognito inicia el flujo OIDC y Clave Tributaria eventualmente devuelve el *Código de Autorización*.
5. Amazon Cognito invoca una función AWS Lambda para reemplazar el *Código de Autorización* original por uno nuevo.
6. La función de AWS Lambda auto-genera el nuevo *Código de Autorización*, produce el documento con el mapeo del *Código de Autorización* original al nuevo y lo envía a Amazon DynamoDB para ser almacenado.
7.  La función de AWS Lambda retorna el nuevo *Código de Autorización*.
8.  Amazon Cognito continua el flujo OIDC con el nuevo *Código de Autorización*, y Clave Tributaria responde el *Token de Identidad* y el *Token de Acceso*.
9.  Amazon Cognito devuelve los tokens a la aplicación junto con el resultado del proceso de autenticación.
10. La aplicación le informa al usuario que el inicio de sesión fue realizado correctamente.

## Conclusiones

Con una arquitectura completamente serverless, la TGR consiguió desacoplar sus aplicaciones del mecanismo de autenticación. Esto les permite escalar a miles de usuarios concurrentes, mejorar su time-to-market para nuevas funcionalidades y reducir el costo operativo de sus sistemas.

Junto a la TGR seguimos trabajando en brindar más servicios a la ciudadanía, con arquitecturas seguras, que reducen la complejidad, costos operativos y que permiten escalar a medida que la demanda lo requiera.

Si quieres aprender más de Amazon Cognito y de arquitecturas serverless, te recomendamos el siguiente [enlace](https://aws.amazon.com/getting-started/projects/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-2/).
