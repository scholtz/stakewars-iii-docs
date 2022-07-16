# Run CLI

compose-cli.sh

```
if [ "$ver" == "" ]; then
ver=shardnet-near-cli
fi

echo "docker build -t \"scholtz2/stakewars:$ver\" -f Dockerfile ."
docker build -t "scholtz2/stakewars:$ver" -f cli.Dockerfile .|| error_code=$?
if [ "$error_code" != "" ]; then
echo "$error_code";
    echo "failed to build";
	exit 1;
fi

docker push "scholtz2/stakewars:$ver" || error_code=$?
if [ "$error_code" != "" ]; then
echo "$error_code";
    echo "failed to push";
	exit 1;
fi

echo "Image: scholtz2/stakewars:$ver"
```

cli.Dockerfile

```
FROM ubuntu:latest
USER root
ENV DEBIAN_FRONTEND noninteractive
RUN apt update && apt dist-upgrade -y && apt install -y mc wget telnet git curl iotop atop vim jq && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN curl -ysSf https://sh.rustup.rs | sh
WORKDIR /tools
RUN curl -L https://raw.githubusercontent.com/tj/n/master/bin/n -o n
RUN bash n 18
RUN npm install -g n
RUN node -v
RUN npm install -g npm@8.14.0
RUN npm -v
RUN npm install -g near-cli
ENV NEAR_ENV=shardnet
RUN useradd -d /app -ms /bin/bash app
USER app
WORKDIR /app
RUN echo 'export NEAR_ENV=shardnet' >> /app/.bashrc
```

near-cli-deployment.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: stakewars

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: stakewars-cli-deployment
  namespace: stakewars
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stakewars-cli-deployment
  template:
    metadata:
      labels:
        app: stakewars-cli-deployment
    spec:
      nodeName: sch-h-02
      containers:
        - name: stakewars-cli-deployment
          image: registry.k8s.aramid.finance/stakewars:shardnet-near-cli
          imagePullPolicy: IfNotPresent
          #command: ["/root/start.sh", "run", "testnet"]
          command: ["/bin/sh", "-ec", "while :; do date; sleep 60 ; done"]
          #command: ["neard", "--home", "/app/.near", "run"]

          resources:
            requests:
              memory: "1000Mi"
              cpu: "100m"
            limits:
              memory: "2000Mi"
              cpu: "1000m"
          env:
            - name: NEAR_ENV
              value: shardnet
          lifecycle:
            preStop:
              exec:
                command: ["sleep","20"]
          volumeMounts:
            - name: stakewars-cli-data-pvc
              mountPath: /app/.near-credentials
      volumes:
        - name: stakewars-cli-data-pvc
          persistentVolumeClaim:
            claimName: stakewars-cli-data-pvc


---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stakewars-cli-data-pvc
  namespace: stakewars
spec:
  storageClassName: stakewars-cli-data
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: stakewars-cli-data-pv
spec:
  storageClassName: stakewars-cli-data
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/nvme1/near-stakewars-cli"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - sch-h-02
```

sch-h-02 is the node name where it runs

you might want to use other PVC which automatically creates PV

```
# apply config to k8s
kubectl apply -f near-cli-deployment.yaml
```

In K8S dashboard exec to pod

![](<../.gitbook/assets/image (3).png>)
