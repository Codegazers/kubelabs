
>NOTE:
>For this Lab, we will use [Codegazers K8s-vagrant Quick Environment](https://github.com/Codegazers/k8s-vagrant)

1. Create 2 separated namespaces
~~~
$ kubectl apply -f colors-namespaces.yml
namespace "red-blue" configured
namespace "black-white" configured
~~~

2. Create 4 applications
~~~
$ kubectl apply -f colors-deployments.yml
deployment.extensions "red-app" created
deployment.extensions "blue-app" created
deployment.extensions "black-app" created
deployment.extensions "white-app" created
~~~
3. Deploy NodePort type services
~~~
$ kubectl apply -f colors-services.yml
service "red-svc" created
service "blue-svc" created
service "black-svc" created
service "white-svc" created

$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
black-svc    NodePort    10.101.65.70     <none>        80:32133/TCP   7s
blue-svc     NodePort    10.110.152.243   <none>        80:32055/TCP   7s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        14m
red-svc      NodePort    10.96.82.18      <none>        80:32173/TCP   7s
white-svc    NodePort    10.111.56.137    <none>        80:30879/TCP   6s
~~~

$ kubectl get pods --namespace red-blue

$ kubectl get pods --namespace black-white

$ kubectl get svc --namespace black-white

$ kubectl get svc --namespace red-blue


6. Open a shell in one of the black application pods, for example
~~~
$ kubectl exec -ti $(kubectl get pods --namespace black-white --selector color==black -o name|awk -F "/" 'END { print $2 }') -n black-white -- sh
~~~