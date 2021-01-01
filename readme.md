# Montar cluster de kubernetes en una raspberry (kubeberry)

## ¿Qué necesitamos??

- X raspberry
- X tarjetas de memoria (yo recomiendo como poco 32 Gb)
- X cables de alimentación para las raspberry

## Instalación de sistema operativo

Vamos a usar ubuntu server de 64 bits porque ahora mismo no esta disponible raspbian de 64 bits.
Instalamos la imagen en la tarjeta con "Raspberry pi imagier" y luego desde la tarjeta sd podemos configurar las redes (wifi) y la conexión por ssh 

Para configurar el wifi desde la sd podemos modificar el fichero `network-config` como podemos ver en esta guía
 https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#3-wifi-or-ethernet
 
 ```
 wifis:
  wlan0:
    dhcp4: no
    addresses: [192.168.0.22/24]
    gateway4: 192.168.0.1
    nameservers:
      addresses: [192.168.0.163, 8.8.8.8]
    optional: true
    access-points:
      "<wifi network name>":
        password: "<wifi password>"

```

En principio el sistema anterior funciona sin problemas pero para que vaya todo bien hay que reiniciar la raspberry. En el boot inicial se conecta configura la red pero para que los cambios
hagan efecto hay que reiniciar. Si estas conectado por pantalla puedes hacer un `sudo reboot`. Yo como la tenía sin pantalla simplemente le he quitado la alimentación 
a los 5 minutos aprox. Luego la he vuelto a encender y aunque tarda parece que ha ido todo correctamente

### Activar memory groups 

Esto también se puede hacer desde la SD antes de arrancar.


Para que k3s funcione es necesario activar unos parámetros en el fichero de startup.

Para ver que fichero tenemos que modificar podemos verlo en: `cat /boot/firmware/config.txt`

Vemos la línea: `cmdline=cmdline.txt`

Modificamos el fichero y añadimos los parámetros de memoria.

`boot/firmware/cmdline.txt`

net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc **cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1**

### Cambiar contraseña y actualizar SO

La primera vez que nos conectamos nos va a pedir que cambiemos la contraseña. La contraseña por defecto es "ubuntu" y tras realizar el cambio lo ideal es hacer un `sudo apt-get upgrade` para instalar todas las 
actualizaciones disponibles.

## Instalación software de monitorización

Aunque esto no es obligatorio, vamos a instalar un software de monitorización para ver como van de cargas las distintas raspberrys y poder consultarlo con un panel de grafana.
Para hacer todo esto más fácil vamos a usar **Ansible**, que es un software que nos permite lanzar una serie de comandos que configuremos en todos los dispositivos. 

Yo la verdad que no tengo mucha idea de Ansible así que basicamente he buscado por internet y copia un poco el script con las cosas que me hacen falta. Me he guiado sobre todo 
de esta web: https://www.dinofizzotti.com/blog/2020-04-10-raspberry-pi-cluster-part-1-provisioning-with-ansible-and-temperature-monitoring-using-prometheus-and-grafana/#static-ips


### Instalar ansible

Para guiarme un poco con los primeros pasos he usado esta guía: https://blog.deiser.com/es/primeros-pasos-con-ansible

Basicamente instalamos Ansible, en mi caso lo voy a instalar en ubuntu. Este software los vamos a instalar desde la máquina en la que vamos a ejecutar los comandos. No en las raspberries

```
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```

### Copiar cable ssh a los nodos

Copiamos la cable ssh a las raspberries para que ansible se pueda conectar sin necesidad de contraseña por ssh. Para ello ejecutamos el siguiente comando para cada nodo:

```
ssh-copy-id user@192.168.0.XXX
```

Si no tenemos clave generada, la podemos generar antes con el comando `ssh-keygen`. 

### Creamos el fichero de inventario de Ansinble

Ansible puede usar un fichero de inventario donde listamos todos los nodos que vamos a usar. Este fichero es `inventory.yml` dento de la carpeta ansinble. Tiene el siguiente formato. 

```
---
all:
  hosts:
    main:
      ansible_host: 192.168.0.22
    minion1:
      ansible_host: 192.168.0.33
  children:
    raspberry_pi:
      hosts:
        main: {}
        minion1: {}
    monitoring_server:
      hosts:
        main: {}
  vars:
    ansible_python_interpreter: /usr/bin/python3
    remote_user: ubuntu
```

En este fichero definimos los distintos nodos con sus ips y grupos de hosts.

Finalmente para comprobar si todo se ha inicializado correctamente podemos lanzar el siguiente comando:

```
ansible all -m ping -u ubuntu -i inventory.yml
```

Si todo ha ido bien obtendremos esta respuesta:

```
minion1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
main | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Instalar software monitorización

Para instalar el software en ambdos nodos podemos usar el palybook que está creado en la carpeta ansible. Esta copiado casi todo de la guía del primer enlace añadiendo algunas modificaciones. Basicamente tenemos una tarea llamada `up.yml` que va a intentar consolidar los roles en definidiso en los distintos nodos.

El comando es el siguiente:

```
ansible-playbook up.yml -i inventory.yml
```
