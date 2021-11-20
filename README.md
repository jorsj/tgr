# Cómo la Tesorería General de la Republica de Chile usa Amazon Cognito, Lambda y DynamoDB para abstraer la lógica de autenticación de sus aplicaciones

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

## Abstrayendo la autenticación de la lógica de negocio

Sin embargo, implementar el flujo OIDC tiene varios desafíos: 1. Todas las aplicaciones que necesitan autenticar al usuario final deben implementarlo, 2. Mantener actualizados los componentes que ejecutan el flujo OIDC para que sea seguro y complacente no es trivial, y 3. Cada vez que queremos integrar un nuevo proveedor de identidad se vuelve necesario intervenir las aplicaciones.

Para solucionar estos temas, la TGR utiliza Amazon Cognito. Cognito es un servico que permite incorporar de manera rápida y sencilla el registro, inicio de sesión y control de acceso a aplicaciones web y móviles. Cognito puede escalar a millones de usuarios y soporta el inicio de sesión de proveedores de identidad como OIDC, SAML y redes sociales.

En el caso de la TGR, Cognito crea una capa de abstracción al actuar como intermediario entre ClaveÚnica y las aplicaciones, esto le permite a la TGR desacoplar la autenticación de la lógica de negocio. Esta capa de abstracción, además de separar las responsabilidades de los componentes, le da mayor agilidad a los desarolladores de la TGR -porque ya no deben preocuparse por mantener el flujo OIDC- y le permite agregar nuevos proveedores de identidad sin tener que modificar las aplicaciones. El siguiente diagrama de arquitectura muestra el flujo OIDC con Cognito:

<p align="center">
  <br/>
  <img src="https://raw.githubusercontent.com/jorsj/tgr/master/cognito.svg" alt="Amazon Cognito"/>
  <br/>
</p>

1. La aplicación le instruye a 
2. 

## Interceptando el flujo OIDC

TODO:

* [ ] Introducir contexto de por qué necesitamos interceptar el protocolo OIDC.
* [ ] Revisar el diagrama de arquitectura final con el cliente.

<p align="center">
  <br/>
  <img src="https://raw.githubusercontent.com/jorsj/tgr/master/final.svg" alt="Arquitectura final"/>
  <br/>
</p>

1. El usuario carga la aplicación web de la TGR.
2. Antes de dibujar la página, la aplicación consulta a Cognito por los proveedores de identidad que habilitados y usa la respuesta para mostrar las opciones.
3. Luego de que el usuario selecciona un proveedor de identidad, la aplicación invoca a Cognito para que que inicie el flujo de autenticación. 
4. Cognito inicia el flujo OIDC contra ClaveÚnica.
5. ClaveÚnica devuelve el *Código de Autorización* como parte del flujo OIDC.
6. Antes de continuar con el intercambio del *Código de Autorización* por el *Token de Identidad* y el *Token de Acceso*, Cognito invoca una función Lambda. 
7. La función Lambda mapea el *Código de Autorización* enviado por ClaveÚnica a un nuevo *Código de Autorización* generado aleatoriamente. El mapeo del *Token de Autorización* original al nuevo *Token de Autorización* lo persiste en DynamoDB.
8. La función Lambda retorna el nuevo *Código de Autorización*.
9. Cognito continua el flujo OIDC con el *Código de Autorización*
10. ClaveÚnica responde el *Token de Identidad* y el *Token de Acceso*
11. Cognito devuelve los tokens a la aplicación, junto con el resultado del proceso de autenticación.
12. La aplicación le presenta al usuario
