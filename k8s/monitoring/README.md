# Monitoring hands-on

## Setup DNS names

Append the following line on the /etc/hosts on your machine (your laptop):

```
<your-olss-vm-ip> minikube grafana.minikube prometheus.minikube olss-demo-app.minikube ingress-example.minikube
```

This is what I have on my mac:
```
❯ tail -1 /etc/hosts
131.154.97.229 minikube grafana.minikube prometheus.minikube olss-demo-app.minikube tutor-01.olss ingress-example.minikube
```

If you want to also contact the minikube ingress from within the OLSS VM,
login into the OLSS VM, collect the minikube ip with:

```
minikube ip
```

and append the following on the /etc/hosts on that machine, e.g.:

```
<your-minikube-ip> minikube grafana.minikube prometheus.minikube olss-demo-app.minikube ingress-example.minikube
```

On my tutor-1 machine I have:

```
[centos@tutor1 ~]$ minikube ip
192.168.49.2
[centos@tutor1 ~]$ tail -1 /etc/hosts
192.168.49.2 minikube grafana.minikube prometheus.minikube olss-demo-app.minikube ingress-example.minikube
```

Make sure the minikube ingress add is enabled:
```
[centos@tutor1 monitoring]$ minikube addons list | grep ingress
| ingress                     | minikube | enabled ✅   |
```

If is not enabled, enable it:
```
minikube addons enable ingress
```

# Install the OLSS demo app in your cluster

```
git clone https://github.com/andreaceccanti/olss-demo-app/
```

If have already cloned in the past days, or made your own fork, clone it 
again in another repo and use that one:

```
git clone https://github.com/andreaceccanti/olss-demo-app/
olss-demo-app-orig
```

Go to the `k8s/olss-demo-app` folder and type the following command:

```
kubectl apply -k .
```

This command will install the demo app on your cluster.

```
[centos@tutor1 olss-demo-app]$ kubectl -n olss-demo-app get po
NAME                            READY   STATUS    RESTARTS   AGE
olss-demo-app-58cd985fc-8drr8   1/1     Running   0          58s
```

# Monitoring probes

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

Deploy the OLSS demo app without probes:

```
kubectl apply -f olss-app.deploy.no-probes.yaml
```

# Create a kube config context for the olss-demo-app namespace

```
kubectl config set-context --cluster=minikube --user=minikube --namespace=olss-demo-app olss-demo-app
kubectl config use-context olss-demo-app
```

# Install the multitool container

```
kubectl run multitool --image=praqma/network-multitool --restart Never
```

Enter the multitool container:

```
kubectl exec -ti multitool -- sh
```

Within the mulitool container run the following command:

```
watch -n.5 curl olss-demo-app:8080/api/hello
```

The above commands issue 2 requests per second against the OLSS demo app.

Now, in another terminal, scale up the olss-demo-app deployment to 3 replicas:

```
kubectl scale deployment --replicas=2 olss-demo-app
```

You will observe that while containers start up some requests fail.
This is because we do not have probes enabled.

Scale down the deployment back to 1 replica:

```
kubectl scale deployment --replicas=1 olss-demo-app
```

Deploy the olss-demo-app with probes:

```
kubectl apply -f olss-app.deploy.yaml
```

# Install prometheus on your cluster 


```
# Install prometheus community helm charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# create a namespace for prometheus and monitoring-related stuff
kubectl create namespace monitoring

# install the prometheus stack in the monitoring namespace
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --values=prometheus-kube-stack.values.yml
```

# Setup port forwarding to reach the ingress controller

In another terminal running on the OLSS VM, type the following command
(and leave it running):


```
kubectl --namespace=ingress-nginx port-forward --address 0.0.0.0
svc/ingress-nginx-controller 8080:80
```


# Open the grafana dashboard in your browser

Point your browser to:

http://grafana.minikube:8080

You should see the Grafana login page.

You can log into the system with the following credentials:

- username: admin
- password: prom-operator

# Open prometheus dashboard in your browser

Point your browser to:

http://promethes.minikube:8080

You should see the Prometheus dashboard.

# Setup monitoring for the olss-demo-app

Go back to the olss-demo-app repo you cloned, cd into `k8s/olss-demo-app` and
run the following command:

```
kubectl apply -f olss-app.servicemonitor.yaml
```

In a while prometheus should start scraping metrics from the demo app.

