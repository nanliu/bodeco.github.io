---
layout: default
title:  "Fun with Kubernetes"
date:   2015-11-06 11:00:48
categories: kubernetes devops gce
---

# Fun with Kubernetes

Finally had a chance to have a look at Kubernetes by deploying Keystone into Google Compute Engine using Kubernetes. Lots of work automating Openstack deployments this year with Puppet, so it’s pretty fresh in my mind. Keystone is the most standalone part of openstack in terms of dependencies, so it’s the easiest place to start.

## Why do I care?

Curious how Kubernetes compares to a deployment system that I had built for Openstack based on Puppet, Consul, and an Openstack UnderCloud controller.

Already started looking into Docker to see how it might meet engineering requirements for dependency isolation of openstack components, and to see how it might simplify the continuous integration part of deploying openstack (ie: how it compares to building packages based from source).

Interested in seeing if moving to a full container based micro service architecture would allow me to plugin directly into things like kubernetes or mesos to get deployment orchestration and features like rolling upgrades for free!

## What is Kubernetes

Kubernetes is a orchestration engine that allows users to define complex application stacks using higher level constructs than hosts.

Instead of defnining roles (ie: db, keystone) and assigning them to hosts, you specify policies for how
many of each service should be running.

    Ie: run 3 dbs and 3 api endpoints

Kubernetes acts as the scheduler, continuously monitoring the state of your resources to ensure that they conform to these policies.

It allows users to define application stacks using three constructs:

### Pods

Pods are the units of work that are scheduled by kubernetes. They describe a set of containers that are used to model a single application. In general, this can be viewed as all services that you would want to be deployed to the same physical host.

The containers specified as a part of a pod are all based on Docker containers.

Pods are also the entity that is scaled out by replication controllers.

### Services

Services are used to describe the endoints of an application that provide both the user interface as well as the internal communication channel between services in a Service Oriented Architecture.

They are implemented as a vip where Kubernetes is responsible for managing how that vip directs traffic to the actual api endpoints providing the service (as defined by containers launched in pods).

This vip is set as a magic environment variable that matches the name of the service being provided
(basically using Docker links under the covers)

### Replication Controllers

Replication Controllers allow applications to be managed based on a policy. They can be used to describe a collection of pods as well as how many of those pods should exist.

Replication Controllers are also the things that controls rolling upgrade support.

## A real example

In my example, I wanted to deploy N number of keystone APIs running with a local memcached server that all use the same mysql db to store data.

