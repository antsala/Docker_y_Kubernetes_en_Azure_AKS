# Laboratorio 80: ***Troubleshooting en AKS.***
 

Requisitos:

1. Una máquina virtual con Ubuntu 20.04 LTS a la que poder hacer ssh o escritorio remoto.
2. Cluster AKS iniciado.

Hay dos áreas donde las cosas pueden ir mal. O bien hay problemas en el cluster, o la aplicación tiene problemas.

Un nodo del cluster puede caerse. Se debe a un error en la infraestructura de Azure o un problema con la VM del nodo. En cualquier caso, K8s monitoriza el cluster buscando fallos en los nodos y los recuperará automáticamente.

Un segundo problema típico son los fallos debidos a que no hay recursos disponibles.

Un tercero está relacionado con el montaje del almacenamiento, que ocurre cuando un nodo tiene problemas. K8s NO desconectará los discos que tiene asignado el nodo con problemas. Esto significa que esos discos no se pueden usar en otro nodo por los pods que lo necesiten.


## Ejercicio 1. Problemas cuando cae un nodo del cluster.

Cambiamos al directorio de trabajo:

```
cd ~/Docker_y_Kubernetes_en_Azure_AKS/80
```

Nos aseguramos que el cluster tiene al menos dos nodos.

```
kubectl get nodes
```

Entramos en directorio de trabajo.

```
cd ./Problemas_en_el_cluster
```

Volvemos a lanzar la app.

```
kubectl create -f guestbook-all-in-one.yaml
```

Esperamos hasta que la app se haya desplegado.

```
kubectl get all
```

Copiar la IP Externa.

```
IP_EXTERNA=<Poner aquí la IP Externa del balanceador>
```

Comprobamos en qué nodos se están ejecutando los pods.

```
kubectl get pods -o wide
```

Creamos un script.

```
echo "while true; do"                   > script.sh
echo "   curl -m 1 http://$IP_EXTERNA;" >> script.sh
echo "   sleep 5;"                      >> script.sh
echo "   clear;"                        >> script.sh
echo "done"                             >> script.sh
```

Lo hacemos ejecutable.

```
chmod +x script.sh
```

Lo ejecutamos.

```
./script.sh
```

El script hace una request cada 5 segundos. Borra la pantalla para ver la response. Mientras k8s rebalancea el cluster (cuando hay algun error de nodo) puede ser normal orservar alguna latencia en la response.

Vamos a simular el fallo de un nodo. Para ello procederemos a apagarlo. Apagamos el nodo que tenga más pods. 

En otra terminal, con el siguiente comando podemos ver el nodo con más pods.

```
kubectl get pods -o wide
```

Tomamos en nombre del grupo de recursos donde reside el scale set del cluster.

```
vmssRG=$(az aks show \
            --name myaks \
            --resource-group myaks-rg \
            --query nodeResourceGroup \
            --output tsv)
```

Comprobamos

```
echo $vmssRG
```

Listamos los nodos del cluster y nos quedamos con el nombre del que más pods contiene.

```
kubectl get nodes
```

Supongamos que el nodo 'aks-nodepool1-XXXXXXXX-vmss000001' es quien más pods contiene. 'aks-nodepool1-XXXXXXXX-vmss' es el nombre del vmss. '000001', es el id de la instancia.

```
VMMS_NAME=<Poner aquí el 'nombre del vmms'>
```

```
INSTANCE_ID=<Poner aquí el 'id de la instancia'>
```

Comprobamos.

```
echo $VMMS_NAME
echo $INSTANCE_ID
```

Paramos la instancia (nodo) que tiene más pods.

```
az vmss stop \
    --resource-group $vmssRG  \
    --name $VMMS_NAME \
    --instance-id $INSTANCE_ID
```

Observar, en la otra terminal, como 'curl' no obtiene respuesta mientras se rebalancea el cluster. Comprobar como el estado del nodo es 'NotReady'.

