# Cluster storage

## Montar disco duro externo

Vamos a montar el disco duro externo en el nodo master

```
sudo fdisk -l (nos devuelve el listado de discos y nos quedamos con el que sea, por ejemplo /dev/sda)

sudo mkdir /media
sudo chown -R ubuntu:ubuntu /media
sudo mount /dev/sda1 /media
```

Para que el disco se monte siempre que encedamos la pi tenemos que ponerlo en `fstab`. Para ello buscamos el id y lo añadimos al fichero

```
sudo blkid
/dev/sda: UUID="c7fd3f1f-8a19-4785-b303-bd7fb837c8bc" TYPE="ext4"

sudo nano /etc/fstab
UUID=c7fd3f1f-8a19-4785-b303-bd7fb837c8bc /media ext4 defaults 0 0
```

## Compatir por NFS

#### Compatir por NFS (servidor)

Primero instalamos el software necesario

```
sudo apt-get install nfs-kernel-server -y
```

Lo siguiente es configurar el server

```
sudo nano /etc/exports

-----
/media *(rw,no_root_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)
```

Finalmente iniciamos el servidor

```
sudo exportfs -ra
```

Reiniciamos

#### Compatir por NFS (cliente)

Primero instalamos el software necesario

```
sudo apt-get install nfs-common -y
```

Creamos el directorio donde vamos a montar el NFS

```
sudo mkdir /media
sudo chown -R ubuntu:ubuntu /media
```

Montar el NFS share

```
sudo nano /etc/fstab
-------------
192.168.0.22:/media   /media   nfs    rw  0  0
```

Reiniciamos

#### Configurar  local storage k3s

Para configurar la ruta del local storage debemos de modificar el config del servicio de `local-path`. El config map es `local-path-config` y podemos modificarlo así:

```
apiVersion: v1
data:
  config.json: |-
    {
            "nodePathMap":[
            {
                    "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                    "paths":["/media"]
            }
            ]
    }
```
