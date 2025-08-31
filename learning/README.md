## Índide de Semanas
- [Week 01](#week-01)

---

## 🔥 Week 01 <a name="week-01"></a>

### Índice Week 01
- [ ¿Qué es SAML Federation?](#local-01)
- [¿OAuth Device Flow hace autentificación o autorización?](#local-02)
- [Texto 02](#local-03)
- [Texto 02](#local-04)
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

