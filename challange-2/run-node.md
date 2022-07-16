# Run node

neard.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: stakewars

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: stakewars-nonarchival-deployment
  namespace: stakewars
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: stakewars-nonarchival-deployment
  template:
    metadata:
      labels:
        app: stakewars-nonarchival-deployment
    spec:
      nodeName: sch-h-02
      containers:
        - name: stakewars-nonarchival-deployment
          image: registry.k8s.aramid.finance/stakewars:shardnet-neard-1.28.0-dev
          imagePullPolicy: IfNotPresent
          command: ["/bin/sh", "/app/start.sh"]
          #command: ["/bin/sh", "-ec", "while :; do date; sleep 60 ; done"]
          #command: ["/tools/nearcore/target/release/neard", "--home", "/app/.near", "run"]
          #command: ["neard", "--home", "/app/.near", "run"]

          resources:
            requests:
              memory: "10000Mi"
              cpu: "1000m"
            limits:
              memory: "20000Mi"
              cpu: "8000m"
          env:
            - name: NEAR_ENV
              value: shardnet
          ports:
          -
            containerPort: 3030
            protocol: "TCP"  
          -
            containerPort: 24567
            protocol: "TCP"  
          startupProbe:
              tcpSocket:
                port: 24567
              initialDelaySeconds: 600
              periodSeconds: 10
          readinessProbe:
              tcpSocket:
                port: 24567
              initialDelaySeconds: 600
              periodSeconds: 10
          livenessProbe:
              tcpSocket:
                port: 24567
              initialDelaySeconds: 600
              periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["sleep","1"]
          volumeMounts:
            - name: stakewars-nonarchival-data-pvc
              mountPath: /app/.near/
      volumes:
        - name: stakewars-nonarchival-data-pvc
          persistentVolumeClaim:
            claimName: stakewars-nonarchival-data-pvc


---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stakewars-nonarchival-data-pvc
  namespace: stakewars
spec:
  storageClassName: stakewars-nonarchival-data
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: stakewars-nonarchival-data-pv
spec:
  storageClassName: stakewars-nonarchival-data
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/nvme1/near-stakewars-nonarchival"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - sch-h-02
```

Apply to K8S

```
kubectl apply -f neard.yaml
```
