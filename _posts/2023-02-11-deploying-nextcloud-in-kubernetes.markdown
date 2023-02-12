---
layout: post
title:  "Deploying Nextcloud in Kubernetes"
date:   2023-02-11 11:51:00 -0500
categories: kubernetes nextcloud collabora
published: true
---
This is the first part in my series of posts regarding [running Nextcloud and Collabora CODE in Kubernetes][root-post].

1. [Deploying Nextcloud in Kubernetes][nextcloud-post]
2. [Deploying Collabora CODE in Kubernetes][collabora-post]
3. [Configuring Nextcloud with Collabora Code in Kubernetes][configuration-post]

---

## Preparation

I have twelve files that deploy Nextcloud into my cluster to break out the deployment into different steps.  These are for the three main component required for a Nextcloud installation:

* PostreSQL Database
* Redis
* Nextcloud

While deploying all of these services, I have a terminal window open and run the following command so that I can monitor the services as the cluster deploys them.

```bash
watch -n 3 kubectl get all,pvc,ingress,certificaterequests,orders,certificates,secret,cronjob,configmap -n nextcloud
```

## Deployment

### Namespace

Create a namespace for Nextcloud in your Kubernetes cluster.

```bash
kubectl create namespace nextcloud
```

### Secrets

I started with a single secrets file, `secrets.yml` that contains the secrets data that will be used for all three tools.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nextcloud-secrets
  namespace: nextcloud
type: Opaque
stringData:
  POSTGRES_DB: $DB
  POSTGRES_USER: $DB_USER 
  POSTGRES_PASSWORD: $DB_NEXTCLOUD_PASSWORD
  NEXTCLOUD_ADMIN_USER: $NEXTCLOUD_ADMIN_USER
  NEXTCLOUD_ADMIN_PASSWORD: $NEXTCLOUD_ADMIN_PASSWORD
```

Each of the variables is set by an environment variable on my local machine so that the actual secrets are not committed in plain text.  Then, I run the following command to deploy the secrets file.

```bash
envsubst < secrets.yml | kubectl apply -f -
```

### PostgreSQL

Create three files for PostgreSQL:

1. `postgresql_pvc.yml`
2. `postgresql_service.yml`
3. `postgresql_deployment.yml`

Database storage is provided by Longhorn.  I have started with 2 Gigabytes of storage, which Longhorn will allow me to increase in time if the database grows too large.  `postgresql_pvc.yml` is the following YAML.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
  namespace: nextcloud
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

Next, define the service in `postgresql_service.yml`.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: nextcloud
  labels:
    app: postgresql
spec:
  ports:
    - port: 5432
  selector:
    app: postgresql
```

Lastly, configure the deployment `postgresql_deployment.yml`.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: nextcloud
  labels:
    app: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_DB
                  name: nextcloud-secrets
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_USER
                  name: nextcloud-secrets
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_PASSWORD
                  name: nextcloud-secrets
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: TZ
              value: America/New_York
          volumeMounts:
            - name: postgresql-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgresql-data
          persistentVolumeClaim:
            claimName: postgresql-pvc
```

Once these files are created, then you can apply them all to your cluster.

```bash
kubectl apply -f \
postgresql_pvc.yml \
postgresql_service.yml \
postgresql_deployment.yml
```

Watch your terminal window until your PostgreSQL pod shows that it is in a `Running` state.

### Redis

Redis will be setup in a similar fashion.  The files are:

1. `redis_service.yml`
2. `redis_deployment.yml`

Create the `redis_service.yml` file as follows.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: nextcloud
  labels:
    app: redis
spec:
  ports:
    - port: 6379
  selector:
    app: redis
```

And then the `redis_deployment.yml` file.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: nextcloud
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - image: redis:alpine
          name: redis
          ports:
            - containerPort: 6379
          env:
            - name: TZ
              value: America/New_York
      restartPolicy: Always
