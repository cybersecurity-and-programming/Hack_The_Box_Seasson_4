Hack The Box - Mist
Windows
Insane

Sistema Operativo:
Dificultad:
Release:

30/03/2024

Skills Learned

  Advanced ADCS exploitation
  LDAP and NTLM relay attacks
  Abusing WebDAV
  Kerberos S4U exploitation with shadow credentials
  BloodHound and Certipy usage

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

24 de febrero de 2025

1

La dirección IP de la máquina víctima es 10.129.231.20. Por tanto, envié 5 trazas ICMP para verificar que
existe conectividad entre las dos máquinas.

Enumeración

Una vez que identificada la dirección IP de la máquina objetivo, utilicé el comando nmap -p- -sS -sC -sV
--min-rate  5000  -vvv  -Pn  10.129.231.20  -oN  scanner_mist  para  descubrir  los  puertos  abiertos  y  sus
versiones:











(-p-): realiza un escaneo de todos los puertos abiertos.
(-sS): utilizado para realizar un escaneo TCP SYN, siendo este tipo de escaneo el más común y rápido,
además de ser relativamente sigiloso ya que no llega a completar las conexiones TCP. Habitualmente
se conoce esta técnica como sondeo de medio abierto (half open). Este sondeo consiste en enviar un
paquete SYN, si recibe un paquete SYN/ACK indica que el puerto está abierto, en caso contrario, si
recibe un paquete RST (reset), indica que el puerto está cerrado y si no recibe respuesta, se marca
como filtrado.
(-sC): utiliza los scripts por defecto para descubrir información adicional y posibles vulnerabilidades.
Esta opción es equivalente a --script=default. Es necesario tener en cuenta que algunos de estos scripts
se consideran intrusivos ya que podría ser detectado por sistemas de detección de intrusiones, por lo
que no se deben ejecutar en una red sin permiso.
(-sV): Activa la detección de versiones. Esto es muy útil para identificar posibles vectores de ataque
si la versión de algún servicio disponible es vulnerable.
(--min-rate 5000): ajusta la velocidad de envío a 5000 paquetes por segundo.
(-Pn): asume que la máquina a analizar está activa y omite la fase de descubrimiento de hosts.

24 de febrero de 2025

2

El  análisis  inicial  del  servicio  HTTP  revela  la  exposición  del  puerto  80,  donde  se  ejecuta  un  servidor
Apache 2.4.52 desplegado sobre un entorno Windows. La enumeración pasiva y activa del contenido web
permite identificar la presencia del CMS Pluck 4.7.18, cuya superficie de ataque ha sido objeto de diversas
investigaciones de seguridad en los últimos años.

Web Application

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

24 de febrero de 2025

3

Mediante el abuso del  endpoint vulnerable, fue  posible  recuperar su  contenido íntegro, exponiendo una
cadena extensa con apariencia de hash criptográfico. Dado que el nombre del archivo sugiere su relación
con configuraciones administrativas, se planteó la hipótesis de que dicho valor pudiera corresponder a una
credencial protegida.

El análisis preliminar mediante hash-identifier permitió clasificarlo como un hash SHA-512, abriendo la
puerta a un proceso de cracking orientado a obtener acceso autenticado al panel de administración.

Una  vez  recuperado  el  hash  SHA-512  procedente  del  archivo  admin_backup.php,  se  procedió  a  su
sometimiento a un proceso de cracking mediante Hashcat, empleando como diccionario la archiconocida
wordlist rockyou.txt. La operación resultó exitosa, revelando que el valor original correspondía a la cadena
lexypoo97, lo que confirma que el archivo contenía efectivamente una credencial administrativa en formato
cifrado.

24 de febrero de 2025

4

Con esta información, se accedió al endpoint /login.php, donde el CMS Pluck presenta un mecanismo de
autenticación extremadamente simplificado, basado exclusivamente en la introducción de una contraseña
sin necesidad de usuario asociado. Tras suministrar la credencial recuperada, se obtuvo acceso pleno a la
interfaz administrativa del gestor de contenidos, habilitando así la posibilidad de explotar vectores que
requieren autenticación previa.

