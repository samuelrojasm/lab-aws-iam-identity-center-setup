# 🧪 Guía de OAuth 2.0 Device Authorization Flow

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

## Contenido
- [¿Qué es OAuth 2.0 Device Authorization Flow](#ques-es)
- [El problema de usar Authorization Code Flow en CLI](#problema)
- [¿Cómo Device Flow resueve el problema?](#solucion)
- [¿Cómo funciona en la práctica?: Escenario típico con AWS CLI](#practica)
- [¿Por qué CLI es el caso de uso perfecto? ](#caso)
- [Ejemplos de dispositivos con capacidades de entrada limitadas](#limitadas)
- [Device Authorization Flow sin navegador web en el dispositivo](#navegador)


- [](#)
- [](#)

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
        - Útil en situaciones donde el dispositivo primario no tiene internet

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
> [!CAUTION]
> **Problemas:** Keys permanentes, riesgo si se comprometen, difíciles de rotar

    - Profiles con credenciales:
        ```bash
        [default]
        aws_access_key_id = AKIAIOSFODNN7EXAMPLE  
        aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        ```
> [!CAUTION]
> **Problemas:** Credenciales en texto plano en archivos locales
- Con Device Flow (método moderno):
    ```bash
    aws sso login --profile my-sso-profile
    # Autorización una vez, tokens temporales automáticos
    # Renovación transparente
    # Sin credenciales permanentes almacenadas
    ```
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

## ⚙️ Device Authorization Flow sin navegador web en el dispositivo <a name="nevegador"></a> 
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






## ⚙️ Instalar
1. Instalar en macOS:
    ```bash
    # Instalar act (en MacOS con Homebrew)
    brew install act
    act --version
    ```
> [!NOTE]  
> - `xxx` usa ....

## ⚙️ Comandos
-  Iniciar Docker Desktop en macOS:
    ```bash
    open -a Docker
    ```
> [!NOTE]  
> **xxxxxxx:**<br>
> ✓ xxxxxx<br>
> ✓ xxxxxxxx<br> 
> ✗ NO ejecuta nada realmente  

---

## ⚙️ Ejecución por primera vez
- Cuando se ejecuta por primera vez ....
    <p align="center">
        <img src="../../imagenes/act-select-image.png" alt="act-select-image" width="90%">
    </p>


## ⚙️ Elección de ...
- xxxxxx:
    ```bash
    act -P ubuntu-latest=nektos/act-environments-ubuntu:22.04
    ```
---


---

## ⚙️ Ejemplo
- Tenemos ....:
    ```yaml
    name: CI
    on: [push]
    jobs:
        build:
            runs-on: ubuntu-latest
            steps:
                - uses: actions/checkout@v4
                - run: echo "Hola desde GitHub Actions en act"
    ```
- Salida al ejecuarlo
    > [CI/build] 🚀  Start image=nektos/act-environments-ubuntu:22.04<br>
    > [CI/build]   ✅  Success - actions/checkout@v4<br>
    > [CI/build]   ✅  Run echo "Hola desde GitHub Actions en act"<br>
    > Hola desde GitHub Actions en act

---


## 🔗 Referencias
- []()

---