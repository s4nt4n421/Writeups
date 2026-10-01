# Hack The Box — TwoMillion

> **Plataforma:** Hack The Box  
> **Máquina:** TwoMillion  
> **Dificultad:** Easy  
> **SO:** Linux  
> **IP de la instancia:** dinámica — sustituir por la IP asignada por HTB  
> **Host:** `2million.htb`

## 1. Resumen

TwoMillion es una máquina Linux de dificultad **Easy** centrada principalmente en una aplicación web/API.

La cadena de compromiso utilizada fue:

```text
Reconocimiento
    ↓
Virtual Host 2million.htb
    ↓
Enumeración de la API
    ↓
Mass Assignment → privilegios de administrador
    ↓
Command Injection → RCE
    ↓
Credenciales de MySQL
    ↓
Reutilización de credenciales → SSH como admin
    ↓
Enumeración local
    ↓
CVE-2023-0386
    ↓
Root
```

La resolución se realizó únicamente contra la instancia de laboratorio asignada por HTB.

---

## 2. Reconocimiento

### 2.1 Escaneo de puertos

Se realizó un escaneo completo de TCP para identificar la superficie expuesta.

```bash
nmap -p- --min-rate 1000 -Pn <TARGET_IP>
```

Puertos relevantes encontrados:

| Puerto | Estado | Servicio | Versión |
|---|---|---|---|
| `22/tcp` | open | SSH | OpenSSH 8.9p1 Ubuntu |
| `80/tcp` | open | HTTP | nginx |

Posteriormente se realizó fingerprinting sobre los puertos encontrados:

```bash
nmap -sC -sV -p22,80 <TARGET_IP>
```

### 2.2 Enumeración HTTP

El acceso directo por IP redirigía a un virtual host:

```text
http://<TARGET_IP>/  → 301 → http://2million.htb/
```

Por tanto, se añadió el nombre al archivo `/etc/hosts`:

```text
<TARGET_IP>    2million.htb
```

La aplicación pasó a ser accesible mediante:

```text
http://2million.htb/
```

---

## 3. Enumeración de la aplicación web

Se identificaron varias rutas interesantes:

```text
/login
/invite
/register
/api
/home
```

La aplicación utilizaba una API bajo `/api/v1/`.

La enumeración de endpoints permitió identificar una superficie administrativa que resultó ser especialmente interesante.

---

## 4. Escalada de privilegios en la API — Mass Assignment

Uno de los endpoints encontrados fue:

```text
/api/v1/admin/settings/update
```

Al probar diferentes métodos HTTP se observó que el endpoint aceptaba `PUT`.

La respuesta mostraba que el parámetro `is_admin` podía ser modificado sin comprobar correctamente si el usuario que realizaba la petición tenía realmente privilegios administrativos.

La vulnerabilidad se validó modificando el atributo:

```json
{
  "email": "<USER_EMAIL>",
  "is_admin": 1
}
```

Después de realizar la modificación, una nueva consulta al endpoint de autenticación confirmó el cambio:

```text
is_admin: true
```

### Vulnerabilidad

Se trata de un problema de **Mass Assignment / Broken Access Control**: el servidor permitía al cliente modificar un atributo sensible (`is_admin`) que debería estar controlado exclusivamente por la lógica del servidor.

### Impacto

El usuario normal pasó a tener privilegios administrativos dentro de la aplicación.

---

## 5. Command Injection — RCE

Una vez obtenido el rol de administrador se continuó enumerando la API.

Se identificó:

```text
POST /api/v1/admin/vpn/generate
```

El endpoint generaba configuraciones relacionadas con la VPN utilizando el valor proporcionado en `username`.

Antes de intentar una shell, se realizó una prueba benigna para confirmar ejecución de comandos.

La evidencia obtenida confirmó ejecución de comandos en el servidor.

Por tanto, se validó una **Command Injection** que permitía conseguir **Remote Code Execution (RCE)**.

### Enfoque utilizado

Se siguió el principio:

```text
hipótesis
   ↓
prueba mínima
   ↓
confirmación
   ↓
RCE
```

Esto permitió comprobar la vulnerabilidad antes de pasar a una explotación más invasiva.

---

## 6. Extracción de credenciales de la aplicación

Con RCE se revisó la configuración de la aplicación y se encontraron credenciales de la base de datos.

Datos relevantes encontrados durante la explotación:

```text
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

La base de datos MySQL estaba escuchando únicamente en `127.0.0.1`, por lo que no estaba directamente expuesta a la red.

Con las credenciales recuperadas se pudo consultar la base de datos desde el contexto comprometido.

---

## 7. Extracción de hashes

La tabla `users` contenía tres cuentas con sus correspondientes hashes bcrypt.

Entre ellas estaba la cuenta utilizada por el atacante durante el compromiso.

Los hashes eran del tipo:

```text
$2y$...
```

Se valoró el cracking offline, pero la velocidad observada era demasiado baja para que `rockyou.txt` fuese una vía eficiente.

El cálculo aproximado durante la prueba fue de alrededor de **19 hashes/segundo**, con una ETA de varios días.

En lugar de continuar con un proceso costoso, se priorizó una alternativa basada en la evidencia ya obtenida: la posible reutilización de la contraseña de la base de datos.

---

## 8. Acceso SSH como `admin`

La contraseña recuperada de la configuración de la base de datos resultó reutilizable para el usuario del sistema `admin`.

Con ello fue posible acceder mediante SSH:

```bash
ssh admin@<TARGET_IP>
```

Después se confirmó el contexto:

```bash
id
whoami
hostname
```

El usuario obtenido fue:

```text
admin
uid=1000(admin)
```

La flag de usuario se obtuvo mediante:

```bash
cat ~/user.txt
```

> **Nota:** no incluyo aquí las flags de la instancia para que el writeup sea reutilizable aunque HTB asigne una instancia distinta.

---

## 9. Enumeración post-explotación

Ya como `admin` se inició la enumeración local para identificar posibles vías de escalada.

Las comprobaciones relevantes incluyeron:

```bash
id
uname -a
sudo -l
```

Además, se revisaron posibles vectores habituales:

```bash
find / -perm -4000 -type f 2>/dev/null
```

También se verificaron servicios y otros componentes relevantes del sistema.

La cuenta `admin` no disponía de privilegios sudo suficientes para convertirse directamente en root.

---

## 10. Privilege Escalation — CVE-2023-0386

Durante la enumeración del sistema se identificó un kernel vulnerable a **CVE-2023-0386**, una vulnerabilidad relacionada con **OverlayFS** que puede permitir una escalada local de privilegios.

Antes de ejecutar el exploit se comprobó la versión del kernel y la aplicabilidad del vector.

La explotación permitió elevar el contexto desde:

```text
admin
```

a:

```text
root
```

La elevación se confirmó mediante:

```bash
id
```

con un resultado equivalente a:

```text
uid=0(root) gid=0(root)
```

Finalmente se obtuvo la flag de root con:

```bash
cat /root/root.txt
```

---

## 11. Kill Chain completa

```text
┌───────────────────────────────┐
│       Reconocimiento          │
│  22/SSH + 80/HTTP             │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│      Virtual Host             │
│      2million.htb             │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│       Enumeración API         │
│       /api/v1/...             │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│       Mass Assignment         │
│       is_admin = 1            │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│       Command Injection       │
│       → RCE                    │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│       Credenciales DB         │
│       MySQL                    │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│     Reutilización credencial  │
│       → SSH admin             │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│       Enumeración local       │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│       CVE-2023-0386           │
│       OverlayFS                │
└───────────────┬───────────────┘
                ↓
┌───────────────────────────────┐
│             ROOT              │
└───────────────────────────────┘
```

---

## 12. Vulnerabilidades principales

| Vulnerabilidad | Descripción | Impacto |
|---|---|---|
| Broken Access Control / Mass Assignment | El cliente podía modificar `is_admin` | Escalada a administrador de la aplicación |
| Command Injection | Entrada controlada por el usuario ejecutada por el servidor | RCE |
| Credential Reuse | Credencial de aplicación/DB reutilizada en SSH | Acceso como `admin` |
| CVE-2023-0386 | Vulnerabilidad de OverlayFS | Escalada a root |

---

## 13. Lecciones aprendidas

### 13.1 Enumerar métodos HTTP

Un endpoint que responde `405 Method Not Allowed` con un método puede aceptar correctamente otro. Por ello no conviene asumir que un endpoint solo funciona con `GET`.

### 13.2 Revisar controles de autorización

Cuando una API acepta objetos JSON actualizables, es importante comprobar si el cliente puede modificar campos que deberían estar controlados exclusivamente por el servidor.

### 13.3 Validar antes de explotar

La command injection se confirmó inicialmente con una prueba de bajo impacto antes de intentar obtener una shell.

### 13.4 Estimar el coste del cracking

No siempre compensa crackear un hash. Si la velocidad es baja y existe una alternativa respaldada por la evidencia, conviene cambiar de estrategia.

### 13.5 Buscar reutilización de credenciales

Una credencial obtenida de una aplicación o base de datos puede ser reutilizada en otros servicios. Debe comprobarse de forma controlada dentro del alcance.

### 13.6 Enumeración local orientada a evidencias

Después de obtener una shell, comprobar usuario, grupos, kernel, sudo, SUID, capabilities y servicios permite priorizar las posibles vías de escalada.

---

## 14. Recomendaciones de mitigación

### API

- Implementar allowlists de atributos modificables.
- Ignorar atributos sensibles enviados por el cliente.
- Validar autorización en cada endpoint administrativo.
- No confiar en valores proporcionados por el usuario para establecer privilegios.

### Command Injection

- Nunca construir comandos del sistema concatenando entrada del usuario.
- Utilizar APIs/funciones seguras en lugar de ejecutar shell.
- Aplicar validación estricta de entrada.
- Ejecutar los servicios con el mínimo privilegio posible.

### Credenciales

- No reutilizar contraseñas entre aplicación, base de datos y sistema operativo.
- Utilizar secretos independientes por servicio.
- Evitar almacenar credenciales en archivos accesibles por el proceso web.

### Sistema operativo

- Mantener el kernel actualizado.
- Monitorizar vulnerabilidades de componentes como OverlayFS.
- Aplicar el principio de mínimo privilegio.

---

## 15. Referencias

- Hack The Box — TwoMillion: https://www.hackthebox.com/machines/twomillion
- NVD — CVE-2023-0386: https://nvd.nist.gov/vuln/detail/CVE-2023-0386

---

## 16. Flags

Las flags se omiten deliberadamente de este documento porque son específicas de la instancia de HTB utilizada.

```text
user.txt  → obtenido correctamente
root.txt  → obtenido correctamente
```

---

## 17. Resumen final

La máquina se resolvió encadenando una vulnerabilidad de autorización en la API con una command injection, utilizando después las credenciales recuperadas para acceder por SSH y finalmente explotando una vulnerabilidad local del kernel para obtener root.

La parte más relevante desde el punto de vista metodológico fue cambiar de estrategia cuando el cracking de bcrypt resultó demasiado costoso y priorizar la reutilización de una credencial ya obtenida.
