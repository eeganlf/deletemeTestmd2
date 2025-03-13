## Exercise 10.1: Deploy A New Service

### Overview

Services (also called microservices) are objects which declare a policy to access a logical set of Pods. They are typically assigned with labels to allow persistent access to a resource, when front or back end containers are terminated and replaced.

Native applications can use the Endpoints API for access. Non-native applications can use a Virtual IP-based bridge to access back end pods. Service Types Type could be:

- ClusterIP default - exposes on a cluster-internal IP. Only reachable within cluster
- NodePort Exposes node IP at a static port. A ClusterIP is also automatically created.
- LoadBalancer Exposes service externally using cloud providers load balancer. NodePort and ClusterIP automatically created.
- ExternalName Maps service to contents of externalName using a CNAME record.

We use services as part of decoupling such that any agent or object can be replaced without interruption to access from client to back end application.

1.  Deploy two nginx servers using kubectl and a new .yaml file. The kind should be Deployment and label it with nginx.
Create two replicas and expose port 8080. What follows is a well documented file. There is no need to include the
comments when you create the file. This file can also be found among the other examples in the tarball.

```bash
student@cp: ̃$ cp /home/student/LFS458/SOLUTIONS/s_10/nginx-one.yaml .
student@cp: ̃$ vim nginx-one.yaml
```

```yaml
apiVersion: apps/v1
# Determines YAML versioned schema.
kind: Deployment
# Describes the resource defined in this file.
metadata:
  name: nginx-one
  labels:
    system: secondary
# Required string which defines object within namespace.
namespace: accounting
# Existing namespace resource will be deployed into.
spec:
  selector:
    matchLabels:
      system: secondary
# Declaration of the label for the deployment to manage
  replicas: 2
# How many Pods of following containers to deploy
  template:
    metadata:
      labels:
        system: secondary
    spec:
      containers:
        # Array of objects describing containerized application with a Pod.
        # Referenced with shorthand spec.template.spec.containers
        - image: nginx:1.20.1
          # The Docker image to deploy
          imagePullPolicy: Always
          name: nginx
          # Unique name for each container, use local or Docker repo image
          ports:
            - containerPort: 8080
              protocol: TCP
          # Optional resources this container may need to function.
          nodeSelector:
            system: secondOne
            # One method of node affinity.
```

2.  View the existing labels on the nodes in the cluster.

```bash
student@cp: ̃$ kubectl get nodes --show-labels
```

```
NAME     STATUS   ROLES           AGE   VERSION   LABELS
cp       Ready    control-plane   14h   v1.31.1   beta.kubernetes.io/arch=amd64,
beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cp,
kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,
node.kubernetes.io/exclude-from-external-load-balancers=
worker   Ready    <none>          14h   v1.31.1   beta.kubernetes.io/arch=amd64,
beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker,
kubernetes.io/os=linux
```

3.  Run the following command and look for the errors. Assuming there is no typo, you should have gotten an error about
about the accounting namespace.

```bash
student@cp: ̃$ kubectl create -f nginx-one.yaml
```

```
Error from server (NotFound): error when creating
"nginx-one.yaml": namespaces "accounting" not found
```

4.  Create the namespace and try to create the deployment again. There should be no errors this time.

```bash
student@cp: ̃$ kubectl create ns accounting
```

```
namespace/accounting" created
```

```bash
student@cp: ̃$ kubectl create -f nginx-one.yaml
```

```
deployment.apps/nginx-one created
```

5.  View the status of the new pods. Note they do not show a Running status.

```bash
student@cp: ̃$ kubectl -n accounting get pods
```

```
NAME                         READY     STATUS    RESTARTS   AGE
nginx-one-74dd9d578d-fcpmv   0/1       Pending   0          4m
nginx-one-74dd9d578d-r2d67   0/1       Pending   0          4m
```

6.  View the node each has been assigned to (or not) and the reason, which shows under events at the end of the output.

```bash
student@cp: ̃$ kubectl -n accounting describe pod nginx-one-74dd9d578d-fcpmv
```

```
Name:           nginx-one-74dd9d578d-fcpmv
Namespace:      accounting
Node:           <none>
...
Events:
Type     Reason            Age         From      ....
----     ------            ----        ----
Warning  FailedScheduling  <unknown>   default-scheduler
0/2 nodes are available: 2 node(s) didn't match node selector.
```

