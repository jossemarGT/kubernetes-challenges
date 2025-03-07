---
title: Kubernetes Challenges Parte 1
author: Kubernetes Guatemala
date: 2023-03-17
extensions:
- file_loader
- terminal

---
## ***Quien es Marvin Velasques?  ***

  Self thought engineer trabajando para PayPal

  Apasionado de la tecnologia y lo que tenga que ver con automatizacion

󱃾  CKA

   󱈏 

               @PuesSiVos | 󰌻 @mvlsqz |  @mvlsqz 

---
## De que se trata esta charla
Una serie de ejercicios para desenpolvar nuestras habilidades
con Kubernetes

Descubrir cosas que tal vez no recordabamos o no sabiamos

Y para los nuevos una demo de como funciona Kubernetes


---
## Disclaimer 
La idea es original de la plataforma KodeKloud


---
## Prerequisitos para el reto 1
### Configurar autentication de usuario
```bash
# revisamos el estado de nuestro cluster
kubectl cluster-info
```

```bash
mkdir csr/
```
```bash
openssl genrsa -out $(pwd)/martin.key 4096
```

---
### Archivo de configuracion para el CSR

creamos el archivo `csr/martin.csr.cnf` con el siguiente contenido
```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256 # algoritmo de encriptacion
distinguished_name = dn
[ dn ]
CN = martin
O = developers
[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```
---
### Creamos el Certificate Signing Request
```bash
openssl req -config csr/martin.csr.conf -new -key martin.key -nodes -out csr/martin.csr
```
#### ... Y verificamos

```bash
openssl req -in csr/martin.csr
```
---
### CSR para Kubernetes
Creamos el archivo `martin-csr.yaml` con el siguiente contenido

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: martin-cluster-authentication
spec:
  groups:
    - system:authenticated
  request: $(cat $(pwd)/csr/martin.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

... Y aplicamos el manifiesto
```bash
kubectl apply -f martin-csr.yaml
```

---

Revisamos que fue lo que se creo en cluster
```bash
kubectl get csr
```
Procedemos a firmar el Certificate Signing Request

```bash
kubectl certificate approve martin-cluster-authentication
```
... Y descargamos el certificado

```bash
kubectl get csr martin-cluster-authentication -o jsonpath='{.status.certificate}' | base64 --decode > martin.pem
```

Verificamos el contenido del certificado
```bash
openssl x509 -in martin.pem -noout -text
```
En nuestro workspace tenemos que encontrar los siguientes archivos
```bash
key.crt martin.crt
```

Documentacion: **[managing-tls-in-a-cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)** 

---
## Reto 1 | Configurando RBAC para que un desarrollador pueda desplegar una aplicacion con persistencia

<!-- stop -->

- Developer
  * Configurar user data para el usuario `martin` en kubeconfig
  * Configurar un nuevo contexto `developer` para el usuario `martin`
  * Utilizar el contexto creado `developer`

<!-- stop -->

- Permisos
  - Crear `developer role`
  - Crear `developer role binding`

<!-- stop -->

- Aplicacion Jekyll
  * Revisar el Persistent Volume
  * Crear un Persistent Volume Claim que use el Persistent Volume anterior
  * Crear el Pod
    - Usar el Volume Claim
    - Configurar Jekyll
  * Exponer el Pod via Service

---
## Configurar user data para el usuario `martin` en kubeconfig

<!-- stop -->

```bash
kubectl config set-credentials martin \
--client-certificate=/root/martin.crt \
--client-key=/root/martin.key \
--embed-certs=true
```

<!-- stop -->
`--client-certificate` Indica la ruta del certificado del cliente que fue firmado por el cluster
`--client-key` Indica la ruta de la llave del certificado del cliente

<!-- stop -->

Documentacion y ejemplos
[Kubernetes RBAC with examples](https://sysdig.com/learn-cloud-native/kubernetes-security/kubernetes-rbac/)

*Terminos que puedo buscar

* Kubernetes User Accounts
* RBAC
* SSL Basics

---
## Configurar un nuevo contexto `developer` con el usuario `martin` y el cluster

<!-- stop -->

```bash
kubectl config set-context developer \
--cluster=kubernetes \
--user=martin \
--namespace=development
```

---
## Utilizar el contexto `developer`

<!-- stop -->

```bash
kubectl config-context development
```

Documentacion: [organize-cluster-access-kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

---
## Permisos

<!-- stop -->

### Crear `developer role`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
  - apiGroups
    - ""
    resources:
    - persistentvolumeclaims
    - services
    - pods
    verbs:
    - "*"
```
Documentacion: [rbac-good-practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
---
### Crear `developer role binding`

<!-- stop -->

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: development
subjects:
- kind: User
  name: martin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

Documentacion: [rbac](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

---
##  Aplicacion Jekyll

<!-- stop -->
## Creamos un Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
name: jekyll-site
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  local:
    path: /mnt/jekyll-site
  nodeAffinity:
    required:
    nodeSelectorTerms:
    - matchExpressions:
    - key: kubernetes.io/hostname
    operator: In
    values:
    - minikube
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
```

Revisar el Persistent Volume
<!-- stop -->

```bash
kubectl get persistentvolumes -n development
```

---
### Crear un Persistent Volume Claim que use el Persistent Volume anterior

<!-- stop -->

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jekyll-site
  namespace: development
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: local-storage
  resources:
    requests:
      storage: 1Gi
```
Documentacion: [persistent-volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 

---
### Crear el Pod

#### El Pod

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  namespace: development
  name: jekyll
  labels:
    run: jekyll
spec:
  containers:
    - name: jekyll
      image: kodekloud/jekyll-serve
      volumeMounts:
        - mountPath: /site
          name: site
```
---
#### Usar el Volume Claim

<!-- stop -->

```yaml
  volumes:
    - name: site
      persistentVolumeClaim:
        claimName: jekyll-site
```

---
#### Configurar Jekyll

<!-- stop -->

```yaml
  initContainers:
    - name: copy-jekyll-site
      image: kodecloud/jekill
      command: [ "jekyll", "new", "/site"]
      volumeMounts:
      - mountPath: /site
        name: site
```
Documentacion: [pods](https://kubernetes.io/docs/concepts/workloads/pods/)

---

### Exponer el Pod via un Service

<!-- stop -->


```yaml
---
kind: Service
apiVersion: v1
metadata:
  namespace: development
  name: jekyll
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 4000
      nodePort: 30097
  selector:
    run: jekyll
```
Documentacion: [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
