---
layout: post
title:  "Deploying PrivateBin on K3s"
date:   2023-02-18 16:45:00 -0500
categories: kubernetes privatebin
published: true
---
Continuing my line of posts about services that I run in K8s, I wanted to blog about my deployment of [PrivateBin](https://github.com/PrivateBin/PrivateBin).  PrivateBin is described as:

> A minimalist, open source online pastebin where the server has zero knowledge of pasted data. Data is encrypted/decrypted in the browser using 256 bits AES.

I decided to deploy PrivateBin as an alternative to texting secrets or sharing them directly from a password manager with my family.  This service doesn't get used that often, but it's light weight and there when we need it.  PrivateBin publishes a container to DockerHub, so it was straightforward to setup with a small PVC backend for storage.

## Deploying PrivateBin in K3s

I run K3s in my home lab and leverage most of the built in utilities that come with it, such as Traefik ingress.  I also have Longhorn installed for block storage provisioning.

### Persistent Volume Claim

The only contents shared over PrivateBin are small secrets, so a `2GB` PVC is well more than this service needs for sharing small secrets for a short amount of time.  The following `pvc.yml` file defines the PVC.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: privatebin-pvc
  namespace: privatebin
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

### Service

The PrivateBin container exposes its interface over port 8080, so `service.yml` exposes the same port.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: privatebin-service
  namespace: privatebin

spec:
  ports:
    - name: web
      port: 8080
      targetPort: 8080

  selector:
    app: privatebin
```

### Ingress

The `ingress.yml` file implements Traefik annotations to expose HTTP and HTTPS ports.  I also have a middleware defined in the default namespace that redirects HTTP traffic to HTTPS.  Lastly, I used a certmanager annotation to issue an TLS certificate that Traefik will use for TLS termination.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: privatebin-ingress
  namespace: privatebin
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    cert-manager.io/cluster-issuer: letsencrypt-aws
spec:
  rules:
    - host: privatebin.domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: privatebin-service
                port:
                  number: 8080
  tls:
    - secretName: ssl-cert
      hosts:
        - privatebin.domain
```

### Deployment

Lastly, `deployment.yml` implements the deployment.  The PVC is mounted to the pod at `/srv/data`, which is the container's default storage location.  The important configuration is the `securityContext` which defines the user and group that the pod will run as.  If you do not set these values, then PrivateBin will present an error when attempting to save secrets due to an inability to write to the file system.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: privatebin
  name: privatebin-deployment
  namespace: privatebin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: privatebin
  template:
    metadata:
      labels:
        app: privatebin
    spec:
      securityContext:
        runAsUser: 65534
        runAsGroup: 82
        fsGroup: 82
      terminationGracePeriodSeconds: 40
      containers:
        - name: privatebin
          image: privatebin/nginx-fpm-alpine:stable
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            - name: TZ
              value: America/New_York
            - name: PHP_TZ
              value: America/New_York
          livenessProbe:
            httpGet:
              path: /
              port: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
          volumeMounts:
            - name: privatebin-volume
              mountPath: /srv/data
              readOnly: false
      volumes:
        - name: privatebin-volume
          persistentVolumeClaim:
            claimName: privatebin-pvc
```

### Applying the Configuration

Now, just apply all of the files to your cluster.  PrivateBin should be running in your cluster once the services start and the TLS certificate is issued.

```bash
kubectl apply -f \
pvc.yml \
service.yml \
ingress.yml \
deployment.yml
```
