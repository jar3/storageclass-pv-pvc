# storageclass-pv-pvc
### ¿Qué es una Storage Class en Kubernetes?
Una Storage Class es como un plano que define cómo se debe crear y manejar el almacenamiento para tus aplicaciones en Kubernetes. Esto permite que el almacenamiento se cree automáticamente (de forma dinámica) cuando una aplicación lo necesita.

Por ejemplo:
Si tienes una aplicación que necesita almacenamiento rápido (como un SSD), puedes usar una Storage Class para decirle a Kubernetes:

"Crea discos SSD de 10 GB cuando la aplicación lo pida".
Esto evita que tengas que configurar manualmente los discos cada vez que los necesites.

# Componentes clave de una Storage Class:
### Provisioner:
Es el controlador que realmente crea el volumen de almacenamiento.
Ejemplos:

AWS EBS (Elastic Block Store) → kubernetes.io/aws-ebs
Google Persistent Disk → kubernetes.io/gce-pd
Azure Disk → kubernetes.io/azure-disk

### Parameters (Parámetros):
Son configuraciones específicas del almacenamiento, como:

### Tipo de disco (SSD o HDD).
Zonas donde se creará el volumen.
Ejemplo: En AWS puedes especificar type: gp2 para discos SSD generales.

### Reclaim Policy:
Define qué pasa con el volumen cuando ya no se usa:

* Retain: Conserva el volumen para limpieza manual.
* Delete: Borra automáticamente el volumen.
* Recycle: Limpia los datos y lo deja disponible para otro uso.

### Volume Binding Mode:
Define cuándo se asigna un volumen al Persistent Volume Claim (PVC):

* Immediate: El volumen se crea inmediatamente cuando el PVC lo solicita.
* WaitForFirstConsumer: Espera a que un pod use el PVC para crear el volumen, asegurándose de que esté en la misma zona que el pod.

Ejemplo de una Storage Class en AWS:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mi-storage-class
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zones: "us-east-1a, us-east-1b"
reclaimPolicy: Retain
volumeBindingMode: Immediate
```
Explicación: 

* provisioner: Usa AWS EBS para crear el almacenamiento.
* parameters: Crea discos SSD (gp2) en las zonas us-east-1a y us-east-1b.
* reclaimPolicy: El volumen no se borra automáticamente; tendrás que eliminarlo manualmente si ya no lo necesitas.
* volumeBindingMode: El volumen se asigna tan pronto como el PVC lo solicita.

### Cómo usar una Storage Class:
Para usar una Storage Class, necesitas un Persistent Volume Claim (PVC), que es como una solicitud de almacenamiento.

Ejemplo de PVC:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: mi-storage-class
```

Explicación:

* accessModes: Define que el volumen puede ser montado como lectura/escritura por un solo nodo.
* resources.requests.storage: Pide 10 GB de almacenamiento.
* storageClassName: Indica que se usará la Storage Class llamada mi-storage-class.

Cuando aplicas este PVC, Kubernetes automáticamente crea un volumen de 10 GB usando los parámetros definidos en la Storage Class.

Escenario práctico:
Supongamos que tienes una aplicación de e-commerce que almacena imágenes de productos y necesitas que estas imágenes se almacenen en un disco SSD para que sean rápidas de acceder.

Pasos:
Creas una Storage Class con discos SSD (como el ejemplo de AWS).
Tu aplicación solicita almacenamiento usando un PVC (como en el ejemplo).
Kubernetes automáticamente crea un volumen SSD y lo asigna a tu aplicación.

### Otros conceptos importantes:
* Dynamic Provisioning: Kubernetes crea volúmenes automáticamente cuando un PVC lo solicita. Esto es útil para aplicaciones que necesitan almacenamiento dinámico.
* Reclaim Policy: Define si el volumen se borra automáticamente (Delete), se conserva para limpieza manual (Retain) o se limpia y reutiliza (Recycle).
* Volume Binding Mode: Si usas WaitForFirstConsumer, el volumen se crea en la misma zona que el pod que lo va a usar, lo cual es útil en clústeres multizona.

### MINIKUBE (si llegaste hasta aca te haces un fork de este repo habilitas un codespace y probas todo en un entorno de pruebas)
Minikube tiene una Storage Class llamada standard que usa el provisioner k8s.io/minikube-hostpath (crea almacenamiento local en el nodo Minikube).

```
/workspaces/storageclass-pv-pvc (main) $ k get storageclass.storage.k8s.io
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  29m
```

PVC (Usando la Storage Class por defecto):
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard
```

Qué hace:
Solicita 2 GB de almacenamiento con acceso de lectura/escritura por un nodo.
Usa la Storage Class standard que viene con Minikube.

Cómo probarlo:
Aplica el archivo con kubectl apply -f nombre-del-archivo.yaml.
Verifica que se haya creado con:
```
kubectl get pvc
kubectl get pv
```

Ejemplo 2: Crear tu propia Storage Class, PV y PVC
Puedes crear un flujo personalizado de Storage Class, Persistent Volume y Persistent Volume Claim.
Storage Class personalizada:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: k8s.io/minikube-hostpath
parameters:
  type: hostPath
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

Qué hace:
Define una Storage Class llamada local-storage que usa el almacenamiento local en Minikube.
Los volúmenes no se eliminan automáticamente (Retain).

Persistent Volume (PV):
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data
```

Qué hace:
Define un volumen persistente de 5 GB con almacenamiento local en el nodo Minikube.
Usa la ruta /mnt/data en el host como punto de almacenamiento.
Persistent Volume Claim (PVC):
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: local-storage
```

Qué hace:
Solicita 2 GB del volumen definido por la Storage Class local-storage.
Probar el ejemplo completo
Aplica los tres archivos en orden:
```
kubectl apply -f storage-class.yaml
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml
```

Verifica que el PVC esté enlazado al PV:
```
kubectl get pv
kubectl get pvc
```
Usa el PVC en un Pod para montarlo como volumen:
Pod usando el PVC:
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
    volumeMounts:
    - mountPath: "/data"
      name: test-volume
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: local-pvc
```
Qué hace:
Crea un Pod con un contenedor que usa el PVC local-pvc para montar un volumen en la ruta /data.
Cómo probarlo:
Aplica el Pod:
```
kubectl apply -f pod.yaml
```

Accede al contenedor para verificar el volumen:
```
kubectl exec -it test-pod -- sh
ls /data
```

