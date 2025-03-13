## Exercise 5.1: Configuring TLS Access

### Overview

Using the Kubernetes API, `kubectl` makes API calls for you. With the appropriate TLS keys you could run `curl` as well use a `golang` client. Calls to the `kube-apiserver` get or set a PodSpec, or desired state. If the request represents a new state the Kubernetes Control Plane will update the cluster until the current state matches the specified state. Some end states may require multiple requests. For example, to delete a ReplicaSet, you would first set the number of replicas to zero, then delete the ReplicaSet.

An API request must pass information as JSON. `kubectl` converts .yaml to JSON when making an API request on your behalf. The API request has many settings, but must include apiVersion, kind and metadata, and spec settings to declare what kind of container to deploy. The spec fields depend on the object being created.

We will begin by configuring remote access to the `kube-apiserver` then explore more of the API.

1. Begin by reviewing the `kubectl` configuration file. We will use the three certificates and the API server address.

```bash
student@cp:~$ less $HOME/.kube/config
```

```
<output_omitted>
```

2. We will create a variables using certificate information. You may want to double-check each parameter as you set it. Begin with setting the `client-certificate-data` key.

```bash
student@cp:~$ export client=$(grep client-cert $HOME/.kube/config |cut -d" " -f 6)
student@cp:~$ echo $client
```

```
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJ
BZ0lJRy9wbC9rWEpNdmd3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0
ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB4TnpFeU1UTXhOelEyTXpKY
UZ3MHhPREV5TVRNeE56UTJNelJhTURReApGekFWQmdOVkJBb1REbk41YzNS
<output_omitted>
```

3. Almost the same command, but this time collect the `client-key-data` as the `key` variable.

```bash
student@cp:~$ export key=$(grep client-key-data $HOME/.kube/config |cut -d " " -f 6)
student@cp:~$ echo $key
```

```
<output_omitted>
```

4. Finally set the `auth` variable with the `certificate-authority-data` key.

```bash
student@cp:~$ export auth=$(grep certificate-authority-data $HOME/.kube/config |cut -d " " -f 6)
student@cp:~$ echo $auth
```

```
<output_omitted>
```

5. Now encode the keys for use with `curl`.

```bash
student@cp:~$ echo $client | base64 -d - > ./client.pem
student@cp:~$ echo $key | base64 -d - > ./client-key.pem
student@cp:~$ echo $auth | base64 -d - > ./ca.pem
```

6. Pull the API server URL from the config file. Your hostname or IP address may be different.

```bash
student@cp:~$ kubectl config view |grep server
```

```
server: https://k8scp:6443
```

7. Use `curl` command and the encoded keys to connect to the API server. Use your hostname, or IP, found in the previous command, which may be different than the example below.

```bash
student@cp:~$ curl --cert ./client.pem \
--key ./client-key.pem \
--cacert ./ca.pem \
https://k8scp:6443/api/v1/pods
```

```
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods",
    "resourceVersion": "239414"
  },
<output_omitted>
```

8. If the previous command was successful, create a JSON file to create a new pod. Remember to use `find` and search for this file in the tarball output, it can save you some typing.

```bash
student@cp:~$ cp /home/student/LFS458/SOLUTIONS/s_05/curlpod.json .
student@cp:~$ vim curlpod.json
```

```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata":{
    "name": "curlpod",
    "namespace": "default",
    "labels": {
      "name": "examplepod"
    }
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx",
      "ports": [{"containerPort": 80}]
    }]
  }
}
```

9. The previous `curl` command can be used to build a `XPOST` API call. There will be a lot of output, including the scheduler and taints involved. Read through the output. In the last few lines the phase will probably show `Pending`, as it's near the beginning of the creation process.

```bash
student@cp:~$ curl --cert ./client.pem \
--key ./client-key.pem --cacert ./ca.pem \
https://k8scp:6443/api/v1/namespaces/default/pods \
-XPOST -H'Content-Type: application/json'\
-d@curlpod.json
```

```
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "curlpod",
<output_omitted>
```

10. Verify the new pod exists and shows a `Running` status.

```bash
student@cp:~$ kubectl get pods
```

```
NAME       READY     STATUS    RESTARTS   AGE
curlpod    1/1       Running   0          45s
```

## Exercise 5.2: Explore API Calls

1. One way to view what a command does on your behalf is to use `strace`. In this case, we will look for the current endpoints, or targets of our API calls. Install the tool, if not present.

```bash
student@cp:~$ sudo apt-get install -y strace
student@cp:~$ kubectl get endpoints
```

