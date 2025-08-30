# 🧪 Guía de cómo funciona la Autenticación en AWS SSO: SAML, OAuth Device Flow y STS

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)


### Descripción
- ¿Te has preguntado qué pasa realmente cuando haces `aws sso login --profile`? Esta guía explica todo el proceso: desde la autenticación en tu IdP corporativo hasta obtener credenciales temporales de AWS, pasando por SAML assertions, OAuth tokens y STS role assumption.
- Guía conceptual que explica cómo interactúan SAML Federation, OAuth 2.0 Device Authorization Flow y AWS Security Token Service (STS) para proporcionar autenticación unificada en AWS SSO. 
- Explicación detallada de la arquitectura de autenticación federada en AWS SSO, cubriendo los flujos SAML 2.0 para navegadores web, OAuth 2.0 Device Authorization Flow para herramientas CLI, y el rol central de AWS STS como orquestador de credenciales temporales.
- Diseñada para entender rápidamente qué sucede "detrás de escenas" cuando ejecutas `aws sso login --profile` y cómo los diferentes protocolos trabajan juntos para servir usuarios web, CLI tools y aplicaciones nativas desde un mismo proveedor de identidad empresarial.
- ¿Cómo Funciona SAML Federation con OAuth Device Flow y AWS STS?

> **Enfoque:** Conceptual y técnico, sin ejemplos extensos de código.
> Sin código complejo, solo la explicación clara de cómo funciona todo junto.
> Ideal para comprender la integración entre protocolos de autenticación modernos y servicios AWS.