En este punto, resultó pertinente retomar el CVE-2023-50564, cuya explotación depende precisamente de
disponer de privilegios administrativos. Esta vulnerabilidad afecta al subsistema de instalación de módulos,
el  cual  carece  de  controles  adecuados  sobre  la  estructura  y  el  contenido  de  los  paquetes  suministrados.
Mediante la construcción de un módulo malicioso —en este caso, un archivo ZIP conteniendo una web
shell  en  PHP  encapsulada  dentro  de  un  directorio—  es  posible  inducir  al  CMS  a  desplegar  archivos
arbitrarios en el directorio /data/modules, lo que deriva en una ejecución remota de código plenamente
funcional.

Para materializar este vector, se generó una web shell en PHP y se empaquetó dentro de un archivo shell.zip,
respetando la estructura esperada por el instalador de módulos. Desde la consola administrativa, se navegó
a Options → Manage modules → Install a module…, donde se procedió a cargar el paquete malicioso.

24 de febrero de 2025

5

El  CMS,  al  no  implementar  ningún  mecanismo  de  validación  robusto,  descomprimió  el  contenido  sin
restricciones, creando el directorio /data/modules/shell/ y depositando en su interior el archivo shell.php.

Con el acceso a la interfaz administrativa y la capacidad de desplegar archivos arbitrarios en el servidor, el
siguiente objetivo consistió en obtener una reverse shell plenamente interactiva a través de la web shell
previamente implantada. Para ello, se optó por utilizar un payload en PowerShell, dado que el entorno
subyacente  es Windows  y  este  lenguaje  proporciona  un  canal  nativo,  versátil  y  altamente  flexible  para
establecer comunicaciones inversas.

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

24 de febrero de 2025

6

Shell as Brandon.Keywarp

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

Durante la enumeración del sistema de archivos, se identificó un directorio inusual en la raíz del volumen
C:\ denominado Common Applications. Su contenido resultó especialmente interesante: una colección de
accesos directos de Windows (.lnk) sobre los cuales el usuario actual disponía de permisos de escritura.
Este vector es particularmente significativo, ya que los accesos directos pueden ser manipulados para alterar
el ejecutable subyacente que invocan, convirtiéndolos en un mecanismo viable para secuestro de ejecución
(execution hijacking).

Una revisión de documentación técnica reveló que los accesos directos pueden ser modificados mediante
PowerShell para redefinir el binario objetivo, permitiendo sustituir la aplicación legítima por un payload
controlado por  el atacante. Siguiendo esta técnica,  se  seleccionó el  acceso directo  Calculator.lnk como
candidato  para  la  sustitución,  configurándolo  para  invocar un  script  de  PowerShell  con  reverse  shell,
análogo al empleado en la fase inicial de compromiso.

24 de febrero de 2025

7

Tras un periodo de espera razonable —presumiblemente hasta que un usuario legítimo interactuó con el
acceso directo manipulado— se estableció una nueva conexión en nuestro listener. Esta vez, la sesión se
ejecutaba bajo el contexto del usuario brandon.keywarp, un usuario perteneciente al dominio y, por tanto,
con un nivel de integración significativamente superior dentro del entorno corporativo.

Análisis de Active Directory

Con el compromiso de un usuario perteneciente al dominio —brandon.keywarp— se abrió la posibilidad
de realizar una enumeración estructural del entorno Active Directory. Para ello, se optó por emplear
BloodHound, una herramienta ampliamente utilizada para el análisis de relaciones de privilegio, rutas de
ataque y configuraciones delegadas dentro de dominios Windows.

El  primer  paso  consistió  en  transferir  y  ejecutar  SharpHound  en  el  host  comprometido,  con  el  fin  de
recolectar información sobre usuarios, grupos, ACLs, políticas y objetos del dominio. Una vez completada
la fase de recolección, el archivo resultante fue exfiltrado hacia la máquina del atacante utilizando el propio
navegador web, aprovechando la conectividad ya establecida.

24 de febrero de 2025

8

Con  los  datos  cargados  en  BloodHound,  se  procedió  a  analizar  las  capacidades  del  usuario
brandon.keywarp dentro del dominio. La enumeración inicial reveló un hallazgo significativo: el usuario,
como miembro del grupo Authenticated Users, posee permisos para solicitar certificados en la Autoridad
Certificadora  MIST-DC01-CA.  Este  tipo  de  permisos  puede  derivar  en  escenarios  de  abuso  de  PKI,
especialmente si existen plantillas de certificado mal configuradas o susceptibles de escalada.

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

