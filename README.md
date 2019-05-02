# storm-kube
Kubernetes managed Storm/Trident docker cluster

# Storm example

Following this example, you will create a functional [Apache
Storm](http://storm.apache.org/) cluster using Kubernetes and
[Docker](http://docker.io).

You will setup an [Apache ZooKeeper](http://zookeeper.apache.org/)
service, a Storm master service (a.k.a. Nimbus server), and a set of
Storm workers (a.k.a. supervisors).

For the impatient expert, jump straight to the [tl;dr](#tldr)
section.

### Sources

Storm source is freely available at:
* Docker image - https://hub.docker.com/_/storm/

## Step Zero: Prerequisites

This example assumes you have a Kubernetes cluster installed and
running, and that you have installed the ```kubectl``` command line
tool somewhere in your path. See next section on how to get a functioning cluster with Vagrant

## Setup k8s cluster in a Vagrant VM
If you have VirtualBox and Vagrant installed, get the Vagrantfile from: 

https://gist.github.com/ImranSuhailCubic/2866342487688b8361101bbf0e0f671f 

Save it in a new directory for the VM (/vagrantdir)

Go to /vagrantdir and run "vagrant up"

SSH into the VM with "vagrant ssh"

Run the next two commands to start weave net and to untaint the nodes:
```shell
kubectl apply -f https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')
```
```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```
### Start storm cluster
```shell
cd /vagrant
cd storm-kube/
./setup.sh
```
Wait for 6-10 minutes for setup.sh to finish creating services and pods

You will see output similar to this:
```shell
NAME                                READY   STATUS              RESTARTS   AGE
pod/nimbus                          1/1     Running             0          3m1s
pod/storm-ui                        1/1     Running             0          61s
pod/storm-worker-controller-f4d7n   0/1     ContainerCreating   0          30s
pod/zookeeper                       1/1     Running             0          5m2s

NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
service/kubernetes             ClusterIP      10.96.0.1        <none>        443/TCP                               58m
service/nimbus                 ClusterIP      10.109.2.46      <none>        6627/TCP                              61s
service/storm-ui               LoadBalancer   10.100.24.102    <pending>     8080:31007/TCP                        30s
service/storm-worker-service   ClusterIP      10.108.121.47    <none>        6700/TCP,6701/TCP,6702/TCP,6703/TCP   0s
service/zookeeper              ClusterIP      10.104.192.204   <none>        2181/TCP                              3m1s

NAME                                            DESIRED   CURRENT   READY   AGE
replicationcontroller/storm-worker-controller   1         1         0       30s
```

The UI service for Storm will listen on port 30001. 
Get IP for the Vagrant VM and from our host os broswer access storm-ui with http://vagrant-vm-ip:30001
Example: http://172.28.128.4:31007

To change the port for storm ui, edit storm-ui-service.json and change the port under "nodePort"


# Manual setup for storm cluster
(If  you didn't use setup.sh)


## Step One: Start your ZooKeeper service

ZooKeeper is a distributed coordination service that Storm uses as a
bootstrap and for state storage.

Use the [`zookeeper.json`](zookeeper.json) file to create a pod running
the ZooKeeper service.

```shell
$ kubectl create -f storm-kube/zookeeper.json
```

Then, use the [`zookeeper-service.json`](zookeeper-service.json) file to create a
logical service endpoint that Storm can use to access the ZooKeeper
pod.

```shell
$ kubectl create -f storm-kube/zookeeper-service.json
```

You should make sure the ZooKeeper pod is Running and accessible
before proceeding.

### Check to see if ZooKeeper is running

```shell
$ kubectl get pods
POD                 IP                  CONTAINER(S)        IMAGE(S)             HOST                          LABELS                      STATUS
zookeeper           192.168.86.4        zookeeper           mattf/zookeeper      172.18.145.8/172.18.145.8     name=zookeeper              Running
```

### Check to see if ZooKeeper is accessible

```shell
$ kubectl get services
NAME                LABELS                                    SELECTOR            IP                  PORT
kubernetes          component=apiserver,provider=kubernetes   <none>              10.254.0.2          443
kubernetes-ro       component=apiserver,provider=kubernetes   <none>              10.254.0.1          80
zookeeper           name=zookeeper                            name=zookeeper      10.254.139.141      2181

$ echo ruok | nc 10.254.139.141 2181; echo
imok
```

## Step Two: Start your Nimbus service

The Nimbus service is the master (or head) service for a Storm
cluster. It depends on a functional ZooKeeper service.

Use the [`storm-nimbus.json`](storm-nimbus.json) file to create a pod running
the Nimbus service.

```shell
$ kubectl create -f storm-kube/storm-nimbus.json
```

Then, use the [`storm-nimbus-service.json`](storm-nimbus-service.json) file to
create a logical service endpoint that Storm workers can use to access
the Nimbus pod.

```shell
$ kubectl create -f storm-kube/storm-nimbus-service.json
```

Ensure that the Nimbus service is running and functional.

### Check to see if Nimbus is running and accessible

```shell
$ kubectl get services
NAME                LABELS                                    SELECTOR            IP                  PORT
kubernetes          component=apiserver,provider=kubernetes   <none>              10.254.0.2          443
kubernetes-ro       component=apiserver,provider=kubernetes   <none>              10.254.0.1          80
zookeeper           name=zookeeper                            name=zookeeper      10.254.139.141      2181
nimbus              name=nimbus                               name=nimbus         10.254.115.208      6627

$ kubectl exec -it -p nimbus -c nimbus bash
bash-4.3# cd /opt/apache-storm
bash-4.3# ./bin/storm list
...
No topologies running.
```

## Step Three: Start your Storm workers

The Storm workers (or supervisors) do the heavy lifting in a Storm
cluster. They run your stream processing topologies and are managed by
the Nimbus service.

The Storm workers need both the ZooKeeper and Nimbus services to be
running.

Use the [`storm-worker-controller.json`](storm-worker-controller.json) file to create a
ReplicationController that manages the worker pods.

```shell
$ kubectl create -f examples/storm/storm-worker-controller.json
```

### Check to see if the workers are running

One way to check is to list the Replication Controllers.

```shell
$  kubectl get rc
```

## tl;dr

```kubectl create -f storm-kube/zookeeper.json```

```kubectl create -f storm-kube/zookeeper-service.json```

Make sure the ZooKeeper Pod is running (use: ```kubectl get pods```).

```kubectl create -f storm-kube/storm-nimbus.json```

```kubectl create -f storm-kube/storm-nimbus-service.json```

Make sure the Nimbus Pod is running.

```kubectl create -f storm-kube/storm-ui.json```

```kubectl create -f storm-kube/storm-ui-service.json```

Get the Storm-UI IP and visit it on port 8080.

```kubectl create -f storm-kube/storm-worker-controller.json```

```kubectl create -f storm-kube/storm-worker-service.json```