7.  Label the secondary node. Note the value is case sensitive. Verify the labels.

```bash
student@cp: ̃$ kubectl label node <worker_node_name> system=secondOne
```

```
node/worker labeled
```

```bash
student@cp: ̃$ kubectl get nodes --show-labels
```

```
NAME     STATUS   ROLES           AGE   VERSION   LABELS
cp       Ready    control-plane   14h   v1.31.1   beta.kubernetes.io/arch=amd64,
beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cp,
kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,
node.kubernetes.io/exclude-from-external-load-balancers=
worker   Ready    <none>          14h   v1.31.1   beta.kubernetes.io/arch=amd64,
beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker,
kubernetes.io/os=linux
```

8.  View the pods in the accounting namespace. They may still show as Pending. Depending on how long it has been
since you attempted deployment the system may not have checked for the label. If the Pods show Pending after a
minute delete one of the pods. They should both show as Running after a deletion. A change in state will cause the
Deployment controller to check the status of both Pods.

```bash
student@cp: ̃$ kubectl -n accounting get pods
```

```
NAME                         READY     STATUS    RESTARTS   AGE
nginx-one-74dd9d578d-fcpmv   1/1       Running   0          10m
nginx-one-74dd9d578d-sts5l   1/1       Running   0          3s
```

9.  View Pods by the label we set in the YAML file. If you look back the Pods were given a label of app=nginx.

```bash
student@cp: ̃$ kubectl get pods -l system=secondary --all-namespaces
```

```
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE
accounting  nginx-one-74dd9d578d-fcpmv  1/1     Running   0          20m
accounting  nginx-one-74dd9d578d-sts5l  1/1     Running   0          9m
```

10. Recall that we exposed port 8080 in the YAML file. Expose the new deployment.

```bash
student@cp: ̃$ kubectl -n accounting expose deployment nginx-one
```

```
service/nginx-one exposed
```

11. View the newly exposed endpoints. Note that port 8080 has been exposed on each Pod.

```bash
student@cp: ̃$ kubectl -n accounting get ep nginx-one
```

```
NAME        ENDPOINTS                           AGE
nginx-one   192.168.1.72:8080,192.168.1.73:8080   47s
```

12. Attempt to access the Pod on port 8080, then on port 80. Even though we exposed port 8080 of the container the
application within has not been configured to listen on this port. The nginx server listens on port 80 by default. A curl
command to that port should return the typical welcome page.

```bash
student@cp: ̃$ curl 192.168.1.72:8080
```

```
curl: (7) Failed to connect to 192.168.1.72 port 8080: Connection refused
```

```bash
student@cp: ̃$ curl 192.168.1.72:80
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

13. Delete the deployment. Edit the YAML file to expose port 80 and create the deployment again.

```bash
student@cp: ̃$ kubectl -n accounting delete deploy nginx-one
```

```
deployment.apps "nginx-one" deleted
```

```bash
student@cp: ̃$ vim nginx-one.yaml
```

```yaml
1....
2ports:
3- containerPort: 8080     #<-- Edit this line
4protocol: TCP
5....
6
```

```bash
student@cp: ̃$ kubectl create -f nginx-one.yaml
```

```
deployment.apps/nginx-one created
```

## Exercise 10.2: Configure a NodePort

In a previous exercise we deployed a LoadBalancer which deployed a ClusterIP and NodePort automatically. In this exercise
we will deploy a NodePort. While you can access a container from within the cluster, one can use a NodePort to NAT traffic
from outside the cluster. One reason to deploy a NodePort instead, is that a LoadBalancer is also a load balancer resource
from cloud providers like GKE and AWS.

1.  In a previous step we were able to view the nginx page using the internal Pod IP address. Now expose the deployment
using the --type=NodePort. We will also give it an easy to remember name and place it in the accounting namespace.
We could pass the port as well, which could help with opening ports in the firewall.

```bash
student@cp: ̃$ kubectl -n accounting expose deployment nginx-one --type=NodePort --name=service-lab
```

```
service/service-lab exposed
```

2.  View the details of the services in the accounting namespace. We are looking for the autogenerated port.

```bash
student@cp: ̃$ kubectl -n accounting describe services
```

```
....
NodePort:                 <unset>  32103/TCP
....
```

3.  Locate the exterior facing hostname or IP address of the cluster. The lab assumes use of GCP nodes, which we access
via a FloatingIP, we will first check the internal only public IP address. Look for the Kubernetes cp URL. Whichever
way you access check access using both the internal and possible external IP address

```bash
student@cp: ̃$ kubectl cluster-info
```

```
Kubernetes control plane is running at https://k8scp:6443
CoreDNS is running at https://k8scp:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

