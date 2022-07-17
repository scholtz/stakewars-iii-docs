# Linode

### Pricing

Cheapest possible variant for stakewars cost $30 per server plus $50 per storage  /  2 CPU , 4 GB DDR4 RAM, 500 GB storage (might be incemented in time)

### Create cluster&#x20;

![](<../.gitbook/assets/image (8).png>)![](<../.gitbook/assets/image (1).png>)

Download config yaml file to local machine

![](<../.gitbook/assets/image (9).png>)

```
export KUBECONFIG=/mnt/c/linode/k8s-kubeconfig.yaml
KUBECONFIG=/mnt/c/linode/k8s-kubeconfig.yaml

#test with
kubectl get pods -A
```
