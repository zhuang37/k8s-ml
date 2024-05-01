## Exercise: Deploy image classification as a web service

We will start by deploying an image classification model as a web service. Users can upload an image to a basic web app (in Flask) in their browser, then the app will:

* resize the image to the correct input dimensions
* pass it as input to an image classification model that has been fine tuned for food classification
* and return the most likely class (and estimated confidence), as well as the wall time of the `model.predict` call.

### Containerize the basic web app

To make it easy to deploy our basic application, we will containerize it - package the source code together with an operating system, software dependencies, and anything else it needs to run. This is a convenient way to distribute the code along with everything it needs. This will also make it much easier to deploy multiple copies of the application (to handle heavier load) in future exercises.

The source code for our basic application is inside the `app` subdirectory inside `k8s-ml`. It has the following materials:

```
-   app
    -   instance
    -   static
    -   templates
    -   app.py
    -   Dockerfile
    -   requirements.txt
    -   model.keras
```

You can browse these files in the web interface [here](https://github.com/teaching-on-testbeds/k8s-ml/tree/gh-pages/app). 

Note that a saved model - `model.keras` - is inside this directory.  Then, inside `app.py`, 

* when `app.py` runs, it loads the saved model: `model = load_model("model.keras")`
* when a user uploads an images file to the app, or when a special "test" path in the URL is used, a `model_predict` function is called that returns the predicted class of the image:

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


* what "base" container should be used (we'll use a Python 3.9 base container)
* what needs to run inside the container when it starts (we'll install the libraries listed in `requirements.txt` using `pip`)
* what files should be copied to the container (everything in `app`)
* and what command to run on the container when it is ready (we will run `python app.py` to start our web app)

We will use this Dockerfile to build the container (naming it `ml-app`) and then push it to a local distribution "registry" of containers (which is already running on node-0, on port 5000).

In an SSH session on "node-0", run

```
docker build -t ml-app:0.0.1 ~/k8s-ml/app
docker tag ml-app:0.0.1  node-0:5000/ml-app:0.0.1
docker push node-0:5000/ml-app:0.0.1
```

### Deploy the basic web app

Now that we have containerized our application, we can run it! Let's run it now, and will indicate that we want incoming requests on port 32000 to be passed to port 5000 on the container (where our Flask application is listening). In an SSH session on "node-0", run

```
docker run -d -p 32000:5000 node-0:5000/ml-app:0.0.1
```

Here, 

-   `-d` is for detach mode - so we can leave it running in the background. (If you need to run a container in the foreground, for debugging purposes, you will omit this argument.)
-   `-p` is to assign the mapping between "incoming request port" (32000) and "container port' (5000).


You can see the list of running containers with 

```
docker ps
```

which will show containers related to the docker register and Kubernetes deployment, but will also show one running container using the `node-0:5000/ml-app:0.0.1` image. To restrict the output to just this image, we can use

```
docker ps -f ancestor=node-0:5000/ml-app:0.0.1
```

The firewall on the node will block access to our app from outside, so let us temporarily disable it:

```
sudo systemctl stop firewalld.service
```


Now we can visit our web service and try it out! Run this command on "node-0" to get the URL to use:

```
echo http://$(curl -s ifconfig.me/ip):32000
```

Then, open your browser, paste this URL into the address bar, and hit Enter.

When the web app has loaded, upload an image to your classification service, and check its prediction.

You can also see the resource usage - in terms of CPU and memory - of your container, with 

```
docker stats $(docker ps -q -f ancestor=node-0:5000/ml-app:0.0.1)
```

(which uses command substitution to get the ID of any running container using our `ml-app` image, then gets its statistics). Use Ctrl+C to stop this live display.


When you are finished, you can stop the container by running:


```
docker stop $(docker ps -q -f ancestor=node-0:5000/ml-app:0.0.1)
```

(which uses command substitution to get the ID of any running container using our `ml-app` image, then stops it).

