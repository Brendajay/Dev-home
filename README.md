To deploy two containers that scale independently from one another, we will create two Deployments, each with its own Pod template, and each Deployment will target its own database. We will use a separate Service for each Deployment, so that they can be accessed independently from each other.

We will create a Deployment for the users container, which will run an API that returns users from a database.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users
spec:
  replicas: 3 # set the initial number of replicas to 3
  selector:
    matchLabels:
      app: users
  template:
    metadata:
      labels:
        app: users
    spec:
      containers:
      - name: users
        image: <user-image>
        ports:
        - containerPort: 80
        env:
        - name: DATABASE_URL
          value: <users-database-url>

We will create another Deployment for the shifts container, which will run an API that returns shifts from a database.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shifts
spec:
  replicas: 3 # set the initial number of replicas to 3
  selector:
    matchLabels:
      app: shifts
  template:
    metadata:
      labels:
        app: shifts
    spec:
      containers:
      - name: shifts
        image: <shift-image>
        ports:
        - containerPort: 80
        env:
        - name: DATABASE_URL
          value: <shifts-database-url>

To auto-scale the service when the average CPU usage reaches 70%, we will create a Horizontal Pod Autoscaler (HPA) for each Deployment.


apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: users-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: users
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70

apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: shifts-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shifts
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70

To ensure that the deployment can handle rolling deployments and rollbacks, we will use RollingUpdate Deployment strategy, which allows us to update the Deployment without downtime. If the update fails, we can easily rollback to the previous version.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: users
spec:
  replicas: 3
  selector:
    matchLabels:
      app: users
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: users
    spec:
      containers:
      - name: users



apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app-image:latest
          ports:
            - containerPort: 8080
          env:
            - name: MONGODB_URI
              value: mongodb://mongodb-service:27017/my-db
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: my-app-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi



In this example, the deployment specifies three replicas of an application container named my-app that runs the my-app-image Docker image, exposes port 8080, and sets the MONGODB_URI environment variable to connect to the MongoDB database service.

The mongodb-service is created as a Kubernetes service for the MongoDB database, with a port specification that maps the MongoDB port 27017 to the service port 27017.

Finally, a PersistentVolumeClaim is defined to create a persistent storage volume for the my-app container, allowing the application to maintain its state even if the container is restarted or rescheduled.