Con  la  ruta  excluida  de  Windows  Defender  ya  identificada,  procedimos  a  cargar  Chisel  en  el  host
comprometido y establecer un túnel inverso entre MS01 y nuestra máquina de control. Este canal permitió
enrutar  tráfico  a  través  del  sistema  comprometido  y  extender  nuestra  superficie  de  enumeración  hacia
segmentos internos de la red.

24 de febrero de 2025

9

Una vez operativo el túnel, se ejecutó netexec smb a través de ProxyChains, lo que reveló que el sistema
192.168.100.100  correspondía  al  controlador  de  dominio  del  entorno.  Este  hallazgo  confirmó  la
arquitectura  sospechada  durante  las  fases  iniciales  y  subrayó  la  necesidad  de  obtener  credenciales  o
artefactos que permitieran autenticarnos de forma legítima frente a los servicios del dominio.

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

24 de febrero de 2025

10

El análisis de las plantillas disponibles mediante Certify.exe reveló que la plantilla User admite el uso de
Client Authentication, lo que la convierte en un vector idóneo para la obtención de un certificado válido
asociado  a  la  identidad  de  brandon.keywarp. A  partir  de  esta  plantilla,  se  procedió  a  generar  tanto  el
certificado como la clave privada correspondientes, elementos esenciales  para la explotación  del flujo
PKINIT descrito previamente.

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

24 de febrero de 2025

11

Este resultado constituye un punto de inflexión en la intrusión, ya que disponer del hash NTLM permite
autenticarse  frente  a  múltiples  servicios  del  dominio  y  utilizar  herramientas  ofensivas  con  plena
funcionalidad.  Para  validar  la  integridad  del  hash  obtenido,  se  ejecutó  netexec  nuevamente,  esta  vez
autenticando  explícitamente  como  brandon.keywarp.  La  autenticación  exitosa  confirmó  la  validez  del
hash y habilitó la siguiente fase de enumeración y movimiento lateral dentro del dominio.

Gaining Access as MS01$

Con  el  hash  NTLM  de  brandon.keywarp  ya  en  nuestro  poder,  se  procedió  a  realizar  una  serie  de
comprobaciones  orientadas  a  identificar  configuraciones  de  dominio  susceptibles  de  abuso.  Entre  ellas
destacó una especialmente crítica: la desactivación de LDAP Signing en el controlador de dominio. Esta
configuración, aún presente en numerosos entornos corporativos, abre la puerta a ataques de NTLM relay
siempre que se consiga forzar la autenticación NTLM de un usuario o servicio hacia un endpoint controlado
por el atacante.

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

24 de febrero de 2025

12

Con  el  túnel  operativo,  se  lanzó  ntlmrelayx  con  soporte  para  SMB2,  configurado  para  retransmitir
cualquier  autenticación  NTLM  entrante  hacia  el  servicio  LDAP  del  controlador  de  dominio.  Este  paso
constituye la base del ataque: si se consigue inducir una autenticación NTLM desde cualquier máquina del
dominio  hacia  nuestro  endpoint,  ntlmrelayx  podrá  retransmitirla  al  DC  y  obtener  acceso  LDAP  bajo  la
identidad del usuario víctima.

Tras  configurar  ntlmrelayx  con  soporte  SMB2  a  través  del  túnel  establecido,  se  procedió  a  ejecutar
PetitPotam, especificando nuestra dirección IP como listener y habilitando todas las pipes disponibles para
maximizar las posibilidades de coerción.

El análisis de la salida generada por ntlmrelayx confirmó la hipótesis inicial: el ataque fracasó debido a
que el SMB Signing se encuentra habilitado en el controlador de dominio, lo que impide la retransmisión
de autenticaciones NTLM capturadas.

24 de febrero de 2025

13

A modo de referencia, si el mismo intento de autenticación se dirigiera hacia MS01, el ataque sería viable,
ya  que  dicho  host  no  implementa  SMB  Signing;  sin  embargo,  los  machine  accounts  no  pueden  iniciar
sesiones interactivas sobre recursos de red, por lo que este vector no resulta aprovechable.

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

24 de febrero de 2025

14

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

