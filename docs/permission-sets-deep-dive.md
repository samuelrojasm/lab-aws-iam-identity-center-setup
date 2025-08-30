# 🧪 Guía de Permission Sets y AWS IAM Roles

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

## Contenido
- [¿Qué son los Permission Sets?](#que-es)
- [Arquitectura: Permission Sets → IAM Roles](#arquitectura) 
- [Naming Convention de IAM Roles](#name)
- [Componentes de un Permission Set](#componentes)
- [Tipos de políticas en Permission Sets](#tipos)
- [ Permission Sets comunes (ejemplos prácticos)](#ejemplos)
- [Account Assignments](#account)
- [Trust Relationships automáticas](#trust)
- [Propagación y sincronización](#propagacion)
- [Limitaciones importantes](#limitaciones)
- [Terraform para Permission Sets](#terraform)
- [Troubleshooting común](#troubleshooting)
- [ Mejores prácticas](#mejores)

---

## ⚙️ ¿Qué son los Permission Sets? <a name="que-es"></a> 
- Los Permission Sets son **plantillas** de permisos en AWS IAM Identity Center que definen QUÉ puede hacer un usuario una vez autenticado. Son el equivalente de **"roles"** pero específicamente para Identity Center.
- Concepto clave:
    - Identity Center users ≠ IAM users (son sistemas separados)
    - Permission Sets = plantillas que generan IAM Roles automáticamente
    - Account Assignments = asignar Permission Sets a usuarios en cuentas específicas
- Analogía:
    - Permission Set = "Plantilla de trabajo" (ej: "Developer", "ReadOnly", "Admin")
    - IAM Role = "Gafete de acceso" generado automáticamente para cada cuenta
    - Account Assignment = "Juan puede usar la plantilla 'Developer' en la cuenta Production"

> [!IMPORTANT]]
> Los Permission Sets son templates que generan IAM Roles automáticamente. No edites los IAM Roles directamente - siempre hazlo a través de Identity Center para mantener sincronización.

## ⚙️ Arquitectura: Permission Sets → IAM Roles <a name="arquitectura"></a> 
- Diagrama de relación:
    ```mermaid
    flowchart TD
    PS1[Permission Set: DeveloperAccess] --> Role1[IAM Role: AWSReservedSSO_DeveloperAccess_abc123]
    PS2[Permission Set: ReadOnlyAccess] --> Role2[IAM Role: AWSReservedSSO_ReadOnlyAccess_def456]
    
    Role1 --> Acc1[Account: 111111111111]
    Role1 --> Acc2[Account: 222222222222]
    Role2 --> Acc1
    Role2 --> Acc2
    
    User1[user@company.com] -.-> Assignment1[Assignment: DeveloperAccess to Acc1]
    User2[admin@company.com] -.-> Assignment2[Assignment: ReadOnlyAccess to Acc2]
    ```
    
    ```mermaid
    flowchart TD
    PS1[DeveloperAccess] --> Role1[Role_Developer_abc123]
    PS2[ReadOnlyAccess] --> Role2[Role_ReadOnly_def456]

    Role1 --> Acc1[Account_111111111111]
    Role1 --> Acc2[Account_222222222222]
    Role2 --> Acc1
    Role2 --> Acc2
    
    User1[user_company_com] -.-> Assignment1[Dev_to_Acc1]
    User2[admin_company_com] -.-> Assignment2[ReadOnly_to_Acc2]
     ```


- Lo que pasa automáticamente:
    - Creas Permission Set → Identity Center lo almacena como template
    - Haces Account Assignment → Identity Center crea IAM Role en esa cuenta
    - Usuario hace SSO login → STS asume el IAM Role generado
    - Usuario accede recursos → Con permisos del IAM Role

## ⚙️ Naming Convention de IAM Roles <a name="name"></a>
- Patrón automático:
    ```bash
    AWSReservedSSO_{PermissionSetName}_{RandomSuffix}
    ```
- Ejemplos reales:
    ```bash
    # Permission Set: "DeveloperAccess"
    → IAM Role: "AWSReservedSSO_DeveloperAccess_a1b2c3d4e5f6"

    # Permission Set: "ReadOnlyAccess"  
    → IAM Role: "AWSReservedSSO_ReadOnlyAccess_f6e5d4c3b2a1"

    # Permission Set: "DatabaseAdmin"
    → IAM Role: "AWSReservedSSO_DatabaseAdmin_123456789abc"
    ```
- Características del naming:
    - **Prefix fijo**: AWSReservedSSO_ (no se puede cambiar)
    - **Permission Set name**: Exactamente como lo nombras
    - **Random suffix**: 12 caracteres aleatorios únicos
    - **Inmutable**: No se puede renombrar una vez creado

## ⚙️ Componentes de un Permission Set <a name="componentes"></a> 
### Estructura básica:
- Estructura
    ```json
    {
         "Name": "DeveloperAccess",
        "Description": "Developer access with S3, EC2, and CloudWatch permissions",
        "SessionDuration": "PT2H",
        "RelayState": "https://console.aws.amazon.com/ec2/",
        "Tags": [
         {
            "Key": "Environment",
            "Value": "Development"
         }
        ]
    }
    ```
1. Session Duration
    - Configuración:
        ```json
        "SessionDuration": "PT8H"  // ISO 8601 format
        ```
    - Opciones típicas:
        - PT15M = 15 minutos (mínimo)
        - PT1H = 1 hora (común para desarrollo)
        - PT2H = 2 horas (balance típico)
        - PT8H = 8 horas (workday completo)
        - PT12H = 12 horas (máximo)
    - Consideraciones:
        - ✅ Más corto = Más seguro, usuarios deben re-autenticar
        - ❌ Más largo = Más conveniente, pero mayor riesgo
        - Recomendación: 1-2 horas para desarrollo interactivo

2. Relay State
    - **Propósito**: URL donde redirigir al usuario después del login
        ```json
        "RelayState": "https://console.aws.amazon.com/s3/"
        ```
    - Ejemplos útiles:
        ```bash
        # Consola EC2
        "RelayState": "https://console.aws.amazon.com/ec2/"

        # Consola S3
        "RelayState": "https://console.aws.amazon.com/s3/"

        # CloudFormation
        "RelayState": "https://console.aws.amazon.com/cloudformation/"

        # Billing (para admins)
        "RelayState": "https://console.aws.amazon.com/billing/"
        ```
3. Tags
    - **Propósito**: Metadata para organización y facturación
        ```json
        "Tags": [
            {
                "Key": "Team",
                "Value": "Platform"
            },
            {
                "Key": "Environment", 
                "Value": "Production"
            },
            {
                "Key": "CostCenter",
                "Value": "Engineering"
            }
        ]
        ```

## ⚙️ Tipos de políticas en Permission Sets  <a name="tipos"></a> 
1. AWS Managed Policies
    - Más común y recomendado:
        ```json
        {
          "ManagedPolicies": [
            "arn:aws:iam::aws:policy/PowerUserAccess",
            "arn:aws:iam::aws:policy/IAMReadOnlyAccess"
          ]
        }
        ```
    - Políticas populares:
        ```bash
        # Para developers
        arn:aws:iam::aws:policy/PowerUserAccess
        arn:aws:iam::aws:policy/AmazonS3FullAccess  
        arn:aws:iam::aws:policy/AmazonEC2FullAccess

        # Para read-only
        arn:aws:iam::aws:policy/ReadOnlyAccess
        arn:aws:iam::aws:policy/ViewOnlyAccess

        # Para admins
        arn:aws:iam::aws:policy/AdministratorAccess
        ```
2. Customer Managed Policies
    - Para políticas reutilizables customizadas:
        ```json
        {
          "CustomerManagedPolicies": [
            {
                "Name": "CompanyDeveloperPolicy",
                "Path": "/"
            }
          ]
        }
        ```
    - Ventajas:
        - Reutilizable entre múltiples Permission Sets
        - Control de versiones
        - Centralizado en IAM
3. Inline Policies
    - Para permisos específicos del Permission Set:
        ```json
        {
            "InlinePolicy": {
              "Version": "2012-10-17",
              "Statement": [
                {
                  "Effect": "Allow",
                  "Action": [
                    "s3:GetObject",
                    "s3:PutObject"
                  ],
                  "Resource": "arn:aws:s3:::company-dev-bucket/*"
                },
                {
                  "Effect": "Deny",
                  "Action": "s3:DeleteBucket",
                  "Resource": "*"
                }
              ]
         }
    }
    ```
4. Permissions Boundary
    - Para límites máximos de permisos:
        ```json
        {
            "PermissionsBoundary": {
                "CustomerManagedPolicyReference": {
                    "Name": "DeveloperBoundary",
                    "Path": "/"
                }
            }
        }
        ```
    - Casos de uso:
        - Prevenir escalation de privilegios
        - Compliance requirements
        - Límites por departamento

## ⚙️ Permission Sets comunes (ejemplos prácticos) <a name="ejemplos"></a>
1. DeveloperAccess
    ```json
    {
        "Name": "DeveloperAccess",
        "Description": "Standard developer permissions",
        "SessionDuration": "PT2H",
        "RelayState": "https://console.aws.amazon.com/",
        "ManagedPolicies": [
            "arn:aws:iam::aws:policy/PowerUserAccess"
        ],
        "InlinePolicy": {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Deny",
                "Action": [
                    "iam:CreateUser",
                    "iam:DeleteUser",
                    "iam:CreateRole",
                    "iam:DeleteRole"
                ],
                "Resource": "*"
              }
            ]
        }
    }
    ```
2. ReadOnlyAccess
    ```json
    {
        "Name": "ReadOnlyAccess", 
        "Description": "Read-only access to all AWS services",
        "SessionDuration": "PT8H",
        "ManagedPolicies": [
            "arn:aws:iam::aws:policy/ReadOnlyAccess"
        ]
    }
    ```
3. S3AdminAccess
    ```json
    {
        "Name": "S3AdminAccess",
        "Description": "Full access to S3 and related services",
        "SessionDuration": "PT4H", 
        "ManagedPolicies": [
            "arn:aws:iam::aws:policy/AmazonS3FullAccess",
            "arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess"
        ],
        "InlinePolicy": {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Allow",
                "Action": [
                    "kms:Decrypt",
                    "kms:GenerateDataKey"
                ],
                "Resource": "arn:aws:kms:*:*:key/s3-encryption-key-*"
              }
            ]
        }
    }
    ```
4. EmergencyAccess
    ```json
    {
        "Name": "EmergencyAccess",
        "Description": "Break-glass access for incidents",
        "SessionDuration": "PT1H",
        "ManagedPolicies": [
            "arn:aws:iam::aws:policy/AdministratorAccess"
        ],
        "Tags": [
            {
                "Key": "AccessType",
                "Value": "Emergency"
            },
            {
                "Key": "RequiresApproval",
                "Value": "true"
            }
        ]
    }
    ```

## ⚙️ Account Assignments <a name="account"></a>
- ¿Qué son?
    - Los Account Assignments conectan usuarios/grupos con Permission Sets en cuentas específicas.
- Estructura:
    ```json
    {
        "PrincipalType": "USER",           // USER or GROUP
        "PrincipalId": "user@company.com", // User email or Group ID
        "PermissionSetArn": "arn:aws:sso:::permissionSet/ssoins-123/ps-abc456",
        "TargetType": "AWS_ACCOUNT",
        "TargetId": "123456789012"         // Account ID
    }
    ```
- Ejemplos de assignments:
    ```bash
    # Juan puede usar DeveloperAccess en cuenta Development
    User: juan@company.com
    Permission Set: DeveloperAccess  
    Account: 111111111111 (Development)

    # Grupo DevTeam puede usar DeveloperAccess en múltiples cuentas
    Group: DevTeam
    Permission Set: DeveloperAccess
    Accounts: 111111111111, 222222222222, 333333333333

    # María puede usar AdminAccess solo en cuenta Sandbox
    User: maria@company.com
    Permission Set: AdminAccess
    Account: 444444444444 (Sandbox)
    ```

## ⚙️ Trust Relationships automáticas <a name="trust"></a>
- IAM Role Trust Policy generada:
    - Cuando Identity Center crea un IAM Role, automáticamente genera esta trust policy:
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::123456789012:saml-provider/AWSSSO_abc123_DO_NOT_DELETE"
                },
                "Action": "sts:AssumeRoleWithSAML",
                "Condition": {
                    "StringEquals": {
                        "SAML:aud": "https://signin.aws.amazon.com/saml"
                    }
                }
              }
            ]
        }
- Componentes clave:
    - **Federated Principal**: El SAML provider de Identity Center
    - **AssumeRoleWithSAML**: Permite que STS asuma el role via SAML
    - **SAML Condition**: Valida que el request viene de AWS Console

## ⚙️ Propagación y sincronización <a name="propagacion"></a>
- Tiempo de propagación:
    |Acción|Tiempo típico|Impacto|
    |------|-------------|-------|
    |Crear Permission Set|Inmediato|Solo template, no IAM roles|
    |Account Assignment|1-5 minutos|Crea/actualiza IAM role|
    |Modificar Permission Set|2-10 minutos|Actualiza todos los IAM roles|
    |Eliminar Assignment|1-3 minutos|Elimina IAM role|
- Estados de provisioning:
    ```bash
    # Comando para verificar estado
    aws sso-admin describe-account-assignment-creation-status \
        --instance-arn arn:aws:sso:::instance/ssoins-123 \
        --account-assignment-creation-request-id req-abc123

    # Estados posibles
    IN_PROGRESS  # Creando/actualizando IAM role
    SUCCEEDED    # Completado exitosamente  
    FAILED       # Error en la operación
    ```

## ⚙️  Limitaciones importantes <a name="limitaciones"></a>
- Límites de servicio:
    |Recurso|Límite|Notas|
    |-------|------|-----|
    |Permission Sets por instancia|500|Soft limit, se puede aumentar|
    |Account Assignments|50,000|Por instancia|
    |Políticas inline por PS|1|Máximo 32KB de tamaño|
    Managed policies por PS|20|AWS + Customer managed|
    |Tags por Permission Set|50|Key-value pairs|
- Restricciones técnicas:
    - ❌ No se puede:
        - Cambiar el nombre de un Permission Set (debe recrear)
        - Mover Permission Set entre instancias
        - Usar IAM roles existentes (solo genera automáticamente)
        - Modificar trust policy del role generado
    - ✅ Sí se puede:
        - Clonar Permission Set con nuevo nombre
        - Reutilizar el mismo PS en múltiples cuentas
        - Mezclar diferentes tipos de políticas
        - Usar conditions complejas en inline policies

## ⚙️ Terraform para Permission Sets  <a name="terraform"></a>
- Resource principal:
    ```hcl
    resource "aws_ssoadmin_permission_set" "developer_access" {
        name             = "DeveloperAccess"
        description      = "Developer permissions for non-prod environments"
        instance_arn     = local.sso_instance_arn
        session_duration = "PT2H"
        relay_state      = "https://console.aws.amazon.com/"

        tags = {
            Team        = "Platform"
            Environment = "Development"
        }
    }
    ```
- Managed policies:
    ```hcl
    resource "aws_ssoadmin_managed_policy_attachment" "developer_poweruser" {
        instance_arn       = local.sso_instance_arn
        permission_set_arn = aws_ssoadmin_permission_set.developer_access.arn
        managed_policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
    }
    ```
- Inline policy:
    ```hcl
    data "aws_iam_policy_document" "developer_restrictions" {
      statement {
        effect = "Deny"
        actions = [
            "iam:CreateUser",
            "iam:DeleteUser",
            "iam:AttachUserPolicy"
        ]
        resources = ["*"]
      }
    }

    resource "aws_ssoadmin_permission_set_inline_policy" "developer_restrictions" {
        instance_arn       = local.sso_instance_arn
        permission_set_arn = aws_ssoadmin_permission_set.developer_access.arn
        inline_policy      = data.aws_iam_policy_document.developer_restrictions.json
    }
    ```

- Account assignment:
    ```hcl
    resource "aws_ssoadmin_account_assignment" "developer_dev_account" {
        instance_arn       = local.sso_instance_arn
        permission_set_arn = aws_ssoadmin_permission_set.developer_access.arn
  
        principal_id   = data.aws_identitystore_user.john_doe.user_id
        principal_type = "USER"
  
        target_id   = "123456789012"  # Development account
        target_type = "AWS_ACCOUNT"
    }
    ```

## ⚙️ Troubleshooting común <a name="troubleshooting"></a>
- Error: "User cannot assume role"
    - Diagnóstico:
        ```bash
        # Verificar que existe el assignment
        aws sso-admin list-account-assignments \
            --instance-arn arn:aws:sso:::instance/ssoins-123 \
            --account-id 123456789012

        # Verificar el IAM role en la cuenta target
        aws iam get-role --role-name AWSReservedSSO_DeveloperAccess_abc123
        ```
- Error: "Permission Set provisioning failed"
    - Causas comunes:
        - Política inline mal formateada
        - Managed policy ARN incorrecto
        - Límites de servicio excedidos
        - Permisos insuficientes en cuenta target
- Error: "Access denied" después de login
    - Verificaciones:
        ```bash
        # ¿Qué role estoy asumiendo?
        aws sts get-caller-identity

        # ¿Qué permisos tiene este role?
        aws iam list-attached-role-policies \
            --role-name AWSReservedSSO_DeveloperAccess_abc123

        # ¿Hay políticas inline?
        aws iam get-role-policy \
            --role-name AWSReservedSSO_DeveloperAccess_abc123 \
            --policy-name InlinePolicy
        ```

## ⚙️ Mejores prácticas <a name="mejores"></a>
- Diseño de Permission Sets:
    - ✅ Recmendado:
        - Usar naming convention consistente (ej: {Role}Access, {Team}Permissions)
        - Preferir AWS managed policies cuando sea posible
        - Documentar el propósito en la descripción
        - Usar tags para organización
        - Configurar session duration apropiada
    - ❌ Evitar:
        - Permission Sets muy específicos (crear muchos similares)
        - Inline policies complejas (usar customer managed)
        - Session duration muy larga sin justificación
        - Nombres genéricos como "UserAccess" o "Policy1"
- Organización por equipos:
    ```bash
    # Por función
    DeveloperAccess, AdminAccess, ReadOnlyAccess

    # Por equipo
    PlatformTeamAccess, DataTeamAccess, SecurityTeamAccess

    # Por aplicación  
    WebAppDeveloper, DatabaseAdmin, MonitoringAccess

    # Por entorno
    ProductionAdmin, StagingDeveloper, SandboxFullAccess
    ```

---

## 🔗 Referencias
### Documentación AWS:
- [Manage AWS accounts with permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
- [Account Assignments](https://docs.aws.amazon.com/singlesignon/latest/userguide/useraccess.html)
- [Set session duration for AWS accounts](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtosessionduration.html)
### APIs importantes:
- [IAM Identity Center API Reference](https://docs.aws.amazon.com/singlesignon/latest/APIReference/welcome.html)
- [CreatePermissionSet](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreatePermissionSet.html)
- [CreateAccountAssignment](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreateAccountAssignment.html)
### Terraform Provider:
- [aws_ssoadmin_permission_set](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssoadmin_permission_set)
- [aws_ssoadmin_account_assignment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssoadmin_account_assignment)


---