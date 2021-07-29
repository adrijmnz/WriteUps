# TheNoteBook

Buenas! Hoy os voy a enseñar a como pwnear la maquina de HTB que acaban de retirar llamada TheNoteBook, es una maquina de dificultad media y de sistema operativo linux, os dejo aquí unas estadísticas de la maquina:

![images/Untitled.png](images/Untitled.png)

![images/Untitled%201.png](images/Untitled%201.png)

# **ESCANEO**

# **ENUMERACIÓN CON NMAP**

Lo primero de todo como siempre será enumerar que puertos tiene abiertos la maquina, para ellos vamos a utilizar la herramienta "NMAP" con la que haremos un escaneo exhaustivo de puertos.

Para hacer un "Fast Scan" de puertos, siempre suelo utilizar esta sintaxis:

`nmap -sS —min-rate 5000 -p- —open -n -Pn -vvv <IPMACHINE> -oN <FILENAME>` 

-sS : 

![images/Untitled%202.png](images/Untitled%202.png)

—min-rate :

![images/Untitled%203.png](images/Untitled%203.png)

-p- : Escaneo de todos los puertos

![images/Untitled%204.png](images/Untitled%204.png)

—open : Mostrar unicamente puertos abiertos

-n : 

![images/Untitled%205.png](images/Untitled%205.png)

-Pn : Para que no haga descubrimientos de hosts

![images/Untitled%206.png](images/Untitled%206.png)

-vvv : 

![images/Untitled%207.png](images/Untitled%207.png)

-oN :

![images/Untitled%208.png](images/Untitled%208.png)

![images/Untitled%209.png](images/Untitled%209.png)

Yo lo exporto en formato Grep para extraer los puertos con la utilidad "extractPorts" del Youtuber/Streamer "S4vitaar".

Esta utilidad me copia los puertos en la clipboard. Os comparto el Script por aquí pero recordad dejar una estrellita en el Github de S4vitar: (solo tenéis que tener instalado xclip y pegar este código en la .bashrc o .zshrc)

[https://pastebin.com/tYpwpauW](https://pastebin.com/tYpwpauW) 

![images/Untitled%2010.png](images/Untitled%2010.png)

Ahora vamos a enumerar versiones y servicios de todos los puertos con Nmap:

`nmap -sC -sV -p<PUERTOS> <IPMACHINE> -oN <FILENAME>`

-sC : Lanzar una serie de scripts basicos de enumeración

![images/Untitled%2011.png](images/Untitled%2011.png)

-sV : 

![images/Untitled%2012.png](images/Untitled%2012.png)

![images/Untitled%2013.png](images/Untitled%2013.png)

Con la herramienta `whatweb`podemos sacar algo de información de la pagina web:

![images/Untitled%2014.png](images/Untitled%2014.png)

En la pagina web podemos crearnos un usuario, nos creamos nuestro propio usuario:

tukutu:test123:test@test.com

![images/Untitled%2015.png](images/Untitled%2015.png)

Enumerando un poco la pagina vemos una cookie un tanto extraña:

![images/Untitled%2016.png](images/Untitled%2016.png)

# **EXPLOTACION DE LA VULNERABILIDAD**

Vamos a ver que podemos hacer con ella desde `Burpsuite`

Tiene un formato que se parece a un JSON de JWT, copiamos la cookie y la analizamos en [jwt.io](http://jwt.io) 

Por google encuentro como aprovecharme de esto en este [link](https://blog.pentesteracademy.com/hacking-jwt-tokens-kid-claim-misuse-key-leak-e7fce9a10a9c)

Cambiamos el admin_cap a 1 y nos creamos una key en nuestra maquina local:

![images/Untitled%2017.png](images/Untitled%2017.png)

vamos a sustituir el admin_cap a 1:

![images/Untitled%2018.png](images/Untitled%2018.png)

Importante agregar la privkey.key en el jwt.io:

![images/Untitled%2019.png](images/Untitled%2019.png)

Nos quedaría una cookie de esta forma:

![images/Untitled%2020.png](images/Untitled%2020.png)

La copiamos y cambiamos la cookie por la creada, y hemos ganado acceso al panel de admin:

![images/Untitled%2021.png](images/Untitled%2021.png)

# **GANANDO ACCESO A LA MAQUINA**

Podemos subir un archivo; vamos a subir una rev shell:

![images/Untitled%2022.png](images/Untitled%2022.png)

![images/Untitled%2023.png](images/Untitled%2023.png)

Editamos el php y lo subimos:

![images/Untitled%2024.png](images/Untitled%2024.png)

![images/Untitled%2025.png](images/Untitled%2025.png)

Hemos obtenido una Rev. Shell:

![images/Untitled%2026.png](images/Untitled%2026.png)

Hacemos el tratamiento de la TTY:

`script /dev/null -c bash`

`CTRL + Z`

`stty raw -echo;fg`

`reset`

`xterm`

`export TERM=xterm`

`export SHELL=bash`

`stty rows [] columns []`

# **PIVOTANDO A NOAH**

Ya tenemos una consola interactiva como el usuario www-data, toca pivotar al usuario noah para luego escalar privilegios:

Enumerando el sistema encuentro en la carpeta backups, un archivo llamado home.tar.gz, me lo voy a traspasar a mi equipo para extraerlo:

![images/Untitled%2027.png](images/Untitled%2027.png)

![images/Untitled%2028.png](images/Untitled%2028.png)

Lo descomprimimos dos veces con la herramienta `7z`

`7z x home.tar.gz`

`7z x home.tar`

Una vez descomprimido tenemos acceso a la carpeta home de la maquina, por lo que podemos usar la id_rsa de noah para conectarnos por ssh: (`chmod 600 id_rsa`)

![images/Untitled%2029.png](images/Untitled%2029.png)

![images/Untitled%2030.png](images/Untitled%2030.png)

# **ESCALADA DE PRIVILEGIOS**

Toca escalar privilegios a root, con sudo -l nos muestra los programas o comandos que podemos ejecutar con permisos temporales:

![images/Untitled%2031.png](images/Untitled%2031.png)

Examinando que podemos hacer con el comando:

![images/Untitled%2032.png](images/Untitled%2032.png)

Podemos ejecutar comandos en un contenedor, vamos a expaunear una bash:

![images/Untitled%2033.png](images/Untitled%2033.png)

Busco la version del docker para ver si existe algun exploit:

![images/Untitled%2034.png](images/Untitled%2034.png)

[https://github.com/Frichetten/CVE-2019-5736-PoC](https://github.com/Frichetten/CVE-2019-5736-PoC)

Nos descargamos el main.go y modificamos a nuestro gusto:

![images/Untitled%2035.png](images/Untitled%2035.png)

Lo compilamos con `go build main.go` , y nos pasamos el archivo main al docker:

![images/Untitled%2036.png](images/Untitled%2036.png)

Y cuando veamos el Oerwritten ejecutamos en otra ventana :

![images/Untitled%2037.png](images/Untitled%2037.png)

Ahora /bin/bash es SUID:

![images/Untitled%2038.png](images/Untitled%2038.png)