El intento inicial de añadir un registro DNS utilizando las credenciales de brandon.keywarp resultó fallido.
Aunque  en  muchos  dominios  los  usuarios  autenticados  disponen  de  permisos  para  crear  entradas  DNS
dinámicas, en este entorno dicha capacidad no está habilitada, lo que nos obliga a replantear la estrategia.
Dado  que  no  podemos  inducir  autenticación  NTLM  hacia  un  recurso  externo  controlado  por  nosotros,
debemos aprovechar un activo interno bajo nuestro control: MS01.

24 de febrero de 2025

15

La idea consiste en utilizar un puerto arbitrario del propio MS01 —por ejemplo, el 9999— y redirigir todo
el tráfico entrante en dicho puerto hacia el puerto 80 de nuestra máquina atacante. De este modo, al ejecutar
PetitPotam contra MS01@9999/whatever, la autenticación NTLM generada por MS01 será reenviada a
nuestro listener, permitiendo su posterior retransmisión hacia el controlador de dominio.

Para  implementar  esta  redirección,  se  configuró  una  regla  de  port  forwarding  a  través  del  túnel
previamente  establecido  con  Chisel,  asegurando  que  el  tráfico WebDAV  originado  en  MS01  alcanzara
nuestro host de forma transparente.

Antes de relanzar el ataque de relay, se optó por utilizar un fork de Impacket que incorpora funcionalidades
extendidas  para  la manipulación  de  Shadow  Credentials  directamente  desde  la  shell  interactiva  LDAP
proporcionada  por  ntlmrelayx.  Esta  variante  facilita  enormemente  el  proceso  de  inyección  de  claves
públicas  en  el  atributo  msDS-KeyCredentialLink,  evitando  la  necesidad  de  herramientas  adicionales  o
modificaciones manuales.

Tras  clonar  el  repositorio  y  preparar  un  entorno  virtual  de  Python,  se  ejecutó  ntlmrelayx.py  desde  el
directorio examples/, utilizando ProxyChains para enrutar el tráfico a través del túnel. La herramienta se
configuró para apuntar al servicio LDAPS del controlador de dominio y se habilitó la opción -i con el fin
de obtener una shell LDAP interactiva tras un relay exitoso.

Con el listener preparado, se lanzó nuevamente PetitPotam, esta vez apuntando a MS01@9999. El ataque
tuvo éxito: ntlmrelayx  registró la recepción de  una  autenticación  NTLM válida procedente de la cuenta
MS01$, la retransmitió correctamente hacia LDAPS y abrió una shell interactiva en el puerto 11000 de
nuestra máquina local.

24 de febrero de 2025

16

Al conectarnos mediante nc 127.0.0.1 11000 y ejecutar help, se confirmó la disponibilidad de los comandos
clear_shadow_creds y set_shadow_creds, incorporados en este fork específico.

Con estas herramientas, procedimos a preparar el entorno para la inyección de Shadow Credentials en la
cuenta  de  máquina  MS01$,  comenzando  por  eliminar  cualquier  clave  preexistente  mediante
clear_shadow_creds.

Tras establecer el túnel y disponer de un canal estable hacia el entorno interno, se procedió a obtener el
hash NTLM de la cuenta de máquina MS01$ utilizando Certipy, aprovechando la infraestructura AD
CS  del  dominio.  Una  vez  generado  el  certificado  asociado  a  las  Shadow  Credentials,  se  eliminó  la
protección por contraseña del archivo PFX con el fin de permitir su uso directo en operaciones PKINIT.

En  lugar  de  recurrir  a  un  flujo  basado  en  claves  RC4  derivadas  del  hash  NTLM,  se  optó  por  una
aproximación más robusta: solicitar un TGT para MS01$ empleando directamente el certificado PFX,
lo  que  permite  autenticarse  mediante  PKINIT  utilizando  la clave  privada que  controlamos  gracias  a  las
Shadow Credentials.

24 de febrero de 2025

17

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

El resultado fue un ticket Kerberos en formato kirbi, codificado en Base64. Tras extraerlo, se procedió a
decodificarlo  y  almacenarlo  localmente.  Posteriormente,  se  utilizó  la  herramienta  ticketConverter  de
Impacket  para  transformar  el  archivo  kirbi  en  un  ccache,  formato  compatible  con  las  herramientas  de
autenticación de Impacket.

