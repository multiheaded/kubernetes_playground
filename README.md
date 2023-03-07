# Setup minio server on microk8s

Quite fun, because Kubernetes provides an abstraction to the underlying operating system and 
services like DNS that have become quite familiar. So, let's go and relearn some basic principles :)

## Contents
1. [Installing microk8s](#installing-microk8s)  
   1.1 [Install Ubuntu Server](#11-install-ubuntu-server)  
   1.2 [Install the microk8s distribution](#12-install-the-microk8s-distribution)  
2. [Configuration using YAML files](#configuration-using-yaml-files)  
   2.1 [Local disk storage](#21-local-disk-storage)  
   2.2 [Deployments](#22-deployments)  
   2.3 [Ingress](#23-ingress)  
3. [Useful Tipps](#3-useful-tipps)  
   3.1 [Forward a TCP port](#31-forward-a-tcp-port)

## 1. Installing microk8s
Very straightforward and basically an automatic process following the instructions here:
https://microk8s.io/docs/install-raspberry-pi

### 1.1 Install Ubuntu Server
Install Ubuntu Server 22.04 LTS on a SD Card and jam it into the Raspberry Pi.
Enable SSH, WLAN, Passwords as required. The problem is that the ethernet port might not be configured by default. Login to the pi and open `/etc/netplan/50-cloud-init.yaml`. This is a .yaml file that should contain network entries for wifis __and__ ethernets. If eth0 is missing, add it like so:
```yaml
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    wifis:
        renderer: networkd
        wlan0:
            access-points:
                <yourwlanssid>:
                    password: <yourwlanpassword>
            dhcp4: true
            optional: true
```

### 1.2 Install the microk8s distribution
As outlined in the linked documentation entry:
- adapt `/boot/firmware/cmdline.txt` to support cgroups (`cgroup_enable=memory cgroup_memory=1`)
- sudo snap install microk8s --classic --channel=1.25

The channel is kind of the kubernetes version used by microk8s. This might change in the future, so double check before doing anything.

The installation process takes a while and prints out the status in the end (available versions, etc.).

## 2. Configuration using YAML files
Microk8s ships the tools with a name prefix `microk8s.<toolname>`. This is tedious to type all the time, so make a bash alias first:
```bash
alias kubectl=microk8s.kubectl
```

In the following sections configurations will be _applied_ to the Kubernetes cluster using the following command:
```bash
kubectl apply -f <someconfig.yaml>
```

Changing something can be done by patching:
```bash
kubectl patch <what> --patch-file <someyamlpath.yaml>
```
e.g. to add a namespace to a resource because it was forgotten before:
```yaml
metadata:
  namespace: objstorage
```

### 2.1 Local disk storage
In case of a single-device deployment with some locally attached storage like a USB mass storage device (SSD), some additional configuration is necessary according to https://lapee79.github.io/en/article/use-a-local-disk-by-local-volume-static-provisioner-in-kubernetes/

Kubernetes defines storage classes that can be utilized while configuring persistent volumes. 
#### 2.1.1 StorageClass for local storage
Describe a new storage class `local-storage` as outlined in the Kubernetes documentation (the blog linked above does the same, I think for all further steps as well, don't overthink it): https://kubernetes.io/docs/concepts/storage/storage-classes/#local

Going this way is kind of cool, because it waits for the local storage location to be available before deploying containers, services and stuff. If, for example, the data is supposed to reside on an encrypted disk, boot up your node, decrypt the disk and mount it (e.g. with ZFS). Afterwards, Kubernetes will deploy automatically as soon as the storage resource is available.

`localStorageClass.yaml`:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

TODO: What are provisioners?

TODO: What does WaitForFirstConsumer mean as volumeBindingMode?

TODO: Don't know yet!

Apply the new StorageClass as instructed by the official documentation:
```bash
kubectl apply -f localStorageClass.yaml
```

#### 2.1.2 Prepare the local storage location
Attach your USB mass storage and mount it somewhere. Assume the path to be `/usr/local/objstorage/bucket`.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: objstorage-pv
  namespace: objstorage
spec:
  capacity:
    storage: 500Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /usr/local/objstorage/bucket
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - faaspi
```

First contact with namespaces here. You can define distinct namespaces in Kubernetes containing resources that are separated from resources in other namespaces. To `kubectl get`, `describe`, or `edit` a resource, it is necessary to explicitly outline the target namespace with the `-n <namespace>` option.

`nodeAffinity` will make sure that this type of persistent storage is only available on this one node. Deployment will be carried out accordingly.

`storageClassName` defines the storage class specified above.

TODO: What does `persistentVolumeReclaimPolicy: Retain` mean?

#### 2.1.3 Prepare a persistent volume claim for a pod
A container might require persistent storage. This storage is claimed with a PersistentVolumeClaim of a specified size of a specified storage class.
Here:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: objstorage-claim
  namespace: objstorage
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 500Gi
```

TODO: Can a claim be of unlimited size?

### 2.2 Deployments
According to the documentation, a pod is a single container or a set of containers. This is a service or a "logical" server running various applications that are managed by Kubernetes as a single entity: the pod.

After the deployment has been created, Kubernetes will try to schedule resources and deploy whatever is defined within the configuration.

Let's deploy a minio container to provide object storage for other services.

#### 2.2.1 Shared secrets
Accessing the object storage is supposed to be secure! Therefor, secrets must be declared.

Minio needs at least two secrets: an access key and a secret key.
This is like a username/password combination but for API calls. Anyways, both keys should be stored in a file (preferrably on tmpfs and short-lived!!): `/tmp/MINIO_ROOT_PASSWORD` and `/tmp/MINIO_ROOT_USER`

Create the secret in Kubernetes by loading the contents from these files:
```bash
kubectl create secret generic -n objstorage objstorage-secret --from-file=/tmp/MINIO_ROOT_USER --from-file=/tmp/MINIO_ROOT_PASSWORD
```

This resource can now be accessed in the `objstorage` namespace with the `objstorage-secret` name.

#### 2.2.2 Deploy
Not the label `app: objstorage` in the metadata section. This is important later to identify the pod when defining the service and exposing the service via Ingress.
Besides that, this configuration is quite straight forward. The `spec` section contains a selector for the template. Then again, the template contains a spec comprising containers and volumes. 

Deploy the latest minio container and run it with the specified args. Storage is found at the mountpoint `/storage` and the runtime environment contains the access and secret keys from the secret created earlier.

The volumes section will claim persistent storage according to the claim created earlier and provide a volume named `storage`. This `storage` volume is mounted as `/storage`in the container.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: objstorage-deployment
  namespace: objstorage
  labels:
    app: objstorage
spec:
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio
        args:
        - server
        - --console-address
        - ":9090"
        - /storage
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: objstorage-secret
              key: MINIO_ROOT_USER
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: objstorage-secret
              key: MINIO_ROOT_PASSWORD
        ports:
        - name: http
          containerPort: 9090
        volumeMounts:
        - name: storage
          mountPath: "/storage"
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: objstorage-claim
```

~~We could now add additional containers with additional minio instances, however, the goal is to define the deployment once and spin it up multiple times as needed.~~
This was bollocks!

According to the Kubernetes documentation, a deployment should describe the desired state and the controller changes the system to achieve that state. This means, the deployment should be adapted to add additional containers for additional minio instances.

### 2.3. Ingress
Sounds like something from fantasy literature, but describes how cloud resource can be accessed from the public internet. Sounds funny, because one would assume that everything is public inside the cloud. Anyways, Kubernetes defines how the cloud works and these inner workings should be mostly private. Only selected services should be exposed to the public to reduce attack vectors.

#### 2.3.1 Install nginx-ingress
This requires an addition feature that must be installed via the mikro8s distribution --- nginx-ingress.
```bash
microk8s enable dns ingress
```

TODO: Some blog suggested that dns is needed in addition to ingress. I have not verified that but at least I think that we will need dns in the near future anyways, so just go with it. (https://marcolenzo.eu/how-to-install-nginx-ingress-controller-on-kubernetes-and-microk8s/)

This will take some time, but will basically pull additional software from the internet.

Next, it is desireable to instruct nginx to pass the `X-Forwarded-*` headers. It appears some applications need that, such as OIDC (https://www.mattgerega.com/2022/03/05/tech-tip-turn-on-forwarded-headers-in-nginx/).
To achieve this, edit the configmaps and add the data section as outlined below:
```bash
kubectl -n ingress edit configmaps nginx-load-balancer-microk8s-conf
```
```yaml
apiVersion: v1
data:
  use-forwarded-headers: "true"
kind: ConfigMap
# ...
```
To confirm that the changes were indeed applied, pull up the Ingress logs and check them for `reload` messages.

```bash
kubectl -n ingress logs nginx-ingress-microk8s-controller-n8cd5 | grep reload
```
This command (without the piped grep) is also useful to check whether Ingress forwards requests to services.

#### 2.3.2 Setup a service
The deployment is there, the Ingress tool is there. All that is missing is a service to expose.
Minio provides access to the API (to PUT, GET, ... objects) on port 9000 and the admin console on port 9090. For now, let's expose the admin console as hostnames do not yet work an only a single ip is available. This needs to be described in a formal way to allow Kubernetes to act accordingly. It is very important to configure the right selector. It selects depending on a label `app` set to the value `objectstorage`. This is __not__ the name, it is what has been specified in the deployment metadata below labels, `app: objstorage`!!
If this is wrongly configured, Kubernetes __will not__ automatically create the endpoints and Ingress will complain about it. Additionally, the ports won't be exposed, because the selector won't assign the service to the deployed pod.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: objstorage-service
  namespace: objstorage
spec:
  ports:
  - port: 9090
    targetPort: 9090
    protocol: TCP
    name: console
  - port: 9000
    targetPort: 9000
    protocol: TCP
    name: api
  selector:
    app: objstorage
```

#### 2.3.3 
Here is why we cannot yet expose both, the minio API endpoint and console endpoint. So far we do not have any notion about hostnames. Only IP addresses. The following ingress configuration indicates that we can map paths to specific services. This will configure nginx to proxy traffix from the server/hostname to the backend IP and port. Minio does not support paths (e.g. `http://$IP/api` and `http://$IP/console`). Instead it expects these endpoints to be reachable on  root `/`. Therefor, complying with the previous example, two FQDNs are required `http://api.host/` and `http://console.host/`, both pointing to the same IP. HTTP request will be parsed by nginx and forwarded appropriately to the backend based on the domain name (virtual servers in Apache speech).
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: objstorage-ingress
  namespace: objstorage
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: objstorage-service
            port:
              number: 9090
```

## 3 Useful Tipps
### 3.1 Forward a TCP port
This command will block until killed.
```bash
kubectl port-forward -n storage objstorage-deployment-xxxxxxxxxx-xxxxx :9090
```
