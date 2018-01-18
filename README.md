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

*NOTE*: To make these cluster IPs available on a mac running minikube, you can
run the following command. This will not persist on a reboot.

```
sudo route -n add 172.17.0.0/16 $(minikube ip)
```

# Set up the consul service

The file [consul](consul) directory includes a service
and a deployment for consul. Apply it to kubernetes using the following:

```
cat consul/*.yaml | kubectl create -f -
deployment "consul" created
service "consul" created
```

Wait until the pods are started:

```
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
consul-6fccf6744f-7dg5q   1/1       Running   0          23s
```

The service defintion should look like below.
*Note: The service is required if we want consul to be discoverable
via dns on other pods*.

```
$ kubectl describe services consul
Name:              consul
Namespace:         default
Labels:            app=consul
Annotations:       <none>
Selector:          app=consul
Type:              ClusterIP
IP:                10.109.113.235
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

# Set up kube-consul-register

The [kube-consul-register](kube-consul-register) directory includes a
config map and a deployment for the kube-consul-register container. The config
map includes many of the tuning options for the. For this example, we are using
pod based discovery, which will insert a service entry for each pod.

```
$ cat kube-consul-register/*.yaml | kubectl create -f -
configmap "kube-consul-register" created
deployment "kube-consul-register" created
```

Viewing the logs we can see that it has connected to consul successfully,
but there isn't anything for it to register.

```
$ kubectl get pods
NAME                                    READY     STATUS    RESTARTS   AGE
consul-6fccf6744f-7dg5q                 1/1       Running   0          2m
kube-consul-register-69db9f647d-qgq2b   1/1       Running   0          19s
$ kubectl logs kube-consul-register-69db9f647d-qgq2b
I0118 22:07:01.824554       1 main.go:64] Using build: v0.1.6
I0118 22:07:01.840267       1 main.go:98] Current configuration: Controller: &config.ControllerConfig{ConsulAddress:"consul", ConsulPort:"8500", ConsulScheme:"http", ConsulCAFile:"", ConsulCertFile:"", ConsulKeyFile:"", ConsulInsecureSkipVerify:false, ConsulToken:"", ConsulTimeout:2000000000, ConsulContainerName:"consul", ConsulNodeSelector:"consul=enabled", PodLabelSelector:"", K8sTag:"kubernetes", RegisterMode:"single", RegisterSource:"pod"}, Consul: &api.Config{Address:"127.0.0.1:8500", Scheme:"http", Datacenter:"", HttpClient:(*http.Client)(0xc42000f500), HttpAuth:(*api.HttpBasicAuth)(nil), WaitTime:0, Token:""}
I0118 22:07:01.840465       1 main.go:128] Start syncing...
I0118 22:07:01.855795       1 main.go:133] Synchronization's been ended
I0118 22:07:01.856294       1 main.go:112] Start cleaning...
I0118 22:07:01.868502       1 main.go:117] Cleaning has been ended
```

# Set up the redis the visit pod will use

In this example we want the redis to persist when visit restarts,
so it needs to be in its own pod.

To start it use yaml in the [redis-master](redis-master) directory.

```
$ cat redis-master/*.yaml | kubectl create -f -
deployment "redis-master" created
service "redis-master" created
```

Pods should now look like this:

```
$ kubectl get pods
NAME                                    READY     STATUS    RESTARTS   AGE
consul-6fccf6744f-7dg5q                 1/1       Running   0          3m
kube-consul-register-69db9f647d-qgq2b   1/1       Running   0          1m
redis-master-64d75bff65-xsl5d           1/1       Running   0          10s
```

And the service for redis shows that its ClusterIP is available. We need the service
definition as well as the `visit` application uses dns to find the redis-master.

```
$ kubectl describe services redis-master
Name:              redis-master
Namespace:         default
Labels:            app=redis
                   role=master
Annotations:       <none>
Selector:          app=redis,role=master
Type:              ClusterIP
IP:                10.97.56.142
Port:              <unset>  6379/TCP
TargetPort:        6379/TCP
Endpoints:         172.17.0.6:6379
Session Affinity:  None
Events:            <none>
```

And redis is reachable as well:

```
$ redis-cli -h 172.17.0.6 info | head -2
# Server
redis_version:3.2.11
```

Note that the consul registrator does not pick this container up, as it
is not annotated to do so.

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
[visit](visit). This directory includes a configmap, a service, and a
deployment of a container that's hosted on hub.docker.com.

```
$ cat visit/*.yaml | kubectl create -f -
configmap "visit" created
deployment "visit" created
service "visit" created
```

Showing the logs of the pods should show that the app is running:

```
$ kubectl logs visit-f8c7964d9-56t4l
2018/01/18 22:24:51 starting version 0.4.0 with config: {Host:0.0.0.0 Port:80 RedisKey:visit.count RedisADDR:redis-master:6379 RedisPassword: RedisDB:0}
2018/01/18 22:24:51 listening on "[::]:80"
```

The pod defined for `visit` should include the annotations. These are what tells
kube-consul-register what to register in consul.

```
$ kubectl describe services visit
Name:                     visit
...
Annotations:    consul.register/enabled=true
                consul.register/service.name=visit
...
```

And viewing the logs of our kube-consul-register pod should show that something
was noticed and regstered to consul.

```
$ kubectl logs kube-consul-register-69db9f647d-4kqpb
I0118 22:23:55.243752       1 main.go:64] Using build: v0.1.6
...
I0118 22:24:52.504721       1 controller.go:352] POD UPDATE: Name: visit-f8c7964d9-r2pm2, Namespace: default, Phase: Running, Ready: True
I0118 22:24:52.504749       1 controller.go:365] Container visit in POD visit-f8c7964d9-r2pm2 has status: Ready:true
I0118 22:24:52.504800       1 controller.go:369] Adding service for container visit in POD visit-f8c7964d9-r2pm2 to consul
I0118 22:24:52.506202       1 controller.go:385] Service's been registered, Name: visit, ID: visit-f8c7964d9-r2pm2-visit
I0118 22:24:52.533055       1 controller.go:352] POD UPDATE: Name: visit-f8c7964d9-56t4l, Namespace: default, Phase: Running, Ready: True
I0118 22:24:52.533076       1 controller.go:365] Container visit in POD visit-f8c7964d9-56t4l has status: Ready:true
I0118 22:24:52.533108       1 controller.go:369] Adding service for container visit in POD visit-f8c7964d9-56t4l to consul
I0118 22:24:52.533950       1 controller.go:385] Service's been registered, Name: visit, ID: visit-f8c7964d9-56t4l-visit
I0118 22:24:52.549663       1 controller.go:352] POD UPDATE: Name: visit-f8c7964d9-srvlh, Namespace: default, Phase: Running, Ready: True
I0118 22:24:52.549693       1 controller.go:365] Container visit in POD visit-f8c7964d9-srvlh has status: Ready:true
I0118 22:24:52.549698       1 controller.go:369] Adding service for container visit in POD visit-f8c7964d9-srvlh to consul
I0118 22:24:52.550806       1 controller.go:385] Service's been registered, Name: visit, ID: visit-f8c7964d9-srvlh-visit
...
```

And accessing the consul interface should show the available services:

```
$ curl 172.17.0.4:8500/v1/catalog/services
{
    "consul": [],
    "visit": [
        "app:visit",
        "pod-template-hash:947352085",
        "kubernetes",
        "pod:visit-f8c7964d9-r2pm2",
        "pod:visit-f8c7964d9-srvlh",
        "pod:visit-f8c7964d9-56t4l",
        "node:minikube",
        "container:visit"
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
The current visit count is 1 on visit-f8c7964d9-srvlh running version 0.4.0.
$ curl 172.17.0.7
The current visit count is 2 on visit-f8c7964d9-r2pm2 running version 0.4.0.
$ curl 172.17.0.9
The current visit count is 3 on visit-f8c7964d9-56t4l running version 0.4.0.
```
