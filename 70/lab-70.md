# Laboratorio 70: ***Azure Container Registry (ACR)***
 
En este laboratorio aprenderemos a conectar pods de Kubernetes con servicios que corren ***fuera del cluster*** de Kubernetes.

Requisitos:

1. Una máquina virtual con Ubuntu 20.04 LTS a la que poder hacer ssh o escritorio remoto.
2. Cluster AKS iniciado.


## Ejercicio 1. Creación de un ACR (Azure Container Registry)

Cambiamos al directorio de trabajo:

```
cd ~/Docker_y_Kubernetes_en_Azure_AKS/70
```

Descargamos una imagen para probar (Si se tiene Docker usar docker pull)
```
podman pull docker/whalesay
```


Creo grupo de recursos para el ACR

```
az group create \
    --resource-group myACR-rg \
    --location westeurope
```


Creamos el ACR. Pon una fecha.

```
ACR_NAME=myacrasg<yyyymmdd>
```
```
az acr create \
    --resource-group myACR-rg \
    --name $ACR_NAME \
    --sku Basic
```
```
ACR_LOGIN_SERVER=$(az acr show \
                    --name $ACR_NAME \
                    --query loginServer \
                    --output tsv)
```

Comprobamos.

```
echo $ACR_LOGIN_SERVER
```

Para usar la instancia de ACR, debemos logarnos. Existen diferentes formas de conseguirlo, alguna hace uso del propio demonio de Docker. Leer la documentación https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication?tabs=azure-cli

El método elegido es el de la exposición del token de autenticación, que es mejor para los scripts.

Obtengo el token para acceder al ACR.
```
ACR_TOKEN=$(az acr login \
            --name $ACR_NAME \
            --expose-token \
            --query accessToken \
            --output tsv )
```

Comprobamos.

```
echo $ACR_TOKEN
```

Nos logamos al ACR con el token obtenido. Nota, si tienes Docker usa 'sudo docker'.

```
podman login \
    $ACR_LOGIN_SERVER \
    --username 00000000-0000-0000-0000-000000000000 \
    --password $ACR_TOKEN
```

Etiquetamos la imagen para poder subirla al ACR. 

```
MI_TAG=$ACR_LOGIN_SERVER/antsala/whalesay
```

Comprobamos.
```
echo $MI_TAG
```

Etiquetamos.
```
podman image tag docker/whalesay $MI_TAG
```

Listamos las imágenes.
```
podman image ls
```

Subimos la imagen.
```
podman image push $MI_TAG
```

Borramos las imagenes locales.
```
podman image prune -a -f
```

Lanzamos contenedor descargando imagen desde ACR.
```
podman run --rm $MI_TAG
```



## Ejercicio 2. Conectar AKS al ACR.


Vamos a conectar el cluster al registry que creamos antes.  Aunque este procedimiento se puede hacer en tiempo de creación del cluster, lo explicamos de forma independiente, para aprender también como desconectamos al cluster del ACR.

Comprobamos el valor de la variable.

```
echo $ACR_NAME
```

Conectamos el cluster al ACR.

```
az aks update \
    --name myaks \
    --resource-group myaks-rg \
    --attach-acr $ACR_NAME
```

Vamos a lanzar un deployment que use imágenes desde el ACR.

```
cd ./AKS_ACR
```

Editamos el archivo 'deployment_smartwhale.yaml'

```
code deployment_smartwhale.yaml
```

Importante !!! En la línea 19: Poner el nombre del ACR que hemos creado antes ($ACR)

Implementamos.

```
kubectl apply -f deployment_smartwhale.yaml
```


¡¡¡¡¡¡IMPORTANTE!!!!!!
La imagen de contenedor tiene una aplicación que al ejecutarse, muestra un mensaje y finaliza su ejecución. Si miramos el deployment, sus pods están detenidos por la razón explicada. El replicaSet hace su trabajo y los vuelve a lanzar. 

Mostramos los objetos.

```
kubectl get all
```

Mostramos el log (salida estándar del pod).

```
kubectl logs <Poner aquí el nombre de uno de los pods>
```

Ya está demostrado cómo conectar AKS con ACR. Ahora limpiamos.

Eliminamos el deployment.

```
kubectl delete -f deployment_smartwhale.yaml
```

Comprobamos que han eliminado los objetos.

```
kubectl get all
```

Desconectamos el cluster del ACR.

```
az aks update \
    --name myaks \
    --resource-group myaks-rg \
    --detach-acr $ACR_NAME
```

Eliminamos el ACR.

```
az group delete \
    --resource-group myACR-rg \
    --yes
```
