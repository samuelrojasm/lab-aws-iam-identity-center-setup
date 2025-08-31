## Índide de Semanas
- [Week 01](#week-01)

---

## 🔥 Week 01 <a name="week-01"></a>

### Índice Week 01
- [ ¿Qué es SAML Federation?](#local-01)
- [¿OAuth Device Flow hace autentificación o autorización?](#local-02)
- [¿AWS STS hace autentificación o autorización? ](#local-03)
- [¿En que casos STS recibe de SAML o de OAuth?](#local-04)
- [¿AWS STL es parte de AWS Identity Center?](#local-05)
- [¿Cúal es el flujo de Identity Center con OpenID Connect (OIDC)?](#local-06)
- [](#local-07)
- [](#local-08)
- [](#local-09)
- [](#local-10)

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

### ⚡ ¿AWS STL es parte de AWS Identity Center? <a name="local-05"></a>
- NO, AWS STS NO es parte de AWS Identity Center.
- Son servicios separados que trabajan juntos.
#### La Arquitectura Real:
    
    ┌─────────────────────────────────────────────────────┐
    │                AWS Account                          │
    │                                                     │
    │  ┌─────────────────────┐    ┌─────────────────────┐ │
    │  │  AWS Identity       │    │     AWS STS         │ │
    │  │  Center             │    │  (Security Token    │ │
    │  │  (SSO Service)      │    │   Service)          │ │
    │  │                     │    │                     │ │
    │  │ • SAML/OAuth        │───▶│ • AssumeRole*       │ │
    │  │ • User Portal       │    │ • Token Generation  │ │
    │  │ • Identity Sources  │    │ • Credential Mgmt   │ │
    │  └─────────────────────┘    └─────────────────────┘ │
    │                                                     │
    └─────────────────────────────────────────────────────┘
    

#### ¿Cuál es la diferencia?
- AWS Identity Center:
    - 🎯 **Propósito**: Gestión centralizada de identidades y SSO
    - 📍 **Alcance**: Una instancia por organización
    - 🔧 **Funciones**:
        - Portal de usuario (donde ves las cuentas)
        - Gestión de usuarios y grupos
        - Configuración de Identity Providers
        - Permission Sets
        - Mapeo de atributos SAML/OAuth
- AWS STS:
    -  🎯 **Propósito**:** Emisión de credenciales temporales
    - 📍 **Alcance**: Servicio global de AWS, existe en cada cuenta
    - 🔧 **Funciones**:
        - `AssumeRole`, `AssumeRoleWithSAML`, `AssumeRoleWithWebIdentity`
        - Validación de tokens y assertions
        - Generación de credenciales temporales
        - Aplicación de trust policies
#### ¿Cómo Interactúan?
- El Flujo Completo:
    ```bash
    1. 🏢 IdP autentica usuario → SAML assertion
    2. 🌐 AWS Identity Center recibe SAML → procesa y mapea
    3. 🌐 AWS Identity Center → llama a AWS STS
    4. 🔐 AWS STS valida y emite credenciales temporales
    5. ✅ Usuario obtiene acceso temporal a AWS
    ```
- En código (conceptual):
    ```python
    # Lo que hace AWS Identity Center internamente:
    def handle_saml_login(saml_assertion):
        # Identity Center procesa la assertion
        user_info = parse_saml_attributes(saml_assertion)
        role_arn = map_user_to_role(user_info)
        
        # Identity Center llama a STS
        sts_client = boto3.client('sts')
        credentials = sts_client.assume_role_with_saml(
            RoleArn=role_arn,
            PrincipalArn=saml_provider_arn,
            SAMLAssertion=saml_assertion
        )
    return credentials
    ```
#### Comparación detallada:

|Aspecto|AWS Identity Center|AWS STS|
|-------|-------------------|-------|
|Tipo de servicio|Gestión de identidades|Emisión de tokens|
|Interfaz de usuario|Sí (portal web)|No (solo APIs)|
|Gestión de usuarios|Sí|No|
|Configuración IdP|Sí|No|
|Emite credenciales|No directamente|Sí|
|Scope|Multi-account|Por cuenta|
|Costos|Gratis hasta 50 usuarios|Incluido en AWS|

#### ¿Por qué la Confusión?
- Razones comunes:
    - **Integración estrecha**: Trabajan tan juntos que parecen uno
    - **UI unificada**: El portal de Identity Center oculta las llamadas a STS
    - **Documentación**: A veces se mencionan juntos
    - **Experiencia del usuario**: El usuario no ve la diferencia
#### Analogía del Aeropuerto
- AWS Identity Center = Check-in counter
    - Verifica tu identidad
    - Te asigna asiento (mapea permisos)
    - Te da boarding pass (información de acceso)
- AWS STS = Security Checkpoint
    - Valida tu boarding pass
    - Te da acceso temporal al área segura
    - Emite "pase temporal" para volar
> Son diferentes departamentos, pero trabajas con ambos para volar.
#### En la práctica:
- Lo que haces en Identity Center:
    - Configurar usuarios y grupos
    - Mapear atributos SAML
    - Crear Permission Sets
    - Asignar acceso a cuentas
- Lo que STS hace automáticamente:
    - Validar tokens/assertions
    - Aplicar trust policies
    - Generar credenciales AWS temporales
    - Manejar expiración de tokens
#### ¿Puedes usar STS sin Identity Center?
- ¡SÍ! STS existe independientemente:
    ```bash
    # STS directo (sin Identity Center)
    aws sts assume-role \
        --role-arn arn:aws:iam::123456789012:role/MyRole \
        --role-session-name my-session

    # Cross-account access directo
    aws sts assume-role \
        --role-arn arn:aws:iam::999999999999:role/CrossAccountRole \
        --role-session-name cross-account-session
    ```
#### Resumen
- AWS STS es un servicio INDEPENDIENTE que AWS Identity Center UTILIZA.
    - **Identity Center**: El "orquestador" de identidades
    - **STS**: El "emisor" de credenciales temporales
- **Analogía**: Identity Center es como tu banco, STS es como el cajero automático. El banco gestiona tu cuenta, pero el cajero emite el efectivo.

---

### ⚡ ¿Cúal es el flujo de Identity Center con OpenID Connect (OIDC)? <a name="local-06"></a>
- El flujo de Identity Center con OpenID Connect (OIDC) es muy similar al OAuth Device Flow, pero con algunas diferencias importantes.
#### Identity Center puede usar OIDC de dos maneras:
1. Como Identity Source (reemplaza SAML)
2. Para aplicaciones cliente (como el CLI)
#### Flujo 1: OIDC como Identity Source (Reemplazo de SAML)
- Configuración:

    Identity Provider (Okta, Auth0, Azure AD con OIDC)
                    ↓ 
            AWS Identity Center
                    ↓
                 AWS STS

- El Flujo Paso a Paso:
    - Usuario Accede al Portal
        ```bash
        👤 Usuario → https://empresa.awsapps.com/start
        🌐 Identity Center: "No tienes sesión, necesitas autenticarte"
        ```
    - Redirección OIDC (No SAML)
        ```bash
        🌐 Identity Center → OIDC Provider (Authorization Endpoint)
        URL: https://auth.empresa.com/oauth2/authorize?
            client_id=aws-identity-center&
            response_type=code&
            scope=openid profile email groups&
            redirect_uri=https://empresa.awsapps.com/oidc/callback&
            state=random-state-value
        ```
    -  Usuario se autentica
        ```bash
        🏢 OIDC Provider muestra login
        👤 Usuario ingresa credenciales
        🏢 OIDC Provider valida credenciales
        ```
    - Authorization Code Return
        ```bash
        🏢 OIDC Provider → Identity Center (redirect):
        https://empresa.awsapps.com/oidc/callback?
            code=AUTH_CODE_123&
            state=random-state-value
        ```
    - Token Exchange
        ```bash
        🌐 Identity Center → OIDC Provider (Token Endpoint):
        POST /oauth2/token
        {
            "grant_type": "authorization_code",
            "code": "AUTH_CODE_123",
            "client_id": "aws-identity-center",
            "client_secret": "secret",
            "redirect_uri": "https://empresa.awsapps.com/oidc/callback"
        }

        🏢 OIDC Provider responde:
        {
            "access_token": "AT_xyz789",
            "id_token": "ID_TOKEN_jwt",  ← Este es clave
            "token_type": "Bearer",
            "expires_in": 3600
        }
        ```
    - Extracción de Claims del ID Token
        ```bash
        🌐 Identity Center decodifica ID Token (JWT):
        {
            "sub": "12345",
            "email": "juan.perez@empresa.com",
            "name": "Juan Pérez",
            "groups": ["developers", "aws-admin"],
            "department": "engineering",
            "iss": "https://auth.empresa.com",
            "aud": "aws-identity-center"
        }
        ```
    - STS con Web Identity
        ```bash
        🌐 Identity Center → AWS STS:
        AssumeRoleWithWebIdentity(
            RoleArn="arn:aws:iam::123456789012:role/IdentityCenterRole",
            WebIdentityToken=ID_TOKEN_jwt,
            RoleSessionName="juan.perez-session"
        )

        🔐 STS valida JWT y emite credenciales AWS
        ```
#### Flujo 2: CLI con OIDC (aws sso login --profile)
- Cuando Identity Center usa OIDC como source:
1. CLI Inicia Device Flow
    ```bash
    💻 aws sso login --profile dev
    🌐 Identity Center responde con device code (igual que antes)
    ```
2. Usuario Autoriza en Navegador
     ```bash
    👤 Usuario va a https://device.sso.aws.com → código: ABCD-1234
    🌐 Identity Center: "Necesito autenticar este usuario"
    ```
3. OIDC Flow en el Navegador
    ```bash
    🌐 Identity Center → OIDC Provider (Authorization Code Flow)
    🏢 OIDC Provider autentica usuario
    🏢 OIDC Provider → Identity Center: ID Token con claims
    ```
4. Device Authorization Complete
    ```bash
    🌐 Identity Center asocia device_code con identity del usuario
    💻 CLI obtiene OAuth tokens de Identity Center (no del OIDC provider)
    ```
5. CLI usa Tokens con STS
    ```bash
    💻 CLI → AWS STS: AssumeRoleWithWebIdentity
    (Usando tokens de Identity Center, no del OIDC provider original)
    ```
#### Diferencias: SAML vs OIDC en Identity Center
|Aspecto|SAM|OIDC|
|-------|---|----|
|Protocolo base|XML, HTTP POST/Redirect|JSON, HTTP REST|
|Token format|XML Assertion|JWT (JSON Web Token)|
|Flow típico|POST/Redirect binding|Authorization Code Flow|
|Información de usuario|SAML Attributes|JWT Claims|
|Complejidad|Media (XML parsing)|Baja (JSON)|
|Estándar moderno|Maduro pero legacy|Moderno y preferido|

#### Configuración OIDC en Identity Center
- Trust Policy para OIDC Provider:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::123456789012:oidc-provider/auth.empresa.com"
                },
                "Action": "sts:AssumeRoleWithWebIdentity",
                "Condition": {
                    "StringEquals": {
                        "auth.empresa.com:aud": "aws-identity-center",
                        "auth.empresa.com:sub": "validated-subject"
                    },
                    "StringLike": {
                        "auth.empresa.com:email": "*@empresa.com"
                    }
                }
            }
        ]
    }
    ```
- Attribute Mapping (OIDC Claims → Identity Center):
    ```json
    {
        "mappings": {
            "email": "${path:email}",
            "firstName": "${path:given_name}",
            "lastName": "${path:family_name}",
            "displayName": "${path:name}",
            "groups": "${path:groups}",
            "department": "${path:department}"
        }
    }
    ```
#### ¿Cuándo usar OIDC vs SAML?
- Usar OIDC cuando:
    - **Aplicaciones modernas**: APIs, SPAs, mobile apps
    - **Simplicidad**: Menos complejidad de configuración
    - **JWT tokens**: Mejor para APIs y servicios
    - **Cloud-native IdPs** Auth0, Okta modern, Cognito
    - **Microservicios**: Better token introspection

- Usar SAML cuando:
    - **Enterprise legacy**: Sistemas tradicionales
    - **Rich attributes**: Necesitas muchos atributos complejos
    - **Compliance**: Estándares enterprise establecidos
    - **Active Directory**: ADFS y sistemas Microsoft legacy
    - **Complex claims**: Conditional logic en assertions
#### JWT vs SAML Assertion - Ejemplo:
- SAML Assertion (XML):
    ```xml
    <saml:AttributeStatement>
        <saml:Attribute Name="email">
            <saml:AttributeValue>juan.perez@empresa.com</saml:AttributeValue>
        </saml:Attribute>
        <saml:Attribute Name="groups">
            <saml:AttributeValue>developers</saml:AttributeValue>
            <saml:AttributeValue>aws-admin</saml:AttributeValue>
        </saml:Attribute>
    </saml:AttributeStatement>
    ```
- OIDC ID Token (JWT payload):
    ```json
    {
        "sub": "12345",
        "email": "juan.perez@empresa.com",
        "groups": ["developers", "aws-admin"],
        "iss": "https://auth.empresa.com",
        "aud": "aws-identity-center",
        "exp": 1640995200,
        "iat": 1640991600
    }
    ```
#### Ventajas de OIDC en Identity Center
1. Simplicidad:
    ```bash
    SAML: XML parsing, certificate validation, complex bindings
    OIDC: JSON parsing, JWT validation, simple HTTP REST
    ```
2. Performance:
    ```bash
    SAML: Larger XML payloads, POST forms
    OIDC: Compact JWT tokens, REST APIs
    ```
3. Developer Experience:
    ```bash
    SAML: Need specialized libraries
    OIDC: Standard HTTP + JWT libraries
    ```
4. Modern Standards:
    ```bash
    SAML: 2005 standard, XML-based
    OIDC: 2014 standard, built on OAuth 2.0
    ```
#### Resumen
- El flujo OIDC en Identity Center es prácticamente idéntico al SAML, pero:
    - En lugar de SAML Assertions → usa JWT ID Tokens
    - En lugar de XML/POST → usa JSON/REST
    - Mismo resultado final: Credenciales AWS temporales vía STS

> La experiencia del usuario es la misma, pero la implementación técnica es más moderna y simple.

---

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