4.  Test access to the nginx web server using the combination of cp URL and NodePort.

```bash
student@cp: ̃$ curl http://k8scp:32103
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

5.  Using the browser on your local system, use the public IP address you use to SSH into your node and the port. You
should still see the nginx default page. You may be able to use curl to locate your public IP address.

```bash
student@cp: ̃$ curl ifconfig.io
```

```
104.198.192.84
```

## Exercise 10.3: Working with CoreDNS

1.  We can leverage CoreDNS and predictable hostnames instead of IP addresses. A few steps back we created the
service-lab NodePort in the Accounting namespace. We will create a new pod for testing using Ubuntu. The pod
name will be named ubuntu.

```bash
student@cp: ̃$ cp /home/student/LFS458/SOLUTIONS/s_10/nettool.yaml .
student@cp: ̃$ vim nettool.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
name: ubuntu
spec:
containers:
- name: ubuntu
image: ubuntu:latest
command: [
"sleep" ]
args: ["infinity" ]
```

2.  Create the pod and then log into it.

```bash
student@cp: ̃$ kubectl create -f nettool.yaml
```

```
pod/ubuntu created
```

```bash
student@cp: ̃$ kubectl exec -it ubuntu -- /bin/bash
```

On Container

(a) Add some tools for investigating DNS and the network. The installation will ask you the geographic area
and timezone information. Someone in Austin would first answer 2. America, then 37 for Chicago, which
would be central time

```bash
root@ubuntu:/#  apt-get update ; apt-get install curl dnsutils -y
```

(b) Use the dig command with no options. You should see root name servers, and then information about the
DNS server responding, such as the IP address.

```bash
root@ubuntu:/# dig
```

```
; <<>> DiG 9.16.1-Ubuntu <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3394
```

(c) Also take a look at the /etc/resolv.conf file, which will indicate nameservers and default domains to search
if no using a Fully Qualified Distinguished Name (FQDN). From the output we can see the first entry is
default.svc.cluster.local..

```bash
root@ubuntu:/# cat /etc/resolv.conf
```

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
c.endless-station-188822.internal google.internal
options ndots:5
```

(d) Use the dig command to view more information about the DNS server. Us the -x argument to get the
FQDN using the IP we know. Notice the domain name, which uses .kube-system.svc.cluster.local.,
to match the pod namespaces instead of default. Also note the name, kube-dns, is the name of a service
not a pod.

```bash
root@ubuntu:/# dig @10.96.0.10 -x 10.96.0.10
```

```
...
;; QUESTION SECTION:
;10.0.96.10.in-addr.arpa.        IN        PTR
;; ANSWER SECTION:
10.0.96.10.in-addr.arpa.
30        IN        PTR        kube-dns.kube-system.svc.cluster.local.,→
;; Query time: 0 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Thu Aug 27 23:39:14 CDT 2024
;; MSG SIZE  rcvd: 139
```

(e) Recall the name of the service-lab service we made and the namespaces it was created in. Use this
information to create a FQDN and view the exposed pod.

```bash
root@ubuntu:/# curl service-lab.accounting.svc.cluster.local.
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
body {
width: 35em;
margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif;
}
...
```

(f) Attempt to view the default page using just the service name. It should fail as nettool is in the default
namespace.

```bash
root@ubuntu:/# curl service-lab
```

```
curl: (6) Could not resolve host: service-lab
```

(g) Add the accounting namespaces to the name and try again. Traffic can access a service using a name,
even across different namespaces.