```
kubectl get nodes
```

La app puede seguir funcionando porque tiene alta disponibilidad en sus microservicios, con una excepción importante, redis-master. Este pod NO usa PVC, por lo tanto si su nodo cae, será iniciado en otro, perdiendose su base de datos que reside en la capa reescribible del contenedor.

Esto demuestra la necesidad de almacenar el estado en un PVC.

El redespliegue de los pods en el único nodo vivo puede tardar unos minutos (5 al menos). Esperar y volver a comprobar con:

```
kubectl get pods -o wide -w
```


## Ejercicio 2:  Problemas cuando no hay recursos.


Cuando un cluster no tiene suficiente CPU o memoria para planificar los pods, éstos se quedan en un estado pendiente. K8s usa 'requests' para calcular cuánta CPU o memoria necesita un pod concreto.

En el archivo 'guestbook-all-in-one.yaml' aparece...

```
      63 kind: Deployment
      64 metadata:
      65 name: redis-replica
      ...
      83 resources:
      84 requests:
      85 cpu: 200m
      86 memory: 100Mi
```

Cada pod de 'redis-replica' requiere 200 milésimas de un core (20% de un core) y 100 MiB de RAM.

El el cluster de 2 nodos (con un nodo apagado en este momento), escalar a 10 pods causará problemas de recursos de CPU.

```
kubectl scale deployment/redis-replica --replicas=10
```

Algunos pods se quedarán en estado 'pending' y no se le asigna nodo.

```
kubectl get pods -o wide
```

Podemos ver más detalle con el siguiente comando. 

```
PENDING_POD=<Poner el nombre de un pod que esté en estado 'pending'>
```

Veremos la causa de por qué se queda en estado pendiente.

```
kubectl describe pod $PENDING_POD
```

Volvemos a levantar el nodo que detuvimos.

```
az vmss start \
    --resource-group $vmssRG  \
    --name $VMMS_NAME \
    --instance-id $INSTANCE_ID
```

Comprobamos que los dos nodos están corriendo.

```
kubectl get nodes -w
```

Vemos si se están replanificandos los pods pendientes.

```
kubectl get pods -o wide -w
```

Si volvemos a describir el pod que no se iniciaba, comprobaremos que ha sido planificado.

```
kubectl describe pod $PENDING_POD
```

Limpiamos recursos. Volver a la consola principal y detener con CTRL+C.

```
kubectl delete -f guestbook-all-in-one.yaml
```

Comprobamos.

```
kubectl get all
kubectl get pv
kubectl get pvc
```



# Ejercicio 3: Arreglando problemas de montaje de almacenamiento.


Vamos a ver un ejemplo de cómo se puede usar un PVC para prevenir la pérdida de datos cuando k8s mueve un pod a otro nodo. Para ello reutilizamos la app WordPress.

```
helm install wp bitnami/wordpress
```

Comprobamos que se han desplegando los objetos.

```
kubectl get all
```

Vamos a ver los PVCs que se han creado.

```
kubectl get pvc
```

Un 'PersistentVolumeClaim' creara un 'PersistentVolume' o PV. El PV es el enlace al recurso físico creado, que en Azure es un disco. El siguiente comando muestra los PVs.

```
kubectl get pv
```

Podemos tener más información de un PV con el siguiente comando. 

```
kubectl describe pv <Poner aquí el nombre de un PV>
```

Vamos a entrar como administrador en la aplicación WordPress. Para ello # necesitamos el password del administrador. Esto y se vio en un módulo previo. El comando siguiente muestra como obtener este password.
Concretamente en el apartado 3.

```
helm status wp
```

```
SERVICE_IP=$(kubectl get svc wp-wordpress \
                --namespace default \
                --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")

PASSWORD=$(kubectl get secret wp-wordpress \
            --namespace default \
            --output jsonpath="{.data.wordpress-password}" \
            | base64 --decode)

echo "WordPress URL: http://$SERVICE_IP/"
echo "WordPress Admin URL: http://$SERVICE_IP/admin"
echo Username: user
echo Password: $PASSWORD
```

