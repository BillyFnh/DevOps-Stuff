# Kubernetes Resources

## Examples

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: aks-defacement-detection
  name: mongodb-0
  labels:
    env: test
    service: mongodb-0
spec:
  containers:
  - image: mtlbillyfong/mongodb-replica-set:20200330-stable-1
    name: mongodb-0
    resources:
      requests:
        memory: "1024Mi"
        cpu: "500m"
      limits:
        memory: "2048Mi"
        cpu: "1000m"
    env:
    - name: "MONGO_INITDB_ROOT_USERNAME"
      value: "admin"
    - name: "MONGO_INITDB_ROOT_PASSWORD"
      value: "password"
    - name: "MONGODB_ID"
      value: "mongo-0"
    livenessProbe:
      exec:
        command:
        - /bin/bash
        - -c
        - mongo --quiet --eval "db.getMongo()"
      initialDelaySeconds: 5
      periodSeconds: 5
    ports:
    - containerPort: 27017
    volumeMounts:
    - name: vol-nfs-mongodb-rs0-0
      mountPath: "/data/db"
    - name: tz-hongkong
      mountPath: /etc/localtime
  volumes:
    - name: vol-nfs-mongodb-rs0-0
      persistentVolumeClaim:
        claimName: azure-managed-disk-mongodb-0
    - name: tz-hongkong
      hostPath:
        path: /etc/localtime
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: defacement-detection
  name: rabbitmq-server
  labels:
    env: test
    defacementComponent: rabbitmq-server
spec:
  replicas: 1
  selector:
    matchLabels:
      defacementComponent: rabbitmq-server
  template:
    metadata:
      labels:
        defacementComponent: rabbitmq-server
    spec:
      containers:
        - image: mtlbillyfong/rabbitmq-server:20201030-stable-1
          name: rabbitmq-server
          ports:
          - name: metrics
            containerPort: 15692
          resources:
            requests:
              cpu: "1500m"
              memory: "1024Mi"
              ephemeral-storage: "2Gi"
            limits:
              cpu: "2000m"
              memory: "2048Mi"
              ephemeral-storage: "4Gi"
          volumeMounts:
            - name: tz-hongkong
              mountPath: /etc/localtime
            - name: netappnfs-test-defacement-detection-rabbitmq-server
              mountPath: "/var/lib/rabbitmq"
      volumes:
        - name: tz-hongkong
          hostPath:
            path: /etc/localtime
        - name: netappnfs-test-defacement-detection-rabbitmq-server
          nfs:
            server: 172.31.52.20
            path: "/mnt/NFS1/cluster5/defacement-detection-test-deployment/rabbitmq"
```

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
  namespace: defacement-detection
spec:
  selector:
    matchLabels:
      role: mongo
      environment: test
      defacementComponent: mongodb
  serviceName: "mongo"
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: test
        defacementComponent: mongodb
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mongo
        image: mongo:4.2.3
        command: ["/bin/bash", "-c"]
        args:
          - mongod --replSet rs0 --port 27017 --bind_ip localhost,$(hostname -i) --oplogSize 128
        ports:
          - containerPort: 27017
        livenessProbe:
          exec:
            command:
              - /bin/bash
              - -c
              - mongo --quiet --eval "db.getMongo()"
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          exec:
            command:
              - /bin/bash
              - -c
              - mongo --quiet --eval "db.getMongo()"
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
      - name: mongo-sidecar
        image: cvallance/mongo-k8s-sidecar
        env:
          - name: MONGO_SIDECAR_POD_LABELS
            value: "role=mongo,environment=test"
          - name: KUBE_NAMESPACE
            value: "defacement-detection"
  volumeClaimTemplates:
    - metadata:
        name: mongo-persistent-storage
      spec:
        storageClassName: "managed-nfs4k8s-storage"
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: defacement-detection
  name: rabbitmq-server-service
  labels:
    run: rabbitmq-server-service
spec:
  selector:
    defacementComponent: rabbitmq-server
  ports:
    - name: '4369'
      port: 4369
      targetPort: 4369
      protocol: TCP
    - name: '5671'
      port: 5671
      targetPort: 5671
      protocol: TCP
    - name: '5672'
      port: 5672
      targetPort: 5672
      protocol: TCP
    - name: '15672'
      port: 15672
      targetPort: 15672
      protocol: TCP
    - name: '25672'
      port: 25672
      targetPort: 25672
      protocol: TCP
```

### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq-server
  namespace: defacement-detection
  labels:
    defacementComponent: rabbitmq-metric-service
spec:
  selector:
    matchLabels:
      run: rabbitmq-server-metric-service
  endpoints:
  - targetPort: 15692
    interval: 5s
```

### PersistentVolume/PersistentVolumeClaim (NFS,Azure,GKE)

#### PVC - Azure AKS

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: aks-defacement-detection
  name: azure-managed-disk-mongodb-0
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: default
  resources:
    requests:
      storage: 5Gi
```
