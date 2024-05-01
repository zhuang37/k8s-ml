# Present the results of your experiments

Your manager has asked for a PDF set of slides, that they can use to present your overall findings to the senior executives at GourmetGram. This page describes the exact contents of these slides that your manager has requested.

## Slide 1: Table of model training choices

On the first slide, you will prepare a table enumerating your model training choices.

* it should have three columns. The second and third columns should be named "Optimizing Accuracy" and "Optimizing Latency". 
* in the first column, list the following, each in its own row:
  * transforms used in data augmentation
  * base model (include name, size, top-1 accuracy, CPU inference time - from the table in the Keras docs)
  * number of epochs. optimizer, and learning rate used to train classification head  (e.g. "77 epochs Adam @ 0.53")
  * number of layers un-frozen
  * number of epochs, optimizer, and learning rate used to further fine-tune the model
  * final accuracy on evaluation set (test set)
* then fill in each cell in the "Optimizing Accuracy" and "Optimizing Latency" columns.
* highlight any rows for which you made different choices for the "accuracy model" vs the "latency model".

Your manager will need to explain to the senior executives *why* the highlighted choices were made differently for the "accuracy model" vs the "latency model". You'll prepare some speaker notes to help them explain this.

## Slide 2: Performance of models when deployed as a single pod

Using the results of your deployment experiments, you are going to create the following figures, and put them (side-by-side) on the next slide.

**Response time of each model, deployed as a single pod**: Using the results of your siege test from "Check the response time of a pod (under load)", you will create a scatter plot with:

* evaluation set (test set) accuracy on the vertical axis
* response time (from the siege test) on the horizontal axis, with the axis inverted so that low response time is on the right and high response time is on the left.
* plot three points on these axes - one for each of your two models (model optimized for accuracy, model optimized for latency), and one for the predecessor's model
* use an appropriate range for comparing the models, so that you can see the data well. You should *not* start an axis at zero, if the data is more easily understood by choosing a different axis range.
* label each axis and each point, and give the plot an appropriate title.

**Transaction rate of each model, deployed as a single pod**: Using the results of your siege test from "Check the response time of a pod (under load)", you will create a scatter plot with:

* evaluation set (test set) accuracy on the vertical axis
* transaction rate (from the siege test) on the horizontal axis, a high transaction rate on the right and a low transaction rate on the left.
* plot three points on these axes - one for each of your two models (model optimized for accuracy, model optimized for latency), and one for the predecessor's model
* use an appropriate range for comparing the models, so that you can see the data well. You should *not* start an axis at zero, if the data is more easily understood by choosing a different axis range.
* label each axis and each point, and give the plot an appropriate title.

You'll have to prepare some speaker notes again - what should your manager say when presenting this slide? Also in the speaker notes, comment on the "availability" results, which are not visualized, but which your manager should discuss if relevant.


## Slide 3: Performance of models when deployed as a "max-size" deployment

Using the results of your deployment experiments, you are going to create the following figures, and put them (side-by-side) on the next slide.


**Response time of each model, deployed as a "max-size" deployment**: Using the results of your siege test from "Check the response time of a "max-size" deployment (under load)", you will create a scatter plot with:

* evaluation set (test set) accuracy on the vertical axis
* response time (from the siege test) on the horizontal axis, with the axis inverted so that low response time is on the right and high response time is on the left.
* plot three points on these axes - one for each of your two models (model optimized for accuracy, model optimized for latency), and one for the predecessor's model
* use an appropriate range for comparing the models, so that you can see the data well. You should *not* start an axis at zero, if the data is more easily understood by choosing a different axis range.
* label each axis and each point, and give the plot an appropriate title.

**Transaction rate of each model, deployed as a "max-size" deployment**: Using the results of your siege test from "Check the response time of a "max-size" deployment (under load)", you will create a scatter plot with:

* evaluation set (test set) accuracy on the vertical axis
* transaction rate (from the siege test) on the horizontal axis, a high transaction rate on the right and a low transaction rate on the left.
* plot three points on these axes - one for each of your two models (model optimized for accuracy, model optimized for latency), and one for the predecessor's model
* use an appropriate range for comparing the models, so that you can see the data well. You should *not* start an axis at zero, if the data is more easily understood by choosing a different axis range.
* label each axis and each point, and give the plot an appropriate title.


You'll have to prepare some speaker notes again - what should your manager say when presenting this slide? Also in the speaker notes, comment on the "availability" results, which are not visualized, but which your manager should discuss if relevant.


## Slide 4: Table showing replicas and resource request configurations for "max-size" deployment

On the next slide, you will prepare a table explaining what a "max-size" deployment is for each of the three models you evaluated.

