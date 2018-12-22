# Kubernetes Storage Lab
## Nginx with Host Local Filesystem Persistence

1. Create a datastore directory in minikube host. In this example we will use /data/storage
    ~~~
    $ minikube ssh sudo mkdir /data/storage
    ~~~

2. Deploy a local Store Class
    ~~~
    $ kubectl create -f local-storage-class.yml                                 
    storageclass.storage.k8s.io "local-storage-class" created
    ~~~
    We can review deployed Store Class
    ~~~   
    $ kubectl get sc
    NAME                  PROVISIONER                    AGE
    local-storage-class   kubernetes.io/no-provisioner   11s
    standard (default)    k8s.io/minikube-hostpath       14h
    ~~~
3. Deploy a Persistent Volume using local-storage-class
    ~~~    
    $ kubectl create -f local-persistvol-10gb.yml 
    persistentvolume "local-persistvol-10gb" created
    ~~~
    We review this 10GB Persisten Volume
    ~~~
    $ kubectl get pv
    NAME                    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS          REASON    AGE
    local-persistvol-10gb   10Gi       RWO            Retain           Available             local-storage-class             10s
    ~~~
4. And now we create a Volume Claim to associate created volume with new pods
    ~~~
    $ kubectl create -f local-persistvol-1gb-claim.yml 
    persistentvolumeclaim "local-persistvol-1gb-claim" created
    ~~~
    And we review this Persistent Volume Claim
    ~~~
    $ kubectl get pvc
    NAME     STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    local-persistvol-1gb-claim   Pending                   local-storage-class   5s
    ~~~
5. We have prepared a deployment that mounts host defined local volume (directory) in /usr/share/nginx/html.
    ~~~
    $ kubectl apply -f deploy-nginx-with-persitvol.yml 
    deployment.apps "nginx-deployment" created

    $ kubectl get pods
    NAME                              READY     STATUS    RESTARTS   AGE
    nginx-deployment-fd6dfd46-6pccp   2/2       Running   0          1m

    ~~~
6. And now volume claim is bounded
    ~~~
    $ kubectl get pvc
    NAME                   STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    local-persistvol-1gb-claim   Bound     local-persistvol-10gb   10Gi       RWO            local-storage-class   39s
    ~~~
7. We now create a service (NodePort) to access our deployed Nginx
    ~~~
    $ kubectl expose deployment nginx-deployment --type=NodePort
    service "nginx-deployment" exposed

    $ kubectl get svc nginx-deployment
    NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    nginx-deployment   NodePort   10.96.228.209   <none>        80:32519/TCP   10s
    ~~~
8. And now we can just access Nginx (remember that we are using minikube...).
    ~~~
    $ curl $(minikube ip):32519
    <html>
    <head><title>403 Forbidden</title></head>
    <body bgcolor="white">
    <center><h1>403 Forbidden</h1></center>
    <hr><center>nginx/1.7.9</center>
    </body>
    </html>
    ~~~
    As expected, we don't have a default landing page, just an empty volume...
9. We create an index.html file on /data/storage (on minikube host).
    ~~~
    $ minikube ssh "echo TEST |sudo tee /data/storage/index.html"

    $ minikube ssh ls /data/storage
    index.html
    ~~~
10. And now we can try again the Nginx service.
    ~~~
    $ curl $(minikube ip):32519
    TEST
    ~~~
11. We can delete the running pod. Kubernetes will create a new one as the deployment needs at least one.
    ~~~
    $ kubectl delete pod $(kubectl get pods --selector app==nginx -o name|awk -F "/" 'END { print $2 }')
    pod "nginx-deployment-fd6dfd46-6pccp" deleted
    ~~~
12. We test again and we get same index.html because of persist volume.
    ~~~
    $ curl $(minikube ip):32519
    TEST
    ~~~

