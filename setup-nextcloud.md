As in previous cases, there is a "nextcloud-values.yaml" file containing all the needed configuration parameters.
Some relevant parameters to change:
  - `.persistence.storageClass` -> "topolvm-provisoner"
  - `.persistence.size` -> size of the nextcloud storage
  - `.redis.password` -> choose yours/random
  - `.redis.global.storageClass` -> "topolvm-provisoner"
  - `.mariadb.auth.rootPassword` -> choose yours/random
  - `.mariadb.auth.password` -> choose yours/random
  - `.mariadb.primary.persistence.storageClass` -> "topolvm-provisoner"
  - `.mariadb.primary.persistence.size` -> size of the database
  - `.nextcloud.trustedDomains` -> hostnames that nextcloud will identify as valid
  - `.nextcloud.username` -> username or admin for example
  - `.nextcloud.password` -> password for your username or admin
  - `.nextcloud.host` -> main hostname of the netcloud instance

When you are more comfortable with the configuration options you can for example add email configuration so nextcloud can send information emails, better adjust the resources requests and limits to better adjust to your deployment or tune the DB options, or change the storage to object storage... don't get too mad with the hell of parameters and try small changes in a test enviroment before doing the final deployment.

Now the installation steps:

```
helm repo add nextcloud https://nextcloud.github.io/helm/
helm install nextcloud nextcloud/nextcloud -f nextcloud-values.yaml --create-namespace -n nextcloud
```

We need to create the ingress object to allow external connections:
```
kubectl -n nextcloud create ingress nextcloud --class=haproxy --rule="nextcloud.local.net/*=nextcloud:8080"
```

Once the deployment is done you can access nextcloud interface:
```
core@test:~$ helm install nextcloud nextcloud/nextcloud -f nextcloud-values.yaml --create-namespace -n nextcloud
NAME: nextcloud
LAST DEPLOYED: Tue Jun 24 12:29:20 2025
NAMESPACE: nextcloud
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the nextcloud URL by running:

  export POD_NAME=$(kubectl get pods --namespace nextcloud -l "app.kubernetes.io/name=nextcloud" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080/
  kubectl port-forward --namespace nextcloud $POD_NAME 8080:80

2. Get your nextcloud login credentials by running:

  echo User:     admin
  echo Password: $(kubectl get secret --namespace nextcloud nextcloud -o jsonpath="{.data.nextcloud-password}" | base64 --decode)
core@test:~$ kubectl -n nextcloud get pods   
NAME                         READY   STATUS    RESTARTS   AGE
nextcloud-8597db65bd-lb7rw   2/2     Running   0          68m
nextcloud-mariadb-0          1/1     Running   0          136m
nextcloud-redis-master-0     1/1     Running   0          136m
```

Now its time to see the results and access from your browser with: `http://nextcloud.local.net` and login with "admin" user and the password set in the `nextcloud-values.yaml"
