# Deploy a machine learning model on Kubernetes

This sequence of exercises will introduce you to some basic principles of cloud computing, in the context of deploying a machine learning model to the cloud. You will learn the following:

*Deploy an image classification as a web service*. After completing this section, you should be able to:

* describe advantages of deploying a web service using containers 
* incorporate a machine learning model into a web application using Flask 
* build, deploy, and administer containers to serve a web application

*Deploy a service using container orchestration (Kubernetes)*. After completing this section, you should be able to:

* describe benefits of using a container orchestration framework, versus directly deploying containers 
* configure, deploy, and test a service deployed on a Kubernetes cluster

*Deploy a load balanced service on Kubernetes*. After completing this section, you should be able to:

* explain tradeoffs of load balancing with respect to service time and resource usage 
* configure, deploy, and test a load balanced service on Kubernetes

*Deploy a service with dynamic scaling*. After completing this section, you should be able to:

* explain tradeoffs of dynamic scaling with respect to service time and resource usage 
* configure, deploy, and test a service with dynamic scaling on Kubernetes

This sequence assumes that you have already deployed a Kubernetes cluster on Chameleon, using [this recipe](https://chameleoncloud.org/experiment/share/9bae9a8a-68fa-402c-ae51-41431eb78732), and that you can SSH into the primary node in your cluster.

On the primary node in your cluster, "node-0", run

```
git clone --single-branch --branch gh-pages https://github.com/teaching-on-testbeds/k8s-ml.git
```

to get the materials you will need for these exercises.

---