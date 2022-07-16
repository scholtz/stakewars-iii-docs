# Cron job

cli-job.yaml

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ping
  namespace: stakewars
spec:
  schedule: "37 */2 * * *"
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      backoffLimit: 0
      parallelism: 1
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: c
            image: scholtz2/stakewars:shardnet-near-cli
            command: ["/bin/sh", "-ec", "/app/run.sh"]
            volumeMounts:
            - name: pings-cli-conf
              mountPath: /app/.near-credentials/shardnet/scholtz.shardnet.near.json
              subPath: scholtz.shardnet.near.json
            - name: pings-cli-conf
              mountPath: /app/run.sh
              subPath: run.sh
          volumes:
            - name: pings-cli-conf
              configMap:
                defaultMode: 0777
                name: pings-cli-conf

```

cli-conf/run.sh

```
date
echo "ping scholtz.factory.shardnet.near"
near call scholtz.factory.shardnet.near ping '{}' --accountId scholtz.shardnet.near --gas=300000000000000
date
```

cli-conf/scholtz.shardnet.near.json

```
:)
```

near-cli.sh

```
kubectl apply -f cli-job.yaml -n stakewars
kubectl delete configmap pings-cli-conf -n stakewars
kubectl create configmap pings-cli-conf --from-file=cli-conf -n stakewars
```

Run&#x20;

```
./near-cli.sh
```
