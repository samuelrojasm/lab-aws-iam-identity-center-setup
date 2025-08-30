# 🧪 Guía de AWS STS (Security Token Service)

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

## Contenido
- [¿Qué es AWS  STS (Security Token Service)?](#que-es)
- [¿Cómo se relaciona STS con Identity Center?](#relacion)
- [Tipos de operaciones STS relevantes para Identity Center](#operaciones)
- [Anatomía de las credenciales temporales](#anatomia)
- [Duración y expiración de tokens](#duracion)
- [Refresh de credenciales](#refresh)
- [Seguridad de tokens STS](#sec)
- [Troubleshooting común con STS](#troubleshooting)
- [Comparación: STS vs Access Keys permanentes](#comparar)
- [Monitoreo y auditoría](#monitoreo)
- [Integración con otros servicios AWS](#integracion)
- [Comandos útiles de referencia rápida](#comandos)

---

## ⚙️ ¿Qué es AWS  STS (Security Token Service)? <a name="que-es"></a> 
- AWS Security Token Service (STS) es un servicio web que permite **solicitar credenciales temporales y de privilegios limitados** para usuarios de AWS Identity and Access Management (IAM) o para usuarios que autenticas (usuarios federados).
- Características principales:
    - **Credenciales temporales**: Expiran automáticamente (15 minutos a 12 horas)
    - **Privilegios limitados**: Solo los permisos específicos del role asumido
    - **Global**: Disponible en todas las regiones AWS
    - **Sin costo adicional**: No hay charges por usar STS

> [!NOTE]
> STS es un servicio fundamental que opera transparentemente cuando usas Identity Center. Aunque no interactúas directamente con STS, entender su funcionamiento te ayuda a diagnosticar problemas y optimizar la configuración de duraciones de sesión.

## ⚙️ ¿Cómo se relaciona STS con Identity Center? <a name="relacion"></a> 
- **Identity Center actúa como un intermediario** que utiliza AWS STS por detrás para generar las credenciales que tú usas para acceder a recursos AWS.
### Arquitectura simplificada:
- Diagrama:
    ```mermaid
    flowchart TD
    User[Usuario] --> IC[Identity Center]
    IC --> STS[AWS STS]
    STS --> Creds[Credenciales Temporales]
    Creds --> AWS[Recursos AWS]
    
    IC -.- PS[Permission Sets]
    PS -.- Roles[IAM Roles]
    STS -.-> Roles
    ```
### El flujo completo:
1. **Usuario se autentica** → Identity Center (via Device Flow)
2. **Identity Center valida** → Usuario y sus permission sets
3. **Identity Center llama** → STS AssumeRoleWithSAML
4. **STS genera** → Credenciales temporales basadas en el IAM Role
5. **Identity Center retorna** → Credenciales al usuario
6. **Usuario accede** → Recursos AWS con esas credenciales

## ⚙️ Tipos de operaciones STS relevantes para Identity Center  <a name="operaciones"></a> 
1. AssumeRoleWithSAML
- La operación principal que usa Identity Center:
    ```json
    {
        "RoleArn": "arn:aws:iam::123456789012:role/AWSReservedSSO_DeveloperAccess_abc123",
        "PrincipalArn": "arn:aws:iam::123456789012:saml-provider/ExampleProvider",
        "SAMLAssertion": "PHNhbWw6QXNzZXJ0aW9uLi4u...",
        "DurationSeconds": 3600
    }
    ```
- Response:
    ```json
    {
        "Credentials": {
            "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
            "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY",
            "SessionToken": "FwoGZXIvYXdzEDo...",
            "Expiration": "2024-01-15T23:28:33Z"
        },
        "AssumedRoleUser": {
            "AssumedRoleId": "AROA123456789012:user@example.com",
            "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_DeveloperAccess_abc123/user@example.com"
        }
    }
    ```

2. GetCallerIdentity
- Para verificar qué credenciales estás usando:
    ```bash
    aws sts get-caller-identity
    ```
    ```json
    {
        "UserId": "AROA123456789012:user@example.com",
        "Account": "123456789012",
        "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_DeveloperAccess_abc123/user@example.com"
    }
    ```

## ⚙️ Anatomía de las credenciales temporales <a name="anatomia"></a> 
- Componentes de las credenciales STS:
    ```bash
    # Las credenciales que ves en ~/.aws/credentials después de aws sso login
    [profile my-sso-profile]
    aws_access_key_id = ASIAIOSFODNN7EXAMPLE
    aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY
    aws_session_token = FwoGZXIvYXdzEDo...
    ```
- Significado de cada componente:
    - **aws_access_key_id** (comienza con "ASIA"):
        - Identifica las credenciales temporales
        - Diferente de access keys permanentes (que comienzan con "AKIA")
        - Contiene información sobre cuándo expiran
    - **aws_secret_access_key**:
        - Clave secreta para firmar requests
        - Generada específicamente para esta sesión
        - Se invalida automáticamente al expirar
    - **aws_session_token**:
        - Token que debe incluirse en todas las llamadas API
        - Contiene información sobre el role asumido
        - AWS lo valida en cada request

## ⚙️ Duración y expiración de tokens  <a name="duracion"></a> 
- Configuración de duración:
    - En Permission Sets (Identity Center):
        ```json
        {
            "SessionDuration": "PT1H",  // ISO 8601: 1 hora
            "MaxSessionDuration": "PT12H"  // Máximo: 12 horas
        }
        ```
    - En IAM Roles:
        ```json
        {
            "MaxSessionDuration": 43200  // 12 horas en segundos
        }
        ```
- Jerarquía de límites:
    - **Permission Set** → Define duración por defecto
    - **IAM Role** → Define duración máxima permitida
    - **STS Request** → Puede solicitar menos tiempo, nunca más
- Tabla de duraciones típicas:
    |Escenario|Duración recomendada|Justificación|
    |---------|--------------------|-------------|
    |Desarrollo interactivo|1-2 horas|Balance conveniencia/seguridad|
    |Scripts automatizados|15 minutos|Principio de menor exposición|
    |Troubleshooting|4-8 horas|Sesiones de debugging largas|
    |CI/CD pipelines|1 hora|Tiempo suficiente para builds|

## ⚙️ Refresh de credenciales <a name="refresh"></a> 
- Proceso automático de AWS CLI:
    ```mermaid
    flowchart LR
    CLI[AWS CLI] --> Check{¿Credenciales<br/>válidas?}
    Check -->|Sí| Use[Usar credenciales]
    Check -->|No| Refresh[aws sso login<br/>automático]
    Refresh --> STS[Llamada a STS]
    STS --> New[Nuevas credenciales]
    New --> Use
    ```
- Verificación manual:
    ```bash
    # Verificar expiración
    aws configure list
    aws sts get-caller-identity

    # Si falló, refresh manual
    aws sso login --profile my-sso-profile
    ```
- Archivos de cache:
    ```bash
    # Ubicación de credenciales cacheadas
    ~/.aws/sso/cache/
    ├── abc123def456.json          # Access token de Identity Center
    └── 789ghi012jkl.json         # Credenciales STS temporales
    ```

## ⚙️ Seguridad de tokens STS <a name="sec"></a> 
- Ventajas de credenciales temporales:
    - **Auto-expiración**: Riesgo limitado si se comprometen
    - **Permisos específicos**: Solo los del role asumido
    - **Auditoría completa**: CloudTrail registra todo uso
    - **Revocación**: Pueden invalidarse centralmente
- Mejores prácticas:
    - Recomendaciones:
        - Usar duraciones cortas para scripts automatizados
        - Rotar tokens frecuentemente en entornos compartidos
        - Monitorear uso inusual en CloudTrail
        - Usar MFA para operaciones sensibles
    - Evitar:
        - Hardcodear tokens STS (expiran anyway)
        - Duraciones muy largas sin justificación
        - Compartir tokens entre usuarios/sistemas
        - Almacenar tokens en repositorios de código

## ⚙️ Troubleshooting común con STS <a name="troubleshooting"></a> 
- **Error**: "The security token included in the request is invalid"
   - Causas comunes:
        ```bash
        # Token expirado
        $ aws s3 ls
        An error occurred (InvalidToken) when calling the ListBuckets operation: 
        The provided token has expired.

        # Solución
        $ aws sso login --profile my-sso-profile
        ```
- **Error**E: "Unable to locate credentials"
    - Diagnóstico:
        ```bash
        # Verificar configuración
        $ aws configure list --profile my-sso-profile

        # Verificar archivos de cache
        $ ls -la ~/.aws/sso/cache/

        # Re-login si es necesario
        $ aws sso login --profile my-sso-profile
        ```
- **Error**: "Access Denied" con credenciales válidas
    - Verificación de permisos:
        ```bash
        # ¿Qué role estoy usando?
        $ aws sts get-caller-identity

        # ¿Qué permisos tiene este role?
        $ aws iam get-role --role-name AWSReservedSSO_DeveloperAccess_abc123
        ```
- **Error**: "Session token is invalid"
    - Causa: Tokens STS requieren session token en TODOS los requests
    - Verificación:
        ```bash
        # Debe mostrar las 3 variables
        $ env | grep AWS_
        AWS_ACCESS_KEY_ID=ASIA...
        AWS_SECRET_ACCESS_KEY=...
        AWS_SESSION_TOKEN=...    # ← Esta es crítica
        ```

## ⚙️ Comparación: STS vs Access Keys permanentes <a name="comparar"></a> 

|Aspecto|STS (Temporal)|Access Keys (Permanente)|
|-------|--------------|------------------------|
|Duración|15min - 12hrs|Hasta rotación manual|
|Rotación|Automática|Manual|
|Riesgo si comprometen|Limitado por tiempo|Alto|
|Permisos|Limitados al role|Todos los permisos del usuario|
|Auditoría|Completa con contexto|Limitada|
|Revocación|Automática al expirar|Requiere acción manual|
|Uso recomendado|CLI, aplicaciones|Legacy, casos específicos|

## ⚙️ Monitoreo y auditoría <a name="monitoreo"></a> 
- CloudTrail events importantes:
    - AssumeRoleWithSAML:
        ```json
        {
            "eventName": "AssumeRoleWithSAML",
            "sourceIPAddress": "203.0.113.12",
            "userAgent": "aws-cli/2.0.0",
            "requestParameters": {
                "roleArn": "arn:aws:iam::123456789012:role/AWSReservedSSO_DeveloperAccess_abc123",
                "durationSeconds": 3600
            },
            "responseElements": {
                "assumedRoleUser": {
                    "arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_DeveloperAccess_abc123/user@example.com"
                }
            }
        }
        ```
- Métricas útiles:
    - **Frecuencia de AssumeRole**: ¿Usuarios solicitando tokens muy seguido?
    - **Duración promedio**: ¿Tokens con duraciones inusuales?
    - **Errores de expiración**: ¿Usuarios con tokens expirando frecuentemente?
    - **Ubicaciones IP**: ¿Requests desde ubicaciones inusuales?

## ⚙️ Integración con otros servicios AWS  <a name="integracion"></a> 
- Con AWS CLI:
    ```bash
    # CLI maneja STS automáticamente
    aws s3 ls --profile my-sso-profile
    # Internamente: CLI verifica token → si expiró → llama a STS → obtiene nuevas credenciales
    ```
- Con SDKs (ejemplo Python):
    ```python
    import boto3

    # Usando perfil SSO
    session = boto3.Session(profile_name='my-sso-profile')
    s3 = session.client('s3')

    # SDK maneja refresh automáticamente
    buckets = s3.list_buckets()
    ```
- Con Terraform:
    ```hcl
    provider "aws" {
        profile = "my-sso-profile"
        region  = "us-east-1"
    }

    # Terraform usa las mismas credenciales STS
    resource "aws_s3_bucket" "example" {
        bucket = "my-example-bucket"
    }
    ```


## ⚙️ Comandos útiles de referencia rápida <a name="comandos"></a> 
- Lista de comandos:
    ```bash
    # Ver credenciales actuales
    aws sts get-caller-identity

    # Ver configuración de perfil
    aws configure list --profile my-sso-profile

    # Refresh de credenciales SSO
    aws sso login --profile my-sso-profile

    # Verificar expiración de token
    aws sts decode-authorization-message --encoded-message <token>

    # Limpiar cache de credenciales
    aws sso logout
    rm -rf ~/.aws/sso/cache/

    # Verificar permisos del role actual
    aws iam simulate-principal-policy \
    --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
    --action-names s3:ListBucket \
    --resource-arns arn:aws:s3:::my-bucket
    ```

---

## 🔗 Referencias
### Documentación AWS STS:
- [AWS Security Token Service API Reference](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)
- [Temporary security credentials in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
- [AssumeRoleWithSAML](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithSAML.html)
### Identity Center y STS:
- [What is IAM Identity Center?](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [Manage AWS accounts with permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
### Seguridad:
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Use temporary credentials with AWS resources](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)

---