Con  el  ccache  preparado,  se  ejecutó  un  remote  secretsdump  contra  MS01,  aprovechando  el  ticket  de
servicio  obtenido  mediante  la  cadena  S4U.  Este  procedimiento  permitió  recuperar  el  hash  NTLM  del
usuario Administrator, consolidando así el acceso privilegiado al host.

24 de febrero de 2025

18

Finalmente, utilizando dicho hash, se estableció una sesión interactiva sobre WinRM mediante evil-winrm,
obteniendo  control  administrativo  completo  sobre  MS01  y  cerrando  de  forma  exitosa  la  cadena  de
explotación.

Gaining Access as op_Sharon.Mullard on DC01

Durante  la  enumeración  del  sistema  comprometido  con  privilegios  de  Administrator,  se  identificaron
varios elementos de interés en el directorio de usuario de sharon.mullard. Entre ellos destacaba un archivo
KeePass (.kdbx) denominado sharon.kdbx, junto con dos imágenes almacenadas en la carpeta Pictures.
Dado que una base de datos KeePass suele contener credenciales de alto valor, se procedió a descargar los
tres archivos para su análisis local.

El archivo image_20022024.png resultó especialmente relevante: mostraba lo que parecía ser un intento de
la usuaria de manipular o transformar una contraseña utilizando CyberChef, concretamente aplicando una
conversión a Base64.

24 de febrero de 2025

19

Como  primer  intento,  se  utilizó  esta  cadena  directamente  para  desbloquear  la  base  de  datos  KeePass
mediante keepassxc, pero la operación falló. Un examen más detallado de la imagen reveló que parte de la
contraseña parecía estar oculta tras la ventana del bloc de notas, lo que sugería que la cadena visible era
incompleta.

Para recuperar los caracteres faltantes, se optó por realizar un ataque de tipo mask attack con hashcat,
utilizando como base la porción conocida de la contraseña. Antes de ello, fue necesario extraer el hash de
la base de datos KeePass mediante keepass2john, generando un formato compatible con hashcat.

Una  vez  obtenido  el  hash,  se  ejecutó  hashcat  con  una  máscara  adaptada  a  la  estructura  parcial  de  la
contraseña, permitiendo recuperar la cadena completa tras un proceso de cracking exitoso.

Con la contraseña íntegra ya identificada, se procedió finalmente a desbloquear la base de datos KeePass,
habilitando el acceso a las credenciales almacenadas en su interior.