```

Once these files are created, then you can apply them all to your cluster.

```bash
kubectl apply -f \
redis_service.yml \
redis_deployment.yml
```

Watch your terminal window until your Redis pod shows that it is in a `Running` state.

### Nextcloud

This section has a few more deployment files.

| File                       | Purpose                                                      |
| -------------------------- | ------------------------------------------------------------ |
| `nextcloud_pvc.yml`        | Requests storage from Longhorn and NFS provisioners.         |
| `nextcloud_headers.yml`    | Sets Traefik to enable HTTP headers that Nextcloud requires. |
| `nextcloud_service.yml`    | Configures the Nextcloud Kubernetes service.                 |
| `nextcloud_ingress.yml`    | Configures the Kubernetes ingress to the Nextcloud Service.  |
| `nextcloud_deployment.yml` | Configures the Kubernetes deployment of Nextcloud.           |
| `nextcloud_cron.yml`       | Sets the web cron job for Nextcloud to run every 5 minutes.  |

Create `nextcloud_pvc.yml` in order to request storage.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-pvc
  namespace: nextcloud
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-pvc-nfs
  namespace: nextcloud
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 100Gi
```

Create `nextcloud_headers.yml` to create Traefik middleware that will set HTTP headers that Nextcloud cares about, and to enable the `.well-known` URL paths for DAV-based services.

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: headers
  namespace: nextcloud
spec:
  headers:
    frameDeny: true
    browserXssFilter: true
    customResponseHeaders:
      Strict-Transport-Security: "15552000"
      X-Frame-Options: SAMEORIGIN
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirects
  namespace: nextcloud
spec:
  redirectScheme:
    permanent: true
    scheme: https
  redirectRegex:
    regex: https://(.*)/.well-known/(card|cal)dav
    replacement: https://$1/remote.php/dav/
```

Create the `nextcloud_service.yml` to setup the service.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: nextcloud
  labels:
    app: nextcloud
spec:
  ports:
    - port: 80
  selector:
    app: nextcloud
```

The service will operate on port 80, but Traefik was previously configured in the headers file to redirect requests to HTTPS.  The ingress configuration will then serve Nextcloud from HTTPS.

Create `nextcloud_ingress.yml` with the following.  Note that you will need to setup the proper domain for your Nextcloud instance.  This configuration assumes that you have a working CertManager deployment on your cluster; as you can see, I am using a cluster issuer that I setup previously called `letsencrypt-aws`.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress
  namespace: nextcloud
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: nextcloud-headers@kubernetescrd,nextcloud-redirects@kubernetescrd
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    cert-manager.io/cluster-issuer: letsencrypt-aws
spec:
  rules:
    - host: your.nextcloud.domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nextcloud
                port:
                  number: 80
  tls:
    - secretName: ssl-cert
      hosts:
        - your.nextcloud.domain
