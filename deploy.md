# Design and test a deployment "recipe" for your model


One you have trained a model, you will design and evaluate a deployment "recipe" for this model. The goal is to

* minimize the prediction latency,
* subject to not using more resources than necessary. You should assume that there is a "cost" to resource usage, i.e. you will pay for memory and CPU that is "reserved" for your application.


## Exercise: Prepare to evaluate your model and "recipe"

This sequence assumes that you have already deployed a Kubernetes cluster on Chameleon, using [this recipe](https://chameleoncloud.org/experiment/share/9bae9a8a-68fa-402c-ae51-41431eb78732), and that you can SSH into the primary node in your cluster.

On the primary node in your cluster, "node-0", run

```
git clone --single-branch --branch gh-pages https://github.com/teaching-on-testbeds/k8s-ml.git
```

to get the materials you will need for these exercises.

Also, install Siege if it is not already installed:

    sudo apt-get update; sudo apt-get -y install siege

And, run

```
sudo apt update; sudo apt -y install python3-pip # install pip if not already installed
python3 -m pip install kubernetes
```

to install the Python Kubernetes client, which you will need for monitoring your deployment.


## Exercise: Build a container with *your* model

Next, you will upload a saved model ("model.keras" file) to "node-0".

The easiest way is to use a free file upload service, then download the file on the remote host. For example:

* upload your model.keras file to https://www.file.io/
* copy the download link that is provided
* in an SSH session on "node-0", run


<pre>
wget https://file.io/<mark>XXXXXXXXXXXX</mark> -O k8s-ml/app/model.keras
</pre>

substituting your *own* download link in place of the highlighted portion above.

Check the size of this model on disk with

```
ls -l k8s-ml/app/model.keras
```

Next, you will build (or re-build) a container with this model and push it to the distribution registry. In an SSH session on "node-0", run


    docker build -t ml-app:0.0.1 ~/k8s-ml/app
    docker tag ml-app:0.0.1  node-0:5000/ml-app:0.0.1
    docker push node-0:5000/ml-app:0.0.1

Check that your container works, and serves your model. First, run

    echo http://$(curl -s ifconfig.me/ip):32000

to get the URL that your service will run on. Also, temporarily disable the firewall:

    sudo systemctl stop firewalld.service

Then run

    docker run -p 32000:5000 node-0:5000/ml-app:0.0.1

to start your container in the foreground - this way, if the web app crashes, you will see any error messages that may help with debugging. 

Open the URL in your browser, and test the service by uploading a food image. Make sure it returns a predicted class.

When you have tested the service, use Ctrl+C to stop it. Check with 

    docker ps -f ancestor=node-0:5000/ml-app:0.0.1

to be sure that the container is no longer running.


## Exercise: Check the response time of a pod (under load)

Now, you will see how quickly your app returns a response, when under heavy load.

Run

    kubectl apply -f ~/k8s-ml/deploy_k8s/deployment_k8s.yaml

to deploy a pod with the container that you just built.

Monitor the output of 

    kubectl get pods -o wide

until the pod is ready. Then, run

    kubectl top pod

and save the output. In particular, make a note of the memory used by the pod when it is *not* under load.

Open a second SSH session on node-0. In one, run

    watch -n 5 kubectl top pod

to monitor the pod's resource usage in real time (This will be updated every 5 seconds). In the second SSH session, run

    siege -c 10 -t 60s http://$(curl -s ifconfig.me/ip):32000/test


When the siege is finished (after 60 seconds), use Ctrl+C to stop monitoring the pod's resource usage. Save the results of the siege test.

---

A note about Siege results:

* 'Availability' tells you what percent of requests were satisfied. You want this number to be 100%, or as close to it as possible.
* 'Response time' tells you, on average, how long a successful request had to wait for a response. You want this number to be very small.
* 'Transaction rate' tells you, on average, the number of transactions served per unit time. You want this number to be large. 

---

Stop your deployment with

    kubectl delete -f ~/k8s-ml/deploy_k8s/deployment_k8s.yaml

and wait until the deployment is terminated.

## Exercise: Check the response time of a "max-sized" deployment (under load)

Next, you are going to measure a "maximum" deployment size - you are going to try and balance the load across as many replicas as possible, and then measure the response time of the load balanced service.

First, use

    nano ~/k8s-ml/deploy_lb/deployment_lb.yaml

to open the load-balanced configuration file for editing. Change the value of `replicas` to 20.

Use Ctrl+O to save the file, then hit Enter to accept. Use Ctrl+X to exit.

To start the deployment, run

    kubectl apply -f ~/k8s-ml/deploy_lb/deployment_lb.yaml

and use 

    kubectl get pods -o wide

to watch as the pods are brought up.

You will notice that after a while, you will have some "Running" pods and some that are "Pending". The cluster cannot bring up the 20 replicas you asked for, because it does not have enough CPU or memory available (across the entire cluster) to support that number of replicas with the resource requests specifed in the configuration file.

Run

    kubectl describe nodes

and scroll through the output. This tells you about resource usage for each node in the cluster. For example,

```
  Resource           Requests         Limits
  --------           --------         ------
  cpu                3800m (95%)      6300m (157%)
  memory             6425636Ki (79%)  13594617088 (165%)
```

indicates that the pods running on this node have cumulatively requested (lower limit on resource usage) 95% of the available CPU time and 79% of the available memory.

To test the deployment, run

    siege -c 10 -t 60s http://$(curl -s ifconfig.me/ip):32000/test


When the siege is finished (after 60 seconds), save the results. Stop the deployment, with

    kubectl delete -f ~/k8s-ml/deploy_lb/deployment_lb.yaml

and wait until all pods are terminated.

To deploy more replicas, you can try to reduce the CPU and memory requested. Use

    nano ~/k8s-ml/deploy_lb/deployment_lb.yaml

to open the load-balanced configuration file for editing. Change the CPU and memory values in the resource "requests" section. (Note: CPU time can be specified as a float, e.g. "0.5" is half of one CPU. Memory can be specified in Gi or Mi.)

Use Ctrl+O to save the file, then hit Enter to accept. Use Ctrl+X to exit.

To start the deployment, run

    kubectl apply -f ~/k8s-ml/deploy_lb/deployment_lb.yaml

and use 

    kubectl get pods -o wide

to watch as the pods are brought up. Note the number of pods that are "Running" and reach the "Ready" state - if you decreased the CPU and memory allocated to each pod, you should see that you are able to deploy more pods. )