Durante la revisión de la base de datos KeePass, se identificó una única entrada que contenía la contraseña
ImTiredOfThisJob:(.

Sin  embargo,  la  entrada  no  especificaba  un  nombre  de  usuario  asociado,  lo  que  impedía  determinar
directamente a qué identidad del dominio pertenecía. Para resolver esta incertidumbre, se decidió realizar
un  password  spraying  controlado  contra  todos  los  usuarios  del  dominio,  con  el  fin  de  identificar  si  la
contraseña correspondía a alguna cuenta válida.

Como  primer  paso,  se  generó  un  listado  completo  de  usuarios  del  dominio  utilizando  el  módulo
GetADUsers de Impacket, obteniendo así un inventario exhaustivo de identidades potenciales.

24 de febrero de 2025

20

Con  el  listado  de  usuarios  disponible,  se  procedió  a  ejecutar  un  ataque  de  password  spraying  mediante
netexec, proporcionando la contraseña recuperada de la base de datos KeePass. El objetivo era identificar
cualquier coincidencia válida sin provocar bloqueos de cuenta ni generar ruido innecesario en el entorno.
El resultado reveló que la contraseña correspondía a la cuenta op_Sharon.Mullard.

Es  importante  destacar  que  esta  identidad  es  distinta  de  la  cuenta  Sharon.Mullard,  lo  que  sugiere  la
existencia de un usuario operativo o de servicio asociado a la misma persona, pero con un rol diferenciado
dentro del dominio.

Con estas credenciales válidas, se procedió a establecer una sesión remota contra DC01 mediante WinRM,
consolidando así acceso directo al controlador de dominio bajo una identidad legítima del entorno.

Gaining control over the svc_ca$ account

Tras  obtener  acceso  interactivo  al  controlador  de  dominio  mediante  la  cuenta  op_Sharon.Mullard,
resultaba  imprescindible  volver  a  consultar  BloodHound  para  identificar  posibles  configuraciones
delegadas o permisos sensibles asociados a este nuevo usuario. El análisis reveló un hallazgo especialmente
relevante:  como  miembro  del  grupo  OPERATIVES,  la  cuenta  op_sharon.mullard  posee  el  privilegio
ReadGMSAPassword sobre el objeto SVC_CA$, un Group Managed Service Account (gMSA) utilizado
en el entorno.

24 de febrero de 2025

21

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

Gaining control over svc_cabackup account

Tras obtener acceso a la cuenta SVC_CA$, resultaba imprescindible volver a consultar BloodHound para
identificar posibles  rutas adicionales de escalada  asociadas a este nuevo  principal.  El análisis reveló un
hallazgo  especialmente  significativo:  SVC_CA$  posee  el  privilegio  AddKeyCredentialLink  sobre  la
cuenta SVC_CABACKUP.

Este permiso es crítico porque permite modificar el atributo msDS-KeyCredentialLink del objeto objetivo,
lo que habilita la inyección de Shadow Credentials. En la práctica, esto significa que podemos replicar
exactamente la misma técnica utilizada previamente contra MS01$: añadir una clave pública controlada
por nosotros al objeto SVC_CABACKUP, y posteriormente utilizar la clave privada correspondiente para
autenticarnos mediante PKINIT como dicho principal.

Una  vez  añadidas  las  Shadow  Credentials,  la  cuenta  SVC_CABACKUP  pasa  a  ser  completamente
controlable desde el punto de vista criptográfico. Esto permite ejecutar la misma cadena TGT → U2U que
se  utilizó  anteriormente  para  el  usuario  brandon.keywarp,  obteniendo  finalmente  el  hash  NTLM  de  la
cuenta de servicio.

24 de febrero de 2025

22

Para llevar a cabo esta operación, se empleó Certipy a través de ProxyChains, aprovechando su capacidad
para gestionar Shadow Credentials y realizar autenticación PKINIT de forma transparente sobre el túnel
establecido. Certipy automatiza tanto la inyección de claves como la solicitud de TGTs y la ejecución del
flujo  U2U,  lo  que  simplifica  considerablemente  el  proceso  respecto  a  la  combinación  manual  de
herramientas utilizada en fases anteriores.

Obtaining Domain Admin Privileges

Tras  comprometer  la  cuenta  SVC_CABACKUP,  se  revisaron  nuevamente  sus  pertenencias  a  grupos
mediante  BloodHound,  con  el  objetivo  de  identificar  rutas  adicionales  de  escalada  dentro  de  la
infraestructura de Active Directory.

El  análisis  reveló  que  SVC_CABACKUP  es  miembro  del  grupo  CERTIFICATE  SERVICES,  un
conjunto de identidades con privilegios avanzados sobre la infraestructura de certificación del dominio. A
partir de esta información, se ejecutaron varias de las consultas Cypher predefinidas en BloodHound, en
particular la que evalúa plantillas vulnerables a ADCS ESC13 mediante la relación Enrollment Rights on
CertTemplates  with  OIDGroupLink.  Esta  consulta  permite  identificar  plantillas  de  certificación  que,
combinadas con permisos delegados, pueden ser abusadas para obtener privilegios elevados.

24 de febrero de 2025

23

El resultado fue especialmente relevante: los miembros del grupo CERTIFICATE SERVICES pueden
obtener  membresía  efectiva  en  el  grupo  CERTIFICATE  MANAGERS  abusando  de  la  plantilla
MANAGERAUTHENTICATION,  la  cual  es  vulnerable  a  ESC13.  Esta  relación  es  crítica,  ya  que
CERTIFICATE MANAGERS forma parte del grupo CA_BACKUP, ampliando aún más la superficie de
ataque.

A  su  vez,  los  miembros  del  grupo  CA_BACKUP  pueden  obtener  membresía  en  el  grupo
SERVICEACCOUNTS  mediante  la  plantilla  BACKUPSVCAUTHENTICATION,  que  también
presenta  vulnerabilidades  asociadas  a  ESC13.  Esta  cadena  de  relaciones  permite  escalar  privilegios  de
forma progresiva a través de la infraestructura de certificación, aprovechando plantillas mal configuradas
y permisos delegados entre grupos.

24 de febrero de 2025

24

Finalmente,  el  grupo  SERVICEACCOUNTS  es  miembro  del  grupo  BACKUP  OPERATORS,  un
conjunto de identidades con privilegios altamente sensibles dentro del dominio, incluyendo la capacidad de
leer  archivos  protegidos,  manipular  servicios  críticos  y,  en  algunos  casos,  realizar  operaciones  que
conducen directamente a la toma de control del controlador de dominio.

Tras identificar que la cadena de privilegios derivada de SVC_CABACKUP finalizaba en el grupo Backup
Operators, se evaluó el impacto real de alcanzar dicho nivel de acceso. Los miembros de este grupo poseen
permisos  de  lectura  sobre  todos  los  archivos  del  sistema,  incluidos  los  hives  SAM,  SYSTEM  y
SECURITY, lo que permite extraer las credenciales locales almacenadas en el controlador de dominio.
Este vector constituye una vía directa para comprometer completamente el host y, por extensión, el dominio.

A partir de la información recopilada mediante BloodHound, se diseñó una cadena de explotación basada
en  el  abuso  de  ADCS  ESC13,  aprovechando  plantillas  de  certificación  mal  configuradas  y  relaciones
delegadas  entre  grupos.  El  primer  paso  consistió  en  solicitar  un  certificado  para  la  cuenta
SVC_CABACKUP utilizando la plantilla ManagerAuthentication, vulnerable a ESC13.

La emisión de este certificado otorgó membresía efectiva en el grupo Certificate Managers, habilitando
el acceso a plantillas adicionales que no estaban disponibles inicialmente.

24 de febrero de 2025

25

Con este certificado, se obtuvo un TGT mediante PKINIT y se utilizó para solicitar un segundo certificado
basado en la plantilla BackupSvcAuthentication, igualmente vulnerable a ESC13 y accesible únicamente
gracias a la membresía adquirida en el paso anterior.

La  emisión  de  este  segundo  certificado  elevó  los  privilegios  efectivos  de  la  cuenta  hasta  el  grupo
ServiceAccounts, que a su vez deriva en Backup Operators, consolidando así el acceso necesario para
interactuar con los hives del sistema.

Una  vez  obtenido  el  nuevo  TGT  asociado  a  estos  privilegios,  se  procedió  a  extraer  los  hives  SAM,
SYSTEM y SECURITY utilizando el módulo reg de Impacket.

Posteriormente, los archivos fueron descargados a través de evil-winrm y procesados localmente mediante
secretsdump, lo que permitió recuperar los hashes locales del  sistema, incluido  el correspondiente a la
cuenta de máquina DC01$.

24 de febrero de 2025

26

Con el hash NTLM de DC01$ en nuestro poder, se ejecutó un ataque DCSync, solicitando directamente
los secretos del dominio y obteniendo el hash NTLM del Domain Administrator. Este paso consolidó el
control total sobre la infraestructura de Active Directory.

Finalmente, utilizando evil-winrm, se estableció una sesión interactiva como Domain Administrator, lo
que permitió acceder al controlador de dominio y recuperar la flag root.txt, completando así el compromiso
integral del entorno.

Curiosidad técnica

Como observación adicional, una vez completado el compromiso del entorno y obtenidos privilegios de
administrador de dominio, fue posible establecer una conexión RDP directa hacia el sistema comprometido.
Aunque no formaba parte del flujo principal de explotación, este acceso remoto permitió interactuar con el
host de manera gráfica, sin necesidad de recurrir exclusivamente a sesiones de PowerShell o WinRM.

24 de febrero de 2025

27

El acceso por RDP no aportó ventajas operativas significativas respecto a las técnicas ya empleadas, pero
sí  ofreció  una  perspectiva  interesante  del  sistema  desde  el  punto  de  vista  del  usuario  final,  permitiendo
validar visualmente el estado del entorno y confirmar la extensión del compromiso. Este tipo de acceso
suele resultar útil en escenarios reales para realizar comprobaciones rápidas, revisar configuraciones locales
o verificar el impacto de determinadas acciones sin depender únicamente de la línea de comando

24 de febrero de 2025

28


