In order to allow petitions to enter the cluster we need an entry point, in this case i choose an haproxy service that will create an entry in its configuration for every service that will accept petitions from outside of the "cluster".

The installation is pretty simple, fist add the repository:
```
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
```

Create the file with values as we did before with topolvm, verify/adapt them to your needs, ([here is](haproxy-ingress-values.yaml) an example configuration that i used), and install it:
```
helm install haproxy-ingress haproxy-ingress/haproxy-ingress --create-namespace --namespace ingress-controller --version 0.14.7 -f haproxy-ingress-values.yaml
```

You can verify the installation for example with these commands:
```
core@test:~$ kubectl -n ingress-controller get pods
NAME                    READY   STATUS    RESTARTS   AGE
haproxy-ingress-dwgt6   1/1     Running   0          3m5s
core@test:~$ sudo netstat -tunelp | grep haproxy     
tcp        0      0 0.0.0.0:1936            0.0.0.0:*               LISTEN      0          1045440    47193/haproxy       
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      0          1045439    47193/haproxy       
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      0          1045438    47193/haproxy       
tcp        0      0 0.0.0.0:10253           0.0.0.0:*               LISTEN      0          1045441    47193/haproxy       
tcp6       0      0 :::10254                :::*                    LISTEN      0          1044387    47174/haproxy-ingre 
```

Now every petition that is directed to your host ip and port 443/tcp (https) or 80/tcp (http) can be directed to a pod depending on the service name. Lets try a simple example to demostrate it with a simple example:
```
echo "---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP" | kubectl apply -f -
```
This will create a service only accesible inside the cluster, but to expose it through the ingress you need to create the ingress object, in this case we have selected a hostname called `nginx.local.net` that exists in our DNS and points to the node IP (well... that would be great, the reallity is that the entry is in our /etc/hosts :) )
```
kubectl create ingress nginx-ingress --class=haproxy --rule="nginx.local.net/=nginx-service:80"
```
```
core@test:~$ grep nginx /etc/hosts
192.168.122.116 test.local.net       test    nginx.local.net 
core@test:~$ curl http://nginx.local.net/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...
```

Once you are done with the testing you can delete the resources:
```
kubectl delete deployment/nginx-deployment service/nginx-service ingress/nginx-ingress
```

