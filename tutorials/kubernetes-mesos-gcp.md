---
layout: doc
title: Kubernetes on Mesos on Google Cloud Platform
---

_Kubernetes and Mesos, better together._

*UPDATE:* For the latest version of this tutorial see [Getting started with Kubernetes on Mesos](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/getting-started-guides/mesos.md).

[Kubernetes](https://github.com/googlecloudplatform/kubernetes) brings cluster scheduling concepts straight from the Google Cloud Platform team to your Mesos cluster. Mesos allows dynamic sharing of cluster resources between Kubernetes and other first-class Mesos frameworks such as [Hadoop](/tutorials/run-hadoop-on-mesos-using-installer), [Spark](/tutorials/run-spark-on-mesos), and [Chronos](/tutorials/run-chronos-on-mesos). Mesos ensures applications running on your cluster are isolated and that resources are allocated fairly.

Running Kubernetes on Mesos allows you to easily move Kubernetes workloads from one cloud provider to another to your own physical datacenter.

This tutorial will walk you through setting up Kubernetes on a Mesosphere cluster on [Google Cloud Platform](http://cloud.google.com).   It provides a step by step walk through of adding Kubernetes to a Mesosphere cluster and running the classic GuestBook demo application.

## Prerequisites

* Mesosphere cluster on [Google Compute Engine](https://google.mesosphere.com)
* [VPN connection to the cluster](/getting-started/cloud/google/#vpn-setup)

## Deploy Kubernetes-Mesos

Log into the master node over SSH, replacing the placeholder below with the correct IP address.

{% highlight bash %}
ssh jclouds@${ip_address_of_master_node}
{% endhighlight %}

***

Build the Kubernetes-Mesos executables.

{% highlight bash %}
$ dpkg -l | grep -e mesos
ii    mesos    0.21.0-1.0.debian77     amd64     Cluster resource manager ...
{% endhighlight %}

{% highlight bash %}
$ git clone https://github.com/mesosphere/kubernetes-mesos k8sm
$ mkdir -p bin && sudo docker run --rm \
  -v $(pwd)/bin:/target -v $(pwd)/k8sm:/snapshot \
  jdef/kubernetes-mesos:build-mesos-0.21.0-compat
{% endhighlight %}

***

Set some environment variables.
The internal IP address of the master is visible via the cluster details page on the Mesosphere launchpad:

{% highlight bash %}
$ export servicehost=${mesos_master_internal_ip_address}
$ export KUBERNETES_MASTER=http://${servicehost}:8888
{% endhighlight %}

***

Start etcd and verify that it is running:

{% highlight bash %}
$ sudo docker run -d --net=host coreos/etcd go-wrapper run \
       -advertise-client-urls=http://${servicehost}:4001 \
       -listen-client-urls=http://${servicehost}:4001 \
       -initial-advertise-peer-urls=http://${servicehost}:7001 \
       -listen-peer-urls=http://${servicehost}:7001
{% endhighlight %}

{% highlight bash %}
$ sudo docker ps
CONTAINER ID  IMAGE               COMMAND               CREATED  STATUS  PORTS  NAMES
4026e139abd2  coreos/etcd:latest  "/etcd go-wrapper ru  9s ago   Up 9s          silly_bill
{% endhighlight %}

***

Start the kubernetes-mesos framework:

{% highlight bash %}
$ ./bin/kubernetes-mesos \
      -address=${servicehost} \
      -mesos_master=${servicehost}:5050 \
      -etcd_servers=http://${servicehost}:4001 \
      -executor_path=$(pwd)/bin/kubernetes-executor \
      -proxy_path=$(pwd)/bin/kube-proxy \
      -portal_net=10.10.10.0/24 \
      -mesos_user=root \
      -v=2 >master.log 2>&1 &
{% endhighlight %}

***

Start a replication controller:

{% highlight bash %}
$ ./bin/controller-manager -master=${KUBERNETES_MASTER} \
    -v=2 >controller.log 2>&1 &
{% endhighlight %}

***

Start a kube-proxy:

{% highlight bash %}
$ sudo ./bin/kube-proxy -bind_address=${servicehost} -etcd_servers=http://${servicehost}:4001 \
  -logtostderr=true -v=2 >proxy.log 2>&1 &
{% endhighlight %}

***

Disown your background jobs so that they'll stay running if you log out.

{% highlight bash %}
$ disown -a
{% endhighlight %}

***

Interact with the kubernetes-mesos framework via `kubecfg`:

{% highlight bash %}
$ bin/kubecfg list pods
Name                Image(s)            Host             Labels            Status
----------          ----------          ----------       ----------        ----------
{% endhighlight %}

{% highlight bash %}
$ bin/kubecfg list services                            # your service IPs will likely differ
Name           Labels           Selector                                 IP            Port
----------     ----------       ----------                               ----------    -----
kubernetes-ro                   component=apiserver,provider=kubernetes  10.10.10.227  80
kubernetes                      component=apiserver,provider=kubernetes  10.10.10.153  443
{% endhighlight %}

## Spin up a pod

Write a JSON pod description to a local file:

{% highlight bash %}
$ cat <<EOPOD >nginx.json
{ "kind": "Pod",
"apiVersion": "v1beta1",
"id": "nginx-id-01",
"desiredState": {
  "manifest": {
    "version": "v1beta1",
    "containers": [{
      "name": "nginx-01",
      "image": "dockerfile/nginx",
      "ports": [{
        "containerPort": 80,
        "hostPort": 31000
      }],
      "livenessProbe": {
        "enabled": true,
        "type": "http",
        "initialDelaySeconds": 30,
        "httpGet": {
          "path": "/index.html",
          "port": "8081"
        }
      }
    }]
  }
},
"labels": {
  "name": "foo"
} }
EOPOD
{% endhighlight %}

***

Send the pod description to Kubernetes using the `kubecfg` CLI:

{% highlight bash %}
$ bin/kubecfg -c nginx.json create pods
Name                Image(s)            Host                Labels            Status
----------          ----------          ----------          ----------        ----------
nginx-id-01         dockerfile/nginx    /                   name=foo          Waiting
{% endhighlight %}

Wait a minute or two while `dockerd` downloads the image layers from the internet.
We can use the `kubecfg` interface to monitor the status of our pod:

{% highlight bash %}
$ bin/kubecfg list pods
Name              Image(s)            Host                        Labels      Status
----------        ----------          ----------                  ----------  ----------
nginx-id-01       dockerfile/nginx    ${slave_ip}/${slave_ip}     name=foo    Running
{% endhighlight %}

***

Verify that the pod task is running in the Mesos web console.

Now we can interact with the pod running on the Mesos cluster:

{% highlight bash %}
$ curl http://${slave_ip}:31000/
	... (HTML, Welcome to nginx on Debian!)
{% endhighlight %}

## Run the Example Guestbook App

Following the instructions from the kubernetes-mesos
[examples/guestbook](https://github.com/mesosphere/kubernetes-mesos/tree/v0.2.2/examples/guestbook):

{% highlight bash %}
$ export ex=k8sm/examples/guestbook
$ bin/kubecfg -c $ex/redis-master.json create pods
$ bin/kubecfg -c $ex/redis-master-service.json create services
$ bin/kubecfg -c $ex/redis-slave-controller.json create replicationControllers
$ bin/kubecfg -c $ex/redis-slave-service.json create services
$ bin/kubecfg -c $ex/frontend-controller.json create replicationControllers
$ cat <<EOS >/tmp/frontend-service
{
  "id": "frontend",
  "kind": "Service",
  "apiVersion": "v1beta1",
  "port": 9998,
  "selector": {
    "name": "frontend"
  },
  "publicIPs": [
    "${servicehost}"
  ]
}
EOS
$ bin/kubecfg -c /tmp/frontend-service create services
{% endhighlight %}

Watch your pods transition from `Waiting` to `Running`:

{% highlight bash %}
$ watch 'bin/kubecfg list pods'
{% endhighlight %}

Review your Mesos cluster's tasks:

{% highlight bash %}
$ mesos ps
   TIME   STATE    RSS     CPU    %MEM  COMMAND USER                   ID
 0:00:03    R    67.62 MB  0.5   105.66   none  root c7315c47-7a61-11e4-8b8c-42010a863922
 0:00:03    R    68.20 MB  0.75  106.57   none  root c731613c-7a61-11e4-8b8c-42010a863922
 0:00:01    R    67.79 MB  0.25  105.93   none  root c7310460-7a61-11e4-8b8c-42010a863922
 0:00:03    R    67.62 MB  0.5   105.66   none  root bda74154-7a61-11e4-8b8c-42010a863922
 0:00:03    R    68.20 MB  0.75  106.57   none  root bda6e97f-7a61-11e4-8b8c-42010a863922
 0:00:03    R    68.20 MB  0.75  106.57   none  root b0e206f1-7a61-11e4-8b8c-42010a863922
{% endhighlight %}

Determine the internal IP address of the frontend
[service portal](https://github.com/GoogleCloudPlatform/kubernetes/blob/release-0.6/docs/services.md#ips-and-portals):

{% highlight bash %}
$ bin/kubecfg list services
Name           Labels           Selector                                 IP            Port
----------     ----------       ----------                               ----------    -----
redismaster                     name=redis-master                        10.10.10.63   10000
redisslave     name=redisslave  name=redisslave                          10.10.10.7    10001
frontend                        name=frontend                            10.10.10.149  9998
kubernetes-ro                   component=apiserver,provider=kubernetes  10.10.10.60   80
kubernetes                      component=apiserver,provider=kubernetes  10.10.10.213  443
{% endhighlight %}

Interact with the frontend application via curl:

{% highlight bash %}
$ curl http://${frontend_service_ip_address}:9998/index.php?cmd=get\&key=messages
{% endhighlight %}

Or via the Redis CLI:

{% highlight bash %}
$ sudo apt-get install redis-tools
$ redis-cli -h ${redis_master_service_ip_address} -p 10000
10.233.254.108:10000> dump messages
"\x00\x06,world\x06\x00\xc9\x82\x8eHj\xe5\xd1\x12"
{% endhighlight %}

Tail the logs of the kubelet-executor:

{% highlight bash %}
$ mesos tail -f c7315c47-7a61-11e4-8b8c-42010a863922 stderr
I1202 20:34:38.749340 03297 log.go:151] GET /podInfo?podID=c7312b26-7a61-11e4-8b8c-42010a863922&podNamespace=default: (3.71265ms) 200
I1202 20:34:38.772047 03297 log.go:151] GET /podInfo?podID=bda7385a-7a61-11e4-8b8c-42010a863922&podNamespace=default: (3.785459ms) 200
...
{% endhighlight %}

Or interact with the frontend application via your browser, in 2 steps:

First, open the firewall on the master machine.

{% highlight bash %}
$ # determine the internal port for the frontend service portal
$ sudo iptables-save|grep -e frontend  # -- port 56640 in this case
-A KUBE-PROXY -d 10.10.10.79/32 -p tcp -m comment --comment frontend -m tcp --dport 9998 -j DNAT --to-destination 10.57.172.200:56640
-A KUBE-PROXY -d 10.57.172.200/32 -p tcp -m comment --comment frontend -m tcp --dport 9998 -j DNAT --to-destination 10.57.172.200:56640

$ # open up access to the internal port for the frontend service portal
$ sudo iptables -A INPUT -i eth0 -p tcp -m state --state NEW,ESTABLISHED -m tcp \
  --dport ${internal_frontend_service_port} -j ACCEPT
{% endhighlight %}

Next, add a firewall rule in Google Cloud Platform Console / Networking:

<img src="{% asset_path learn/k8s-firewall.png %}" title="Google Cloud Platform firewall configuration" alt="" />

Finally, visit the guestbook in your browser!

<img src="{% asset_path learn/k8s-guestbook.png %}" title="Kubernetes Guestbook app running on Mesos" alt="" />
