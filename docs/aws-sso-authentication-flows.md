# 🧪 Guía de cómo funciona la Autenticación en AWS SSO: SAML, OAuth Device Flow y STS

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)


### Descripción
- ¿Te has preguntado qué pasa realmente cuando haces `aws sso login --profile`? Esta guía explica todo el proceso: desde la autenticación en tu IdP corporativo hasta obtener credenciales temporales de AWS, pasando por SAML assertions, OAuth tokens y STS role assumption.
- Guía conceptual que explica cómo interactúan SAML Federation, OAuth 2.0 Device Authorization Flow y AWS Security Token Service (STS) para proporcionar autenticación unificada en AWS SSO. 
- Explicación detallada de la arquitectura de autenticación federada en AWS SSO, cubriendo los flujos SAML 2.0 para navegadores web, OAuth 2.0 Device Authorization Flow para herramientas CLI, y el rol central de AWS STS como orquestador de credenciales temporales.
- Diseñada para entender rápidamente qué sucede "detrás de escenas" cuando ejecutas `aws sso login --profile` y cómo los diferentes protocolos trabajan juntos para servir usuarios web, CLI tools y aplicaciones nativas desde un mismo proveedor de identidad empresarial.
> **Enfoque:** Conceptual y técnico, sin ejemplos extensos de código.
> Sin código complejo, solo la explicación clara de cómo funciona todo junto.
> Ideal para comprender la integración entre protocolos de autenticación modernos y servicios AWS.


## Contenido
- [Introducción y Conceptos Fundamentales](#intro)
- [AWS STS: El Corazón de la Federación](#sts)
- [SAML 2.0: Federación Web Tradicional](#tradicional)
- [OAuth 2.0 Device Authorization Flow](#flow)
- [Integración: SAML + OAuth + STS](#integracion)
- [Configuración Práctica para un laboratoio de pruebas](#lab)

## ⚙️ Introducción y Conceptos Fundamentales <a name="intro"></a> 
### ¿Qué Problemas Resolvemos?

