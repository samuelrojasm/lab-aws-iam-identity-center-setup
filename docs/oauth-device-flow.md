# 🧪 Guía de OAuth 2.0 Device Authorization Flow

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

## Contenido
- [¿Qué es OAuth 2.0 Device Authorization Flow](#ques-es)
- [¿Por qué AWS IAM Identity Center usa Device Flow?](#uso-en-iam)
- [¿Cómo funciona el Device Flow?](#diagrama)
- [Ejemplo práctico: AWS CLI SSO Login](#ejemplo)
- [Ventajas del Device Flow](#ventajas)
- [Consideraciones de seguridad](#seguridad)
- [Diferencias con otros OAuth flows](#diferencia)
- [Troubleshooting común](#problemas)
- [El problema de usar Authorization Code Flow en CLI](#problema)
- [¿Cómo Device Flow resueve el problema?](#solucion)
- [¿Cómo funciona en la práctica?: Escenario típico con AWS CLI](#practica)
- [¿Por qué CLI es el caso de uso perfecto? ](#caso)
- [Ejemplos de dispositivos con capacidades de entrada limitadas](#limitadas)
- [¿Por qué Device Authorization Flow evita usar el navegador web en el dispositivo?](#navegador)

---

## ⚙️ ¿Qué es OAuth 2.0 Device Authorization Flow? <a name="que-es"></a> 
- El OAuth 2.0 Device Authorization Flow (también conocido como `Device Flow`) es un flujo de **autorización** diseñado específicamente para dispositivos que tienen capacidades de entrada limitadas o no tienen un navegador web disponible.
- Características principales:
    - Diseñado para dispositivos con entrada limitada (CLI, IoT devices, smart TVs)
    - No requiere un navegador web en el dispositivo
    - El usuario autoriza en un dispositivo separado (smartphone, laptop)
    - Especialmente útil para aplicaciones de línea de comandos
- OAuth 2.0 tiene varios flujos para diferentes situaciones:
    - Authorization Code Flow - Para aplicaciones web
    - Client Credentials Flow - Para server-to-server
    - Device Authorization Flow - Para dispositivos limitados
    - Implicit Flow - Para SPAs (ya no recomendado)
- Comparación de uso de flujos de OAuth

    | Escenario    |Flujo OAuth recomendado|¿Por qué?                               |
    |--------------|-----------------------|----------------------------------------|
    |Aplicación web|Authorization Code Flow|Tiene navegador, puede manejar redirects|
    |API server-to-server|Client Credentials Flow|No hay usuario final|
    |CLI/TV/IoT|Device Authorization Flow|Sin navegador o entrada limitada|

## ⚙️ ¿Por qué AWS IAM Identity Center usa Device Flow?  <a name="uso-en-iam"></a>
- AWS IAM Identity Center utiliza este flujo para el AWS CLI porque:
    - **CLI no tiene interfaz gráfica**: No puede abrir ventanas de navegador
    - **Experiencia de usuario mejorada**: Usuario puede autorizar desde cualquier dispositivo
    - **Seguridad**: No se almacenan credenciales permanentes en el CLI
    - **Multiplataforma**: Funciona igual en Windows, macOS, Linux

## ⚙️ ¿Cómo funciona el Device Flow? <a name="diagrama"></a>
### Diagrama del proceso:
```mermaid
sequenceDiagram
    participant AWS CLI
    participant Identity Center
    participant Usuario
    participant Navegador
    
    AWS CLI->>Identity Center: 1. Solicita device code
    Identity Center-->>AWS CLI: 2. Retorna device_code + user_code + verification_uri
    AWS CLI->>Usuario: 3. Muestra user_code y URL
    Usuario->>Navegador: 4. Abre verification_uri
    Usuario->>Navegador: 5. Ingresa user_code
    Navegador->>Identity Center: 6. Autoriza el dispositivo
    loop Polling
        AWS CLI->>Identity Center: 7. Polling por tokens
        Identity Center-->>AWS CLI: 8. pending/authorized
    end
    Identity Center-->>AWS CLI: 9. Retorna access_token
```
### Paso a paso detallado:
1. Solicita device code
- **Qué hace AWS CLI**: Envía una petición HTTP POST al endpoint `/device_authorization` de Identity Center
- **Datos enviados**: `client_id` (identificador de la aplicación AWS CLI) y `scope` (permisos solicitados como `sso:account:access`)
- **Autenticación**: No se requieren credenciales de usuario en este punto, solo identificación de la aplicación
- **Timing**: Ocurre inmediatamente cuando ejecutas `aws sso login`
- **Request ejemplo**:
    ```http
    POST /device_authorization
    Content-Type: application/x-www-form-urlencoded

    client_id=arn:aws:sso::123456789012:application/ssoins-abc123&scope=sso:account:access
    ```

> AWS CLI solicita un device code a Identity Center
> No se requieren credenciales en este punto

2. Retorna device_code + user_code + verification_uri
- **Response de Identity Center**: JSON con múltiples campos esenciales
- **device_code**: Cadena larga y única (ej:`abcdef123456789...`) que identifica esta sesión específica
- **user_code**: Código corto y amigable (ej: `QWER-TYUI`) que el usuario puede escribir fácilmente
- **verification_uri**: URL donde el usuario debe ir (ej: `https://device.sso.us-east-1.amazonaws.com/`)
- **verification_uri_complete**: URL que incluye el user_code pre-poblado (opcional)
- **expires_in**: Tiempo en segundos (típicamente 600 = 10 minutos)
- **interval**: Frecuencia recomendada para polling (típicamente 5 segundos)
- **Response ejemplo**:
    ```json
    {
        "device_code": "abcdef123456789abcdef123456789abcdef123456789",
        "user_code": "QWER-TYUI",
        "verification_uri": "https://device.sso.us-east-1.amazonaws.com/",
        "verification_uri_complete": "https://device.sso.us-east-1.amazonaws.com/?user_code=QWER-TYUI",
        "expires_in": 600,
        "interval": 5
    }

> Identity Center retorna:
>   - `device_code`: Código único para el dispositivo
>   - `user_code`: Código que el usuario debe ingresar
>   - `verification_uri`: URL donde el usuario debe ir
>   - `expires_in`: Tiempo de expiración

3. Muestra user_code y URL
- **Output del CLI**: Formatea la información de manera user-friendly
- **Auto-browser**: AWS CLI intenta abrir automáticamente el navegador usando el comando del sistema
- **Fallback**: Si no puede abrir el navegador, muestra las instrucciones manuales
- **Información mostrada**: URL, código de usuario, y instrucciones claras
- **Mensaje típico**:
    ```bash
    Attempting to automatically open the SSO authorization page in your default browser.
    If the browser does not open or you wish to use a different device to authorize this request, 
    open the following URL:

    https://device.sso.us-east-1.amazonaws.com/

    Then enter the code: QWER-TYUI

    Waiting for authorization to complete...
    ```

> AWS CLI muestra al usuario el `user_code` y la `verification_uri`

4. Abre verification_uri
- **Usuario toma acción**: Puede usar cualquier dispositivo con navegador (smartphone, tablet, otra laptop)
- **Página inicial**: Identity Center muestra una página de bienvenida para verificación de dispositivos
- **Validaciones**: La página verifica que la URL es válida y está activa
- **Experiencia**: Página limpia con un campo para ingresar el código de usuario
- **Seguridad**: HTTPS obligatorio, validación del dominio AWS oficial
- **UI típica**: Formulario simple con campo de texto para el user_code

> Usuario abre la URL en cualquier navegador

5. Ingresa user_code
- **Input del usuario**: Escribe el código exacto mostrado por CLI (case-sensitive)
- **Validación inmediata**: Identity Center verifica que el código existe y no ha expirado
- **Feedback**: Página confirma si el código es válido o muestra error específico
- **Estados posibles**:
    - ✅ Código válido → Procede al siguiente paso
    - ❌ Código inválido → "Code not found or expired"
    - ❌ Código expirado → "This code has expired, please restart the process"
- **UX**: Típicamente el código se formatea automáticamente (QWER-TYUI)

> Usuario ingresa el `user_code`

6. Autoriza el dispositivo
- **Autenticación del usuario**: Pantalla de login estándar de Identity Center
- **Credenciales**: Username/password del usuario, más MFA si está configurado
- **Información del dispositivo**: Muestra detalles sobre qué está solicitando autorización:
    - Aplicación: "AWS CLI"
    - Dispositivo: Información del sistema operativo/hostname (si disponible)
    - Permisos: Lista de accounts/roles que tendrá acceso
- **Pantalla de confirmación**: "¿Autorizar AWS CLI en [nombre-dispositivo] para acceder a tus cuentas AWS?"
- **Opciones**: Botones "Authorize" y "Deny"
- **Audit logging**: Identity Center registra la autorización para auditoría

> Usuario se autentica en Identity Center
> Usuario autoriza al dispositivo (AWS CLI)

7. Polling por tokens (loop)
- **Proceso en background**: AWS CLI ejecuta un loop mientras el usuario completa la autorización
- **HTTP Requests**: Peticiones POST continuas al endpoint `/token`
- **Frecuencia**: Cada 5 segundos (según el `interval` recibido en paso 2)
- **Request formato**:
    ```http
    POST /token
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:device_code
    &device_code=abcdef123456789abcdef123456789abcdef123456789
    &client_id=arn:aws:sso::123456789012:application/ssoins-abc123
    ```
- **Rate limiting**: AWS aplica límites para evitar abuse del polling
- **Timeout**: Continúa hasta expiración del device_code (10 minutos típicamente)

> Mientras tanto, AWS CLI hace polling al endpoint de tokens
> Verifica si el usuario ya autorizó

8. Pending/authorized
- **Response estados**: Identity Center responde con diferentes HTTP status codes y mensajes
- **authorization_pending**:
    - HTTP 400 Bad Request
    - Body: `{"error": "authorization_pending"}`
    - Significado: Usuario aún no ha completado el proceso
- **slow_down**:
    - HTTP 400 Bad Request
    - Body: `{"error": "slow_down"}`
    - Significado: CLI está haciendo polling muy rápido, debe incrementar intervalo
- **access_denied**:
    - HTTP 400 Bad Request
    - Body: `{"error": "access_denied"}`
    - Significado: Usuario rechazó la autorización
- **expired_token**:
    - HTTP 400 Bad Request
    - Body: `{"error": "expired_token"}`
    - Significado: Device code expiró, debe reiniciar proceso
- **Autorización exitosa**: 
    - HTTP 200 OK con tokens (paso 9)

9. Retorna access_token
- **Response exitoso**: HTTP 200 OK con JSON conteniendo tokens
- **Tokens incluidos**:
    - `access_token`: Token JWT para hacer llamadas a AWS APIs
    - `token_type`: Típicamente "Bearer"
    - `expires_in`: Duración del token en segundos (típicamente 3600 = 1 hora)
    - `refresh_token`: Para renovar el access_token sin re-autenticación
- **Almacenamiento local**: AWS CLI guarda los tokens en `~/.aws/sso/cache/`
- **Formato de archivo**: JSON cifrado con metadata adicional
- **Configuración de perfil**: CLI actualiza la configuración para usar estos tokens
- **Response ejemplo**:
    ```json
    {
        "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
        "token_type": "Bearer",
        "expires_in": 3600,
        "refresh_token": "def50200abcdef123456789..."
    }
    ```
- **Confirmación**: CLI muestra mensaje "Successfully logged in" y está listo para usar

> Una vez autorizado, Identity Center retorna los tokens
> AWS CLI puede ahora hacer llamadas autenticadas

## ⚙️ Ejemplo práctico: AWS CLI SSO Login <a name="ejemplo"></a> 
- Comando inicial:
    ```bash
    aws sso login --profile my-sso-profile
    ```
- Output típico:
    ```bash
    Attempting to automatically open the SSO authorization page in your default browser.
    If the browser does not open or you wish to use a different device to authorize this request, open the following URL:

    https://device.sso.us-east-1.amazonaws.com/

    Then enter the code:

    QWER-TYUI
    ```
- Lo que pasa por detrás:
1. Request inicial (simplificado):
    ```http
    POST https://oidc.us-east-1.amazonaws.com/device_authorization
    Content-Type: application/x-www-form-urlencoded

    client_id=arn:aws:sso::123456789012:application/ssoins-1234567890abcdef
    scope=sso:account:access
    ```
2. Response (simplificado):
    ```json
    {
        "device_code": "abcdef123456...",
        "user_code": "QWER-TYUI",
        "verification_uri": "https://device.sso.us-east-1.amazonaws.com/",
        "verification_uri_complete": "https://device.sso.us-east-1.amazonaws.com/?user_code=QWER-TYUI",
        "expires_in": 600,
        "interval": 5
    }
    ```
3. Polling requests:
    ```http
    POST https://oidc.us-east-1.amazonaws.com/token
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:device_code
    &device_code=abcdef123456...
    &client_id=arn:aws:sso::123456789012:application/ssoins-1234567890abcdef
    ```
## ⚙️ Ventajas del Device Flow<a name="ventajas"></a> 
### Para usuarios:
- **Conveniencia**: Pueden autorizar desde cualquier dispositivo
- **Seguridad**: No ingresan credenciales en la terminal
- **Flexibilidad**: Pueden usar un dispositivo más cómodo para la autenticación
### Para aplicaciones:
- **Sin almacenamiento de credenciales**: No necesita guardar passwords
- **Multiplataforma**: Funciona en cualquier sistema operativo
- **Sin navegador**: No requiere integración con browsers
### Para administradores:
- **Control centralizado**: Todas las autorizaciones pasan por Identity Center
- **Auditoría**: Registro completo de autorizaciones
- **Políticas**: Pueden aplicar políticas de acceso condicional

## ⚙️ Consideraciones de seguridad<a name="seguridad"></a> 
### Buenas prácticas:
- **Códigos únicos**: Los user_codes son únicos y de un solo uso
- **Expiración**: Los device codes expiran (típicamente 10-15 minutos)
- **Rate limiting**: Límites en el polling para evitar abuso
- **Validación**: Verificación del dispositivo antes de emitir tokens
### Posibles ataques y mitigaciones:
- **Code interception**: Los códigos son de corta duración
- **Phishing**: URLs oficiales verificables
- **Brute force**: Rate limiting en attempts

## ⚙️ Diferencias con otros OAuth flows<a name="diferencia"></a> 

|Aspecto|Device Flow|Authorization Code|Client Credentials|
|-------|-----------|------------------|------------------|
|Uso típico|CLI, TV, IoT|Web apps|Server-to-server|
|Navegador requerido|En otro dispositivo|En el mismo dispositivo|No|
Interacción usuario|Sí (en otro dispositivo)|Sí|No|
|Redirect URI|No aplicable|Requerida|No aplicable|
|Complejidad|Media|Baja|Muy baja|

## ⚙️ Troubleshooting común<a name="problemas"></a>
### Error: "authorization_pending"
> **Causa:** Usuario aún no ha completado la autorización<br>
> **Solución:** Continuar polling, verificar que usuario complete el proceso
### Error: "slow_down"
> **Causa**: Polling muy frecuente<br>
> **Solución**: Incrementar el intervalo de polling
### Error: "expired_token"
> **Causa**: Device code expiró<br>
> **Solución**: Reiniciar el flujo completo
### Error: "access_denied"
> **Causa**: Usuario rechazó la autorización<br>
> **Solución**: Usuario debe reintentar y aceptar

## ⚙️ El problema de usar Authorization Code Flow en CLI <a name="problema"></a> 
- Usar `Authorization Code Flow` para CLI es problemático.
- ¿Como funciona `Authorization Code Flow`?
    > 1.- Aplicación → "Ve a esta URL para loggearte"<br>
    > 2.- Usuario → Abre navegador y va a la URL<br>
    > 3.- Usuario → Se loggea en el navegador<br>
    > 4.- Proveedor → Redirige de vuelta a la aplicación<br>
    > 5.- Aplicación → Recibe el código y lo intercambia por tokens<br>
- ¿Cuál es el probema de usar el flujo `Authorization Code Flow` con CLI?
    > - **Paso 1: "Ve a esta URL"**
    >   - CLI no puede "abrir" un navegador de forma confiable
    >   - En servidores SSH puede no haber navegador disponible
    > - **Paso 4: "Redirige de vuelta a la aplicación"**
    >   - ¿A dónde redirigir? CLI no tiene una URL de callback donde redireccionar
    >   - ¿Cómo captura el CLI la respuesta del redirect?
- Flujo visual de Authorization Code Flow
    ```bash
    CLI ←--[redirect]-- Navegador ←--[auth]-- Usuario
    ↑                     ↑
    [abrir browser]    [¿cómo capturar?]
    ```
- Ejemplo del problema real:
    ```bash
    # Lo que NO puede hacer un CLI tradicional:
    $ aws s3 ls
    Error: Necesitas autenticarte
    Ve a: https://auth.aws.com/login?redirect_uri=???
    # ¿Redirect a dónde? ¿http://localhost:8080?
    # ¿Qué pasa si el puerto está ocupado?
    # ¿Qué pasa si no hay navegador disponible?
    ```

## ⚙️ ¿Cómo Device Flow resueve el problema?<a name="solucion"></a> 
- Device Flow elimina la necesidad de redirect:
    > 1.- CLI → "Necesito autorización"<br>
    > 2.- Proveedor → "Aquí tienes un código: ABCD-1234"<br>
    > 3.- CLI → "Usuario, ve a esta URL e ingresa: ABCD-1234"<br>
    > 4.- Usuario → Va a la URL en CUALQUIER dispositivo<br>
    > 5.- Usuario → Ingresa el código y se autentica<br>
    > 6.- CLI → Hace polling: "¿Ya me autorizó?"<br>
    > 7.- Proveedor → "Sí, aquí están tus tokens"<br>
- Flujo visual Device Authorization Flow
    ```bash
    CLI ←--[polling]-- Proveedor ←--[auth]-- Navegador ←-- Usuario
                                                ↑
                                        [cualquier dispositivo]
    ```
> [!NOTE]  
> **Ventaja clave de uso de Device Flow:**
> - No hay redirect → No necesita capturar respuestas HTTP
> - No abre navegador → Funciona en cualquier entorno
> - Usuario decide qué dispositivo usar → Máxima flexibilidad

## ⚙️ ¿Cómo funciona en la práctica?: Escenario típico con AWS CLI<a name="practica"></a>
### El usuario autoriza en un dispositivo separado
- Dispositivo de incio de sesión (pueder estar limitado)
    ```bash
    # En nuestra laptop/servidor (dispositivo limitado)
    $ aws sso login --profile my-profile
    # CLI muestra:
    Attempting to automatically open the SSO authorization page...
    If the browser does not open, go to: https://device.sso.amazonaws.com/
    Then enter the code: BCFG-HJKL
    Waiting for authorization to complete...
    ```
### Lo que hacemos como usuarios
1. En nuestro smartphone (dispositivo con capacidades completas):
    - Abrimos el link: https://device.sso.amazonaws.com/
    - Vemos una pantalla que dice "Enter device code"
    - Escribimos: BCFG-HJKL
    - Nos loggeamos con  usuario/password normal
    - Confirmar: "¿Autorizar AWS CLI en nuestra laptop?"
2. De vuelta en nuestra laptop:
    - CLI detecta que autorizaste
    - Descarga los tokens automáticamente
    - Ya podemos usar comandos AWS
### Ventajas de separ dispositivos (capacidades completas/capacidades limitadas):
- Comodidad:
    - Usas tu teléfono (teclado táctil, autocompletado, biometrics)
    - No tienes que escribir passwords complejos en terminal
- Seguridad:
    - Credenciales nunca pasan por el dispositivo limitado
    - Puedes usar 2FA, biometrics, etc. en tu teléfono
- Flexibilidad:
    - Puedes autorizar desde cualquier dispositivo con navegador
    - Útil cuando el dispositivo no tiene interfaz gráfica o navegador disponible

> [!NOTE] 
> - OAuth 2.0 = Protocolo flexible y bueno
> - Device Flow = Una herramienta específica dentro de OAuth 2.0
> - Authorization Code Flow = Otra herramienta, pero para casos diferentes

## ⚙️ ¿Por qué CLI es el caso de uso perfecto? <a name="caso"></a>
### Especialmente útil para aplicaciones de línea de comandos
- Antes del Device Flow
    - Access Keys hardcodeados:
        ```bash
        export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
        export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        ```
    - Profiles con credenciales:
        ```bash
        [default]
        aws_access_key_id = AKIAIOSFODNN7EXAMPLE  
        aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        ```

> [!CAUTION]
> - Problema: Access Keys hardcodeados<br>
>   - Keys permanentes, riesgo si se comprometen, difíciles de rotar<br>
> - Problema: Profiles con credenciales<br>
>   - Credenciales en texto plano en archivos locales

- Con Device Flow (método moderno):
    ```bash
    aws sso login --profile my-sso-profile
    ```

> [!NOTE]
> - Beneficios método: Device Flow<br>
>   - Autorización una vez, tokens temporales automáticos
>   - Renovación transparente
>   - Sin credenciales permanentes almacenadas

### Comparación práctica:

|Método|Seguridad|Comodidad|Mantenimiento|
|------|---------|---------|-------------|
|Access Keys|❌ Permanentes|✅ Simple setup|❌ Rotación manual|
|IAM Roles|✅ Temporales|❌ Complejo setup|❌ Configuración compleja|
|Device Flow|✅ Temporales|✅ Simple UX|✅ Automático|

### Otros ejemplos de CLI que usan Device Flow:
- GitHub CLI: `gh auth login`
- Azure CLI: `az login --use-device-code`
- Google Cloud CLI: `gcloud auth login --no-launch-browser`
 - Docker CLI: Para acceder a registries privados

## ⚙️ Ejemplos de dispositivos con capacidades de entrada limitadas <a name="limitadas"></a> 
### Smart TVs:
- Solo tienen control remoto (botones arriba/abajo/izquierda/derecha)
- No pueden mostrar un teclado QWERTY completo
- Escribir un email y password sería muy lento y frustrante
### Dispositivos IoT:
- Termostatos inteligentes, cámaras de seguridad
- Pueden tener solo una pantalla pequeña y pocos botones
- No hay forma práctica de ingresar credenciales complejas
### Consolas de videojuegos:
- Controladores de juego no son ideales para escribir texto
- Navegar por teclados en pantalla es lento

## ⚙️ ¿Por qué Device Authorization Flow evita usar el navegador web en el dispositivo? <a name="navegador"></a> 
### El problema tradicional usando Authorization Code
- En flujos OAuth normales (como Authorization Code), el proceso es:
    - Aplicación redirige al usuario a una página de login
    - Usuario ingresa credenciales en el navegador
    - Proveedor redirige de vuelta a la aplicación
### ¿Por qué esto no funciona en CLI?
- AWS CLI ejemplo:
    ```bash
    $ aws s3 ls
    # ¿Cómo debería abrirse un navegador aquí?
    # ¿En qué URL redireccionar después del login?
    # ¿Cómo capturar la respuesta en la terminal?
    ```
### Problemas específicos:
- Terminal no puede "abrir" un navegador de forma confiable
- No hay una URL de callback donde redireccionar
- Diferentes sistemas operativos manejan browsers diferente
- En servidores remotos (SSH) puede no haber interfaz gráfica

---

## 🔗 Referencias
### Especificaciones:
- [RFC 8628: OAuth 2.0 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628)
- [RFC 6819: OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/rfc6819)
### Documentación AWS:
- [AWS IAM Identity Center OpenID Connect (OIDC  API Reference)](https://docs.aws.amazon.com/singlesignon/latest/OIDCAPIReference/Welcome.html)
- [Configuring IAM Identity Center authentication with the AWS CLI]()
### Herramientas de desarrollo:
- [OAuth 2.0 Debugger](https://oauthdebugger.com/)
- [JSON Web Token (JWT) Debugger-Para inspeccionar tokens JWT](https://www.jwt.io/)

## 📖 Conceptos relacionados
- **OpenID Connect (OIDC)**: Extensión de OAuth 2.0 que AWS usa.
- **PKCE (Proof Key for Code Exchange)**: Extensión de seguridad para OAuth.
- **JWT (JSON Web Tokens)**: Formato común para tokens de acceso.
- **Refresh Tokens**: Para obtener nuevos access tokens sin re-autorización

---