# Customized Wordpress Deployment on Kubernetes


## Deploying Contour Ingress


Run in your cluster:
```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

This command creates:

* A new namespace projectcontour
* Two instances of Contour in the namespace
* A Kubernetes Daemonset running Envoy on each node in the cluster listening on host ports 80/443
* A Service of type: LoadBalancer that points to the Contour’s Envoy instances
* Depending on your deployment environment, new cloud resources – for example, a cloud load balancer

```
kubectl get svc -n projectcontour
+ kubectl get svc envoy -n projectcontour
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP                                                               PORT(S)                      AGE
envoy     LoadBalancer   10.97.50.224   a84132ad3fdde4ee68ecde27fd5b44e1-1706667444.us-east-1.elb.amazonaws.com   80:32048/TCP,443:32560/TCP   163m
```

Create a CNAME for your ingress like:

```
*.apps.<your-domain-name> pointing to your LoadBalancer external IP
```
## Deploying MySQL

Create a MySQL namespace:
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: mysql
EOF
```

Create a MySQL secret. Remember that the password needs to be Base64 enconded. 

```
cat << EOF | kubectl apply -f -
apiVersion: v1
data:
  password: cGFzc3dvcmQ=
kind: Secret
metadata:
  name: mysql-pass
  namespace: mysql
type: Opaque
EOF
```

Deploy a MySQL service

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  namespace: mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
EOF
```

Create a persistent volume claim

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: mysql
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF
```

Deploy the database application

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress-mysql
  namespace: mysql  
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
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
EOF
```

## Deploying WordPress

Create a namespace for the application

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
EOF
```

Create a secret for accessing the database. (It must contain the same value from the MySQL secret)

```
cat << EOF | kubectl apply -f -
apiVersion: v1
data:
  password: cGFzc3dvcmQ=
kind: Secret
metadata:
  name: mysql-pass-7tt4f27774
  namespace: wordpress
type: Opaque
EOF
```

Create the application service

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
  namespace: wordpress    
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: ClusterIP
EOF
```

Create the persistent volume claim

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  namespace: wordpress
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF
```

Create the Wordpress deployment

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql.mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass-7tt4f27774
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
EOF
```

Create the application ingress

```
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: wpress-ingress
  namespace: wordpress
spec:
  rules:
  - host: wpress.apps.gdambor.com
    http:
      paths:
      - backend:
          serviceName: wordpress
          servicePort: 80
EOF
```