The source for this project can be found here: (https://github.com/bodepd/openstack-kubernetes)

I’ll be launching these containers into GCE, b/c it’s the easiest way to get started.

First, you have to [setup](https://cloud.google.com/container-engine/docs/before-you-begin) an environment and [create the cluster](https://cloud.google.com/container-engine/docs/clusters/operations) that will hold your applications:

Once you have those things ready, you can start modeling and deploying the application.

### mysql pod

First we’ll build out a pod to represent our mysql backend for keystone. The definition below uses
the standard docker mariadb container.

NOTE: for simplicity, the definition does not set up an external mount point for the database data.

{% highlight yaml %}
# keystone-db.yaml
apiVersion: v1
kind: Pod
metadata:
  name: keystone-db
  labels:
    name: keystone-db
spec:
  containers:
    - resources:
        limits:
          cpu: 0.5
      image: mariadb
      name: keystone-db
      env:
        - name: MYSQL_USER
          value: keystone
        - name: MYSQL_PASSWORD
          value: keystone
        - name: MYSQL_DATABASE
          value: keystone
        - name: MYSQL_ROOT_PASSWORD
          value: yourpassword
      ports:
        - containerPort: 3306
          name: keystone-db
{% endhighlight %}

The above definition does the following:
* creates a pod
* selects docker standard mariadb image
* passes some ENV vars into docker to setup the root password and keystone database
* specifies port to expose
* labels the pod with "name: keystone-db"

The following commands are used to create and inspect the resulting container.

First create it.

{% highlight bash %}
> kubectl create -f keystone-db.yaml
{% endhighlight %}

Once you’ve launched the pod, you can check it’s status:

{% highlight bash %}
> kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
keystone-db   1/1       Running   0          1m
{% endhighlight %}

The following command can log into the container to inspect it

{% highlight bash %}
> kubectl exec keystone-db -c keystone-db -i -t -- bash -il
{% endhighlight %}

### keystone db service

Now that we’ve created a pod with our db container, we need to assign it to a service so that we can
link it to our keystone pod to manage how keystone willl connect to it’s database.

The following yaml defines the db service.

{% highlight yaml %}
# keystone-db-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: keystone-db
  name: db
spec:
  ports:
    # The port that this service should serve on.
    - port: 3306
  # Label keys and values that must match in order to receive traffic for this service.
  selector:
    name: keystone-db
{% endhighlight %}

it does the following:
* Names the service
* Labels the service
* Adds a port
* Uses a selector to determine which containers the service will map to (using the container’s label)

First create the service:

{% highlight bash %}
> kubectl create -f keystone-db-service.yaml
services/db
{% endhighlight %}

Check the status of that service

{% highlight bash %}
> kubectl get services
NAME         LABELS              SELECTOR           IP(S)           PORT(S)
db           name=keystone-db    name=keystone-db   10.87.250.253   3306/TCP
{% endhighlight %}

### keystone replication controller

The following replication controller creates a pod for keystone with 2 containers, one for the keystone api and one for memcache.

{% highlight yaml %}
# keystone-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: keystone-rc
  spec:
    replicas: 3
      selector:
      name: keystone
      template:
        metadata:
          labels:
            name: keystone
      spec:
        containers:
          - image: bodepd/keystone
            name: keystone
            env:
              - name: MYSQL_USER
                value: keystone
              - name: MYSQL_PASSWORD
                value: keystone
              - name: MYSQL_DATABASE
                value: keystone
              - name: ADMIN_TOKEN
                value: token123
            ports:
              - containerPort: 5000
                name: keystone-public
                containerPort: 35357
                name: keystone-admin
            imagePullPolicy: Always
          - image: memcached
            name: memcached
            ports:
              - containerPort: 11211
                name: memcached
{% endhighlight %}

it does the following:
* defines two containers for the pod (keystone api and memcached)
* passes in env vars to keystone
* selects my custom docker image (bodepd/keystone)
* defines the 2 ports that the keystone service exposes
* specifies that 3 instances of this pod should exist

### pod to service magic mappings

One thing not expliclty specified here is how the keystone containers leverage the defined service
and configure itself to use the correct database.

This relies on the fact that services are made visiable as environment variables inside of containers.

The following environment variable is exposed to our containers at runtime.

{% highlight bash %}
if [ -z "${DB_PORT_3306_TCP_ADDR}" ]; then
  echo 'DB_PORT_3306_TCP_ADDR must be set, maybe you did not link it?'
  exit 1
fi
…
connection="mysql:keystone:passwd:@{DB_PORT_3306_TCP_ADDR}/keystone"
crudini --set /etc/keystone/keystone.conf database connection $connection
{% endhighlight %}

Let's create the replication controller.

{% highlight bash %}
> kubectl create -f keystone-rc.yaml
{% endhighlight %}

Using get, we can see the defined replication controller.

{% highlight bash %}
> kubectl get rc                                                                         1 ↵
CONTROLLER    CONTAINER(S)   IMAGE(S)          SELECTOR        REPLICAS
keystone-rc   keystone       bodepd/keystone   name=keystone   3
              memcached      memcached
{% endhighlight %}

Now have a look at what pods exist:

{% highlight bash %}
> kubectl get pods
NAME                READY     STATUS    RESTARTS   AGE
keystone-db         1/1       Running   0          3h
keystone-rc-3kjkt   2/2       Running   0          3m
keystone-rc-pusyc   2/2       Running   0          3m
keystone-rc-srnx7   2/2       Running   0          3m
{% endhighlight %}

Lastly, let's log in to see that our database connection did get set up correctly.

{% highlight bash %}
> kubectl exec keystone-rc-3kjkt -c keystone -i -t -- bash -il
root@keystone-rc-3kjkt$ grep connection | grep mysql
root@keystone-rc-3kjkt$ connection = mysql://keystone:keystone@10.87.250.253/keystone
{% endhighlight %}

### keystone api endpoint

The last thing to do is to define is the service

{% highlight yaml %}
# keystone-public-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: keystone-public
  name: keystone-public
spec:
  type: LoadBalancer
  ports:
    - port: 5000
      targetPort: 5000
      protocol: TCP
  selector:
    name: keystone
{% endhighlight %}

Create the service.

{% highlight bash %}
> kubectl create -f keystone-public-service.yaml
{% endhighlight %}

Check that the service exists.

{% highlight bash %}
> kubectl get services
NAME              LABELS                                    SELECTOR           IP(S)           PORT(S)
db                name=keystone-db                           name=keystone-db   10.87.250.253   3306/TCP
keystone-public   name=keystone-public                      name=keystone        10.87.245.132   5000/TCP
                                                                                                                        104.197.79.72
kubernetes        component=apiserver,provider=kubernetes   <none>           10.87.240.1     443/TCP
{% endhighlight %}

You can see that the keystone public endpoint has an additional public ip address. This is because it was specified as "type: LoadBalancer".

Now you curl the resulting keystone api via the loadbalancer vip

{% highlight json %}
curl http://104.197.79.72:5000
{"versions": {"values": [{"status": "stable", "updated": "2013-03-06T00:00:00Z", "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v3+json"}, {"base": "application/xml", "type": "application/vnd.openstack.identity-v3+xml"}], "id": "v3.0", "links": [{"href": "http://104.197.79.72:5000/v3/", "rel": "self"}]}, {"status": "stable", "updated": "2014-04-17T00:00:00Z", "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v2.0+json"}, {"base": "application/xml", "type": "application/vnd.openstack.identity-v2.0+xml"}], "id": "v2.0", "links": [{"href": "http://104.197.79.72:5000/v2.0/", "rel": "self"}, {"href": "http://docs.openstack.org/api/openstack-identity-service/2.0/content/", "type": "text/html", "rel": "describedby"}, {"href": "http://docs.openstack.org/api/openstack-identity-service/2.0/identity-dev-guide-2.0.pdf", "type": "application/pdf", "rel": "describedby"}]}]}}%
{% endhighlight %}

