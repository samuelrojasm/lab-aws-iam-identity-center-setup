# 🧪 Guía de Integración Completa de Autenticación AWS
### SAML Federation, OAuth 2.0 Device Flow y AWS STS - De Conceptos Básicos a Implementación Avanzada
### Integración Completa de Autenticación AWS (SAML, OAuth 2.0 Device Flow y STS)

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

## Contenido
- [Introducción y Conceptos Fundamentales](#intro)
- [AWS STS: El Corazón de la Federación](#sts)
- [SAML 2.0: Federación Web Tradicional](#tradicional)
- [OAuth 2.0 Device Authorization Flow](#flow)
- [Integración: SAML + OAuth + STS](#integracion)
- [Configuración Práctica para un laboratoio de pruebas](#lab)

## ⚙️ Introducción y Conceptos Fundamentales <a name="intro"></a> 
### ¿Qué Problemas Resolvemos?
- En el mundo moderno de aplicaciones distribuidas, enfrentamos varios desafíos:
    - Usuarios móviles que necesitan acceso desde dispositivos sin navegador
    - Aplicaciones de línea de comandos que requieren autenticación
    - Sistemas IoT que no pueden mostrar interfaces web
    - Federación empresarial que debe integrarse con sistemas legacy
    - Gestión centralizada de credenciales y permisos
### Los Tres Pilares de Nuestra Solución
- Diagrama
    ```mermaid
    graph TB
    A[Usuario/Dispositivo] --> B[SAML Federation]
    A --> C[OAuth 2.0 Device Flow]
    B --> D[AWS STS]
    C --> D
    D --> E[AWS Resources]
    
    B --> F[Web Browsers<br/>Enterprise SSO]
    C --> G[CLI Tools<br/>IoT Devices<br/>Mobile Apps]
    
    style D fill:#52796F,stroke:#354F52,color:#fff
    style B fill:#84A98C,stroke:#52796F,color:#000
    style C fill:#CAD2C5,stroke:#84A98C,color:#000
    ```
### Arquitectura de Autenticación y Autorización
- Flujo Conceptual Completo
    ```bash
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │                 │    │                 │    │                 │
    │   Identity      │    │   OAuth 2.0     │    │   SAML 2.0      │
    │   Provider      │    │   Authorization │    │   Service       │
    │   (IdP)         │    │   Server        │    │   Provider      │
    │                 │    │                 │    │                 │
    └─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
              │                      │                      │
              │ Authenticates        │ Issues Tokens        │ Issues Assertions
              │                      │                      │
              ▼                      ▼                      ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                     AWS STS (Security Token Service)            │
    │                                                                 │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
    │  │ AssumeRoleWith  │  │ AssumeRoleWith  │  │ AssumeRoleWith  │  │
    │  │ WebIdentity     │  │ SAML            │  │ OAuth2          │  │
    │  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
    └─────────────────────────────────────────────────────────────────┘
            │                      │                      │
            ▼                      ▼                      ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                        AWS Services                             │
    │   EC2 │ S3 │ RDS │ Lambda │ CloudFormation │ EKS │ ...          │
    └─────────────────────────────────────────────────────────────────┘
    ```

### Conceptos Clave
1. AWS STS (Security Token Service)
    - **Definición**: "Servicio web que permite solicitar credenciales temporales"
    - **Propósito**: "Proporcionar acceso temporal y limitado a recursos AWS"
    - **Beneficios**:
        - "Credenciales que expiran automáticamente",
        - "No requiere crear usuarios IAM permanentes",
        - "Integración con proveedores de identidad externos",
        - "Control granular de permisos"
    - **Tipos de tokens**
        - **AccessKeyId**: "Identificador de clave de acceso temporal",
        - **SecretAccessKey**: "Clave secreta temporal",
        - **SessionToken**: "Token de sesión requerido para uso temporal"
2. SAML 2.0 (Security Assertion Markup Language)
    - Estructura básica de una SAML Assertion:
        ```xml
        <!-- Estructura básica de una SAML Assertion -->
        <saml:Assertion>
            <saml:Subject>
                <saml:NameID Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
                    usuario@empresa.com
                </saml:NameID>
            </saml:Subject>
            
            <saml:AttributeStatement>
                <saml:Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
                    <saml:AttributeValue>
                        arn:aws:iam::123456789012:role/SAMLRole,
                        arn:aws:iam::123456789012:saml-provider/ExampleProvider
                    </saml:AttributeValue>
                </saml:Attribute>
            </saml:AttributeStatement>
        </saml:Assertion>
        ```
3. OAuth 2.0 Device Authorization Flow
    - Device Authorization Flow (RFC 8628):
        - Device → Authorization Server: POST /device_authorization
            - Response: { device_code, user_code, verification_uri }
        - Device displays: "Visita https://device.auth.com e ingresa: ABCD-1234"
        - Usuario → Browser → Authorization Server: Ingresa user_code
        - Device → Authorization Server: POST /token (polling)
            - Request: { grant_type: "device_code", device_code }
            - Response: { access_token, refresh_token }

---

## ⚙️ AWS STS: El Corazón de la Federación <a name="sts"></a> 
### Operaciones Principales de STS
1. AssumeRole - Para Roles Directos
    - Comando:
        ```bash
        aws sts assume-role \
            --role-arn "arn:aws:iam::123456789012:role/MyRole" \
            --role-session-name "my-session" \
            --duration-seconds 3600
        ```
    - Respuesta
        ```json
        {
            "Credentials": {
                "AccessKeyId": "ASIA...",
                "SecretAccessKey": "...",
                "SessionToken": "...",
                "Expiration": "2025-08-30T12:00:00Z"
            },
            "AssumedRoleUser": {
                "AssumedRoleId": "AROA...:my-session",
                "Arn": "arn:aws:sts::123456789012:assumed-role/MyRole/my-session"
            }
        }

        ```
2. AssumeRoleWithSAML - Para Federación SAML
    - Comando
        ```bash
        aws sts assume-role-with-saml \
            --role-arn "arn:aws:iam::123456789012:role/SAMLRole" \
            --principal-arn "arn:aws:iam::123456789012:saml-provider/ExampleProvider" \
            --saml-assertion "$(cat saml-assertion.xml | base64 -w 0)"
        ```
3. AssumeRoleWithWebIdentity - Para OAuth/OpenID Connect
    - Comando
        ```bash
        aws sts assume-role-with-web-identity \
            --role-arn "arn:aws:iam::123456789012:role/WebIdentityRole" \
            --role-session-name "web-identity-session" \
            --web-identity-token "eyJhbGciOiJSUzI1NiIs..."
        ```
### Políticas de Trust para STS
- Trust Policy para SAML
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::123456789012:saml-provider/ExampleProvider"
                },
                "Action": "sts:AssumeRoleWithSAML",
                "Condition": {
                    "StringEquals": {
                        "SAML:aud": "https://signin.aws.amazon.com/saml"
                    },
                    "StringLike": {
                        "SAML:email": "*@empresa.com"
                    }
                }
            }
        ]
    }
    ```
- Trust Policy para OAuth/OpenID Connect
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
                        "auth.empresa.com:aud": "aws-client-id",
                        "auth.empresa.com:sub": "user-subject-id"
                    },
                    "StringLike": {
                        "auth.empresa.com:email": "*@empresa.com"
                    }
                }
            }
        ]
    }
    ````

## ⚙️ SAML 2.0: Federación Web Tradicional <a name="tradicional"></a> 
### Flujo Detallado SAML con STS
- Diagrama:
    ```bash
         ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
         │   Usuario   │     │   Browser   │     │    IdP      │     │   AWS STS   │
         │             │     │             │     │  (SAML)     │     │             │
         └──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
                │                   │                   │                   │
                │ 1. Accede AWS     │                   │                   │
                │ Console/App       │                   │                   │
                ├──────────────────►│                   │                   │
                │                   │ 2. Redirect       │                   │
                │                   │ (SAML Request)    │                   │
                │                   ├──────────────────►│                   │
                │                   │                   │ 3. Authenticate   │
                │ 4. Login          │                   │ User              │
                │ Credentials       │◄──────────────────┤                   │
                ├──────────────────►│                   │                   │
                │                   │ 5. SAML Response  │                   │
                │                   │ (Assertion)       │                   │
                │                   │◄──────────────────┤                   │
                │                   │ 6. Post SAML      │                   │
                │                   │ to AWS STS        │                   │
                │                   ├─────────────────────────────────────► │
                │                   │                   │ 7. Validate &     │
                │                   │                   │ Issue Temp Creds  │
                │ 8. Access AWS     │◄───────────────────────────────────── ┤
                │ with Temp Creds   │                   │                   │
                │◄──────────────────┤                   │                   │
    ```
### Configuración de SAML Identity Provider en AWS
1. Crear SAML Identity Provider
    - Comando
        ```bash
        # Crear el proveedor SAML
        aws iam create-saml-provider \
            --saml-metadata-document file://metadata.xml \
            --name "ExampleProvider"

        # Respuesta
        {
            "SAMLProviderArn": "arn:aws:iam::123456789012:saml-provider/ExampleProvider"
        }
        ```
2. Metadata XML Ejemplo
    - Ejemplo
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <md:EntityDescriptor 
            xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
            entityID="https://idp.empresa.com">
            
            <md:IDPSSODescriptor 
                protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
                
                <md:KeyDescriptor use="signing">
                    <ds:KeyInfo xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
                        <ds:X509Data>
                            <ds:X509Certificate>
                                MIICXjCCAcegAwIBAgIJAL...
                            </ds:X509Certificate>
                        </ds:X509Data>
                    </ds:KeyInfo>
                </md:KeyDescriptor>
                
                <md:SingleSignOnService 
                    Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
                    Location="https://idp.empresa.com/sso/saml2"/>
                    
            </md:IDPSSODescriptor>
        </md:EntityDescriptor>
        ```
3. Configurar Role para SAML
    - Configuración
        ```json
        {
            "Role": {
                "RoleName": "SAMLRole",
                "AssumeRolePolicyDocument": {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Principal": {
                                "Federated": "arn:aws:iam::123456789012:saml-provider/ExampleProvider"
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
            }
        }
        ```

## ⚙️ OAuth 2.0 Device Authorization Flow <a name="flow"></a> 
### ¿Por Qué Device Authorization Flow?
    - Escenarios típicos:
        - CLI tools (AWS CLI, kubectl, terraform)
        - Smart TVs y dispositivos IoT
        - Aplicaciones móviles nativas
        - Sistemas sin navegador web
### Implementación Detallada del Flow
- Paso 1: Device Authorization Request
    - Request
        ```http
        POST /device_authorization HTTP/1.1
        Host: auth.empresa.com
        Content-Type: application/x-www-form-urlencoded

        client_id=aws-cli-client&
        scope=aws:sts:assume-role
        ```
    - Respuesta:
        ```json
        {
            "device_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
            "user_code": "WDJB-MJHT",
            "verification_uri": "https://auth.empresa.com/device",
            "verification_uri_complete": "https://auth.empresa.com/device?user_code=WDJB-MJHT",
            "expires_in": 1800,
            "interval": 5
        }
        ```
- Paso 2: User Authorization
    ```bash
    # El dispositivo muestra:
    echo "Para autorizar este dispositivo, visita:"
    echo "https://auth.empresa.com/device"
    echo "E ingresa el código: WDJB-MJHT"
    ```
- Paso 3: Token Polling
    ```bash
    # El dispositivo hace polling cada 5 segundos
    while true; do
        response=$(curl -s -X POST https://auth.empresa.com/token \
            -H "Content-Type: application/x-www-form-urlencoded" \
            -d "grant_type=urn:ietf:params:oauth:grant-type:device_code&device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS&client_id=aws-cli-client")
        
        if [[ $response == *"access_token"* ]]; then
            echo "¡Autorización exitosa!"
            break
        elif [[ $response == *"authorization_pending"* ]]; then
            echo "Esperando autorización del usuario..."
            sleep 5
        else
            echo "Error en la autorización"
            break
        fi
    done
    ```
- Paso 4: Token Response
    ```json
     {
        "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
        "token_type": "Bearer",
        "expires_in": 3600,
        "refresh_token": "def50200...",
        "scope": "aws:sts:assume-role",
        "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
    }
    ```
### Integración con AWS STS
- Configurar OpenID Connect Provider
    ```bash
    # Crear OIDC Provider
    aws iam create-open-id-connect-provider \
        --url "https://auth.empresa.com" \
        --client-id-list "aws-cli-client" \
        --thumbprint-list "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"
    ```
- Usar Token con STS
    ```python
    import boto3
    import json
    from botocore.exceptions import ClientError

    def assume_role_with_oauth_token(id_token, role_arn):
        """
        Usa un ID token de OAuth para asumir un rol de AWS
        """
        sts_client = boto3.client('sts')
        
        try:
            response = sts_client.assume_role_with_web_identity(
                RoleArn=role_arn,
                RoleSessionName='oauth-device-session',
                WebIdentityToken=id_token,
                DurationSeconds=3600
            )
            
            credentials = response['Credentials']
            return {
                'AccessKeyId': credentials['AccessKeyId'],
                'SecretAccessKey': credentials['SecretAccessKey'],
                'SessionToken': credentials['SessionToken'],
                'Expiration': credentials['Expiration']
            }
            
        except ClientError as e:
            print(f"Error asumiendo rol: {e}")
            return None

    # Uso
    role_arn = "arn:aws:iam::123456789012:role/DeviceAuthRole"
    id_token = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."

    aws_credentials = assume_role_with_oauth_token(id_token, role_arn)
    if aws_credentials:
        print("Credenciales AWS obtenidas exitosamente")
    ```

## ⚙️ Integración: SAML + OAuth + STS <a name="integracion"></a> 
### Arquitectura Híbrida
- Diagrama
    ```bash
                    ┌─────────────────────────────────────┐
                    │        Identity Provider            │
                    │                                     │
                    │  ┌─────────────┐ ┌─────────────┐    │
                    │  │    SAML     │ │   OAuth     │    │
                    │  │  Endpoint   │ │  Endpoint   │    │
                    │  └─────────────┘ └─────────────┘    │
                    └─────────────┬───────────┬───────────┘
                                  │           │
                    ┌─────────────▼───────────▼───────────┐
                    │           AWS STS                   │
                    │                                     │
                    │  ┌─────────────────────────────┐    │
                    │  │     Role Resolution         │    │
                    │  │                             │    │
                    │  │  SAML Attrs → AWS Role      │    │
                    │  │  OAuth Claims → AWS Role    │    │
                    │  └─────────────────────────────┘    │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │          AWS Services               │
                    │                                     │
                    │  Users access through:              │
                    │  • Web Console (SAML)               │
                    │  • CLI Tools (OAuth Device)         │
                    │  • Mobile Apps (OAuth)              │
                    │  • APIs (Temporary Credentials)     │
                    └─────────────────────────────────────┘
    ```
### Caso de Uso: Organización Completa
- Escenario
    - Empleados: Acceso web via SAML SSO
    - Desarrolladores: CLI tools via OAuth Device Flow
    - Aplicaciones: Service-to-service via OAuth Client Credentials
    - Contractors: Acceso limitado via SAML con atributos específicos
- Configuración Multi-Protocol
    - 1. Identity Provider Configuration
        ```yaml
        # Configuración conceptual del IdP
        saml_applications:
        - name: "AWS-Console"
            acs_url: "https://signin.aws.amazon.com/saml"
            entity_id: "urn:amazon:webservices"
            attributes:
            - name: "Role"
                value: "${user.aws_roles}"
            - name: "RoleSessionName"
                value: "${user.email}"

        oauth_clients:
        - client_id: "aws-cli-client"
            client_type: "public"
            grant_types: ["device_code"]
            scopes: ["openid", "aws:sts"]
        
        - client_id: "mobile-app-client"
            client_type: "public"
            grant_types: ["authorization_code"]
            redirect_uris: ["myapp://auth/callback"]
        ```
    - 2. AWS Configuration
        ```bash
        # Crear proveedores de identidad
        aws iam create-saml-provider \
            --saml-metadata-document file://saml-metadata.xml \
            --name "EnterpriseProvider"

        aws iam create-open-id-connect-provider \
            --url "https://auth.empresa.com" \
            --client-id-list "aws-cli-client,mobile-app-client" \
            --thumbprint-list "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"
        ```
    - 3. Roles con Trust Policies Múltiples
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Federated": "arn:aws:iam::123456789012:saml-provider/EnterpriseProvider"
                    },
                    "Action": "sts:AssumeRoleWithSAML",
                    "Condition": {
                        "StringEquals": {
                            "SAML:aud": "https://signin.aws.amazon.com/saml",
                            "SAML:department": "engineering"
                        }
                    }
                },
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Federated": "arn:aws:iam::123456789012:oidc-provider/auth.empresa.com"
                    },
                    "Action": "sts:AssumeRoleWithWebIdentity",
                    "Condition": {
                        "StringEquals": {
                            "auth.empresa.com:aud": ["aws-cli-client", "mobile-app-client"],
                            "auth.empresa.com:department": "engineering"
                        }
                    }
                }
            ]
        }
        ```

## ⚙️ Configuración Práctica para un laboratoio de pruebas <a name="lab"></a> 
### Prerrequisitos
- Requerimentos
    ```bash
    # Herramientas necesarias
    - AWS CLI v2
    - Python 3.8+
    - OpenSSL
    - Docker (opcional, para IdP local)
    - jq (para procesamiento JSON)

    # Permisos AWS requeridos
    - IAM Full Access (para crear roles y proveedores)
    - STS Full Access (para pruebas)
    ```
### Lab Setup: Identity Provider Local
1. Configurar Keycloak como IdP
    ```bash
    # Ejecutar Keycloak con Docker
    docker run -p 8080:8080 \
        -e KEYCLOAK_ADMIN=admin \
        -e KEYCLOAK_ADMIN_PASSWORD=admin \
        quay.io/keycloak/keycloak:latest \
        start-dev

    # Configuración básica
    echo "Keycloak disponible en: http://localhost:8080"
    echo "Admin user: admin"
    echo "Admin pass: xxxxx"
    ```

2. Script de Configuración Automática
     ```python
        #!/usr/bin/env python3
        """
        Script para configurar automáticamente SAML y OAuth en Keycloak
        """
        import requests
        import json
        import sys

        class KeycloakConfigurator:
            def __init__(self, base_url="http://localhost:8080"):
                self.base_url = base_url
                self.admin_token = None
                
            def get_admin_token(self):
                """Obtener token de administrador"""
                token_url = f"{self.base_url}/realms/master/protocol/openid-connect/token"
                data = {
                    'username': 'admin',
                    'password': 'admin',
                    'grant_type': 'password',
                    'client_id': 'admin-cli'
                }
                
                response = requests.post(token_url, data=data)
                if response.status_code == 200:
                    self.admin_token = response.json()['access_token']
                    return True
                return False
            
            def create_realm(self, realm_name="aws-lab"):
                """Crear realm para el lab"""
                if not self.admin_token:
                    return False
                    
                headers = {'Authorization': f'Bearer {self.admin_token}',
                        'Content-Type': 'application/json'}
                
                realm_config = {
                    "realm": realm_name,
                    "enabled": True,
                    "displayName": "AWS Lab Realm",
                    "registrationAllowed": True,
                    "rememberMe": True
                }
                
                response = requests.post(
                    f"{self.base_url}/admin/realms",
                    headers=headers,
                    json=realm_config
                )
                
                return response.status_code == 201
            
            def create_saml_client(self, realm_name="aws-lab"):
                """Crear cliente SAML para AWS"""
                headers = {'Authorization': f'Bearer {self.admin_token}',
                        'Content-Type': 'application/json'}
                
                saml_client = {
                    "clientId": "urn:amazon:webservices",
                    "protocol": "saml",
                    "enabled": True,
                    "attributes": {
                        "saml.assertion.signature": "true",
                        "saml.force.post.binding": "true",
                        "saml.multivalued.roles": "false",
                        "saml.encrypt": "false",
                        "saml_force_name_id_format": "true",
                        "saml_name_id_format": "persistent",
                        "saml.client.signature": "false"
                    },
                    "redirectUris": ["https://signin.aws.amazon.com/saml"]
                }
                
                response = requests.post(
                    f"{self.base_url}/admin/realms/{realm_name}/clients",
                    headers=headers,
                    json=saml_client
                )
                
                return response.status_code == 201
            
            def create_oauth_client(self, realm_name="aws-lab"):
                """Crear cliente OAuth para Device Flow"""
                headers = {'Authorization': f'Bearer {self.admin_token}',
                        'Content-Type': 'application/json'}
                
                oauth_client = {
                    "clientId": "aws-cli-client",
                    "protocol": "openid-connect",
                    "publicClient": True,
                    "enabled": True,
                    "attributes": {
                        "oauth2.device.authorization.grant.enabled": "true",
                        "oidc.ciba.grant.enabled": "false"
                    },
                    "standardFlowEnabled": False,
                    "directAccessGrantsEnabled": False
                }
                
                response = requests.post(
                    f"{self.base_url}/admin/realms/{realm_name}/clients",
                    headers=headers,
                    json=oauth_client
                )
                
                return response.status_code == 201

        def main():
            configurator = KeycloakConfigurator()
            
            print("1. Obteniendo token de administrador...")
            if not configurator.get_admin_token():
                print("Error obteniendo token")
                sys.exit(1)
            
            print("2. Creando realm...")
            if configurator.create_realm():
                print("✓ Realm creado exitosamente")
            else:
                print("✗ Error creando realm")
            
            print("3. Configurando cliente SAML...")
            if configurator.create_saml_client():
                print("✓ Cliente SAML creado exitosamente")
            else:
                print("✗ Error creando cliente SAML")
            
            print("4. Configurando cliente OAuth...")
            if configurator.create_oauth_client():
                print("✓ Cliente OAuth creado exitosamente")
            else:
                print("✗ Error creando cliente OAuth")
            
            print("\nConfiguración completada!")
            print("Siguiente paso: Configurar AWS IAM Identity Center")

        if __name__ == "__main__":
            main()
    ```
---

