# Deploy a machine learning model on Kubernetes

This sequence of exercises will introduce you to some basic principles of cloud computing, in the context of deploying a machine learning model to the cloud. You will learn the following:

*Deploy an image classification as a web service*. After completing this section, you should be able to:

-   describe advantages of deploying a web service using containers
-   incorporate a machine learning model into a web application using Flask
-   build, deploy, and administer containers to serve a web application

*Deploy a service using container orchestration (Kubernetes)*. After completing this section, you should be able to:

-   describe benefits of using a container orchestration framework, versus directly deploying containers
-   configure, deploy, and test a service deployed on a Kubernetes cluster

*Deploy a load balanced service on Kubernetes*. After completing this section, you should be able to:

-   explain tradeoffs of load balancing with respect to service time and resource usage
-   configure, deploy, and test a load balanced service on Kubernetes

*Deploy a service with dynamic scaling*. After completing this section, you should be able to:

-   explain tradeoffs of dynamic scaling with respect to service time and resource usage
-   configure, deploy, and test a service with dynamic scaling on Kubernetes

This sequence assumes that you have already deployed a Kubernetes cluster on Chameleon, using [this recipe](https://chameleoncloud.org/experiment/share/9bae9a8a-68fa-402c-ae51-41431eb78732), and that you can SSH into the primary node in your cluster.

On the primary node in your cluster, "node-0", run

    git clone --single-branch --branch gh-pages https://github.com/teaching-on-testbeds/k8s-ml.git

to get the materials you will need for these exercises.

------------------------------------------------------------------------

## Exercise: Deploy image classification as a web service

We will start by deploying an image classification model as a web service. Users can upload an image to a basic web app (in Flask) in their browser, then the app will:

-   resize the image to the correct input dimensions
-   pass it as input to an image classification model that has been fine tuned for food classification
-   and return the most likely class (and estimated confidence), as well as the wall time of the `model.predict` call.

### Containerize the basic web app

To make it easy to deploy our basic application, we will containerize it - package the source code together with an operating system, software dependencies, and anything else it needs to run. This is a convenient way to distribute the code along with everything it needs. This will also make it much easier to deploy multiple copies of the application (to handle heavier load) in future exercises.

The source code for our basic application is inside the `app` subdirectory inside `k8s-ml`. It has the following materials:

    -   app
        -   instance
        -   static
        -   templates
        -   app.py
        -   Dockerfile
        -   requirements.txt
        -   model.keras

You can browse these files in the web interface [here](https://github.com/teaching-on-testbeds/k8s-ml/tree/gh-pages/app).

Note that a saved model - `model.keras` - is inside this directory. Then, inside `app.py`,

-   when `app.py` runs, it loads the saved model: `model = load_model("model.keras")`

-   when a user uploads an images file to the app, or when a special "test" path in the URL is used, a `model_predict` function is called that returns the predicted class of the image:

          def model_predict(img_path, model):
              im = Image.open(img_path).convert('RGB')
              image_resized = im.resize(target_size, Image.BICUBIC)
              test_sample = np.array(image_resized)/255.0
              test_sample = test_sample.reshape(1, target_size[0], target_size[1], 3)
              classes = np.array(["Bread", "Dairy product", "Dessert", "Egg", "Fried food",
              "Meat", "Noodles/Pasta", "Rice", "Seafood", "Soup",
              "Vegetable/Fruit"])
              test_probs = model.predict(test_sample)
              most_likely_classes = np.argmax(test_probs.squeeze())

              return classes[most_likely_classes], test_probs.squeeze()[most_likely_classes]

The `app` directory also includes a [Dockerfile](https://github.com/teaching-on-testbeds/k8s-ml/blob/gh-pages/app/Dockerfile). This file describes how to build a container for this application, including:

-   what "base" container should be used (we'll use a Python 3.9 base container)
-   what needs to run inside the container when it starts (we'll install the libraries listed in `requirements.txt` using `pip`)
-   what files should be copied to the container (everything in `app`)
-   and what command to run on the container when it is ready (we will run `python app.py` to start our web app)

We will use this Dockerfile to build the container (naming it `ml-app`) and then push it to a local distribution "registry" of containers (which is already running on node-0, on port 5000).

In an SSH session on "node-0", run

    docker build -t ml-app:0.0.1 ~/k8s-ml/app
    docker tag ml-app:0.0.1  node-0:5000/ml-app:0.0.1
    docker push node-0:5000/ml-app:0.0.1

### Deploy the basic web app

Now that we have containerized our application, we can run it! Let's run it now, and will indicate that we want incoming requests on port 32000 to be passed to port 5000 on the container (where our Flask application is listening). In an SSH session on "node-0", run

    docker run -d -p 32000:5000 node-0:5000/ml-app:0.0.1

Here,

-   `-d` is for detach mode - so we can leave it running in the background. (If you need to run a container in the foreground, for debugging purposes, you will omit this argument.)
-   `-p` is to assign the mapping between "incoming request port" (32000) and "container port' (5000).

You can see the list of running containers with

    docker ps

which will show containers related to the docker register and Kubernetes deployment, but will also show one running container using the `node-0:5000/ml-app:0.0.1` image. To restrict the output to just this image, we can use

    docker ps -f ancestor=node-0:5000/ml-app:0.0.1

The firewall on the node will block access to our app from outside, so let us temporarily disable it:

    sudo systemctl stop firewalld.service

Now we can visit our web service and try it out! Run this command on "node-0" to get the URL to use:

    echo http://$(curl -s ifconfig.me/ip):32000

Then, open your browser, paste this URL into the address bar, and hit Enter.

When the web app has loaded, upload an image to your classification service, and check its prediction.

You can also see the resource usage - in terms of CPU and memory - of your container, with

    docker stats $(docker ps -q -f ancestor=node-0:5000/ml-app:0.0.1)

(which uses command substitution to get the ID of any running container using our `ml-app` image, then gets its statistics). Use Ctrl+C to stop this live display.

When you are finished, you can stop the container by running:

    docker stop $(docker ps -q -f ancestor=node-0:5000/ml-app:0.0.1)

(which uses command substitution to get the ID of any running container using our `ml-app` image, then stops it).

## Exercise: Deploy your service using container orchestration (Kubernetes)

In the previous exercise, we deployed a container by running it directly. Now, we will deploy the same container using Kubernetes, a platform for container orchestration.

What are some benefits of using a container orchestration framework like Kubernetes, rather than deploying containers directly?

-   *Container Orchestration:* Kubernetes helps to automate the deployment, scaling and management of containers.
-   *Self-healing:* If a container fails, Kubernetes can automatically detect it and replace it with a functional container.
-   *Load balancing:* Kubernets can distribute traffic for an application across multiple instances of running containers. (We'll try this in the next exercise.)
-   *Resource management:* Kubernetes allows you to set resource limits for containers, to ensure that the application has the resources to run efficiently.

*Pods* are the basic components in Kubernetes, and are used to deploy and manage containerized applications in a scalable and efficient way. They are designed to be ephemeral, meaning they can be created, destroyed, and recreated as needed. They can also be replicated, which allows for load balancing and high availability.

Although we will eventually deploy pods across all three of our "worker" nodes, our deployment will be managed from the "controller" node, which is "node-0".

To deploy an app on a Kubernetes cluster, we use a manifest file, which describes our deployment. For this exercise, we will use the "deployment_k8s.yaml" file inside the "\~/k8s-ml/deploy_k8s" directory, which you can see [here](https://github.com/teaching-on-testbeds/k8s-ml/blob/gh-pages/deploy_k8s/deployment_k8s.yaml).

This manifest file defines a Kubernetes service named "ml-kube-service" and a Kubernetes deployment named "ml-kube-app".

-   Inside the service definition, we create a service of `type: NodePort`. This service passes incoming requests on a specified port, to a (different) port on the pod.
-   Inside the deployment definition, you can see that
    -   the "ml-app" container you built earlier will be retrieved from the local registry ("node-0:5000/ml-app:0.0.1"),
    -   the deployment will include just a single copy of our pod ("replicas: 1").
    -   there is a "readiness" probe defined - the container is considered "Ready" and requests will be forwarded to it only when it responds with a success code to 3 HTTP requests on the `/test` endpoint.

It also defines the resource requirements of the container, in terms of CPU cores and memory. The "request" defines the minimum resource a container may get, and the "limit" defines the maximum resource a container may get.

To start this deployment, we will run:

    kubectl apply -f ~/k8s-ml/deploy_k8s/deployment_k8s.yaml

and make sure the following output appears:

    service/ml-kube-service created
    deployment.apps/ml-kube-app created

Let's check the status of the service. Run the command:

``` shell
kubectl get svc -o wide
```

The output will include a line similar to

    NAME                 TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)         AGE     SELECTOR
    ml-kube-service   NodePort    10.233.37.25   <none>        6000:32000/TCP   12m     app=ml-kube-app

It may take a few minutes for the pod to start running. To check the status of pods, run:

    kubectl get pods -o wide

The output may include a line similar to

    NAME                                               READY   STATUS              RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
    ml-kube-app-7b4c8648c6-r8zvv                    0/1     ContainerCreating   0          22s     <none>        node-2   <none>           <none>

In this example, the status of the pod is `ContainerCreating`, which means the container is getting ready. When it reaches the `Running` state, then it means the pod is healthy and is running. When it shows "1/1" in the "Ready" column, it is ready to accept requests according to the probe we had set up.

(As before, if your model is large, it may take a while before it is ready to accept requests.)

Once the pod is ready, check the resource usage (CPU and memory) of the pod with

    kubectl top pod

Note that the resource usage varies depending on whether or not the pod is currently serving a request!

Get the URL of the service - run

    echo http://$(curl -s ifconfig.me/ip):32000

copy and paste this URL into your browser's address bar, and verify that your app is up and running there.

### Test deployment under load

To test the load on the deployment we will use [siege](https://linux.die.net/man/1/siege), a command-line tool used to test and analyze the performance of web servers. It can generate a significant amount of traffic to test the response of a web server under load.

Install Siege on node-0:

    sudo apt-get update; sudo apt-get -y install siege

Open a second SSH session on node-0. In one, run

    watch -n 5 kubectl top pod

to monitor the pod's resource usage in real time (This will be updated every 5 seconds). In the second SSH session, run

    siege -c 10 -t 30s http://$(curl -s ifconfig.me/ip):32000/test

Here Siege will generate traffic to a "test" endpoint on your website, which requests a prediction for a pre-saved image, for 30 seconds with a concurrency level of 10 users. After it finishes execution, make a note of key results - how many transactions were served successfully, how many failed, the transaction rate, and what the average response time was (note that this includes inference time, as well as several other elements).

Note: you may see some instances of

    [error] socket: unable to connect sock.c:249: Connection refused

in the Siege output. This is an indication that in the tests, some of the connections failed entirely (rather than just waiting a long time!) because the server is under such heavy load.

### Stop the deployment

When you are done with your experiment, make sure to delete the deployment and service. To delete, run the command:

    kubectl delete -f ~/k8s-ml/deploy_k8s/deployment_k8s.yaml

and look for output like

    service "ml-kube-service" deleted
    deployment.apps "ml-kube-app" deleted

Use

    kubectl get pods -o wide

and verify that (eventually) no pods are running your app.

## Exercise: Deploy your service with load balancing

In the previous exercise, we deployed a single replica of a Kubernetes pod. But if the load on the service is high, the single pod will have slow response times. We can address this by deploying multiple "replicas" of the pod, and distributing the traffic across them by assigning each incoming request to a pod. This is called **load balancing**.

The manifest file for deploying a load balanced service is named "deployment_lb.yaml", and it is inside the "\~/k8s-ml/deploy_lb" directory. You can see it [here](https://github.com/teaching-on-testbeds/k8s-ml/blob/gh-pages/deploy_lb/deployment_lb.yaml).

This manifest file defines a Kubernetes service of type `LoadBalancer` with the name "ml-kube-service" and a Kubernetes deployment named "ml-kube-app". There are two major differences between this deployment and the previous deployment:

-   in this one, we specify that the service is of `type: LoadBalancer`, i.e. instead of directly passing incoming requests to one pod, will place a load balancer service in "front" of the pods that will distribute the requests across pods.
-   in this one, we specify `replicas: 5` where previously we used `replicas: 1`.

To start this deployment, we will run:

    kubectl apply -f ~/k8s-ml/deploy_lb/deployment_lb.yaml

and make sure the following output appears:

    service/ml-kube-service created
    deployment.apps/ml-kube-app created

It will take a few minutes for the pods to start running. To check the status of deployment, run:

    kubectl get pods -o wide

and wait until the output shows that all pods are in the "Running" state and show as "1/1" in the "Ready" column. Note that the pods will be deployed across all of the nodes in the cluster. Also note that if some pods are "Ready" but not others, requests will be sent only to the pods that are "Ready".

Once the pods are ready, you can see the resource usage (CPU and memory) of the pods with

    kubectl top pod

Get the URL of the service - run

``` shell
echo http://$(curl -s ifconfig.me/ip):32000
```

copy and paste this URL into your browser's address bar, and verify that your app is up and running there.

### Test deployment under load

As before, we will test this deployment under load. Open a second SSH session on node-0. In one, run

    watch -n 5 kubectl top pod

to monitor the pod's resource usage in real time (This will be updated every 5 seconds). In the second SSH session, run

    siege -c 10 -t 30s http://$(curl -s ifconfig.me/ip):32000/test

After it finishes execution, make a note of key results - how many transactions were served successfully, how many failed, the transaction rate, and what the average response time was (note that this includes inference time, as well as several other elements).

### Stop the deployment

When you are done with your experiment, make sure to delete the deployment and service. To delete, run the command:

    kubectl delete  -f ~/k8s-ml/deploy_lb/deployment_lb.yaml

and look for output like

    service "ml-kube-service" deleted
    deployment.apps "ml-kube-app" deleted

Use

    kubectl get pods -o wide

and verify that (eventually) no pods are running your app.

## Exercise: Deploy a service with dynamic scaling

When we used load balancing to distribute incoming traffic across multiple pods, the response time under load was much faster. But during time intervals when the load is not heavy, it may be wasteful to deploy so many pods. (The application is loaded and uses memory and some CPU even when there are no requests!)

To address this issue, we can use scaling - where the resource deployment changes in response to load on the service. In this exercise, specifically we use **horizontal scaling**, which adds more pods/replicas to handle increasing levels of work, and removes pods when they are not needed. (This is in contrast to **vertical scaling**, which would increase the resources assigned to pods - CPU and memory - to handle increasing levels of work.)

The manifest file for deploying a service with scaling is "deployment_hpa.yaml", and it is inside the "\~/k8s-ml/deploy_hpa" directory. You can see it [here](https://github.com/teaching-on-testbeds/k8s-ml/blob/gh-pages/deploy_hpa/deployment_hpa.yaml).

There are two differences between this deployment and the previous deployment:

-   in this one, we add a service of `type: HorizontalPodAutoscaler`. We specify the minimum number of replicas and maximum number of replicas we want to have in our deployment, and the condition under which to increase the number of replicas. (This `HorizontalPodAutoscaler` service is *in addition to* the `LoadBalancer` service, which is also in place.)
-   in this one, we specify `replicas: 1` again in the deployment - the initial deployment has 1 replica, but it may be scaled up to 5 by the autoscaler.

To start this deployment, we will run:

    kubectl apply -f ~/k8s-ml/deploy_hpa/deployment_hpa.yaml

Let's check the status of the service. Run the command:

    kubectl get svc -o wide

and then

    kubectl get pods -o wide

Initially, you will see one pod in the deployment. Wait until the pod is "Running" and has a "1/1" in the "Ready" column.

Get the URL of the service - run

    echo http://$(curl -s ifconfig.me/ip):32000

copy and paste this URL into your browser's address bar, and verify that your app is up and running there.

### Test deployment under load

You will need two SSH sessions on node-0. In one, run

    kubectl get hpa --watch

to see the current state of the autoscaler.

In the second SSH session, run

    siege -c 10 -t 360s http://$(curl -s ifconfig.me/ip):32000/test

Note that this test is of a longer duration, so that you will have time to observe additional replicas being brought up and becoming ready to use.

### Stop the deployment

When you are done with your experiment, make sure to delete the deployment and services. To delete, run the command:

    kubectl delete -f  ~/k8s-ml/deploy_hpa/deployment_hpa.yaml

Use

    kubectl get pods -o wide

and verify that (eventually) no pods are running your app.

Also, re-enable the firewall:

    sudo systemctl start firewalld.service
