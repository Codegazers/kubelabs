
>NOTE:
>For this Lab, we will use [Codegazers K8s-vagrant Quick Environment](https://github.com/Codegazers/k8s-vagrant)

----
1. Create 2 separated namespaces
~~~
$ kubectl apply -f colors-namespaces.yml
namespace "red-blue" configured
namespace "black-white" configured
~~~
----
2. Create 4 applications
~~~
$ kubectl apply -f colors-deployments.yml
deployment.extensions "red-app" created
deployment.extensions "blue-app" created
deployment.extensions "black-app" created
deployment.extensions "white-app" created

$ kubectl get pods --all-namespaces -o wide|grep -v system
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE       IP                NODE
black-white   black-app-558798b74-4svx7                  1/1       Running   0          1m        192.168.200.193   k8s-2
black-white   black-app-558798b74-tgm5r                  1/1       Running   0          1m        192.168.13.67     k8s-3
black-white   white-app-6cfcdcf5b4-l6898                 1/1       Running   0          1m        192.168.13.68     k8s-3
black-white   white-app-6cfcdcf5b4-p5whf                 1/1       Running   0          1m        192.168.200.194   k8s-2
red-blue      blue-app-7bc9476d9-qcjdz                   1/1       Running   0          1m        192.168.200.196   k8s-2
red-blue      blue-app-7bc9476d9-sdv8r                   1/1       Running   0          1m        192.168.13.69     k8s-3
red-blue      red-app-77cbb6774-hds7p                    1/1       Running   0          1m        192.168.200.195   k8s-2
red-blue      red-app-77cbb6774-kl28h                    1/1       Running   0          1m        192.168.13.66     k8s-3
red-blue      red-app-77cbb6774-rkrf6                    1/1       Running   0          1m        192.168.13.65     k8s-3

~~~

As we can see, applications are deployed on 2 different namespaces:
- red-blue
- black-white
----

3. Now, we deploy NodePort type services
~~~
$ kubectl apply -f colors-services.yml
service "red-svc" created
service "blue-svc" created
service "black-svc" created
service "white-svc" created

$ kubectl get svc --all-namespaces -o wide|egrep -v -e system -e default
NAMESPACE     NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE       SELECTOR
black-white   black-svc     NodePort    10.97.64.85      <none>        80:30981/TCP    2m        color=black
black-white   white-svc     NodePort    10.101.178.196   <none>        80:31300/TCP    2m        color=white
red-blue      blue-svc      NodePort    10.99.251.154    <none>        80:32481/TCP    2m        color=blue
red-blue      red-svc       NodePort    10.100.85.51     <none>        80:32640/TCP    2m        color=red

~~~
----
4. From a black application pod, we can access red service, published on port 80:
~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- curl 10.100.85.51/text

APP_VERSION: 1.0
COLOR: red
CONTAINER_NAME: red-app-77cbb6774-rkrf6
CONTAINER_IP:  192.168.13.65
CLIENT_IP: ::ffff:10.0.2.15
CONTAINER_ARCH: linux
~~~
If we issue a new request, we will get another pod, because we deployed 3 replicas for red application.
~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- curl 10.100.85.51/text

APP_VERSION: 1.0
COLOR: red
CONTAINER_NAME: red-app-77cbb6774-kl28h
CONTAINER_IP:  192.168.13.66
CLIENT_IP: ::ffff:10.0.2.15
CONTAINER_ARCH: linux
~~~

Everything is working as expected, even accesing pods without service's cluster IP, using one of the pods for example:
~~~ 
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- curl 192.168.200.195:3000/text

APP_VERSION: 1.0
COLOR: red
CONTAINER_NAME: red-app-77cbb6774-hds7p
CONTAINER_IP:  192.168.200.195
CLIENT_IP: ::ffff:192.168.13.67
CONTAINER_ARCH: linux
~~~
----
5. We now deploy a Network Policy that only allows traffic to red-blue namespace from pods on same namespace.
~~~
$ kubectl apply -f networkpolicy-disallow-non-red-blue-traffic.yml
networkpolicy.networking.k8s.io "deny-from-other-namespaces" created
~~~
----
6. Testing from a blue pod should work, as expected.
~~~
$ kubectl exec -ti $(kubectl get pods --namespace red-blue --selector color==blue -o name|awk -F "/" 'END { print $2 }') -n red-blue -- curl --connect-timeout 5 10.100.85.51/text

APP_VERSION: 1.0
COLOR: red
CONTAINER_NAME: red-app-77cbb6774-rkrf6
CONTAINER_IP:  192.168.13.65
CLIENT_IP: ::ffff:10.0.2.15
CONTAINER_ARCH: linux
~~~
But same requests from black pod, outside of red-blue namespace will not reach any pod.
~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- curl --connect-timeout 5 10.100.85.51/text
curl: (28) Connection timed out after 5001 milliseconds
~~~
----























5. We now deploy a Network Policy that only allows blue traffic to red pods.
~~~
$ kubectl apply -f networkpolicy-red2blue.yml
networkpolicy.networking.k8s.io "networkpolicy-red2blue" created
~~~
----




6. And now we will test everything again.
>NOTE: We added _--connect-timeout 5_ to curl to get a quick response.

~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- curl --connect-timeout 5 10.100.85.51/text
curl: (28) Connection timed out after 5001 milliseconds
~~~
And same results accesing a red pod.
~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- curl --connect-timeout 5 192.168.200.195:3000/text
curl: (28) Connection timed out after 5000 milliseconds
~~~
7. We now try from a blue pod (which is allowed as defined on NetworkPolicy).



$ kubectl get pods --namespace red-blue

$ kubectl get pods --namespace black-white

$ kubectl get svc --namespace black-white

$ kubectl get svc --namespace red-blue


6. Open a shell in one of the black application pods, for example
~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- sh
~~~