* it should have four columns. The second, third, and fourth columns should be named "Previous", "Accuracy" and "Latency". 
* in the first column, list the following, each in its own row:
  * number of replicas (note: this should be the number of replicas that are actually deployed to "Running" state - don't include those that are "Pending" for lack of resources)
  * CPU resource "requests"
  * memory resource "requests"
  * CPU resource "limits"
  * memory resource "limits"
* then fill in each cell in the "Previous", "Accuracy" and "Latency" columns.

Your manager will need to explain to the senior executives *why* you chose these specific resource requests and why the number of replicas in a "max-size" deployment is not necessarily the same for all models. You'll prepare some speaker notes to help them explain this, with specific reference to *your* models.


## Slide 5: Table showing horizontal scaling configurations

On the next slide, you will prepare a table showing your horizontal scaling configuration ("deployment_hpa.yaml") for each of the two models you trained.

* it should have three columns. The second and third columns should be named "Accuracy" and "Latency". 
* in the first column, list the following, each in its own row:
  * `minReplicas`
  * `maxReplicas`
  * `targetCPUUtilizationPercentage`
  * CPU resource "requests"
  * memory resource "requests"
  * CPU resource "limits"
  * memory resource "limits"
* then fill in each cell in the "Accuracy" and "Latency" columns with the values you used in "deployment_hpa.yaml".

Your manager will need to explain to the senior executives *why* you chose these values for these specific models. You'll prepare some speaker notes to help them explain this, with specific reference to *your* models.

## Slide 6: Visualize the deployment for your "accuracy" model over time

On the next slide, you will prepare a plot with the following subplots, for the evaluation of your "recipe" for the "accuracy" model:

* Row 1, Column 1: Number of requests over time (from the "Trans" column in "load_output.csv". This value is reported every minute.) This represents the load on the service in this variable load scenario.
* Row 1, Column 2: Average response time over time (from the "Resp Time" column in "load_output.csv". This value is reported every minute.)
* Row 1, Column 3: Availability over time (use the "OKAY" column divided by the "Trans" column in "load_output.csv". This value is reported every minute.)

* Row 2, Column 1: Number of replicas over time (from the "n_replica" column in "resource_usage.csv". This value is reported every 5 seconds.)
* Row 2, Column 2: CPU over time - from "resource_usage.csv", include one line each for
  * "cpu_req_core" (the sum "request" value for all deployed pods) 
  * "cpu_lim_core" (the sum "limit" value for all deployed pods) 
  * "cpu_use_core" (the actual usage of all deployed pods)
* Row 2, Column 3: Memory over time - from "resource_usage.csv", include one line each for
  * "mem_req_KB" (the sum "request" value for all deployed pods) 
  * "mem_lim_KB" (the sum "limit" value for all deployed pods) 
  * "mem_use_KB" (the actual usage of all deployed pods)

Each subplot should be a line plot, and each should have appropriately labeled vertical and horizontal axes. When there are multiple lines in the same plot, make sure to include a legend. Use appropriate ranges for each vertical axis.

You'll have to prepare some speaker notes again - what should your manager say when presenting this slide? 

## Slide 7: Visualize the deployment for your "latency" model over time

On the next slide, you will prepare a plot with the following subplots, for the evaluation of your "recipe" for the "latency" model:

* Row 1, Column 1: Number of requests over time (from the "Trans" column in "load_output.csv". This value is reported every minute.) This represents the load on the service in this variable load scenario.
* Row 1, Column 2: Average response time over time (from the "Resp Time" column in "load_output.csv". This value is reported every minute.)
* Row 1, Column 3: Availability over time (use the "OKAY" column divided by the "OKAY" + "Failed" column in "load_output.csv". This value is reported every minute.)

* Row 2, Column 1: Number of replicas over time (from the "n_replica" column in "resource_usage.csv". This value is reported every 5 seconds.)
* Row 2, Column 2: CPU over time - from "resource_usage.csv", include one line each for
  * "cpu_req_core" (the sum "request" value for all deployed pods) 
  * "cpu_lim_core" (the sum "limit" value for all deployed pods) 
  * "cpu_use_core" (the actual usage of all deployed pods)
* Row 2, Column 3: Memory over time - from "resource_usage.csv", include one line each for
  * "mem_req_KB" (the sum "request" value for all deployed pods) 
  * "mem_lim_KB" (the sum "limit" value for all deployed pods) 
  * "mem_use_KB" (the actual usage of all deployed pods)

Each subplot should be a line plot, and each should have appropriately labeled vertical and horizontal axes. When there are multiple lines in the same plot, make sure to include a legend. Use appropriate ranges for each vertical axis.

You'll have to prepare some speaker notes again - what should your manager say when presenting this slide? 

## Slide 8: Summarize your contributions

In the final slide, write a few bullet points summarizing the value of the models and the deployment configurations that you developed in your time at GourmetGram (compared to what your predecessor had left you).