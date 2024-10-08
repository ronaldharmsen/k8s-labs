## Ghost
Ghost is a popular blogging engine with a clean interface written in JavaScript. It can either use a file-based SQLite database or MySQL for storage.

#### Configuring Ghost
Ghost is configured with a simple JavaScript file that describes the server. We will store this file as a configuration map. A simple development configuration for Ghost looks like :

```
var path = require('path'),
    config;

config = {
    development: {
        url: 'http://localhost:2368',
        database: {
            client: 'sqlite3',
            connection: {
                filename: path.join(process.env.GHOST_CONTENT,
                                    '/data/ghost-dev.db')
            },
            debug: false
        },
        server: {
            host: '0.0.0.0',
            port: '2368'
        },
        paths: {
            contentPath: path.join(process.env.GHOST_CONTENT, '/')
        }
    }
};

module.exports = config;
```

Once you have this configuration file saved to ghost-config.js, you can create a Kubernetes ConfigMap object using:

`kubectl create cm --from-file ghost-config.js ghost-config` 

This creates a ConfigMap that is named ghost-config. 

We will mount this configuration file as a volume inside of our container. We will deploy Ghost as a Deployment object, which defines this volume mount as part of the Pod template

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost
spec:
  replicas: 1
  selector:
    matchLabels:
      run: ghost
  template:
    metadata:
      labels:
        run: ghost
    spec:
      containers:
      - image: ghost:5.8.3
        name: ghost
        command:
        - sh
        - -c
        - cp /ghost-config/ghost-config.js /var/lib/ghost/config.js
          && /usr/local/bin/docker-entrypoint.sh node current/index.js
        volumeMounts:
        - mountPath: /ghost-config
          name: config
        resources:
          requests:
            memory: "64Mi"
            cpu: "500m"
          limits:
            memory: "128Mi"
            cpu: "1000m"
      volumes:
      - name: config
        configMap:
          defaultMode: 420
          name: ghost-config
```

One thing to note here is that we are copying the config.js file from a different location into the location where Ghost expects to find it, since the ConfigMap can only mount directories, not individual files. 

Ghost expects other files that are not in that ConfigMap to be present in its directory, and thus we cannot simply mount the entire ConfigMap into /var/lib/ghost.

Run the deployment:
`kubectl apply -f ghost.yaml`

And expose the deployment (creating a service and endpoint)
`kubectl expose deployment ghost --port=2368`

Run:
`kubectl proxy`

And visit the url: `http://localhost:8001/api/v1/namespaces/default/services/ghost/proxy/`

## GHOST + MYSQL

Of course, this example isn’t very scalable, or even reliable, since the contents of the blog are stored in a local file inside the container. A more scalable approach is to store the blog’s data in a MySQL database.

To do this, first copy ghost-config.js into config-mysql.js and modify ghost-config-mysql.js to include:
```
...
database: {
   client: 'mysql',
   connection: {
     host     : 'mysql',
     user     : 'root',
     password : 'root',
     database : 'ghost_db',
     charset  : 'utf8'
   }
 },
...
```

Create a new configmap: 
```
kubectl create configmap ghost-config-mysql --from-file ghost-config.js=ghost-config-mysql.js
```

And update the ghost.yaml file to use the newly created configmap.

#### Create MySQL environment

Before we store any data in MySQL we want to make sure we have a persistent storage setup:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

As we can see this manifest contains multiple items divided by `---`
Apply this manifest to create the persistent volume and proceed with creating the mysql instance:

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:9.0.1
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: root
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

You will need to create the database in the MySQL database:

```
$ kubectl exec -it mysql-<your-instance-here> -- mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
...

mysql> create database ghost_db;
...
```

Now apply the updated ghost.yaml file to start using the mysql database.

Review all steps and monitor the application. Also check logs to see what happened and if ghost is actually using your mysql database or not.

#### Ingress
Instead of using the proxy we might want to use something better.

Setup ingress with a rule to redirect traffic into ghost for `http://<your-ip>/`

Hint:
enable an ingress controller `microk8s.enable ingress`
or
minikube addons enable ingress

Example ingress.yaml file (change to your situation)
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: "ghost.localhost"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ghost
            port:
              number: 2368
```

when you run above with Minikube run ```minikube tunnel``` to make ingress available on localhost
