# 🧪 Lab Personal: Configuración inicial de AWS IAM Identity Center desde cero.

[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?logo=amazon-web-services&logoColor=white)](#)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-623CE4?logo=terraform&logoColor=white)](#)
[![HCL](https://img.shields.io/badge/Language-HCL-blueviolet)](#)
[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

> 🚀 Laboratorio completo para configuración inicial de **AWS IAM Identity Center** desde cero. Incluye Terraform + pasos manuales, cheatsheets de referencia rápida y documentación técnica de conceptos fundamentales.

## 🎯 Objetivos del MPV
### Este laboratorio te permitirá:
- Configurar AWS IAM Identity Center desde cero en una nueva organización de AWS.
- Dominar tanto la automatización con Terraform como los pasos manuales necesarios.
- Entender los conceptos técnicos fundamentales como OAuth 2.0 Device Flow y STS (Security Token Service).
- Aplicar mejores prácticas de seguridad en la configuración de acceso centralizado

---

## ⚙ Tecnolgías usadas
### Servicios AWS
- AWS IAM Identity Center - Servicio principal de SSO y gestión de acceso centralizado
- AWS Organizations - Gestión de múltiples cuentas AWS
- AWS STS (Security Token Service) - Tokens de acceso temporal
- AWS IAM - Gestión de permisos y políticas
### Herramientas
- Terraform - Infrastructure as Code para automatización de recursos
- AWS CLI - Interfaz de línea de comandos para gestión y validación
### Protocolos y Estándares
- OAuth 2.0 Device Authorization Flow - Flujo de autenticación
### Versiones recomendadas:
- Terraform: >= 1.13.1
- AWS CLI: >= 2.28.21
- AWS Provider: ~> 6.0

---

## ⚙ Este MVP del workflow de [...] incluye solo lo esencial

---

## 🛠 Bloques de construcción (building blocks - the basic things that are put together to make something exist)

---

## 🚀 Demostración y Prueba del laboratorio (el MVP Funcional)
- Sección 01
    ```bash
    open -a Docker
    ```

    ```hcl
        resource "aws_iam_role_policy_attachment" "lambda_basic_execution_policy" {
            role       = aws_iam_role.lambda_role.name
            policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
        }
    ```

    ```json
    {"message":"Hola Samuel desde Lambda con HTTP API!"}
    ```

    <p align="center">
        <img src="imagenes/imagen.png" alt="imagen" width="80%">
    </p>

> [!NOTE]
> Este es un bloque de nota.

---

## 🔗 Referencias
- []()

---

### 📝 Licencia

Este repositorio está disponible bajo la licencia MIT.
Puedes usar, modificar y compartir libremente el contenido, incluso con fines comerciales.
Consulta el archivo [`LICENSE`](./LICENSE) para más detalles.

---
Laboratorio para configuración inicial de AWS IAM Identity Center (AWS SSO) - Guías, scripts y mejores prácticas
