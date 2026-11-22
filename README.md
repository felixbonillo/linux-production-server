# 🛡️ Linux Production Server Hardening & Reverse Proxy

Este repositorio documenta el despliegue de una arquitectura de servidor web de nivel producción en **AlmaLinux 9**, aplicando los principios de **Zero Trust**, **Mínimo Privilegio (PoLP)** y **Defensa Activa**. El objetivo fue exponer un microservicio (Python) de forma segura a través de un Reverse Proxy (Nginx), blindando el acceso administrativo y aplicando políticas estrictas de SELinux.

## 🏗️ Arquitectura del Sistema

La arquitectura está diseñada para que el backend (Python) sea completamente invisible para el mundo exterior:

1.  **Cliente (Exterior):** Realiza peticiones HTTP al puerto 80.
2.  **Firewalld (El Guardián):** Permite únicamente el tráfico HTTP (80) y SSH Hardened (2222).
3.  **Nginx (Reverse Proxy):** Recibe la petición y la redirige internamente a `127.0.0.1:9000`.
4.  **SELinux (Guardián Interno):** Autoriza a Nginx a hacer conexiones de red (Booleano `relay`) y permite el acceso a los archivos web (`httpd_sys_content_t`).
5.  **Python Backend (Systemd):** Escucha exclusivamente en Loopback (`127.0.0.1:9000`), sin acceso externo.

## 🛠️ Tecnologías y Herramientas Utilizadas
-   **Sistema Operativo:** AlmaLinux 9 (Familia RHEL)
-   **Servidor Web/Proxy:** Nginx
-   **Backend:** Python3
-   **Gestión de Servicios:** Systemd
-   **Seguridad:** SELinux (Contextos, Booleanos, Puertos), Firewalld
-   **Blindaje SSH:** Llaves ED25519, Fail2Ban

## 🚀 Implementaciones Clave

### 1. Gestión de Servicios (Systemd)
-   Creación de una unidad `prod-app.service` para el backend de Python.
-   Configuración de `Type=simple` y `Restart=always` para alta disponibilidad.
-   Aislamiento del backend usando `--bind 127.0.0.1` (Loopback).

### 2. Políticas de SELinux (Sin desactivar la seguridad)
-   **Contextos de Archivo:** Uso de `semanage fcontext` para etiquetar la carpeta `/var/www/prod-app` con `httpd_sys_content_t`.
-   **Puertos Autorizados:** Uso de `semanage port` para declarar el puerto 9000 (backend) y 2222 (SSH) como válidos.
-   **Booleanos Restrictivos:** Uso de `httpd_can_network_relay` (en lugar del amplio `connect`) para limitar las conexiones salientes de Nginx exclusivamente a puertos web.

### 3. Blindaje de SSH (Hardening) y Anti-Lockout
-   Generación de llaves criptográficas asimétricas `ED25519` (Zero Trust).
-   Deshabilitación de `PermitRootLogin` y `PasswordAuthentication`.
-   Cambio de puerto por defecto (22 -> 2222).
-   **Protocolo Anti-Lockout:** Validación de acceso en una segunda terminal antes de cerrar el puerto 22 en Firewalld.

### 4. Defensa Activa (Fail2Ban)
-   Configuración de un archivo `jail.local` limpio para evitar conflictos.
-   Vigilancia del log `/var/log/secure` en el puerto 2222.
-   Política de bloqueo: 3 intentos fallidos = 10 minutos de baneo a nivel de Firewalld.

## 🧠 Lecciones Aprendidas (Troubleshooting)
-   **Error 502 Bad Gateway:** Ocurre cuando Nginx intenta conectarse al backend pero SELinux lo bloquea (falta de booleano o puerto no autorizado).
-   **El peligro de copiar el `jail.conf`:** Demostró que es mejor crear un `jail.local` desde cero para evitar directivas duplicadas que rompan Fail2Ban.
