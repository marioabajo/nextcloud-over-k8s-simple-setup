
- In order to not deal with modifying lot of yaml files for the deployments we will use HELM, think of it as a kind of repository with yaml templates for products + Jinja like variable substitution so it is easy to deploy an application with one command already configured for your environment. Please follow this simple instrunctions to install helm:
  https://helm.sh/docs/intro/install/
  This is client, so install it in the host/s from where you want to control de cluster:

- For our applications we will need storage... but we are lazy so we want the storage to be managed by kubernetes, we want it to be dynamic, isolated (from the rest of containers, what about LVM volumes?)... we are in a single node deployment... looks like a good place to try "topolvm" a simple lvm provisioner for kubernetes, but before installing it, as a dependcy, we need to install "cert-manager":
```
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.16.2 --set crds.enabled=true --set resources.requests.cpu=10m,resources.requests.memory=32Mi,resources.limits.cpu=1,resources.limits.memory=128Mi --set webhook.resources.requests.cpu=10m,webhook.resources.requests.memory=32Mi,webhook.resources.limits.cpu=1,webhook.resources.limits.memory=128Mi --set cainjector.resources.requests.cpu=10m,cainjector.resources.requests.memory=32Mi,cainjector.resources.limits.cpu=1,cainjector.resources.limits.memory=128Mi --set 'extraArgs={--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53}'
```

- Now lets install `topo-lvm`:
  First we need to add th erepositorty and create the namespace for it:
```
kubectl create ns topolvm-system
helm repo add topolvm https://topolvm.github.io/topolvm
helm repo update
```
  The configuration options for this package can be downloaded for review using this command:
```
helm show values topolvm/topolvm | tee topolvm-values.yaml
```
  For example [this](topolvm-values.yaml) is my `topolvm-values.yaml` with the parameters i choose. Plese keep special attention at the ".lvmd.deviceClasses[0].volme-group" parameter which is the name of the volume group that your system should have.
  
  And now lets add a pair of label to two namesapace to avoid the topolvm webhook from getting stuck ([more info here](https://github.com/topolvm/topolvm/blob/main/docs/getting-started.md#install-instructions)) and finally install topolvm:
```
kubectl label namespace topolvm-system topolvm.io/webhook=ignore
kubectl label namespace kube-system topolvm.io/webhook=ignore
helm install --namespace=topolvm-system topolvm topolvm/topolvm -f topolvm-values.yaml
```

This will create a `storage-class` named `topolvm-provisioner` (the name is configurable in the values, check the `.storageClasses` parameter), this way you have have different storage providers and choose which one you want to provide the volume that you need. In our case we only will have this one so we will make it the default also so eveytime we ask for a volume, even without selecting any storage class, this one will provide us the volume:
```
kubectl patch storageclass topolvm-provisioner -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```
You can then list the storage classes and see which one is the default:
```
core@test:~$ kubectl get sc topolvm-provisioner
NAME                            PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
topolvm-provisioner (default)   topolvm.io    Retain          WaitForFirstConsumer   true                   22h
```

Lets try now a small example to create a pod with a persistent volume attached to see how it works, please run this commands
```
echo "---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pv
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pv-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 10000"]
    volumeMounts:
    - mountPath: /data
      name: my-volume
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: test-pv" | kubectl apply -f -
```
This should create the following resources:
```
core@test:~$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          VOLUMEATTRIBUTESCLASS   AGE
test-pv   Bound    pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a   1Gi        RWO            topolvm-provisioner   <unset>                 11s
core@test:~$ kubectl get pv 
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS          VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a   1Gi        RWO            Retain           Bound    default/test-pv   topolvm-provisioner   <unset>                          12s
core@test:~$ kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
test-pv-pod   1/1     Running   0          18s
```
And you can access the pod and verify the disk is mounted:
```
core@test:~$ kubectl exec -ti test-pv-pod -- /bin/sh
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  13.2G      4.0G      8.4G  32% /
tmpfs                    64.0M         0     64.0M   0% /dev
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                   794.1M      2.7M    791.4M   0% /etc/resolv.conf
tmpfs                   794.1M      2.7M    791.4M   0% /etc/hostname
tmpfs                   794.1M      2.7M    791.4M   0% /run/.containerenv
/dev/mapper/test--vg-e1b7adfa--1c90--4eee--8c01--4ad5a51838e9
                       1014.0M     39.3M    974.7M   4% /data
/dev/mapper/test--vg-root
                         13.2G      4.0G      8.4G  32% /etc/hosts
/dev/mapper/test--vg-root
                         13.2G      4.0G      8.4G  32% /dev/termination-log
tmpfs                     7.7G     12.0K      7.7G   0% /var/run/secrets/kubernetes.io/serviceaccount
udev                      3.9G         0      3.9G   0% /proc/kcore
udev                      3.9G         0      3.9G   0% /proc/keys
udev                      3.9G         0      3.9G   0% /proc/timer_list
/ # ls -l /data/
total 0
/ # exit
core@test:~$ sudo lvs
  LV                                   VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  e1b7adfa-1c90-4eee-8c01-4ad5a51838e9 test-vg -wi-ao----  1.00g                                                    
  home                                 test-vg -wi-ao---- <4.77g                                                    
  root                                 test-vg -wi-ao---- 13.46g                                                 
```
Once you are done you can remove the resources to free the resources:
```
core@test:~$ kubectl delete pod/test-pv-pod pvc/test-pv    
pod "test-pv-pod" deleted
persistentvolumeclaim "test-pv" deleted
```
As the storage class has a reclaim policy of `retain` that means that the persistent volumes will be there available so we need to remove them by hand or change the policy:
```
core@test:~$ kubectl delete pv/pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a
persistentvolume "pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a" deleted
```
TopoLvm create an object called "logicalvolume" that represents the system logical volume created by it, after deleting the PV the logical volume will still be there and in we should also clean it up manually:
```
core@test:~$ sudo lvs 
  LV                                   VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  e1b7adfa-1c90-4eee-8c01-4ad5a51838e9 test-vg -wi-a-----  1.00g                                                    
  home                                 test-vg -wi-ao---- <4.77g                                                    
  root                                 test-vg -wi-ao---- 13.46g                                                    
core@test:~$ kubectl get logicalvolumes
NAME                                       AGE
pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a   9m46s
core@test:~$ kubectl delete logicalvolume/pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a
logicalvolume.topolvm.io "pvc-e9d61069-4a9a-471d-be29-9d58798c8c8a" deleted
core@test:~$ sudo lvs
  LV   VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home test-vg -wi-ao---- <4.77g                                                    
  root test-vg -wi-ao---- 13.46g                                                    
```