However, more pods with less resources allocated to each pod, will not necessarily have better performance than fewer pods with more resources allocated to each pod. 

You should experiment with different configurations to see how to get the best performance: high availability, high transaction rate, and low response time. (Note that different models may have a different "max size" deployment, since they have different resource requirements.)

To test the deployment, run

    siege -c 10 -t 60s http://$(curl -s ifconfig.me/ip):32000/test


When the siege is finished (after 60 seconds), save the results. Stop the deployment, with

    kubectl delete -f ~/k8s-ml/deploy_lb/deployment_lb.yaml

and wait until all pods are terminated.

## Exercise: Design a horizontal scaling configuration for managing performance and resource usage under variable load

> Note: you will do this section for each of "your" two models that you trained, but not for the predecessor's model.

#### Basic principles

Finally, you will design a deployment "recipe" - a configuration for a horizontal scaling deployment, similar to the example [here](deploy_hpa/deployment_hpa.yaml) - for each model that you trained from scratch. 

Your model will have the best response time and best transaction rate in a "max"-size deployment with load balancing. However, this configuration is expensive - assuming you pay according to the resources that are deployed, you do not want to pay for a "max"-size deployment 100% of the time, even when load is low.

You want to balance resource usage and keep costs low when load is low, while still keeping response time low when load is high.

To achieve this, you will modify these values in the configuration for a horizontal scaling deployment - 

First, you will define the resource requests (lower limit) and limits (upper limit) for each copy of the container -

```
resources:
    limits:
    cpu: "2"
    memory: "4Gi"
    requests:
    cpu: "1"
    memory: "2Gi"
```

These determine the resource requirements of the container, in terms of CPU cores and memory. The "request" defines the minimum resource a container may get, and the "limit" defines the maximum resource a container may get. (Note: CPU time can be specified as a float, e.g. "0.5" is half of one CPU. Memory can be specified in Gi or Mi.)

You will also decide on the following values for the autoscaling policy:

* `maxReplicas`
* `minReplicas`
* `targetCPUUtilizationPercentage`

these specify the minimum number of replicas and maximum number of replicas you want to have in your deployment, and the condition on CPU utilization under which to increase the number of replicas. 

The key principle here is: 