## Contenido
- [El problema que resolvemos](#intro)
- [Los Actores Principales](#actores)
- [SAML: La Base de la Federación Web](#base)
- [AWS STS: El Traductor de Tokens](#sts)


- [Integración: SAML + OAuth + STS](#integracion)
- [Configuración Práctica para un laboratoio de pruebas](#lab)

## ⚙️ El problema que resolvemos <a name="intro"></a> 
- Imagina una organización moderna donde los empleados necesitan acceder a AWS desde múltiples contextos:
    - María (Gerente de Proyecto) accede desde su navegador web durante reuniones
    - Carlos (DevOps Engineer) usa la terminal y AWS CLI todo el día
    - Ana (Data Scientist) trabaja desde Jupyter notebooks sin navegador
    - Roberto (Mobile Developer) necesita acceso desde aplicaciones móviles
- **El desafío**: Todos necesitan autenticarse contra el mismo sistema corporativo (Active Directory, Okta, etc.) pero desde interfaces completamente diferentes.
- **La solución tradicional fallida**: Crear usuarios IAM individuales sería un nightmare de seguridad y gestión.
- **La solución elegante:** Una arquitectura que combina SAML (para web) + OAuth Device Flow (para CLI/aplicaciones) + AWS STS (como orquestador central)

## ⚙️ Los Actores Principales  <a name="actores"></a> 
### Identity Provider (IdP)
- **Qué es**: El sistema que "conoce" a los usuarios (Active Directory, Okta, Azure AD)
- **Responsabilidad**: Autenticar usuarios y proporcionar información sobre ellos
- **Analogía**: Como el portero de un edificio que reconoce a los empleados
### AWS IAM Identity Center (antes AWS SSO)
- **Qué es**: El intermediario inteligente entre el IdP y AWS
- **Responsabilidad**: Traducir identidades externas a permisos AWS
- **Analogía**: Como un traductor que convierte "Juan de Marketing" en "permisos de S3 para bucket-marketing"
### AWS STS (Security Token Service)
- **Qué es**: El emisor de "pases temporales" para AWS
- **Responsabilidad**: Convertir autenticaciones en credenciales AWS temporales
- **Analogía**: Como la oficina que emite gafetes temporales de visitante
### Usuario/Aplicación
- **Qué es**: Quien necesita acceso (humano desde navegador, CLI, app móvil)
- **Responsabilidad**: Iniciar el proceso de autenticación

## ⚙️ SAML: La Base de la Federación Web <a name="base"></a> 
### ¿Qué es SAML Realmente?
- **SAML** es como un "certificado digital de identidad" que un sistema puede enviar a otro. Piénsalo como:
    - Un pasaporte que tu IdP emite
    - Contiene "sellos" (atributos) que dicen quién eres y qué puedes hacer
    - AWS confía en este "pasaporte" porque conoce al país (IdP) que lo emitió
### El Flujo SAML Explicado Paso a Paso
    🌐 Usuario → AWS Console → "No te conozco, ve con tu IdP"
        ⬇
    👤 Usuario → IdP → "Soy Juan, aquí están mis credenciales"
        ⬇
    🏢 IdP valida → "Sí, es Juan de Marketing, aquí está su certificado SAML"
        ⬇
    🌐 AWS recibe certificado → "Ok, confío en este IdP, Juan puede entrar"
        ⬇
    🔐 STS genera credenciales temporales → "Juan puede usar S3 por 1 hora"
        ⬇
    ✅ Usuario accede a AWS Console con permisos específicos
### Información Clave en una Assertion SAML
    - Una SAML Assertion es como un documento de identidad que contiene:
        - **Subject (Sujeto)**: "Esta identidad pertenece a juan.perez@empresa.com"
        - **Attributes (Atributos)**:
            - Departamento: "Marketing"
            - Roles: "Marketing-ReadOnly", "S3-FullAccess"
            - Email: "juan.perez@empresa.com"
        - **Conditions (Condiciones)**: "Válido solo por 1 hora, solo para AWS"
        - **Digital Signature**: "Firmado por IdP confiable"
### ¿Por Qué SAML Funciona Tan Bien para Web?
    - **Redirects naturales**: Los navegadores manejan redirects automáticamente
    - **Cookies y sesiones**: Mantiene el estado de autenticación
    - **POST forms**: Puede enviar datos grandes (assertions) fácilmente
    - **Universal**: Funciona en cualquier navegador sin instalaciones

## ⚙️ OAuth Device Flow: Autenticación sin Navegador <a name="base"></a> 
### El Problema del CLI
- Cuando Carlos ejecuta `aws sso login --profile dev-account`, su terminal no tiene un navegador integrado. No puede mostrar una página de login, no puede manejar redirects, no puede procesar JavaScript.
### La Solución Ingeniosa: Device Flow
- El Device Flow es como "autenticación por proxy":
    - **El CLI le dice a AWS**: "Necesito que autentiques este dispositivo"
    - **AWS responde**: "Ok, dile al usuario que vaya a https://device.sso.aws.com e ingrese el código ABCD-1234"
    - **El usuario** abre su **navegador** normal y completa la autenticación
    - Mientras tanto, el CLI está esperando pacientemente preguntando "¿ya terminó?"
    - Una vez completado, AWS le da al CLI los tokens necesarios
### El Flujo Device Authorization Explicado
    💻 CLI ejecuta: aws sso login --profile dev-account
        ⬇
    🌐 AWS SSO responde: "Ve a https://device.sso.aws.com, código: WXYZ-1234"
        ⬇
    👤 Usuario abre navegador → ingresa código WXYZ-1234
        ⬇
    🏢 IdP autentica usuario (SAML flow en el navegador)
        ⬇
    🌐 AWS SSO confirma → "Device autorizado para juan.perez@empresa.com"
        ⬇
    💻 CLI (que estaba esperando) recibe → OAuth tokens
        ⬇
    🔐 CLI usa tokens para obtener credenciales AWS vía STS
        ⬇
    ✅ CLI ahora tiene credenciales temporales de AWS
### Información en los OAuth Tokens
- Los tokens OAuth contienen información diferente a SAML:
    - **Access Token**: "Este token puede acceder a AWS en nombre de Juan"
    - **ID Token**: Contiene claims como:
        - `sub` (subject): "juan.perez@empresa.com"
        - `groups`: ["developers", "marketing-readonly"]
        - `email`: "juan.perez@empresa.com"
    - Refresh Token: "Usa esto para obtener nuevos tokens sin re-autenticar"

## ⚙️ AWS STS: El Traductor de Tokens <a name="sts"></a>
- ¿Qué Hace STS Realmente?