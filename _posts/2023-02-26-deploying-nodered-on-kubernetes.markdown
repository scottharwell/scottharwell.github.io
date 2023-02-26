---
layout: post
title:  "Deploying Node-RED on K3s"
date:   2023-02-26 09:00:00 -0500
categories: kubernetes nodered
published: true
---
[Node-RED][node-red] is another service that I run in my K3s cluster for experimenting with event-driven automation.  Node-RED is described as:

> Node-RED is a programming tool for wiring together hardware devices, APIs and online services in new and interesting ways.
> 
> It provides a browser-based editor that makes it easy to wire together flows using the wide range of nodes in the palette that can be deployed to its runtime in a single-click.

I originally used Node-RED with a few Raspberry Pis, but found the tool interesting and useful enough to run a permanent instance in my cluster.  There are tons of plugins that can extend the value of Node-RED well beyond its out-of-the-box functionality.

## Configuring the Deployment

I run K3s in my home lab and leverage most of the built in utilities that come with it, such as Traefik ingress.  I also have Longhorn installed for block storage provisioning.

I put all files created in this example in their own folder to separate this deployment from other deployments on my cluster.

### Authorization

Node-RED is not a multi-user platform.  Its UI is exposed unless you take action to secure it through other ingress, like your reverse proxy.  In my case, I configured Basic Auth in the out-of-the-box Traefik ingress in K3s.  We'll use the `htpasswd` utility to create the password string and then `base64` to encode it for k8s.

The full command is the following replacing `user` and `password` with the username and password that you want to use:

```bash
htpasswd -nb user password | openssl base64
```

And then we will create a `secrets.yml` file to store the output of that command.

**Note:** The example below contains a dummy output from the command above.  You could use an environment variable and `envsubst` to remove any secrets from the file.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: auth-secret
  namespace: node-red
data:
  users: |2
    dXNlcjokYXByMSRHaFZaT3VJLiRjMWRydDU5Nmo2SzhQSUV4Z21WWnoxCgo=
```

### Headers

Now that the authorization secret is prepared, we can go about configuring the rest of the deployment resources.

First, we will create a `headers.yml` file that will create a Traefik middleware to manage the basic auth authorization.

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: auth
  namespace: node-red
spec:
  basicAuth:
    secret: auth-secret
```

### Storage

Next, create the file `pvc.yml` to request a persistent volume claim so that Node-RED can store its data.  In my example below, I use the Longhorn storage provisioner, but it can be local storage or any other storage provisioner that you have in your cluster.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: node-red-pvc
  namespace: node-red
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

### Service

Next create a `service.yml` file to expose the Node-RED service.  The container exposes port `1880` by default.  We'll use our ingress to terminate TLS traffic later.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: node-red-service
  namespace: node-red

spec:
  ports:
    - name: web
      port: 1880
      targetPort: 1880

  selector:
    app: node-red
```

### Ingress

Create an `ingress.yml` file to define the ingress.  This ingress exposes both HTTP and HTTPS traffic, but I have a redirect middleware defined in the default namespace that reroutes HTTP to HTTPS.  You also find the `auth` middleware defined in the `node-red` namespace, which applies basic auth to the ingress.  I also use CertManager to issue TLS certs for TLS termination.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-red-ingress
  namespace: node-red
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd,node-red-auth@kubernetescrd
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    cert-manager.io/cluster-issuer: letsencrypt-aws
spec:
  rules:
    - host: your.node-red.domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: node-red-service
                port:
                  number: 1880
  tls:
    - secretName: ssl-cert
      hosts:
        - your.node-red.domain
```

### Deployment

Lastly, create a `deployment.yml` file to define the deployment.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: node-red
  name: node-red
  namespace: node-red
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-red
  template:
    metadata:
      labels:
        app: node-red
    spec:
      terminationGracePeriodSeconds: 40
      containers:
        - name: node-red
          image: nodered/node-red:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 1880
              protocol: TCP
          volumeMounts:
            - name: node-red-volume
              mountPath: /data
              readOnly: false
      volumes:
        - name: node-red-volume
          persistentVolumeClaim:
            claimName: node-red-pvc
```

### Apply the Configuration

Once all files are created, we can run the following command to apply them to the cluster.

```bash
kubectl apply -f .
```

Watch the namespace and when the deployment is up and the TLS certs have been issued, then you should be able navigate to the domain name that you used for your ingress.  You'll then be presented with a username and password for the basic auth that you configured.  Once logged in, you will be able to use Node-RED on your kubernetes cluster.

[node-red]: https://nodered.org/