* your deployment will start with `minReplicas` pods.
* If your existing pods become very busy serving requests (have a high CPU utilization, greater than `targetCPUUtilizationPercentage`), then new requests will have to wait in line to be served, increasing their prediction serving latency. 
* Therefore, the autoscaler should increase the number of pods up to `maxReplicas`. 
* However, it will take some time to bring up the new pods - the bigger your model, the more time it will take! - and during this time the prediction serving latency will still be high.

#### Editing the configuration file and starting the deployment

Use

    nano ~/k8s-ml/deploy_hpa/deployment_hpa.yaml

to open the load-balanced configuration file for editing, and make your changes. Use Ctrl+O to save the file, then hit Enter to accept. Use Ctrl+X to exit.

To start this deployment, we will run:

    kubectl apply -f ~/k8s-ml/deploy_hpa/deployment_hpa.yaml

Check the status of the service. Run 

    kubectl get pods -o wide

Initially, you will see `minReplicas` pods in the deployment. Wait until the minimum deployment is "Running" and has a "1/1" in the "Ready" column for each pod.

#### Getting the evaluation scripts

To evaluate your deployment "recipe", you will use two Python scripts:


* one that monitors resource usage over time
* and one that generates *variable* load and measure response time and transaction rate


Get the Python script for monitoring resource usage with

```
wget -O ~/resource_monitor.py https://raw.githubusercontent.com/teaching-on-testbeds/k8s-ml/gh-pages/challenge/resource_monitor.py
```

Once you have downloaded the Python script, you can monitor resource usage at any time with e.g.

```
python3 ~/resource_monitor.py  -d 30 -o ~/resource_usage.csv
```

where 

* the `-d` argument is used to specify monitoring duration, in seconds
* the `-o` argument is used to specify the output file to save results to

A row of data will be written to the file you specified - in this case, `~/resource_usage.csv` - every 5 seconds, with the following statistics:

* time since beginning of monitoring
* number of `ml-app` containers currently deployed
* total number of CPU cores [requested](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-requests-are-scheduled) by all the `ml-app` containers
* total memory in KB [requested](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-requests-are-scheduled) by all the `ml-app` containers
* total [limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits) on CPU cores of all the `ml-app` containers
* total [limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits) on memory in KB of all the `ml-app` containers
* total number of CPU cores actually used by all the `ml-app` containers
* total memory in KB actually used by all the `ml-app` containers

You can use any data analysis tool for CSV files (e.g. Python with `pandas`, `matplotlib`) on the `~/resource_usage.csv` output file.

Get the script for generating a variable load by running:

```
wget -O ~/load_test.sh https://raw.githubusercontent.com/teaching-on-testbeds/k8s-ml/gh-pages/challenge/load_test.sh
```

When you run this script, it will generate a file `~/load_output.csv` with Siege test results reported every minute (average over the previous minute), which you can similarly analyze with the data analysis tool of your choice.

#### Evaluting your deployment (40 minutes)

Now that you are ready with both scripts, you can evaluate your "recipe"! This takes 40 minutes, but is hands-off (you can leave it running and return 40 minutes later to see the results). We will use a utility called `screen` to make sure that the tests keep running even if we close our browser and disconnect the SSH session.

Run


```
screen -d -m python3 /home/cc/resource_monitor.py  -d 2400 -o /home/cc/resource_usage.csv
```

and immediately after, run

```
screen -d -m bash /home/cc/load_test.sh
```

After a few minutes, you will start to see some results reported in the CSV files - you can run

```
tail load_output.csv
```

and 

```
tail resource_usage.csv
```

to see the latest entries.

> Note: If you need to forcibly stop this evaluation before it is finished, you can run: `killall screen`


Once 40 minutes has passed, you can use `scp` to transfer these files from "node-0" to the Jupyter environment, and then download them or write Python code to analyze them further. In the Jupyter instance, open a new terminal (File > New > Terminal). Run

<pre>
scp cc@<mark>IP_ADDRESS</mark>:~/load_output.csv load_output.csv
</pre>

and 

<pre>
scp cc@<mark>IP_ADDRESS</mark>:~/resource_usage.csv resource_usage.csv
</pre>

substituting the IP address of your own instance. You should then be able to find your CSV files in the file browser on the left side.


#### Stopping the deployment

To stop this deployment, run:

    kubectl delete -f ~/k8s-ml/deploy_hpa/deployment_hpa.yaml

and confirm that no pods are left running: 

    kubectl get pods -o wide