### kubernetes is cool!

Kebernetes isn't the easiest thing to use, it requires that you rewrite your entire architecture to use microservices, and the
required yaml files can be a little unwiedly. So the question is: "Are the features it gives you worth the effort"?

I'll give one quick example of a feature that you get for free based off the previous example:

First let's have a look at our pods.

{% highlight bash %}
> kubectl get pods
NAME                READY     STATUS    RESTARTS   AGE
keystone-db         1/1       Running   0          4h
keystone-rc-3kjkt   2/2       Running   0          33m
keystone-rc-pusyc   2/2       Running   0          33m
keystone-rc-srnx7   2/2       Running   0          33m
{% endhighlight %}

Then let's delete one, and immediately take a look at the resulting pods:

{% highlight bash %}
> kubectl delete pod keystone-rc-3kjkt; kubectl get pods;

pods/keystone-rc-3kjkt

NAME                READY     STATUS    RESTARTS   AGE
keystone-db         1/1       Running   0          4h
keystone-rc-2nds0   0/2       Pending   0          1s
keystone-rc-pusyc   2/2       Running   0          34m
keystone-rc-srnx7   2/2       Running   0          34m
{% endhighlight %}

In the above example, kubernetes detects that the pod is missing and instantly replaces it
to meet the SLA of the replication controller. Pretty cool, huh?

## Next steps

I’m just getting started playing around with controllers, I’ll keep posting blogs as I go :)