```

Now, we configure the Nextcloud deployment in `nextcloud_deployment.yml`.  

Note that I use the most-specific tag to call out the latest Nextcloud release, `nextcloud:25.0.3-apache` in the example below.  If you use a pod management strategy that could download and apply a new version of Nextcloud without a manual trigger, like using the tag `nextcloud:25-apache` that will change as new versions are released, then the Nextcloud upgrade process could start unintentionally.  That process can take many minutes.  I prefer to be alerted by Nextcloud when there are new updates and then scheduling an upgrade when convenient to apply the update.

Be sure to set your domain name in the proper variables.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud
  namespace: nextcloud
  labels:
    app: nextcloud
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
        - image: nextcloud:25.0.3-apache
          name: nextcloud
          ports:
            - containerPort: 80
              protocol: TCP
          env:
            - name: REDIS_HOST
              value: redis
            - name: POSTGRES_HOST
              value: postgresql
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_DB
                  name: nextcloud-secrets
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_USER
                  name: nextcloud-secrets
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: POSTGRES_PASSWORD
                  name: nextcloud-secrets
            - name: NEXTCLOUD_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  key: NEXTCLOUD_ADMIN_USER
                  name: nextcloud-secrets
            - name: NEXTCLOUD_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: NEXTCLOUD_ADMIN_PASSWORD
                  name: nextcloud-secrets
            - name: NEXTCLOUD_TRUSTED_DOMAINS
              value: your.nextcloud.domain
            - name: NEXTCLOUD_DATA_DIR
              value: /mnt/data
            # - name: OBJECTSTORE_S3_HOST
            #   value: your.s3.host
            # - name: OBJECTSTORE_S3_REGION
            #   value: gso-rack-1
            # - name: OBJECTSTORE_S3_BUCKET
            #   value: nextcloud
            # - name: OBJECTSTORE_S3_PORT
            #   value: "9000"
            # - name: OBJECTSTORE_S3_SSL
            #   value: "true"
            # - name: OBJECTSTORE_S3_USEPATH_STYLE
            #   value: "true"
            # - name: OBJECTSTORE_S3_KEY
            #   valueFrom:
            #     secretKeyRef:
            #       key: OBJECTSTORE_S3_KEY
            #       name: nextcloud-secrets
            # - name: OBJECTSTORE_S3_SECRET
            #   valueFrom:
            #     secretKeyRef:
            #       key: OBJECTSTORE_S3_SECRET
            #       name: nextcloud-secrets
            - name: TRUSTED_PROXIES
              value: 192.168.4.0/24 10.0.0.0/16 # This includes my router IP address and the CIDR range of the cluster
            - name: APACHE_DISABLE_REWRITE_IP
              value: "1"
            - name: OVERWRITEHOST
              value: your.nextcloud.domain
            - name: OVERWRITEPROTOCOL
              value: https
            - name: OVERWRITECLIURL
              value: https://your.nextcloud.domain
            - name: OVERWRITEWEBROOT
              value: "/"
            - name: PHP_MEMORY_LIMIT
              value: 4G
            - name: PHP_UPLOAD_LIMIT
              value: 1G
            - name: TZ
              value: America/New_York
          volumeMounts:
            - mountPath: /var/www/html
              name: nextcloud-storage
              readOnly: false
            - mountPath: /mnt/data
              name: nextcloud-storage-nfs
              readOnly: false
      volumes:
        - name: nextcloud-storage
          persistentVolumeClaim:
            claimName: nextcloud-pvc
        - name: nextcloud-storage-nfs
          persistentVolumeClaim:
            claimName: nextcloud-pvc-nfs
```

We then need to setup the Nextcloud CRON job in `nextcloud_cron.yml`.  The Nextcloud container does not contain the CRON utility, so the web-based cron is the best solution here.  I reuse the same Nextcloud container image to run the CRON job as the deployment.

```yaml
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nextcloud-cron
  namespace: nextcloud
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: nextcloud
            image: nextcloud:25.0.3-apache
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - curl https://your.nextcloud.domain/cron.php
          restartPolicy: OnFailure
```

Now, we need to apply all of these files.

```bash
kubectl apply -f \
nextcloud_pvc.yml \
nextcloud_headers.yml \
nextcloud_service.yml \
nextcloud_ingress.yml \
nextcloud_ingress.yml \
nextcloud_deployment.yml \
nextcloud_cron.yml
```

This process can take about five minutes to spin up all of the resources.  In particular, the SSL certificate request can take a minute or two itself.  So, be patient at this step and follow the `watch` command that you started at the beginning of this post.  When all pods are in a `Running` state, and your certificates show the ready state as `True`, then you should be able to access your nextcloud instance.

Next, we need to [deploy Collabora Code][collabora-post].

[root-post]: {% post_url 2023-02-11-nextcloud-and-collabora-on-kubernetes %}
[nextcloud-post]: {% post_url 2023-02-11-deploying-nextcloud-in-kubernetes %}
[collabora-post]: {% post_url 2023-02-11-deploying-collabora-code-in-kubernetes %}
[configuration-post]: {% post_url 2023-02-11-configuring-nextcloud-and-collabora-on-kubernetes %}
