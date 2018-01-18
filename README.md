Kube Consul Register Example
============================

This repo is an example of consul integration using [tczekajlo/kube-consul-register](https://github.com/tczekajlo/kube-consul-register).

It uses the project at [partkyle/go-visit](https://github.com/partkyle/go-visit), a small HTTP that uses redis to store and update a visitor count.

This example was tested in minikube.

```
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    3m        v1.8.0
```

# Set up the consul service

The file [config/consul.yaml](config/consul.yaml) includes a service
and a deployment for consul. Apply it to kubernetes using the following:

```
$ kubectl apply -f config/consul.yaml
service "consul" created
deployment "consul" created
```

Wait until the pods are started:

```
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
consul-6fccf6744f-vjktx   1/1       Running   0          40s
```

The service should be exposed via NodePort:

```
$ kubectl describe services consul
Name:              consul
Namespace:         default
Labels:            app=consul
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"consul"},"name":"consul","namespace":"default"},"spec":{"ports":[{"na...
Selector:          app=consul
Type:              ClusterIP
IP:                10.99.232.29
Port:              http  8500/TCP
TargetPort:        8500/TCP
Endpoints:         172.17.0.4:8500
Port:              dns  8600/TCP
TargetPort:        8600/TCP
Endpoints:         172.17.0.4:8600
Session Affinity:  None
Events:            <none>
```

Consul should be available on http://172.17.0.4:8500/ in this example, and
should have only `consul` as a serivce.

*NOTE*: To make these addresses available on a mac running minikube, you can
run the following command. This will not persist on a reboot.

```
sudo route -n add 172.17.0.0/16 $(minikube ip)
```

# Set up kube-consul-register

The [config/kube-consul-register.yaml](config/kube-consul-register.yaml) file
includes a config map and a replicaset for the kube-consul-register container.

```
$ kubectl create -f config/kube-consul-register.yaml
configmap "kube-consul-register" created
replicaset "kube-consul-register" created
$ kubectl get pods
NAME                         READY     STATUS              RESTARTS   AGE
consul-6fccf6744f-vjktx      1/1       Running             0          16m
kube-consul-register-d2jkl   0/1       ContainerCreating   0          4s
```

Viewing the logs we can see that it has connected to consul successfully,
but there isn't anything for it to register.

```
$ kubectl logs kube-consul-register-d2jkl
I0117 22:15:48.862424       1 main.go:64] Using build: v0.1.6
I0117 22:15:48.889853       1 main.go:98] Current configuration: Controller: &config.ControllerConfig{ConsulAddress:"consul", ConsulPort:"8500", ConsulScheme:"http", ConsulCAFile:"", ConsulCertFile:"", ConsulKeyFile:"", ConsulInsecureSkipVerify:false, ConsulToken:"", ConsulTimeout:2000000000, ConsulContainerName:"consul", ConsulNodeSelector:"consul=enabled", PodLabelSelector:"", K8sTag:"kubernetes", RegisterMode:"single"}, Consul: &api.Config{Address:"127.0.0.1:8500", Scheme:"http", Datacenter:"", HttpClient:(*http.Client)(0xc42032a0f0), HttpAuth:(*api.HttpBasicAuth)(nil), WaitTime:0, Token:""}
I0117 22:15:48.890136       1 main.go:128] Start syncing...
I0117 22:15:48.898982       1 main.go:133] Synchronization's been ended
I0117 22:15:48.899048       1 main.go:112] Start cleaning...
I0117 22:15:48.902840       1 main.go:117] Cleaning has been ended
```

# Set up the redis the visit pod will use

In this example we want the redis to persist when visit restarts,
so it needs to be in its own pod.

To start it use [config/redis.yaml](config/redis.yaml).

```
$ kubectl apply -f config/redis-master.yaml
service "redis-master" created
deployment "redis-master" created
```

Pods should now look like this:

```
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
consul-6fccf6744f-vjktx         1/1       Running   0          10m
redis-master-64d75bff65-9cd22   1/1       Running   0          16s
```

And the service for redis shows that its NodePort is available. This is not
necessary for the application, since redis doesn't need to be exposed. I just
left it in to allow for easy debugging.

```
$ kubectl describe services redis-master
Name:              redis-master
Namespace:         default
Labels:            app=redis
                   role=master
Annotations:       <none>
Selector:          app=redis,role=master
Type:              ClusterIP
IP:                10.111.193.74
Port:              <unset>  6379/TCP
TargetPort:        6379/TCP
Endpoints:         172.17.0.6:6379
Session Affinity:  None
Events:            <none>
```

And redis is reachable as well:

```
$ redis-cli -h 172.17.0.6
172.17.0.6:6379> info
# Server
redis_version:3.2.11
```

Note that the consul registrator does not pick this container up, as it
is not annotated.

```
$ kubectl logs kube-consul-register-d2jkl
...
I0117 22:19:48.907559       1 main.go:128] Start syncing...
I0117 22:19:48.915098       1 main.go:133] Synchronization's been ended
I0117 22:21:48.916198       1 main.go:128] Start syncing...
I0117 22:21:48.923995       1 main.go:133] Synchronization's been ended
```

# Create the visit service to test the consul integration

Now that everything is in place, we can test the consul integration using
[config/visit.yaml](config/visit.yaml). This file includes a configmap, a service,
and a deployment of a container that's hosted on hub.docker.com.

```
$ kubectl create -f config/visit.yaml
configmap "visit" created
service "visit" created
deployment "visit" created
```

Showing the logs of the pods should show that the app is running:

```
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
consul-6fccf6744f-vjktx         1/1       Running   0          48m
kube-consul-register-rm82l      1/1       Running   0          52s
redis-master-64d75bff65-tmwrx   1/1       Running   0          29m
visit-555f6784db-62dk7          1/1       Running   0          5s
visit-555f6784db-8b59q          1/1       Running   0          5s
visit-555f6784db-rsqnd          1/1       Running   0          5s
$ kubectl logs visit-555f6784db-62dk7
2018/01/17 22:47:31 starting version 0.2.0 with config: {Host:0.0.0.0 Port:80 RedisKey:visit.count RedisADDR:redis-master:6379 RedisPassword: RedisDB:0}
2018/01/17 22:47:31 listening on "[::]:80"
```

The pod defined for `visit` should include the annotations. These are what tells
kube-consul-register what to register in consul.

```
$ kubectl describe services visit
Name:                     visit
Namespace:                default
Labels:                   app=visit
Annotations:              <none>
Selector:                 app=visit
Type:                     NodePort
IP:                       10.103.164.68
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31691/TCP
Endpoints:                <none>
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
$ kubectl describe pods visit-f8c7964d9-6rxc8
Name:           visit-f8c7964d9-6rxc8
Namespace:      default
Node:           minikube/192.168.99.100
Start Time:     Wed, 17 Jan 2018 15:29:17 -0800
Labels:         app=visit
                pod-template-hash=947352085
Annotations:    consul.register/enabled=true
                consul.register/service.name=visit
                kubernetes.io/created-by={"kind":"SerializedReference","apiVersion":"v1","reference":{"kind":"ReplicaSet","namespace":"default","name":"visit-f8c7964d9","uid":"3cac1db2-fbde-11e7-81e6-0800275bd293","a...
Status:         Running
...
```

And viewing the logs of our kube-consul-register pod should show that something
was noticed and regstered to consul.

```
$ kubectl logs kube-consul-register-srrt4
...
I0117 23:20:17.757423       1 controller.go:352] POD UPDATE: Name: visit-f8c7964d9-qmhp7, Namespace: default, Phase: Running, Ready: True
I0117 23:20:17.757479       1 controller.go:365] Container visit in POD visit-f8c7964d9-qmhp7 has status: Ready:true
I0117 23:20:17.757485       1 controller.go:369] Adding service for container visit in POD visit-f8c7964d9-qmhp7 to consul
I0117 23:20:17.760500       1 controller.go:385] Service's been registered, Name: visit, ID: visit-f8c7964d9-qmhp7-visit
I0117 23:20:18.342157       1 controller.go:352] POD UPDATE: Name: visit-f8c7964d9-qhzjl, Namespace: default, Phase: Running, Ready: True
I0117 23:20:18.342230       1 controller.go:365] Container visit in POD visit-f8c7964d9-qhzjl has status: Ready:true
I0117 23:20:18.342240       1 controller.go:369] Adding service for container visit in POD visit-f8c7964d9-qhzjl to consul
I0117 23:20:18.343525       1 controller.go:385] Service's been registered, Name: visit, ID: visit-f8c7964d9-qhzjl-visit
I0117 23:20:18.941233       1 controller.go:352] POD UPDATE: Name: visit-f8c7964d9-dknn8, Namespace: default, Phase: Running, Ready: True
I0117 23:20:18.941277       1 controller.go:365] Container visit in POD visit-f8c7964d9-dknn8 has status: Ready:true
I0117 23:20:18.941284       1 controller.go:369] Adding service for container visit in POD visit-f8c7964d9-dknn8 to consul
I0117 23:20:18.942230       1 controller.go:385] Service's been registered, Name: visit, ID: visit-f8c7964d9-dknn8-visit
I0117 23:21:04.518183       1 main.go:128] Start syncing...
I0117 23:21:04.524953       1 main.go:133] Synchronization's been ended
```

And accessing the consul interface should show the available services:

```
$ curl 172.17.0.4:8500/v1/catalog/services
{
    "consul": [],
    "visit": [
        "pod-template-hash:947352085",
        "kubernetes",
        "pod:visit-f8c7964d9-qhzjl",
        "pod:visit-f8c7964d9-qmhp7",
        "pod:visit-f8c7964d9-dknn8",
        "node:minikube",
        "container:visit",
        "app:visit"
    ]
}
```

```
$ dig @172.17.0.4 -p 8600 visit.service.consul +short
172.17.0.8
172.17.0.7
172.17.0.9
```

And hitting any of those ip's should result in hitting the individual pod associated.

```
$ curl 172.17.0.8
The current visit count is 1 on visit-f8c7964d9-6rxc8 running version 0.4.0.
```
