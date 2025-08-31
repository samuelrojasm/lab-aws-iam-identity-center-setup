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
- [OAuth Device Flow: Autenticación sin navegador](#oauth)
- [AWS STS: El Traductor de Tokens](#sts)
- [La Danza Completa: `aws sso login --profile`](#danza)
- [¿Por qué esta arquitectura funciona?](#funciona)
- [Diferencias clave entre los flows](#flows)
- [Consideraciones importantes](#importantes)

## ⚙️ El problema que resolvemos <a name="intro"></a> 
- Imagina una organización moderna donde los empleados necesitan acceder a AWS desde múltiples contextos:
    - María (Gerente de Proyecto) accede desde su navegador web durante reuniones
    - Carlos (DevOps Engineer) usa la terminal y AWS CLI todo el día
    - Ana (Data Scientist) trabaja desde Jupyter notebooks sin navegador
    - Roberto (Mobile Developer) necesita acceso desde aplicaciones móviles
- **El desafío**: Todos necesitan autenticarse contra el mismo sistema corporativo (Active Directory, Okta, etc.) pero desde interfaces completamente diferentes.
- **La solución tradicional fallida**: Crear usuarios IAM individuales sería un nightmare de seguridad y gestión.
- **La solución elegante:** Una arquitectura que combina SAML (para web) + OAuth Device Flow (para CLI/aplicaciones) + AWS STS (como orquestador central)

## ⚙️ Los actores principales  <a name="actores"></a> 
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
### ¿Por Qué SAML Funciona tan bien para Web?
- **Redirects naturales**: Los navegadores manejan redirects automáticamente
- **Cookies y sesiones**: Mantiene el estado de autenticación
- **POST forms**: Puede enviar datos grandes (assertions) fácilmente
- **Universal**: Funciona en cualquier navegador sin instalaciones

## ⚙️ OAuth Device Flow: Autenticación sin navegador <a name="oauth"></a> 
### El problema del CLI
- Cuando Carlos ejecuta `aws sso login --profile dev-account`, su terminal no tiene un navegador integrado. No puede mostrar una página de login, no puede manejar redirects, no puede procesar JavaScript.
### La Solución Ingeniosa: Device Flow
- El Device Flow es como "autenticación por proxy":
    - **El CLI le dice a AWS**: "Necesito que autentiques este dispositivo"
    - **AWS responde**: "Ok, dile al usuario que vaya a https://device.sso.aws.com e ingrese el código ABCD-1234"
    - **El usuario** abre su **navegador** normal y completa la autenticación
    - Mientras tanto, el CLI está esperando pacientemente preguntando "¿ya terminó?"
    - Una vez completado, AWS le da al CLI los tokens necesarios
### El flujo Device Authorization explicado
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

## ⚙️ AWS STS: El traductor de Tokens <a name="sts"></a>
### ¿Qué hace STS Realmente?
- STS es como un "cajero automático de credenciales". No importa si llegas con:
    - Un cheque SAML (desde navegador web)
    - Un token OAuth (desde CLI)
    - Un certificado de otra cuenta AWS (cross-account)
- STS los convierte todos en la misma moneda: credenciales AWS temporales.
### Los Tres "Cajeros" Principales de STS
1. AssumeRoleWithSAML
    - **Input**: SAML Assertion + Role ARN
    - **Output**: Credenciales AWS temporales
    - **Uso típico**: Navegadores web, aplicaciones web
2. AssumeRoleWithWebIdentity
    - **Input:** OAuth/OpenID Connect token + Role ARN
    - **Output**: Credenciales AWS temporales
    - **Uso típico**: CLI tools, aplicaciones móviles, aplicaciones nativas
3. AssumeRole
    - **Input**: Credenciales AWS existentes + Role ARN
    - **Output**: Nuevas credenciales AWS temporales (para otro rol)
    - **Uso típico**: Cross-account access, privilege escalation controlada
### ¿Cómo STS toma decisiones?
- STS no decide arbitrariamente. Cada rol tiene una "Trust Policy" que es como una lista de invitados:
    - **Rol**: "Marketing-S3-Access"
    - **Trust Policy dice**: "Confío en usuarios que vengan de:"
        - IdP corporativo (SAML)
        - Con atributo department="marketing"  
        - Y que el email termine en "@empresa.com"

## ⚙️ La Danza Completa: `aws sso login --profile` <a name="danza"></a>
### Flujo A: Usuario Web (SAML) - María accede desde el navegador
- Paso 1: Acceso Directo a AWS Console
    ```bash
    👤 María → https://mi-empresa.awsapps.com/start
    AWS SSO: "No tienes sesión activa, te redirijo al IdP"
    ```
- Paso 2: Redirección SAML al IdP
    ```bash
    🌐 AWS SSO → IdP: Envía SAML AuthnRequest
    "Por favor autentica a este usuario para AWS"
    ```
- Paso 3: Autenticación en el IdP
    ```bash
    🏢 IdP muestra login → María ingresa credenciales
    IdP valida → "Sí, es María del departamento Marketing"
    ```
- Paso 4: SAML Assertion de Vuelta
    ```bash
    🏢 IdP → AWS SSO: Envía SAML Assertion firmada
    Assertion contiene:
        - Subject: maria.lopez@empresa.com  
        - Attributes: Department=Marketing, Groups=Marketing-ReadOnly
        - Signature: Certificado del IdP
    ```
- Paso 5: STS con SAML
    ```bash
    🌐 AWS SSO → STS: AssumeRoleWithSAML
        - saml_assertion: [la assertion del IdP]
        - role_arn: "arn:aws:iam::123456789012:role/AWSReservedSSO_Marketing-ReadOnly_xyz"

    🔐 STS valida assertion → verifica Trust Policy → genera credenciales
    ```
- Paso 6: Acceso a AWS Console
    ```bash
    ✅ María ve el portal AWS SSO con sus cuentas disponibles
    Selecciona cuenta → obtiene credenciales temporales → accede a AWS Console
    ```

### Flujo B: Usuario CLI (OAuth Device Flow) - Carlos usa `aws sso login --profile`
- Paso 1: CLI Descubre la configuración
    ```bash
    💻 El CLI lee ~/.aws/config:
        [profile dev-account]
        sso_start_url = https://mi-empresa.awsapps.com/start
        sso_region = us-east-1
        sso_account_id = 123456789012
        sso_role_name = DeveloperAccess
    ```
- Paso 2: Inicia Device Authorization Flow
    ```bash
    💻 CLI → AWS SSO: "Necesito autorizar este device para el perfil dev-account"
    🌐 AWS SSO → CLI: {
        "device_code": "secreto-que-solo-el-cli-conoce",
        "user_code": "WXYZ-1234", 
        "verification_uri": "https://device.sso.aws.com"
    }
    ```
- Paso 3: Aqupi entra SAML - Usuario autoriza en navegador
    ```bash
    💻 CLI muestra: "Ve a https://device.sso.aws.com e ingresa: WXYZ-1234"

    👤 Carlos abre navegador → ingresa código WXYZ-1234
    🌐 AWS SSO: "¿Quién eres? Te redirijo al IdP"
    🏢 IdP ejecuta mismo flujo SAML que María:
        - Carlos se autentica
        - IdP genera SAML Assertion  
        - AWS SSO recibe y valida assertion
        - AWS SSO asocia device_code con identity de Carlos
    ```
- Paso 4: CLI Obtiene Tokens OAuth
    ```bash
    💻 CLI (polling cada 5 segundos): "¿Ya terminó la autorización?"
    🌐 AWS SSO: "Sí, Carlos se autenticó vía SAML, aquí están tus tokens OAuth"
    💻 CLI recibe: access_token, id_token, refresh_token
    ```
- Paso 5: Conversión a Credenciales AWS
    ```bash
    💻 CLI → STS: AssumeRoleWithWebIdentity
        - web_identity_token: [el id_token de OAuth que contiene info de Carlos]
        - role_arn: "arn:aws:iam::123456789012:role/AWSReservedSSO_DeveloperAccess_xyz"

    🔐 STS valida token → verifica Trust Policy → genera credenciales temporales
        STS → CLI: {
            "AccessKeyId": "ASIA...",
            "SecretAccessKey": "...",
            "SessionToken": "...",
            "Expiration": "2025-08-30T13:00:00Z"
        }
    ```
- Paso 6: CLI guarda y usa credenciales
    ```bash
    💻 CLI guarda en ~/.aws/cli/cache/
    Futuras llamadas AWS usan estas credenciales automáticamente
    Cuando expiren, CLI usa refresh_token para renovar sin re-autenticar
    ``
### El Punto Clave: SAML Siempre Está Presente**
- **La revelación importante**: Incluso en el flujo OAuth Device, SAML sigue siendo usado para la autenticación real del usuario. OAuth Device Flow es simplemente el mecanismo de autorización del dispositivo, pero la autenticación del usuario sigue siendo SAML.
    ```bash
    Flujo Web:     Usuario → SAML → STS
    Flujo Device:  Usuario → OAuth Device → [SAML en el navegador] → OAuth Tokens → STS
    ```
- **Por eso funciona tan bien**: El IdP no necesita saber si el usuario viene de una web app o de un CLI tool. Siempre usa SAML para autenticar, y AWS SSO se encarga de "traducir" esa autenticación al formato que necesita cada cliente (SAML assertion directa vs OAuth tokens).

### ¿Por Qué este proceso es tan complejo?
- La complejidad existe por buenas razones de seguridad:
    - **Separación de contextos**: CLI no maneja credenciales del usuario directamente
    - **Tokens temporales**: Las credenciales expiran automáticamente
    - **Validación múltiple**: Cada paso valida la legitimidad del request
    - **Audit trail**: Cada paso genera logs para auditoría
    - **Flexibility**: Mismo IdP puede servir web, móvil, CLI, APIs

## ⚙️ ¿Por qué esta arquitectura funciona <a name="funciona"></a>
### Ventajas del modelo Híbrido
1. Experiencia de Usuario Optimizada
    - **Web users**: Login transparente con SSO, sin interrupciones
    - **CLI users**: Setup una vez, luego invisible
    - **Mobile apps**: Native flow sin embebidos browsers
    - **Scripts**: Refresh automático de credenciales
2. Seguridad robusta
    - **No long-term credentials**: Todo expira automáticamente
    - **Principio de menor privilegio**: Cada contexto obtiene solo lo necesario
    - **Audit completo**: Cada autenticación se registra
    - **Revocación centralizada**: Deshabilitar usuario afecta todos los accesos
3. Escalabilidad empresarial
    - **Single source of truth**: Un IdP para toda la organización
    - **Consistent policies**: Mismas reglas para web y CLI
    - **Easy onboarding**: Nuevos usuarios automáticamente obtienen acceso correcto
    - **Cross-account**: Mismo patrón funciona para múltiples cuentas AWS
4. Flexibilidad técnica
    - **Protocol agnóstic**: SAML para web, OAuth para APIs/CLI
    - **IdP independence**: Cambiar IdP no requiere reconfigurar aplicaciones
    - **AWS native**: Integración profunda con servicios AWS
### Casos de uso reales
- Desarrollo de Software
    ```bash
    Developer workflow:
    1. Llega al trabajo → laptop ya tiene credenciales cacheadas
    2. git push → CI/CD pipeline usa service account con OAuth
    3. Debug production → aws sso login --profile prod (requiere MFA adicional)
    4. Code review → Web console con SAML SSO
    ```
- Operaciones Cloud
    ```bash
    SRE workflow:
    1. Incident response → CLI tools con credenciales pre-autorizadas
    2. Infrastructure changes → Terraform con service principal OAuth
    3. Monitoring dashboards → Web access con SAML
    4. Emergency access → Break-glass procedure con temporary elevated permissions
    ```
- Business Intelligence
    ```bash
    Data team workflow:
    1. Data scientists → Jupyter notebooks con SDK usando OAuth refresh tokens
    2. Business users → QuickSight dashboards con SAML SSO
    3. ETL processes → Lambda functions con IAM roles (no user credentials)
    4. Ad-hoc queries → CLI tools con temporary elevated access
    ```

## ⚙️ Diferencias clave entre los flows <a name="flows"></a>
### SAML vs OAuth Device Flow: ¿Cuándo se usa cada uno?

|Aspecto|SAML Federation|OAuth Device Flow|
|--------|--------------|-----------------|
|Contexto de uso|Navegadores web, aplicaciones web|CLI tools, aplicaciones nativas, IoT|
|Experiencia de usuario|Seamless redirects|"Ve a URL X e ingresa código Y"|
|Información transportada|Attributes ricos (department, groups, etc.)|Claims estándar (email, groups)|
|Duración de sesión|Basada en cookies del navegador|Refresh tokens de larga duración|
|Complejidad de implementació|nMedia (manejo de XML, certificates)|Alta (polling, device codes, refresh logic)
|Security model|Basado en redirects y POST forms|Basado en tokens y polling|
|Offline capability|No (requiere conectividad para redirects)|Sí (refresh tokens funcionan offline)|

### ¿Por qué no usar solo SAML para todo?
- Limitaciones técnicas de SAML:
    - **No funciona en CLI**: No hay navegador para manejar redirects
    - **Complejo para móvil**: Embedded browsers son poor UX
    - **No hay refresh**: Cada sesión requiere re-autenticación completa
    - **XML parsing**: Más pesado para aplicaciones simples
### ¿Por qué no usar solo OAuth para todo?
- Limitaciones de OAuth para contextos web:
    - **Menos información**: Claims estándar vs attributes ricos de SAML
    - **Complejidad innecesaria**: Para web, los redirects SAML son más simples
    - **Legacy compatibility**: Muchos IdPs enterprise tienen mejor soporte SAML
    - **Standards maturity**: SAML tiene más años de battle-testing en enterprise
### El Sweet Spot: Híbrido
- La combinación SAML + OAuth + STS da lo mejor de ambos mundos:
    - **SAML para web**: Experiencia optimizada, attributes ricos
    - **OAuth para programmatic**: Refresh tokens, simplicidad para APIs
    - **STS como common backend**: Consistent security model, unified audit

## ⚙️ Consideraciones importantes <a name="importantes"></a>
### Puntos de fallo potenciales
1. Configuración incorrecta de Trust Policies
    - **Síntoma**: "Access Denied" incluso con autenticación exitosa
    - **Causa común**: Role trust policy no permite el IdP o faltan conditions
    - **Detección**: CloudTrail events `AssumeRoleWithSAML` o `AssumeRoleWithWebIdentity` con error
2. Clock Skew
    - **Síntoma**: Tokens válidos rechazados como "expired"
    - **Causa común**: Diferencias de tiempo entre sistemas
3. Certificate Expiration
    - **Síntoma**: SAML assertions rechazadas
    - **Causa común**: Certificados de IdP expirados o rotados
    - **Prevención**: Monitoring de certificate expiration dates
4. Token Refresh Failures
    - **Síntoma**: CLI deja de funcionar después de período de inactividad
    - **Causa común**: Refresh tokens expirados o revocados
    - **Solución**: Re-ejecutar `aws sso login --profile`
### Mejores prácticas de operación
1. Monitoring y Alerting
    - Monitor failed authentication attempts
    - Alert on unusual patterns (geography, time, volume)
    - Track credential expiration and refresh patterns
    - Monitor IdP availability and response times
2. Disaster Recovery
    - Maintain break-glass admin accounts
    - Document emergency access procedures
    - Regular testing of backup authentication methods
    - IdP failover and recovery plans
3. Performance Optimization
    - Cache SAML metadata when possible
    - Optimize token refresh intervals
    - Use connection pooling for STS calls
    - Monitor and tune session timeout values
### Conclusión: El ecosistema completo
- Esta arquitectura representa la evolución natural de la autenticación enterprise:
    - **Era 1**: Usuarios y passwords individuales (nightmare de gestión)
    - **Era 2**: SAML SSO para web (gran mejora, pero limitado)
    - **Era 3**: OAuth para APIs y móvil (flexible, pero fragmentado)
    - **Era 4**: Federación híbrida con STS (lo mejor de todos los mundos)
### ¿Por qué AWS STS es el Game Changer?
- STS actúa como el "traductor universal" que permite que diferentes protocolos de autenticación trabajen juntos seamlessly. Sin STS, necesitarías:
    - Usuarios IAM para CLI access
    - Diferentes sistemas de permisos para web vs API
    - Credenciales de larga duración (security risk)
    - Gestión manual de access patterns
### El futuro de esta Arquitectura
- Las tendencias emergentes que se integran naturalmente:
    - **WebAuthn/FIDO2**: Puede integrarse como factor adicional en SAML flow
    - **Zero Trust**: STS tokens naturalmente support conditional access
    - **Service Mesh**: OAuth tokens pueden usarse para service-to-service auth
    - **Serverless**: Lambda functions pueden asumir roles dinámicamente basado en context

> [!NOTE]
> La belleza de esta arquitectura es que es extensible y future-proof. Los componentes base (IdP, STS, roles) pueden evolucionar independientemente mientras manteniendo compatibilidad.

---