```
NAME         ENDPOINTS          AGE
kubernetes   10.128.0.3:6443    3h
```

2. Run this command again, preceded by `strace`. You will get a lot of output. Near the end you will note several `openat` functions to a local directory, `/home/student/.kube/cache/discovery/k8scp_6443`. If you cannot find the lines, you may want to redirect all output to a file and `grep` for them. This information is cached, so you may see some differences should you run the command multiple times. As well your IP address may be different.

```bash
student@cp:~$ strace kubectl get endpoints
```

```
execve("/usr/bin/kubectl", ["kubectl", "get", "endpoints"], [/*....
....
openat(AT_FDCWD, "/home/student/.kube/cache/discovery/k8scp_6443..
<output_omitted>
```

3. Change to the parent directory and explore. Your endpoint IP will be different, so replace the following with one suited to your system.

```bash
student@cp:~$ cd /home/student/.kube/cache/discovery/
student@cp:~/.kube/cache/discovery$ ls
```

```
k8scp_6443
```

```bash
student@cp:~/.kube/cache/discovery$ cd k8scp_6443/
```

4. View the contents. You will find there are directories with various configuration information for kubernetes.

```bash
student@cp:~/.kube/cache/discovery/k8scp_6443$ ls
```

```
admissionregistration.k8s.io  certificates.k8s.io           node.k8s.io
apiextensions.k8s.io          coordination.k8s.io           policy
apiregistration.k8s.io        cilium.io                     rbac.authorization.k8s.io
apps                          discovery.k8s.io              scheduling.k8s.io
authentication.k8s.io         events.k8s.io                 servergroups.json
authorization.k8s.io          extensions                    storage.k8s.io
autoscaling                   flowcontrol.apiserver.k8s.io  v1
batch                         networking.k8s.io
```

5. Use the find command to list out the subfiles. The prompt has been modified to look better on this page.

```bash
student@cp:./k8scp_6443$ find .
```

```
.
./storage.k8s.io
./storage.k8s.io/v1beta1
./storage.k8s.io/v1beta1/serverresources.json
./storage.k8s.io/v1
./storage.k8s.io/v1/serverresources.json
./rbac.authorization.k8s.io
<output_omitted>
```

6. View the objects available in version 1 of the API. For each object, or kind:, you can view the verbs or actions for that object, such as create seen in the following example. Note the prompt has been truncated for the command to fit on one line. Some are HTTP verbs, such as GET, others are product specific options, not standard HTTP verbs. The command may be `python`, depending on what version is installed.

```bash
student@cp:.$ python3 -m json.tool v1/serverresources.json
```

```json
serverresources.json
1{
2"apiVersion":"v1",
3"groupVersion":"v1",
4"kind":"APIResourceList",
5"resources": [
6{
7"kind":
"Binding",
8"name":"bindings",
9"namespaced":true,
10"singularName":"",
11"verbs": [
12"create"
13]
14},
15<output_omitted>
16
```

7. Some of the objects have `shortNames`, which makes using them on the command line much easier. Locate the `shortName` for endpoints.

```bash
student@cp:.$ python3 -m json.tool v1/serverresources.json | less
```

```json
serverresources.json
1....
2{
3"kind":"Endpoints",
4"name":"endpoints",
5"namespaced":true,
6"shortNames": [
7
"ep"
8],
9"singularName":"",
10"verbs": [
11"create",
12"delete",
13....
14
```

8. Use the `shortName` to view the endpoints. It should match the output from the previous command.

```bash
student@cp:.$ kubectl get ep
```

```
NAME         ENDPOINTS          AGE
kubernetes   10.128.0.3:6443    3h
```

9. We can see there are 37 objects in version 1 file.

```bash
student@cp:.$ python3 -m json.tool v1/serverresources.json | grep kind
```

```
"kind": "APIResourceList",
"kind": "Binding",
"kind": "ComponentStatus",
"kind": "ConfigMap",
"kind": "Endpoints",
"kind": "Event",
<output_omitted>
```

10. Looking at another file we find nine more.

```bash
student@cp:$ python3 -m json.tool apps/v1/serverresources.json | grep kind
```

```
"kind": "APIResourceList",
"kind": "ControllerRevision",
"kind": "DaemonSet",
"kind": "DaemonSet",
"kind": "Deployment",
<output_omitted>
```

11. Delete the `curlpod` to recoup system resources.

```bash
student@cp:$ kubectl delete po curlpod
```

```
pod "curlpod" deleted
```

12. Take a look around the other files in this directory as time permits.
