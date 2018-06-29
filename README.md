# Kubernetes 101

! IMPORTANT - All source code is [here]()

Topics:
- Setup minikube
- Deploying your first application
  - Deploy pod
  - Deploy service
  - Scale rc
- Accessing your application
  - [Ingress controller](#ingress-controller)
    - [Ingress Controller is Needed](#ingress-controller-is-needed)
    - [Why Nginx?](#why-nginx)
    - [Defaultbackend](#defaultbackend)
    - [Nginx-Ingress & defaultbackend](#nginx-ingress-and-defaultbackend)
    - [Simple Application](#simple-application)
    - [RC and Service](#rc-and-service)
    - [Multiple Services](#multiple-services)
    - [Configuring Ingress to Handle HTTPS traffic](#configuring-ingress-to-handle-https-traffic)
    - [Bonus 1 - Mapping Different Services to Different Hosts](#bonus-1---mapping-different-services-to-different-hosts)
    - [Bonus 2 - Use Minikube Addon to Enable Nginx Ingress](#bonus-2---use-minikube-addon-to-enable-nginx-ingress)
  - Routes
- Persistent storage
  - Define PV to Kubernetes (NFS)
  - Create PVC and assign it to PODS

## Ingress controller
*Ingress - the action or fact of going in or entering; the capacity or right of the entrance. Synonyms: entry, entrance, access, means of entry, admittance, admission;*

### Ingress Controller is needed
This is unlike other types of controllers, which typically run as part of the kube-controller-manager binary, and which are typically started automatically as part of cluster creation.

You can choose any Ingress controller:

- haproxy
- Nginx
- ...

### Why Nginx?
- Nginx Ingress Controller officially supported by Kubernetes, as we already now.
- it’s totally free :)
- It’s the default option for minikube, which we will use to test Ingress behavior in Kubernetes.

## Defaultbackend
An Ingress with no rules sends all traffic to a single default backend. Traffic is routed to your default backend if none of the Hosts in your Ingress match the Host in the request header, and/or none of the paths match the URL of the request.

### Nginx-Ingress and Defaultbackend
```
---

apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissible as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.4
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: ingress-nginx
  labels:
    app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app: ingress-nginx
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
        - events
    verbs:
        - create
        - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.16.2
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
                drop:
                - ALL
                add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
```

### Simple application
Conceptually, our setup should look like this:
```
    internet
        |
   [ ingress ]
        |
   [ service ]
        |
      [RC]
   --|--|--|--
     [pods]
```
We're going to use a simple NodeJS application for testing. I've pushed image to DockerHub already (evalle/gordon:v1.0) but you should never-ever download and run unknown images from DockerHub, so let's build them instead.

Here is the app:
```
const http = require('http');
const os = require('os');

console.log("Gordon server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("You've hit gordon v1, my name is " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```
If you're not familiar with NodeJS, this app basically answers with greetings and hostname to any request to port 8080.

Save it as `app-v1.js` and let's create a Dockerfile for it:
```
FROM node:10.5-slim
ADD app-v1.js /app-v1.js
ENTRYPOINT ["node", "app-v1.js"]
```
now build it:
```
$ docker build -t <your_name>/gordon:v1.0
```
And push it either to DockerHub or to your favorite Docker registry.

### RC and Service
Now let's create a ReplicationController and a service: they will be used by Ingress:
```
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ReplicationController
metadata:
  name: gordon-v1
spec:
  replicas: 3
  template:
    metadata:
      name: gordon-v1
      labels:
        app: gordon-v1
    spec:
      containers:
      - image: evalle/gordon:v1.0
        name: nodejs
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: gordon-service-v1
spec:
  selector:
    app: gordon-v1
  ports:
  - port: 80
    targetPort: 8080
EOF
```
Don’t forget to change the image in the YAML file to your docker image:

```
...
containers:
- image: <name_of_your_image_goes_here>
  name: nodejs
...
```
Check that service is up and running:

```
$  kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
gordon-service-v1   ClusterIP   10.111.33.157   <none>        80/TCP    1d
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP   6d
```

Great, now let's create our first version of ingress:

```
$ cat <<EOF | kubectl create -f -
kind: Ingress
metadata:
  name: gordon
spec:
  rules:
  # this ingress maps the gordon.example.com domain name to our service
  # you have to add gordon.example.com to /etc/hosts
  - host: gordon.example.com
    http:
      paths:
      # all requests will be sent to port 80 of the gordon service
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
EOF
```
To be able to access the ingress from the outside we’ll need to make sure the hostname resolves to the IP of the ingress controller.
We can do it via:

- adding the hostname gordon.example.com into /etc/hosts (don't forget to delete it afterwards!)
- changing the hostname `gordon.example.com` to `gordon.example.com.<minikube's_ip>` in the previous YAML file - it will use [nip.io](http://nip.io/) service for resolving the hostname
- in case of production applications we will need to set up DNS resolving properly.

Check that ingress is available:
```
$ kubectl get ing
NAME      HOSTS                ADDRESS        PORTS     AGE
gordon    gordon.example.com   192.168.64.8   80        1h
```

Let's test our ingress:

```
$ for i in {1..10}; do curl http://gordon.example.com/v1; done
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
```

If you will try to request http://gordon.example.com it will give you a default backend's 404:

```
$ curl http://gordon.example.com
default backend - 404
```

This is because we have a rule only for /v1 path in our ingress YAML. An Ingress with no rules sends all traffic to a single default backend. Traffic is routed to your default backend if none of the Hosts in your Ingress match the Host in the request header, and/or none of the paths match the URL of the request.

The biggest advantage of using ingresses is their ability to expose multiple services through a single IP address, so let’s see how to do that.

### Multiple services
Let's create a second app, it's, basically, the same application with the slightly different output:

```
const http = require('http');
const os = require('os');

console.log("Gordon Server is starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("Hey, I'm the next version of gordon; my name is " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

Save it as `app-v2.js` and let's build from the following Dockerfile:
```
FROM node:10.5-slim
ADD app-v2.js /app-v2.js
ENTRYPOINT ["node", "app-v2.js"]
```
Build it:
```
$ docker build -t <your_name>/gordon:v2.0
```
And push it either to DockerHub or to Harbor (don't forget to tag it accordingly).

Let's create a second ReplicationController and a Service:
```
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ReplicationController
metadata:
  name: gordon-v2
spec:
  replicas: 3
  template:
    metadata:
      name: gordon-v2
      labels:
        app: gordon-v2
    spec:
      containers:
      - image: evalle/gordon:v2.0
        name: nodejs
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: gordon-service-v2
spec:
  selector:
    app: gordon-v2
  ports:
  - port: 90
    targetPort: 8080
EOF
```
Again, don’t forget to change the `image` in the YAML file to your docker image.

Let's check our services:
```
$ kubectl get svc
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
default-http-backend   ClusterIP   10.98.23.177     <none>        80/TCP    1h
gordon-service-v1      ClusterIP   10.108.177.42    <none>        80/TCP    1h
gordon-service-v2      ClusterIP   10.105.110.160   <none>        90/TCP    1h
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP   4d
```
And here is our new ingress YAML file:
```
$ cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gordon
spec:
  rules:
  # this ingress maps the gordon.example.com domain name to our service
  # you have to add gordon.example.com to /etc/hosts
  - host: gordon.example.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
      - path: /v2
        backend:
          serviceName: gordon-service-v2
          servicePort: 90
EOF
```
*Don't forget to change the line `  - host: gordon.example.com` in case you're using nip.io service for resolving*

Let's test it:
```
$ for i in {1..5}; do curl http://gordon.example.com/v1; done
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw

$ for i in {1..5}; do curl http://gordon.example.com/v2; done
Hey, I'm the next version of gordon; my name is gordon-v2-g6pll
Hey, I'm the next version of gordon; my name is gordon-v2-c78bh
Hey, I'm the next version of gordon; my name is gordon-v2-jn25s
Hey, I'm the next version of gordon; my name is gordon-v2-g6pll
Hey, I'm the next version of gordon; my name is gordon-v2-c78bh
```
It works!

### Configuring Ingress to Handle HTTPS traffic
Currently, the Ingress can handle incoming HTTPS connections, but it terminates the TLS connection and sends requests to the services unencrypted.
Since the Ingress terminates the TLS connection, it needs a TLS certificate and private key to do that.
The two need to be stored in a Kubernetes resource called a Secret.

Create a certificate, a key and save them into Kubernetes Secret:
```
$ openssl genrsa -out tls.key 2048
$ openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj '/CN=gordon.example.com'
$ kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
secret "tls-secret" created
```

Now let's change our ingress:
```
$ cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gordon
spec:
  tls:
  - hosts:
    - gordon.example.com
    secretName: tls-secret
  rules:
  # this ingress maps the gordon.example.com domain name to our service
  # you have to add gordon.example.com to /etc/hosts
  - host: gordon.example.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
      - path: /v2
        backend:
          serviceName: gordon-service-v2
          servicePort: 90
EOF
```
*Don't forget to change the line `  - host: gordon.example.com` in case you're using nip.io service for resolving*

Let's test it:
```
$ curl http://gordon.example.com/v1
<html>
<head><title>308 Permanent Redirect</title></head>
<body bgcolor="white">
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>


$ curl -k https://gordon.example.com/v1
You've hit gordon v1, my name is gordon-v1-64hhw
```

### Bonus 1 - Mapping Different Services to Different Hosts
Requests received by the controller will be forwarded to either service `foo` or `bar`, depending on the Host header in the request (exactly like how virtual hosts are handled in web servers).
Of course, DNS needs to point both the `foo.example.com` and the `bar.example.com` domain names to the Ingress controller’s IP address.

Example:
```
...
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /                
        backend:
          serviceName: foo
          servicePort: 80      
  - host: bar.example.com
    http:
      paths:
      - path: /                
        backend:
          serviceName: bar
          servicePort: 80
...
```

### Bonus 2 - Use Minikube Addon to Enable Nginx Ingress
```
$ minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.64.8
```

```
$ minikube addons list
- addon-manager: enabled
- coredns: disabled
- dashboard: enabled
- default-storageclass: enabled
- efk: disabled
- freshpod: disabled
- heapster: disabled
- ingress: disabled
- kube-dns: enabled
- metrics-server: disabled
- registry: disabled
- registry-creds: disabled
- storage-provisioner: enabled
```
As you can see, **ingress** add-on is disabled, let’s enable it:
```
$ minikube addons enable ingress
ingress was successfully enabled
```

Wait for a minute and then check that your cluster runs both nginx and default-http-backend:

nginx:
```
$ kubectl get all --all-namespaces | grep nginx
kube-system   deploy/nginx-ingress-controller   1         1         1            1           1h
kube-system   rs/nginx-ingress-controller-67956bf89d   1         1         1         1h
kube-system   po/nginx-ingress-controller-67956bf89d-dbbl4   1/1       Running   2          1h
```

default-backend:
```
$ kubectl get all --all-namespaces | grep default-http-backend
kube-system   deploy/default-http-backend       1         1         1            1           1h
kube-system   rs/default-http-backend-59868b7dd6       1         1         1         1h
kube-system   po/default-http-backend-59868b7dd6-6sh5f       1/1       Running   1          1h
kube-system   svc/default-http-backend   NodePort    10.104.42.209   <none>        80:30001/TCP    1h
```