```bash
root@ubuntu:/# curl service-lab.accounting
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

(h) Exit out of the container and look at the services running inside of the kube-system namespace. From the
output we see that the kube-dns service has the DNS server IP, and exposed ports DNS uses.

```bash
root@ubuntu:/# exit
```

```bash
student@cp: ̃$ kubectl -n kube-system get svc
```

```
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   42h
```

3.  Examine the service in detail. Among other information notice the selector in use to determine the pods the service
communicates with.

```bash
student@cp: ̃$ kubectl -n kube-system get svc kube-dns -o yaml
```

```yaml
...
labels:
k8s-app: kube-dns
kubernetes.io/cluster-service: "true"
kubernetes.io/name: CoreDNS
...
selector:
k8s-app: kube-dns
sessionAffinity: None
type: ClusterIP
...
```

4.  Find pods with the same labels in all namespaces. We see that infrastructure pods all have this label, including coredns.

```bash
student@cp: ̃$ kubectl get pod -l k8s-app --all-namespaces
```

```
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   cilium-5tv9d               1/1     Running   0          136m
kube-system   cilium-gzdk6               1/1     Running   0          54m
kube-system   coredns-5d78c9869d-44qvq   1/1     Running   0          31m
kube-system   coredns-5d78c9869d-j6tqx   1/1     Running   0          31m
kube-system   kube-proxy-lpsmq           1/1     Running   0          35m
kube-system   kube-proxy-pvl8w           1/1     Running   0          34m
```

5.  Look at the details of one of the coredns pods. Read through the pod spec and find the image in use as well as any
configuration information. You should find that configuration comes from a configmap.

```bash
student@cp: ̃$ kubectl -n kube-system get pod coredns-f9fd979d6-4dxpl -o yaml
```

```yaml
...
spec:
containers:
- args:
- -conf
- /etc/coredns/Corefile
image: k8s.gcr.io/coredns:1.7.0
...
volumeMounts:
- mountPath: /etc/coredns
name: config-volume
readOnly: true
...
volumes:
- configMap:
defaultMode: 420
items:
- key: Corefile
path: Corefile
name: coredns
name: config-volume
...
```

6.  View the configmaps in the kube-system namespace.

```bash
student@cp: ̃$ kubectl -n kube-system get configmaps
```

```
NAME                                 DATA   AGE
cilium-config                        4      43h
coredns                              1      43h
extension-apiserver-authentication   6      43h
kube-proxy                           2      43h
kubeadm-config                       2      43h
kubelet-config                       1      43h
```

7.  View the details of the coredns configmap. Note the cluster.local domain is listed.

```bash
student@cp: ̃$ kubectl -n kube-system get configmaps coredns -o yaml
```

```yaml
apiVersion: v1
data:
Corefile: |
.:53 {
errors
health {
lameduck 5s
}
ready
kubernetes cluster.local in-addr.arpa ip6.arpa {
pods insecure
fallthrough in-addr.arpa ip6.arpa
ttl 30
}
prometheus :9153
forward . /etc/resolv.conf {
max_concurrent 1000
}
cache 30
loop
reload
loadbalance
}
kind: ConfigMap
...
```

8.  It is very important to backup our resources before we make changes to it.

```bash
student@cp: ̃$ kubectl -n kube-system get configmaps coredns -o yaml > coredns-backup.yaml
```

9.  While there are many options and zone files we could configure, lets start with simple edit. Add a rewrite statement such
that test.io will redirect to cluster.local More about each line can be found at coredns.io.

```bash
student@cp: ̃$ kubectl -n kube-system edit configmaps coredns
```

```yaml
apiVersion: v1
data:
Corefile: |
.:53 {
rewrite name regex (.*)\.test\.io {1}.default.svc.cluster.local   #<-- Add this line
errors
health {
lameduck 5s
}
ready
kubernetes cluster.local in-addr.arpa ip6.arpa {
pods insecure
fallthrough in-addr.arpa ip6.arpa
ttl 30
}
prometheus :9153
forward . /etc/resolv.conf {
max_concurrent 1000
}
cache 30
loop
reload
loadbalance
}
```

10. Delete the coredns pods causing them to re-read the updated configmap.

```bash
student@cp: ̃$ kubectl -n kube-system delete pod coredns-f9fd979d6-s4j98 coredns-f9fd979d6-xlpzf
```

```
pod "coredns-f9fd979d6-s4j98" deleted
pod "coredns-f9fd979d6-xlpzf" deleted
```

11. Create a new web server and create a ClusterIP service to verify the address works. Note the new service IP to start
with a reverse lookup.

```bash
student@cp: ̃$ kubectl create deployment nginx --image=nginx
```

```
deployment.apps/nginx created
```

```bash
student@cp: ̃$ kubectl expose deployment nginx --type=ClusterIP --port=80
```

```
service/nginx expose
```

```bash
student@cp: ̃$ kubectl get svc
```

```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   3d15h
nginx        ClusterIP   10.104.248.141   <none>        80/TCP    7s
```

12. Log into the ubuntu container and test the URL rewrite starting with the reverse IP resolution.

```bash
student@cp: ̃$ kubectl exec -it ubuntu -- /bin/bash
```

On Container

(a) Use the dig command. Note that the service name becomes part of the FQDN.

```bash
root@ubuntu:/# dig -x 10.104.248.141
```

```
....
;; QUESTION SECTION:
;141.248.104.10.in-addr.arpa.        IN        PTR
;; ANSWER SECTION:
141.248.104.10.in-addr.arpa.
30        IN        PTR        nginx.default.svc.cluster.local.,→
....
```

(b) Now that we have the reverse lookup test the forward lookup. The IP should match the one we used in the
previous step.

```bash
root@ubuntu:/# dig nginx.default.svc.cluster.local.
```

```
....
;; QUESTION SECTION:
;nginx.default.svc.cluster.local. IN        A
;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN        A        10.104.248.141
....
```

(c) Now test to see if the rewrite rule for the test.io domain we added resolves the IP. Note the response
uses the original name, not the requested FQDN.

```bash
root@ubuntu:/# dig nginx.test.io
```

```
....
;; QUESTION SECTION:
;nginx.test.io.                        IN        A
;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN        A        10.104.248.141
....
```

13. Exit out of the container then edit the configmap to add an answer section.

```bash
student@cp: ̃$ kubectl -n kube-system edit configmaps coredns
```

```yaml
....
data:
Corefile: |
.:53 {
rewrite stop {                        #<-- Edit this and following two lines
name regex (.*)\.test\.io {1}.default.svc.cluster.local
answer name (.*)\.default\.svc\.cluster\.local {1}.test.io
}
errors
health {
....
```

14. Delete the coredns pods again to ensure they re-read the updated configmap.

```bash
student@cp: ̃$ kubectl -n kube-system delete pod coredns-f9fd979d6-fv9qn coredns-f9fd979d6-lnxn5
```

```
pod "coredns-f9fd979d6-fv9qn" deleted
pod "coredns-f9fd979d6-lnxn5" deleted
```

15. Log into the ubuntu container again. This time the response should show the FQDN with the requested FQDN.

```bash
student@cp: ̃$ kubectl exec -it ubuntu -- /bin/bash
```

On Container

```bash
root@ubuntu:/# dig nginx.test.io
```

```
....
;; QUESTION SECTION:
;nginx.test.io.                        IN        A
;; ANSWER SECTION:
nginx.test.io.                30        IN        A        10.104.248.141
....
```

16. Exit then delete the DNS test tools container to recover the resources.

```bash
student@cp: ̃$ kubectl delete -f nettool.yaml
```

## Exercise 10.4: Use Labels to Manage Resources

1.  Try to delete all Pods with the system=secondary label, in all namespaces.

```bash
student@cp: ̃$ kubectl delete pods -l system=secondary \
--all-namespaces
```

```
pod "nginx-one-74dd9d578d-fcpmv" deleted
pod "nginx-one-74dd9d578d-sts5l" deleted
```

2.  View the Pods again. New versions of the Pods should be running as the controller responsible for them continues.

```bash
student@cp: ̃$ kubectl -n accounting get pods
```

```
NAME                         READY     STATUS    RESTARTS   AGE
nginx-one-74dd9d578d-ddt5r   1/1       Running   0          1m
nginx-one-74dd9d578d-hfzml   1/1       Running   0          1m
```

3.  We also gave a label to the deployment. View the deployment in the accounting namespace.

```bash
student@cp: ̃$ kubectl -n accounting get deploy --show-labels
```

```
NAME       READY  UP-TO-DATE  AVAILABLE  AGE  LABELS
nginx-one  2/2    2           2          10m  system=secondary
```

4.  Delete the deployment using its label.

```bash
student@cp: ̃$ kubectl -n accounting delete deploy -l system=secondary
```

```
deployment.apps "nginx-one" deleted
```

5.  Remove the label from the secondary node. Note that the syntax is a minus sign directly after the key you want to
remove, or system in this case.

```bash
student@cp: ̃$ kubectl label node worker system-
```

```
node/worker unlabeled
