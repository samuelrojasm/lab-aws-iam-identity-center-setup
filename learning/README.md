## Índide de Semanas
- [Week 01](#week-01)

---

## 🔥 Week 01 <a name="week-01"></a>

### Índice Week 01
- [ ¿Qué es SAML Federation?](#local-01)
- [¿OAuth Device Flow hace autentificación o autorización?](#local-02)
- [¿AWS STS hace autentificación o autorización? ](#local-03)
- [¿En que casos STS recibe de SAML o de OAuth?](#local-04)
- [Texto 02](#local-05)
- [Texto 02](#local-06)

---

### ⚡ ¿Qué es SAML Federation? <a name="local-01"></a>
- SAML (Security Assertion Markup Language) Federation es un estándar que permite el intercambio seguro de datos de autenticación y autorización entre un proveedor de identidad (IdP) y un proveedor de servicios (SP). 
    - Identity Provider (IdP): Sistema externo que autentica usuarios (ej: Active Directory, Okta, Azure AD)
    - Service Provider (SP): AWS IAM Identity Center
- La federación SAML permite a los usuarios autenticarse con un proveedor de identidad externo y acceder a recursos de AWS sin necesidad de crear usuarios IAM individuales.

---

### ⚡ ¿OAuth Device Flow hace autentificación o autorización? <a name="local-02"></a>
- OAuth Device Flow hace SOLO AUTORIZACIÓN, NO autenticación.
- Autenticación vs Autorización:
    - Autenticación = "¿Quién eres?" (Identity)
    - Autorización = "¿Qué puedes hacer?" (Permissions)
- Lo que SÍ hace OAuth Device Flow:
    - ✅ Autoriza al DISPOSITIVO a actuar en nombre del usuario
    - ✅ Autoriza al CLI a obtener tokens
    - ✅ utoriza acceso a recursos específicos (scopes)
    - ✅ Proporciona mecanismo para dispositivos sin navegador
- Lo que NO hace OAuth Device Flow:
    - ❌ NO autentica al usuario (no verifica credenciales)
    - ❌ NO valida password
    - ❌ NO conoce la identidad del usuario
    - ❌ NO accede a directorio de usuarios
- El flujo real separado:
    ```bash
    1. AUTORIZACIÓN DEL DISPOSITIVO (OAuth Device Flow):
        💻 CLI → AWS SSO: "Autoriza este device para actuar por un usuario"
        🌐 AWS SSO → CLI: "OK, device_code: ABC123, user_code: XYZ789"

    2. AUTENTICACIÓN DEL USUARIO (SAML):
        👤 Usuario → Navegador → AWS SSO: "Soy el código XYZ789"
        🌐 AWS SSO → IdP: "¿Quién es este usuario?" (SAML AuthnRequest)
        🏢 IdP: "Dame credenciales"
        👤 Usuario ingresa password
        🏢 IdP → AWS SSO: "Es Juan Pérez" (SAML Assertion)
    3. UNIÓN DE AMBOS:
        🌐 AWS SSO: "Device ABC123 está autorizado para actuar por Juan Pérez"
    ```
- Tabla comparativa

|Aspecto|OAuth Device Flow|SAML|
|-------|-----------------|----|
|Propósito|Autorizar dispositivo|Autenticar usuario|
|Pregunta que responde"|¿Puede este CLI actuar por alguien?"|"¿Quién es este usuario?"|
|Input|Device code, client ID|Credenciales de usuario|
|Output|Tokens de autorización|Identity assertion|
|Conoce al usuario|NO (es agnóstico)|SÍ (valida identidad)|
|Valida credenciales|NO|SÍ|
|Dónde ocurre|Entre CLI y AWS SSO|Entre usuario e IdP|

- Analogía del club nocturno:
    - SAML (Autenticación):
        ```bash
        👤 "Soy Juan Pérez"
        🛂 Bouncer: "Muéstrame tu ID"
        👤 Muestra cédula con foto
        🛂 "Sí, eres Juan Pérez, puedes entrar"
        ```
    - OAuth Device Flow (Autorización):
        ```bash
        📱 "Este celular quiere entrar por Juan Pérez"
        🛂 Bouncer: "¿Juan autorizó este celular?"
        👤 Juan (desde adentro): "Sí, autorizo mi celular"
        🛂 "OK, celular puede actuar por Juan"
        ```
- La secuencia completa en `aws sso login --profile`:
    ```bash
    FASE 1 - AUTORIZACIÓN DE DISPOSITIVO (OAuth):
        💻 "AWS SSO, autoriza este CLI"
        🌐 "OK, pero necesito que un usuario real autorice"

    FASE 2 - AUTENTICACIÓN DE USUARIO (SAML):  
        👤 "Soy yo, aquí están mis credenciales"
        🏢 IdP valida y envía SAML assertion
        🌐 "Confirmado: eres Juan Pérez"
    FASE 3 - UNIÓN:
        🌐 "CLI autorizado + Juan autenticado = Tokens para Juan vía CLI"
        💻 Recibe tokens y puede actuar como Juan
    ```
- Resumen
    - OAuth Device Flow hace SOLO AUTORIZACIÓN:
        - Autoriza al dispositivo/aplicación
        - Autoriza el acceso a recursos
        - Autoriza la delegación de permisos
    - PERO NO hace autenticación del usuario - eso siempre lo hace SAML.

> [!NOTE]
> **OAuth Device Flow es "ciego" a la identidad, solo dice:**<br>
> - Autorizo que ALGUIEN use este dispositivo"<br>
> - "No me importa quién sea ese alguien"<br>
> - "SAML se encarga de decirme quién es"<br>

> [!NOTE]
> **SAML es "ciego" al dispositivo. Solo dice:**<br>
> - "Este es Juan Pérez con estos atributos"
> - "No me importa desde dónde se conecta"
> - "OAuth se encarga de autorizar el dispositivo"

---

### ⚡ ¿AWS STS hace autentificación o autorización? <a name="local-03"></a>
- AWS STS hace AUTORIZACIÓN, NO autenticación.
- ¿Qué hace exactamente STS?
    - STS es el "emisor de permisos temporales":
        - ✅ Autoriza el acceso a recursos AWS
        - ✅ Emite credenciales temporales
        - ✅ Convierte identidades en permisos AWS
        - ✅ Aplica políticas de autorización
    - STS NO autentica usuarios:
        - ❌ NO valida passwords
        - ❌ NO verifica identidades
        - ❌ NO accede a directorios de usuarios
        - ❌ NO hace login
- STS: El "Validador de Tickets"
    - Piensa en STS como el empleado del cine que valida boletos:
        ```bash
        👤 Usuario llega con "boleto" (SAML assertion o OAuth token)
        🎫 STS: "¿Este boleto es válido?"
        ├─ ¿Está firmado por alguien en quien confío?
        ├─ ¿No ha expirado?
        ├─ ¿Tiene los permisos correctos?
        └─ ¿Coincide con las políticas del rol?

        ✅ Si TODO es válido: "Aquí tienes acceso temporal a AWS"
        ❌ Si algo falla: "Access Denied"
        ```
- El flujo completo con responsabilidades:

    |Paso|Componente|Función|Tipo|
    |----|----------|-------|----|
    |1|OAuth Device Flow|Autoriza dispositivo CLI|AUTORIZACIÓN|
    |2|SAML + IdP|Valida credenciales del usuario|AUTENTICACIÓN|
    |3|AWS STS|Valida tokens y emite credenciales AWS|AUTORIZACIÓN|
- ¿Qué Valida STS exactamente?
    - Cuando recibe SAML Assertion:
        ```bash
        🔍 STS verifica:
        ├─ ¿La assertion está firmada por un IdP que confío?
        ├─ ¿El certificado del IdP es válido?
        ├─ ¿La assertion no ha expirado?
        ├─ ¿El role que se quiere asumir permite este IdP?
        ├─ ¿Los conditions del trust policy se cumplen?
        └─ ¿Los atributos SAML coinciden con las condiciones?

        ✅ Todo OK → Emite credenciales AWS temporales
        ```
    - Cuando recibe OAuth Token:
        ```bash
        🔍 STS verifica:
        ├─ ¿El token está firmado por un OIDC provider que confío?
        ├─ ¿El token no ha expirado?
        ├─ ¿El audience (aud) claim es correcto?
        ├─ ¿El subject (sub) claim es válido?
        ├─ ¿El role permite este OIDC provider?
        └─ ¿Las conditions del trust policy se cumplen?

        ✅ Todo OK → Emite credenciales AWS temporales
        ```
- Analogía del Banco:
    - SAML (Autenticación):
        ```bash
        🏦 "Soy Juan Pérez, aquí mi cédula y firma"
        👨‍💼 Cajero valida identidad: "Sí, eres Juan"
        ```
    - STS (Autorización):
        ```bash
        📋 Juan presenta "cheque" (SAML assertion/OAuth token)
        👨‍💼 Cajero del banco (STS): "¿Este cheque es válido?"
            ├─ ¿Está firmado correctamente?
            ├─ ¿Hay fondos suficientes? (permisos)
            ├─ ¿No está vencido?
            └─ ¿Cumple políticas del banco?
        💰 "OK, aquí está tu dinero temporal"
        ```
- La distinción clave:
    - Autenticación (SAML/IdP):
        - "¿Quién eres?"
        - Valida credenciales reales
        - Accede a directorio de usuarios
        - Confirma identidad
    - Autorización (STS):
        - "¿Qué puedes hacer?"
        - Valida tokens/assertions ya emitidos
        - Aplica políticas de acceso
        - Emite permisos temporales
- STS Nunca Autentica directamente:
    ```bash
    ❌ INCORRECTO: Usuario → STS (con password)
    ✅ CORRECTO: Usuario → IdP → SAML → STS
    ✅ CORRECTO: Usuario → IdP → OAuth → STS
    ```
> [!NOTE]
> STS siempre recibe "pruebas de autenticación" ya validadas (SAML assertions, OAuth tokens, etc.), nunca autentica directamente.

- Resumen
    - AWS STS hace AUTORIZACIÓN:
        - ✅ Autoriza el acceso a recursos AWS
        - ✅ Valida tokens y assertions (pero no autentica usuarios)
        - ✅ Aplica políticas de autorización
        - ✅ Emite credenciales temporales basadas en permisos

> [!NOTE]
> STS confía en otros sistemas (SAML IdP, OAuth providers) para la autenticación real.

---

### ⚡ ¿En que casos STS recibe de SAML o de OAuth? <a name="local-04"></a>
- La diferencia está en CÓMO y DESDE DÓNDE el usuario inicia el acceso.
#### STS recibe SAML cuando:
- Acceso Web directo:
    ```bash
    👤 Usuario → Navegador web → AWS Console/Portal
    🌐 "aws.amazon.com" o "empresa.awsapps.com"
    🔄 Redirection automático a IdP para SAML
    🏢 IdP autentica → envía SAML Assertion
    🔐 STS recibe SAML Assertion directamente
    ```
- Casos típicos:
    - ✅ Acceso a AWS Management Console
    - ✅ Aplicaciones web que usan SAML federation
    - ✅ Portal AWS SSO desde navegador
    - ✅ Aplicaciones internas que implementan SAML
#### STS recibe OAuth cuando:
- Acceso programático/API:
    ```bash
    💻 Cliente (CLI/App) → AWS SSO OAuth endpoint
    🔄 Device Flow o Authorization Code Flow  
    👤 Usuario autentica en navegador (vía SAML internamente)
    🌐 AWS SSO convierte autenticación SAML → OAuth tokens
    💻 Cliente usa OAuth tokens
    🔐 STS recibe OAuth tokens (no SAML)
    ```
- Casos típicos:
    - ✅ aws sso login --profile (CLI)
    - ✅ AWS SDK con SSO authentication
    - ✅ Aplicaciones móviles nativas
    - ✅ Herramientas de terceros (Terraform, kubectl)
    - ✅ CI/CD pipelines
    - ✅ Scripts automatizados
- El Flujo Completo por Caso:
    - Caso 1: María accede vía Web Browser
        ```bash
        👤 María → https://empresa.awsapps.com
        🌐 AWS SSO → IdP (SAML AuthnRequest)  
        🏢 IdP autentica María → SAML Assertion
        🌐 AWS SSO recibe SAML → la pasa directo a STS
        🔐 STS.AssumeRoleWithSAML(saml_assertion)
        ```
    > Resultado: STS recibe SAML Assertion
    - Caso 2: Carlos usa AWS CLI
        ```bash
        💻 aws sso login --profile dev
        🌐 AWS SSO inicia OAuth Device Flow
        👤 Carlos autoriza en navegador (internamente usa SAML con IdP)
        🌐 AWS SSO convierte resultado SAML → OAuth tokens
        💻 CLI recibe OAuth tokens
        🔐 STS.AssumeRoleWithWebIdentity(oauth_token)
        ```
    > Resultado: STS recibe OAuth Token
#### Arquitectura Visual

                Identity Provider (IdP)
                           │
                    SAML Authentication
                           │
                           ▼
    ┌─────────────────────────────────────────────────┐
    │              AWS SSO / IAM Identity Center      │
    │                                                 │
    │  ┌─────────────────┐    ┌─────────────────────┐ │
    │  │   SAML          │    │   OAuth             │ │
    │  │   Endpoint      │    │   Endpoint          │ │
    │  │                 │    │                     │ │
    │  │ Direct SAML     │    │ SAML→OAuth          │ │
    │  │ Passthrough     │    │ Translation         │ │
    │  └─────────────────┘    └─────────────────────┘ │
    └─────────────┬─────────────────────┬─────────────┘
                  │                     │
            SAML Assertion        OAuth Tokens
                  │                     │
                  ▼                     ▼
    ┌─────────────────────────────────────────────────┐
    │                  AWS STS                        │
    │                                                 │
    │  AssumeRoleWithSAML   AssumeRoleWithWebIdentity │
    └─────────────────────────────────────────────────┘

####  Ejemplos específicos:
- STS recibe SAML:
    ```bash
    # Usuario accede directamente vía web
    https://123456789012.signin.aws.amazon.com/console
    # → Redirect a IdP → SAML assertion → STS
    ```
- STS recibe OAuth:
    ```bash
    # CLI tools
    aws sso login --profile production
    aws s3 ls

    # SDK con SSO
    import boto3
    session = boto3.Session(profile_name='dev-profile')
    s3 = session.client('s3')

    # Terraform con SSO
    terraform init -backend-config="profile=dev-profile"
    ```
#### ¿Por qué esta diferencia?
- Ventajas SAML directo (Web):
    - ✅ Menos saltos: Menos latencia
    - ✅ Información rica: Atributos SAML completos
    - ✅ Estándar maduro: Bien soportado por browsers
    - ✅ Seamless UX: Redirects transparentes
- Ventajas OAuth tokens (programático):
    - ✅ Refresh capability: Renovación sin re-autenticación
    - ✅ Device flexibility: Funciona sin navegador
    - ✅ API friendly: JSON en lugar de XML
    - ✅ Long-lived sessions: Refresh tokens de larga duración

#### Tabla de decisión:

|Contexto|Protocolo a STS|¿Por qué?|
|--------|---------------|---------|
|AWS Management Console|SAML|Web browser, redirects naturales|
|AWS CLI|OAuth|Device flow, refresh tokens|
|AWS SDK en aplicación|OAuth|Programmatic, long-lived sessions|
|Mobile app nativa|OAuth|Authorization code flow|
|Web app empresarial|SAML|Rich attributes, enterprise SSO|
|CI/CD pipeline|OAuth|Service account, automated|
|Jupyter notebook|OAuth|Programming environment|


#### Resumen
- STS recibe SAML cuando: Acceso directo vía navegador web
- STS recibe OAuth cuando: Acceso vía CLI, SDKs, apps móviles, APIs
- La autenticación del usuario siempre es SAML, pero STS recibe diferentes "formatos" según el cliente que inicia la request.

> [!NOTE]
> **AWS SSO actúa como "traductor inteligente":**<br>
> - Para contextos web: Pasa SAML directo (eficiente)<br>
> - Para contextos programáticos: Convierte SAML → OAuth (flexible)<br>
> - Pero en AMBOS casos, la autenticación real del usuario SIEMPRE es SAML en el IdP.

---

### ⚡ ¿AWS STL es parte de AWS Identity Center?` <a name="local-05"></a>
- NO, AWS STS NO es parte de AWS Identity Center.
- Son servicios separados que trabajan juntos.
#### 


### ⚡ Texto 01 `ss` <a name="local-01"></a>
- Texto01
- Texto02

### ⚡ Texto 01 `ss` <a name="local-01"></a>
- Texto01
- Texto02

> [!NOTE]
> Internamente, `act` crea contenedores **Docker** que simulan<br>
> los **GitHub runners**, por lo que necesitas tener Docker instalado.

#### 🔗 Referencias Texto 01
- []()

---


----


---