Conectamos con el navegador a la URL de admin y nos autenticamos.Escribimos un post en "Write your first blog post", y luego publicamos. La idea es que la base de datos almacene info en el PVC.

Lo primero a probar con los PVCs es eliminar los pods y verificar si los datos persisten.

```
kubectl get pods -w
```

Ahora eliminamos los dos pods que tienen PVCs montados. Esto lo hacemos en otra terminal para comprobar en la primera que los pods pasan a estado 'terminating' y luego se crean de nuevo. Tarda unos minutos (4 ó 5)

```
kubectl delete pod --all
```

Probamos con en navegador que el post sigue estando.

```
echo $SERVICE_IP
```


# Ejercicio 4. Administrar el fallo de un nodo cuando hay PVCs


Comprobar en qué nodos están corriendo los pods. 
```
kubectl get pods -o wide
```

En este ejemplo MariaDB corre en el nodo 1 y WordPress en el nodo 0. Vamos a provocar un fallo en el nodo donde está corriendo WordPress.

Tomamos en nombre del grupo de recursos donde reside el scale set del cluster.

```
export vmssRG=$(az aks show \
                    --name myaks \
                    --resource-group myaks-rg \
                    --query nodeResourceGroup -o tsv)
echo $vmssRG
```

Mostramos los pods y los nodos en los que corren.

```
kubectl get pods -o wide
```

Supongamos que el nodo 'aks-nodepool1-XXXXXXXX-vmss000001' es donde corre 'mariaDB' y 'aks-nodepool1-XXXXXXXX-vmss' es el nombre del vmss. '000001', es el id de la instancia.

```
VMMS_NAME=<Poner aquí el 'nombre del vmms'>
```

```
INSTANCE_ID=<Poner aquí el 'id de la instancia'>
```

```
MARIADB_POD=<Ponr aquí el nombre del pod de 'mariaDB'>
```

Comprobamos.

```
echo $VMMS_NAME
echo $INSTANCE_ID
```

Paramos el nodo donde corre el pod de 'mariaDB'.

```
az vmss stop \
    --resource-group $vmssRG  \
    --name $VMMS_NAME \
    --instance-id $INSTANCE_ID
```

Comprobamos lo que hace Kubernetes. Pueden pasar al menos 5 minutos antes que K8s determine qué hacer con el nodo caído.

```
kubectl get pods -o wide -w
```

NOTA, si tarda mucho 'ayudamos' a la eliminación.

```
kubectl delete pod $MARIADB_POD --grace-period 0 --force
```

El nuevo pod se quedará bloqueado en el estado 'ContainerCreating'. 
Veamos que ha pasado.

```
kubectl describe pod $MARIADB_POD
```

El estado nos dice que hay un problema con el volumen. Veremos 2 errores relacionados con el volumen:

El error 'FailedAttachVolume' nos dice que el volumen está siendo usado por otro nodo. 
El error 'FailedMount' dice que el pod no puede montar el volumen.

Azure liberará el volumen para que pueda ser montado en otro nodo de forma automática, pero llevará bastante tiempo. Comprobamos el estado de los pods durante unos minutos (15 minutos) a que se levante el pod y monte el volumen.

```
kubectl get pods -w
```

Una vez que lo haya levantado, podemos comprobar desde el navegador que todo vuelve a funcionar.

Limpiamos recursos.
```
helm delete wp
kubectl delete pvc --all
kubectl delete pv --all
```

Nota, los discos de Azure no se borran al eliminar el PVC.

Levantamos el nodo caído.

```
az vmss start \
    --resource-group $vmssRG  \
    --name $VMMS_NAME \
    --instance-id $INSTANCE_ID
```

Comprobamos que se levante.

```
kubectl get nodes
```

