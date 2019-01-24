# Scripts-Despliegue_App

## Despliegue de una aplicación escalable.

![pfinal-screenshot](https://user-images.githubusercontent.com/36509669/51669685-44838880-1fc5-11e9-9f1f-91dc1a646921.JPG)

Se utilizan scripts escritos en Python para la puesta en marcha de una base de datos, tres discos de almacenamiento (NAS), tres servidores, un balanceador de carga y la configuración del firewall (se ejecuta para éste último un Shell-script, se le llama desde 7.py).

### Pasos a seguir:
- Guardar todos los archivos en /home/upm
- Abrir una consola
    - Cambiarse a **cd /home/upm**
    - Ejecutar el comando **python3 0** 



# Cómo desplegarlo en AWS (Amazon Web Service)

![aws-screenshot](https://user-images.githubusercontent.com/36509669/51670770-a2b16b00-1fc7-11e9-92bd-edb7b5ca5288.JPG)

-	Crear un VPC (Amazon Virtual Private Cloud) en la sección your VPC (lo llamaremos “MyVPC”). Esto lo hacemos para definir nuestra network.
    -	Dentro metemos subnets, creamos de la siguiente forma 3: 
        -	3 subnet pública: LAN1, LAN2 y LAN3 (las daremos la IP 20.2.1.0/24, 20.2.2.0/24 y 20.2.3.0/24 respectivamente) y que será donde pondremos los servidores webs. Esto lo hacemos así para asegurarnos alta disponibilidad.
        -	1 subnet privada: LAN4 (la daremos la IP tal cual del escenario 20.2.4.0/24) y que será donde conectemos aquello que no debe tener acceso a internet: base de datos y las máquinas de almacenamiento.
    -	Internet Gateway necesitamos una para la VPC, para que tenga salida a internet.
    -	En cada VPC tiene sus propias route tables y se pueden asociar a subnets. Así creamos 2:
        -	Con nombre “MyRoute-public” - asignar a las subnets que queramos hacer públicas
        -	Con nombre “MyRoute-private” - asignar a las subnets que queramos hacer privadas
-	Vamos ahora a crear instancias para nuestros 3 servidores. Con EC2 elegimos en los apartados:
    -	En S1 (servidor1)
        -	Network: “MyVPC”
        -	Subnet: va a ser pública, pues elegimos la subnet donde está LAN1
        -	Dirección IP: hay que configurarlas de forma automática para ello debemos modificar el Configure Security Groups para que nos deje: HTTP, HTTPS y SSH (para acceder luego por aquí a las máquinas que no sean públicas), es decir puertos 80, 443 y 22 respectivamente.
        -	El resto de parámetros no es necesario tocarlos
    -	De la misma manera en S2, S3
        -	Network: “MyVPC”
        -	Subnet: van a ser públicas, pues elegimos las subnet LAN2 y LAN3 respectivamente
        -	Dirección IP: mismo comentario de s1
-	Ahora crearemos varios backend (ya que no van a tener acceso a internet) para alojar nuestra BBDD. Con EC2 de la misma manera:
    -	BBDD 
        -	Network: “MyVPC”
        -	Subnet: va a ser privada, luego elegimos la subnet donde está LAN4
        -	Dirección IP: “20.2.4.31” por ejemplo
        -   NOTA: se podría haber creado con una instancia Database de tipo RDS y haber elegido MySQL
    -	NAS1, NAS2, NAS3 (hacer autoescalado de nas1)
        -   Network: “MyVPC”
        -	Subnet: van a ser privadas, luego elegimos la subnet donde está LAN4
        -   Direcciónes IP: “20.2.4.21”,”20.2.4.22”,”20.2.4.23” por ejemplo
        -   NOTA: se podría haber creado con una instancia Storage.
-   Vamos a pasar a la creación del balanceador de carga, para ello nos dirigimos a la sección que pone Load Balancer agregaremos las subredes que son públicas, en este caso la de los servidores web: LAN1, LAN2 y LAN3. De forma que en availability zone las tendremos en tres lugares distintos, así si surge un desastre en una zona y cae totalmente, aseguramos el servicio. Por defecto el protocolo de este balanceador le asignaremos HTTP puerto 80.

La forma de acceder a los servidores es sencillo: mediante ssh con su IP pública y un keypair, de esa manera una vez conectados podemos instalar los scripts dentro de ellas.

La forma de acceder a los que no tienen IP pública, pues podemos primero acceder a un servidor como hemos comentado y luego hacer otro ssh pero esta vez con la IP privada, de manera que tenemos así una máquina aislada. Una vez conectados podemos empezar la instalación de los scripts dentro de ellas.

Hay que tener mucho cuidado de que se puede acceder entonces desde internet: problema de seguridad. Se soluciona restringiendo el acceso con los Security Groups o network ACL
-	Añadir reglas para denegar el servicio a clientes de accesos indeseados. (Firewall)
    -	En la sección network ACL crearnos uno nuevo que podríamos llamar por ejemplo “public-ACL” de manera que añadimos en el apartado inbound Rules:
        -	Rule 100
        -	Type TCP 
        -	Protocol TCP
        -	Port 80
        -	Source 0.0.0.0/0
        -	Action Allow
        -   NOTA: y como última regla como siempre: al resto de casos * denegar el servicio


