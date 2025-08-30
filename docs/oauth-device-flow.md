# 🧪 Guía de OAuth 2.0 Device Authorization Flow

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-%23FE5196?logo=conventionalcommits&logoColor=white)](https://conventionalcommits.org)

## Contenido
- [¿Qué es OAuth 2.0 Device Authorization Flow](#ques-es)
- [El problema de usar Authorization Code Flow en CLI](#problema)
- [No requiere un navegador web en el dispositivo](#navegador)

- [](#)
- [](#)
- [](#)
- [](#)
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
- Usar `Authorization Code Flow` para CLI es problemático
- ¿como funciona `Authorization Code Flow`?
    > 1.- Aplicación → "Ve a esta URL para loggearte"<br>
    > 2.- Usuario → Abre navegador y va a la URL<br>
    > 3.- Usuario → Se loggea en el navegador<br>
    > 4.- Proveedor → Redirige de vuelta a la aplicación<br>
    > 5.- Aplicación → Recibe el código y lo intercambia por tokens<br>
- ¿cuál es el probema de usar el flujo `Authorization Code Flow` con CLI?
    > - **Paso 1: "Ve a esta URL"**
    >   - CLI no puede "abrir" un navegador de forma confiable
    >   - En servidores SSH puede no haber navegador disponible<
    > - **Paso 4: "Redirige de vuelta a la aplicación"**
    >   - ¿A dónde redirigir? CLI no tiene una URL
    >   - ¿Cómo captura el CLI la respuesta del redirect?

- La solución es usar `Device Authorization Flow`
> [!NOTE]  
> - OAuth 2.0 = Protocolo flexible y bueno
> - Device Flow = Una herramienta específica dentro de OAuth 2.0
> - Authorization Code Flow = Otra herramienta, pero para casos diferentes


## ⚙️ Ejemplos de dispositivos con capacidades de entrada limitadas <a name="limitadas"></a> 


## ⚙️ No requiere un navegador web en el dispositivo <a name="nevegador"></a> 
### El problema tradicional:
- En flujos OAuth normales (como Authorization Code), el proceso es:
    - Aplicación redirige al usuario a una página de login
    - Usuario ingresa credenciales en el navegador
    - Proveedor redirige de vuelta a la aplicación

> [!NOTE]  
> `xxxx` es útil sobre todo ...

---

## ⚙️ ¿Cómo funciona internamente?


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