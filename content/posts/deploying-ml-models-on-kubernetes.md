+++
title = " Deploying machine learning models on kubernetes"
tags = ["python","rust","c"]
date = "2021-03-03"
+++
---

# Deploying Machine Learning Models on Kubernetes

[Bodywork](https://bodywork.readthedocs.io/en/latest/) A common pattern for deploying Machine Learning (ML) models into production environments  - e.g. ML models trained using the SciKit Learn or Keras packages (for Python), that are ready to provide predictions on new data - is to expose these ML as RESTful API microservices, hosted from within [Docker](https://www.docker.com) containers. These can then deployed to a cloud environment for handling everything required for maintaining continuous availability - e.g. fault-tolerance, auto-scaling, load balancing and rolling service updates.

The configuration details for a continuously available cloud deployment are specific to the targeted cloud provider(s) - e.g. the deployment process and topology for Amazon Web Services is not the same as that for Microsoft Azure, which in-turn is not the same as that for Google Cloud Platform. This constitutes knowledge that needs to be acquired for every cloud provider. Furthermore, it is difficult (some would say near impossible) to test entire deployment strategies locally, which makes issues such as networking hard to debug.

[Kubernetes](https://kubernetes.io) is a container orchestration platform that seeks to address these issues. Briefly, it provides a mechanism for defining **entire** microservice-based application deployment topologies and their service-level requirements for maintaining continuous availability. It is agnostic to the targeted cloud provider, can be run on-premises and even locally on your laptop - all that's required is a cluster of virtual machines running Kubernetes - i.e. a Kubernetes cluster.

This README is designed to be read in conjunction with the code in this repository, that contains the Python modules, Docker configuration files and Kubernetes instructions for demonstrating how a simple Python ML model can be turned into a production-grade RESTful model-scoring (or prediction) API service, using Docker and Kubernetes - both locally and with Google Cloud Platform (GCP). It is not a comprehensive guide to Kubernetes, Docker or ML - think of it more as a 'ML on Kubernetes 101' for demonstrating capability and allowing newcomers to Kubernetes (e.g. data scientists who are more focused on building models as opposed to deploying them), to get up-and-running quickly and become familiar with the basic concepts and patterns.

We will demonstrate ML model deployment using two different approaches: a first principles approach using Docker and Kubernetes; and then a deployment using the [Seldon-Core](https://www.seldon.io) Kubernetes native framework for streamlining the deployment of ML services. The former will help to appreciate the latter, which constitutes a powerful framework for deploying and performance-monitoring many complex ML model pipelines.

This work was initially committed in 2018 and has since formed the basis of [Bodywork](https://github.com/bodywork-ml/bodywork-core) - an open-source MLOps tool for deploying machine learning projects developed in Python, to Kubernetes. Bodywork automates a lot of the steps that this project has demonstrated to the many machine learning engineers that have used it over the years - take a look at the [documentation](https://bodywork.readthedocs.io/en/latest/).

## Containerising a Simple Machine Learning Model Scoring Service using Flask and Docker

We start by demonstrating how to achieve this basic competence using the simple Python ML model scoring REST API contained in the `api.py` module, together with the `Dockerfile`, both within the `py-flask-ml-score-api` directory, whose core contents are as follows,

```s
py-flask-ml-score-api/
 |-- Dockerfile
 |-- Pipfile
 |-- Pipfile.lock
 |-- api.py
```

If you're already feeling lost then these files are discussed in the points below, otherwise feel free to skip to the next section.

### Defining the Flask Service in the `api.py` Module

This is a Python module that uses the [Flask](http://flask.pocoo.org) framework for defining a web service (`app`), with a function (`score`), that executes in response to a HTTP request to a specific URL (or 'route'), thanks to being wrapped by the `app.route` function. For reference, the relevant code is reproduced below,

```python
from flask import Flask, jsonify, make_response, request

app = Flask(__name__)


@app.route('/score', methods=['POST'])
def score():
    features = request.json['X']
    return make_response(jsonify({'score': features}))


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

If running locally - e.g. by starting the web service using `python run api.py` - we would be able reach our function (or 'endpoint') at `http://localhost:5000/score`. This function takes data sent to it as JSON (that has been automatically de-serialised as a Python dict made available as the `request` variable in our function definition), and returns a response (automatically serialised as JSON).

In our example function, we expect an array of features, `X`, that we pass to a ML model, which in our example returns those same features back to the caller - i.e. our chosen ML model is the identity function, which we have chosen for purely demonstrative purposes. We could just as easily have loaded a pickled SciKit-Learn or Keras model and passed the data to the approproate `predict` method, returning a score for the feature-data as JSON - see [here](https://github.com/AlexIoannides/ml-workflow-automation/blob/master/deploy/py-sklearn-flask-ml-service/api.py) for an example of this in action.

### Defining the Docker Image with the `Dockerfile`

 A `Dockerfile` is essentially the configuration file used by Docker, that allows you to define the contents and configure the operation of a Docker container, when operational. This static data, when not executed as a container, is referred to as the 'image'. For reference, the `Dockerfile` is reproduced below,

```s
FROM python:3.6-slim
WORKDIR /usr/src/app
COPY . .
RUN pip install pipenv
RUN pipenv install
EXPOSE 5000
CMD ["pipenv", "run", "python", "api.py"]
```

In our example `Dockerfile` we:

 - start by using a pre-configured Docker image (`python:3.6-slim`) that has a version of the [Alpine Linux](https://www.alpinelinux.org/community/) distribution with Python already installed;
 - then copy the contents of the `py-flask-ml-score-api` local directory to a directory on the image called `/usr/src/app`;
 - then use `pip` to install the [Pipenv](https://pipenv.readthedocs.io/en/latest/) package for Python dependency management (see the appendix at the bottom for more information on how we use Pipenv);
 - then use Pipenv to install the dependencies described in `Pipfile.lock` into a virtual environment on the image;
 - configure port 5000 to be exposed to the 'outside world' on the running container; and finally,
 - to start our Flask RESTful web service - `api.py`. Note, that here we are relying on Flask's internal [WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface) server, whereas in a production setting we would recommend on configuring a more robust option (e.g. Gunicorn), [as discussed here](https://pythonspeed.com/articles/gunicorn-in-docker/).

 Building this custom image and asking the Docker daemon to run it (remember that a running image is a 'container'), will expose our RESTful ML model scoring service on port 5000 as if it were running on a dedicated virtual machine. Refer to the official [Docker documentation](https://docs.docker.com/get-started/) for a more comprehensive discussion of these core concepts.

### Building a Docker Image for the ML Scoring Service

We assume that [Docker is running locally](https://www.docker.com) (both Docker client and daemon), that the client is logged into an account on [DockerHub](https://hub.docker.com) and that there is a terminal open in the this project's root directory. To build the image described in the `Dockerfile` run,

```s
$ docker build --tag alexioannides/test-ml-score-api py-flask-ml-score-api
```

Where 'alexioannides' refers to the name of the DockerHub account that we will push the image to, once we have tested it. 

#### Testing

To test that the image can be used to create a Docker container that functions as we expect it to use,

```s
$ docker run --rm --name test-api -p 5000:5000 -d alexioannides/test-ml-score-api
```

Where we have mapped port 5000 from the Docker container - i.e. the port our ML model scoring service is listening to - to port 5000 on our host machine (localhost). Then check that the container is listed as running using,

```s
$ docker ps
```

And then test the exposed API endpoint using,

```s
$ curl http://localhost:5000/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"X": [1, 2]}'
```

Where you should expect a response along the lines of,

```json
{"score":[1,2]}
```

All our test model does is return the input data - i.e. it is the identity function. Only a few lines of additional code are required to modify this service to load a SciKit Learn model from disk and pass new data to it's 'predict' method for generating predictions - see [here](https://github.com/AlexIoannides/ml-workflow-automation/blob/master/deploy/py-sklearn-flask-ml-service/api.py) for an example. Now that the container has been confirmed as operational, we can stop it,

```s
$ docker stop test-api
```

#### Pushing the Image to the DockerHub Registry

In order for a remote Docker host or Kubernetes cluster to have access to the image we've created, we need to publish it to an image registry. All cloud computing providers that offer managed Docker-based services will provide private image registries, but we will use the public image registry at DockerHub, for convenience. To push our new image to DockerHub (where my account ID is 'alexioannides') use,

```s
$ docker push alexioannides/test-ml-score-api
```

Where we can now see that our chosen naming convention for the image is intrinsically linked to our target image registry (you will need to insert your own account ID where required). Once the upload is finished, log onto DockerHub to confirm that the upload has been successful via the [DockerHub UI](https://hub.docker.com/u/alexioannides).

## Installing Kubernetes for Local Development and Testing

There are two options for installing a single-node Kubernetes cluster that is suitable for local development and testing: via the [Docker Desktop](https://www.docker.com/products/docker-desktop) client, or via [Minikube](https://github.com/kubernetes/minikube).

### Installing Kubernetes via Docker Desktop

If you have been using Docker on a Mac, then the chances are that you will have been doing this via the Docker Desktop application. If not (e.g. if you installed Docker Engine via Homebrew), then Docker Desktop can be downloaded [here](https://www.docker.com/products/docker-desktop). Docker Desktop now comes bundled with Kubernetes, which can be activated by going to `Preferences -> Kubernetes` and selecting `Enable Kubernetes`. It will take a while for Docker Desktop to download the Docker images required to run Kubernetes, so be patient. After it has finished, go to `Preferences -> Advanced` and ensure that at least 2 CPUs and 4 GiB have been allocated to the Docker Engine, which are the the minimum resources required to deploy a single Seldon ML component.

To interact with the Kubernetes cluster you will need the `kubectl` Command Line Interface (CLI) tool, which will need to be downloaded separately. The easiest way to do this on a Mac is via Homebrew - i.e with `brew install kubernetes-cli`. Once you have `kubectl` installed and a Kubernetes cluster up-and-running, test that everything is working as expected by running,

```s
$ kubectl cluster-info
```

Which ought to return something along the lines of,

```s
Kubernetes master is running at https://kubernetes.docker.internal:6443
KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### Installing Kubernetes via Minikube

On Mac OS X, the steps required to get up-and-running with Minikube are as follows:

- make sure the [Homebrew](https://brew.sh) package manager for OS X is installed; then,
- install VirtualBox using, `brew cask install virtualbox` (you may need to approve installation via OS X System Preferences); and then,
- install Minikube using, `brew cask install minikube`.

To start the test cluster run,

```s
$ minikube start --memory 4096
```

Where we have specified the minimum amount of memory required to deploy a single Seldon ML component. Be patient - Minikube may take a while to start. To test that the cluster is operational run,

```s
$ kubectl cluster-info
```

Where `kubectl` is the standard Command Line Interface (CLI) client for interacting with the Kubernetes API (which was installed as part of Minikube, but is also available separately).

### Deploying the Containerised Machine Learning Model Scoring Service to Kubernetes

To launch our test model scoring service on Kubernetes, we will start by deploying the containerised service within a Kubernetes [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/), whose rollout is managed by a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), which in in-turn creates a [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) - a Kubernetes resource that ensures a minimum number of pods (or replicas), running our service are operational at any given time. This is achieved with,

```s
$ kubectl create deployment test-ml-score-api --image=alexioannides/test-ml-score-api:latest
```

To check on the status of the deployment run,

```s
$ kubectl rollout status deployment test-ml-score-api
```

And to see the pods that is has created run,

```s
$ kubectl get pods
```

It is possible to use [port forwarding](https://en.wikipedia.org/wiki/Port_forwarding) to test an individual container without exposing it to the public internet. To use this, open a separate terminal and run (for example),

```s
$ kubectl port-forward test-ml-score-api-szd4j 5000:5000
```

Where `test-ml-score-api-szd4j` is the precise name of the pod currently active on the cluster, as determined from the `kubectl get pods` command. Then from your original terminal, to repeat our test request against the same container running on Kubernetes run,

```s
$ curl http://localhost:5000/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"X": [1, 2]}'
```

To expose the container as a (load balanced) [service](https://kubernetes.io/docs/concepts/services-networking/service/) to the outside world, we have to create a Kubernetes service that references it. This is achieved with the following command,

```s
$ kubectl expose deployment test-ml-score-api --port 5000 --type=LoadBalancer --name test-ml-score-api-lb
```

If you are using Docker Desktop, then this will automatically emulate a load balancer at `http://localhost:5000`. To find where Minikube has exposed its emulated load balancer run,

```s
$ minikube service list
```

Now we test our new service - for example (with Docker Desktop),

```s
$ curl http://localhost:5000/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"X": [1, 2]}'
```

Note, neither Docker Desktop or Minikube setup a real-life load balancer (which is what would happen if we made this request on a cloud platform). To tear-down the load balancer, deployment and pod, run the following commands in sequence,

```s
$ kubectl delete deployment test-ml-score-api
$ kubectl delete service test-ml-score-api-lb
```

## Configuring a multi-node Cluster on Google Cloud Platform

In order to perform testing on a real-world Kubernetes cluster with far greater resources than those available on a laptop, the easiest way is to use a managed Kubernetes platform from a cloud provider. We will use Kubernetes Engine on [Google Cloud Platform (GCP)](https://cloud.google.com).

### Getting up-and-running with Google Cloud Platform

Before we can use Google Cloud Platform, sign-up for an account and create a project specifically for this work. Next, make sure that the GCP SDK is installed on your local machine - e.g.,

```s
$ brew cask install google-cloud-sdk
```

Or by downloading an installation image [directly from GCP](https://cloud.google.com/sdk/docs/quickstart-macos). Note, that if you haven't already installed Kubectl, then you will need to do so now, which can be done using the GCP SDK,

```s
$ gcloud components install kubectl
```

We then need to initialise the SDK,

```s
$ gcloud init
```

Which will open a browser and guide you through the necessary authentication steps. Make sure you pick the project you created, together with a default zone and region (if this has not been set via Compute Engine -> Settings).

### Initialising a Kubernetes Cluster

Firstly, within the GCP UI visit the Kubernetes Engine page to trigger the Kubernetes API to start-up. From the command line we then start a cluster using,

```s
$ gcloud container clusters create k8s-test-cluster --num-nodes 3 --machine-type g1-small
```

And then go make a cup of coffee while you wait for the cluster to be created. Note, that this will automatically switch your `kubectl` context to point to the cluster on GCP, as you will see if you run, `kubectl config get-contexts`. To switch back to the Docker Desktop client use `kubectl config use-context docker-desktop`.

### Launching the Containerised Machine Learning Model Scoring Service on GCP

This is largely the same as we did for running the test service locally - run the following commands in sequence,

```s
$ kubectl create deployment test-ml-score-api --image=alexioannides/test-ml-score-api:latest
$ kubectl expose deployment test-ml-score-api --port 5000 --type=LoadBalancer --name test-ml-score-api-lb
```

But, to find the external IP address for the GCP cluster we will need to use,

```s
$ kubectl get services
```

And then we can test our service on GCP - for example,

```s
$ curl http://35.246.92.213:5000/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"X": [1, 2]}'
```

Or, we could again use port forwarding to attach to a single pod - for example,

```s
$ kubectl port-forward test-ml-score-api-nl4sc 5000:5000
```

And then in a separate terminal,

```s
$ curl http://localhost:5000/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"X": [1, 2]}'
```

Finally, we tear-down the replication controller and load balancer,

```s
$ kubectl delete deployment test-ml-score-api
$ kubectl delete service test-ml-score-api-lb
```

## Switching between Kubectl Contexts

If you are running both with Kubernetes locally and with a cluster on GCP, then you can switch Kubectl [context](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) from one cluster to the other, as follows,

```s
$ kubectl config use-context docker-desktop
```

Where the list of available contexts can be found using,

```s
$ kubectl config get-contexts
```

## Using YAML Files to Define and Deploy the Machine Learning Model Scoring Service

Up to this point we have been using Kubectl commands to define and deploy a basic version of our ML model scoring service. This is fine for demonstrative purposes, but quickly becomes limiting, as well as unmanageable. In practice, the standard way of defining entire Kubernetes deployments is with YAML files,  posted to the Kubernetes API. The `py-flask-ml-score.yaml` file in the `py-flask-ml-score-api` directory is an example of how our ML model scoring service can be defined in a single YAML file. This can now be deployed using a single command,

```s
$ kubectl apply -f py-flask-ml-score-api/py-flask-ml-score.yaml
```

Note, that we have defined three separate Kubernetes components in this single file: a [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), a deployment and a load-balanced service - for all of these components (and their sub-components), using `---` to delimit the definition of each separate component. To see all components deployed into this namespace use,

```s
$ kubectl get all --namespace test-ml-app
```

And likewise set the `--namespace` flag when using any `kubectl get` command to inspect the different components of our test app. Alternatively, we can set our new namespace as the default context,

```s
$ kubectl config set-context $(kubectl config current-context) --namespace=test-ml-app
```

And then run,

```s
$ kubectl get all
```

Where we can switch back to the default namespace using,

```s
$ kubectl config set-context $(kubectl config current-context) --namespace=default
```

To tear-down this application we can then use,

```s
$ kubectl delete -f py-flask-ml-score-api/py-flask-ml-score.yaml
```

Which saves us from having to use multiple commands to delete each component individually. Refer to the [official documentation for the Kubernetes API](https://kubernetes.io/docs/home/) to understand the contents of this YAML file in greater depth.

## Using Helm Charts to Define and Deploy the Machine Learning Model Scoring Service

Writing YAML files for Kubernetes can get repetitive and hard to manage, especially if there is a lot of 'copy-paste' involved, when only a handful of parameters need to be changed from one deployment to the next,  but there is a 'wall of YAML' that needs to be modified. Enter [Helm](https://helm.sh//) - a framework for creating, executing and managing Kubernetes deployment templates. What follows is a very high-level demonstration of how Helm can be used to deploy our ML model scoring service - for a comprehensive discussion of Helm's full capabilities (and here are a lot of them), please refer to the [official documentation](https://docs.helm.sh). Seldon-Core can also be deployed using Helm and we will cover this in more detail later on.

### Installing Helm

As before, the easiest way to install Helm onto Mac OS X is to use the Homebrew package manager,

```s
$ brew install kubernetes-helm
```

Helm relies on a dedicated deployment server, referred to as the 'Tiller', to be running within the same Kubernetes cluster we wish to deploy our applications to. Before we deploy Tiller we need to create a cluster-wide super-user role to assign to it, so that it can create and modify Kubernetes resources in any namespace. To achieve this, we start by creating a [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) that is destined for our tiller. A Service Account is a means by which a pod (and any service running within it), when associated with a Service Accoutn, can authenticate itself to the Kubernetes API, to be able to view, create and modify resources. We create this in the `kube-system` namespace (a common convention) as follows,

```s
$ kubectl --namespace kube-system create serviceaccount tiller
```

We then create a binding between this Service Account and the `cluster-admin` [Cluster Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), which as the name suggest grants cluster-wide admin rights,

```s
$ kubectl create clusterrolebinding tiller \
    --clusterrole cluster-admin \
    --serviceaccount=kube-system:tiller
```

We can now deploy the Helm Tiller to a Kubernetes cluster, with the desired access rights using,

```s
$ helm init --service-account tiller
```

### Deploying with Helm

To create a fresh Helm deployment definition - referred to as a 'chart' in Helm terminology - run,

```s
$ helm create NAME-OF-YOUR-HELM-CHART
```

This creates a new directory - e.g. `helm-ml-score-app` as included with this repository - with the following high-level directory structure,

```s
helm-ml-score-app/
 |-- charts/
 |-- templates/
 |--  |--Chart.yaml
 |--  |--values.yaml
```

Briefly, the `charts` directory contains other charts that our new chart will depend on (we will not make use of this), the `templates` directory contains our Helm templates, `Chart.yaml` contains core information for our chart (e.g. name and version information) and `values.yaml` contains default values to render our templates with (in the case that no values are set from the command line).

The next step is to delete all of the files in the `templates` directory (apart from `NOTES.txt`), and to replace them with our own. We start with `namespace.yaml` for declaring a namespace for our app,

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.app.namespace }}
```

Anyone familiar with HTML template frameworks (e.g. Jinja), will be familiar with the use of ``{{}}`` for defining values that will be injected into the rendered template. In this specific instance `.Values.app.namespace` injects the `app.namespace` variable, whose default value defined in `values.yaml`. Next we define a deployment of pods in `deployment.yaml`,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: {{ .Values.app.name }}
    env: {{ .Values.app.env }}
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
        env: {{ .Values.app.env }}
    spec:
      containers:
      - image: {{ .Values.app.image }}
        name: {{ .Values.app.name }}
        ports:
        - containerPort: {{ .Values.containerPort }}
          protocol: TCP
```

And the details of the load balancer service in `service.yaml`,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}-lb
  labels:
    app: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
spec:
  type: LoadBalancer
  ports:
  - port: {{ .Values.containerPort }}
    targetPort: {{ .Values.targetPort }}
  selector:
    app: {{ .Values.app.name }}
```

What we have done, in essence, is to split-out each component of the deployment details from `py-flask-ml-score.yaml` into its own file and then define template variables for each parameter of the configuration that is most likely to change from one deployment to the next. To test and examine the rendered template, without having to attempt a deployment, run,

```s
$ helm install helm-ml-score-app --debug --dry-run
```

If you are happy with the results of the 'dry run', then execute the deployment and generate a release from the chart using,

```s
$ helm install helm-ml-score-app --name test-ml-app
```

This will automatically print the status of the release, together with the name that Helm has ascribed to it (e.g. 'willing-yak') and the contents of `NOTES.txt` rendered to the terminal. To list all available Helm releases and their names use,

```s
$ helm list
```

And to the status of all their constituent components (e.g. pods, replication controllers, service, etc.) use for example,

```s
$ helm status test-ml-app
```

The ML scoring service can now be tested in exactly the same way as we have done previously (above). Once you have convinced yourself that it's working as expected, the release can be deleted using,

```s
$ helm delete test-ml-app
```

## Using Seldon to Deploy the Machine Learning Model Scoring Service to Kubernetes

Seldon's core mission is to simplify the repeated deployment and management of complex ML prediction pipelines on top of Kubernetes. In this demonstration we are going to focus on the simplest possible example - i.e. the simple ML model scoring API we have already been using.

### Building an ML Component for Seldon

To deploy a ML component using Seldon, we need to create Seldon-compatible Docker images. We start by following [these guidelines](https://docs.seldon.io/projects/seldon-core/en/latest/python/python_wrapping_docker.html) for defining a Python class that wraps an ML model targeted for deployment with Seldon. This is contained within the `seldon-ml-score-component` directory, whose contents are similar to those in `py-flask-ml-score-api`,

```s
seldon-ml-score-component/
 |-- Dockerfile
 |-- MLScore.py
 |-- Pipfile
 |-- Pipfile.lock
```

#### Building the Docker Image for use with Seldon

Seldon requires that the Docker image for the ML scoring service be structured in a particular way:

- the ML model has to be wrapped in a Python class with a `predict` method with a particular signature (or interface) - for example, in `MLScore.py` (deliberately named after the Python class contained within it) we have,

```python
class MLScore:
    """
    Model template. You can load your model parameters in __init__ from
    a location accessible at runtime
    """

    def __init__(self):
        """
        Load models and add any initialization parameters (these will
        be passed at runtime from the graph definition parameters
        defined in your seldondeployment kubernetes resource manifest).
        """
        print("Initializing")

    def predict(self, X, features_names):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        feature_names : array of feature names (optional)
        """
        print("Predict called - will run identity function")
        return X
```

- the `seldon-core` Python package must be installed (we use `pipenv` to manage dependencies as discussed above and in the Appendix below); and,
- the container starts by running the Seldon service using the `seldon-core-microservice` entry-point provided by the `seldon-core` package - both this and the point above can be seen the `DockerFile`,

```s
FROM python:3.6-slim
COPY . /app
WORKDIR /app
RUN pip install pipenv
RUN pipenv install
EXPOSE 5000

# Define environment variable
ENV MODEL_NAME MLScore
ENV API_TYPE REST
ENV SERVICE_TYPE MODEL
ENV PERSISTENCE 0

CMD pipenv run seldon-core-microservice $MODEL_NAME $API_TYPE --service-type $SERVICE_TYPE --persistence $PERSISTENCE
```

For the precise details refer to the [official Seldon documentation](https://docs.seldon.io/projects/seldon-core/en/latest/python/index.html). Next, build this image,

```s
$ docker build seldon-ml-score-component -t alexioannides/test-ml-score-seldon-api:latest
```

Before we push this image to our registry, we need to make sure that it's working as expected. Start the image on the local Docker daemon,

```s
$ docker run --rm -p 5000:5000 -d alexioannides/test-ml-score-seldon-api:latest
```

And then send it a request (using a different request format to the ones we've used thus far),

```s
$ curl -g http://localhost:5000/predict \
    --data-urlencode 'json={"data":{"names":["a","b"],"tensor":{"shape":[2,2],"values":[0,0,1,1]}}}'
```

If response is as expected (i.e. it contains the same payload as the request), then push the image,

```s
$ docker push alexioannides/test-ml-score-seldon-api:latest
```

### Deploying a ML Component with Seldon Core

We now move on to deploying our Seldon compatible ML component to a Kubernetes cluster and creating a fault-tolerant and scalable service from it. To achieve this, we will [deploy Seldon-Core using Helm charts](https://docs.seldon.io/projects/seldon-core/en/latest/workflow/install.html). We start by creating a namespace that will contain the `seldon-core-operator`, a custom Kubernetes resource required to deploy any ML model using Seldon,

```s
$ kubectl create namespace seldon-core
```

Then we deploy Seldon-Core using Helm and the official Seldon Helm chart repository hosted at `https://storage.googleapis.com/seldon-charts`,

```s
$ helm install seldon-core-operator \
  --name seldon-core \
  --repo https://storage.googleapis.com/seldon-charts \
  --set usageMetrics.enabled=false \
  --namespace seldon-core
```

Next, we deploy the Ambassador API gateway for Kubernetes, that will act as a single point of entry into our Kubernetes cluster and will be able to route requests to any ML model we have deployed using Seldon. We will create a dedicate namespace for the Ambassador deployment,

```s
$ kubectl create namespace ambassador
```

And then deploy Ambassador using the most recent charts in the official Helm repository,

```s
$ helm install stable/ambassador \
  --name ambassador \
  --set crds.keep=false \
  --namespace ambassador
```

If we now run `helm list --namespace seldon-core` we should see that Seldon-Core has been deployed and is waiting for Seldon ML components to be deployed. To deploy our Seldon ML model scoring service we create a separate namespace for it,

```s
$ kubectl create namespace test-ml-seldon-app
```

And then configure and deploy another official Seldon Helm chart as follows,

```s
$ helm install seldon-single-model \
  --name test-ml-seldon-app \
  --repo https://storage.googleapis.com/seldon-charts \
  --set model.image.name=alexioannides/test-ml-score-seldon-api:latest \
  --namespace test-ml-seldon-app
```

Note, that multiple ML models can now be deployed using Seldon by repeating the last two steps and they will all be automatically reachable via the same Ambassador API gateway, which we will now use to test our Seldon ML model scoring service.

### Testing the API via the Ambassador Gateway API

To test the Seldon-based ML model scoring service, we follow the same general approach as we did for our first-principles Kubernetes deployments above, but we will route our requests via the Ambassador API gateway. To find the IP address for Ambassador service run,

```s
$ kubectl -n ambassador get service ambassador
```

Which will be `localhost:80` if using Docker Desktop, or an IP address if running on GCP or Minikube (were you will need to remember to use `minikuke service list` in the latter case). Now test the prediction end-point - for example,

```s
$ curl http://35.246.28.247:80/seldon/test-ml-seldon-app/test-ml-seldon-app/api/v0.1/predictions \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"data":{"names":["a","b"],"tensor":{"shape":[2,2],"values":[0,0,1,1]}}}'
```

If you want to understand the full logic behind the routing see the [Seldon documentation](https://docs.seldon.io/projects/seldon-core/en/latest/workflow/serving.html), but the URL is essentially assembled using,

```html
http://<ambassadorEndpoint>/seldon/<namespace>/<deploymentName>/api/v0.1/predictions
```

If your request has been successful, then you should see a response along the lines of,

```json
{
  "meta": {
    "puid": "hsu0j9c39a4avmeonhj2ugllh9",
    "tags": {
    },
    "routing": {
    },
    "requestPath": {
      "classifier": "alexioannides/test-ml-score-seldon-api:latest"
    },
    "metrics": []
  },
  "data": {
    "names": ["t:0", "t:1"],
    "tensor": {
      "shape": [2, 2],
      "values": [0.0, 0.0, 1.0, 1.0]
    }
  }
}
```

## Tear Down

To delete a single Seldon ML model and its namespace, deployed using the steps above, run,

```s
$ helm delete test-ml-seldon-app --purge && kubectl delete namespace test-ml-seldon-app
```

Follow the same pattern to remove the Seldon Core Operator and Ambassador,

```s
$ helm delete seldon-core --purge && kubectl delete namespace seldon-core
$ helm delete ambassador --purge && kubectl delete namespace ambassador
```

If there is a GCP cluster that needs to be killed run,

```s
$ gcloud container clusters delete k8s-test-cluster
```

And likewise if working with Minikube,

```s
$ minikube stop
$ minikube delete
```

If running on Docker Desktop, navigate to `Preferences -> Reset` to reset the cluster.

## Where to go from Here

The following list of resources will help you dive deeply into the subjects we skimmed-over above:

- the full set of functionality provided by [Seldon](https://www.seldon.io/open-source/);
- running multi-stage containerised workflows (e.g. for data engineering and model training) using [Argo Workflows](https://argoproj.github.io/argo);
- the excellent '_Kubernetes in Action_' by Marko Lukša [available from Manning Publications](https://www.manning.com/books/kubernetes-in-action);
- '_Docker in Action_' by Jeff Nickoloff and Stephen Kuenzli [also available from Manning Publications](https://www.manning.com/books/docker-in-action-second-edition); and,
- _'Flask Web Development'_ by Miguel Grinberg [O'Reilly](http://shop.oreilly.com/product/0636920089056.do).

This work was initially committed in 2018 and has since formed the basis of [Bodywork](https://www.bodyworkml.com) - an open-source MLOps tool for deploying machine learning projects developed in Python, to Kubernetes.

## Appendix - Using Pipenv for managing Python Package Dependencies

We use [pipenv](https://docs.pipenv.org) for managing project dependencies and Python environments (i.e. virtual environments). All of the direct packages dependencies required to run the code (e.g. Flask or Seldon-Core), as well as any packages that could have been used during development (e.g. flake8 for code linting and IPython for interactive console sessions), are described in the `Pipfile`. Their **precise** downstream dependencies are described in `Pipfile.lock`.

### Installing Pipenv

To get started with Pipenv, first of all download it - assuming that there is a global version of Python available on your system and on the PATH, then this can be achieved by running the following command,

```s
$ pip3 install pipenv
```

Pipenv is also available to install from many non-Python package managers. For example, on OS X it can be installed using the [Homebrew](https://brew.sh) package manager, with the following terminal command,

```s
$ brew install pipenv
```

For more information, including advanced configuration options, see the [official pipenv documentation](https://docs.pipenv.org).

### Installing Projects Dependencies

If you want to experiment with the Python code in the `py-flask-ml-score-api` or `seldon-ml-score-component` directories, then make sure that you're in the appropriate directory and then run,

```s
$ pipenv install
```

This will install all of the direct project dependencies.

### Running Python, IPython and JupyterLab from the Project's Virtual Environment

In order to continue development in a Python environment that precisely mimics the one the project was initially developed with, use Pipenv from the command line as follows,

```s
$ pipenv run python3
```

The `python3` command could just as well be `seldon-core-microservice` or any other entry-point provided by the `seldon-core` package - for example, in the `Dockerfile` for the `seldon-ml-score-component` we start the Seldon-based ML model scoring service using,

```s
$ pipenv run seldon-core-microservice 
```

### Pipenv Shells

Prepending `pipenv` to every command you want to run within the context of your Pipenv-managed virtual environment, can get very tedious. This can be avoided by entering into a Pipenv-managed shell,

```s
$ pipenv shell
```

which is equivalent to 'activating' the virtual environment. Any command will now be executed within the virtual environment. Use `exit` to leave the shell session.

# Further Tutorial
Solutions to Machine Learning (ML) tasks are often developed within Jupyter notebooks. Once a solution is developed you are then faced with an altogether different problem - how to engineer the solution into your product and how to maintain the performance of the solution through time, as new data is generated.

## What is this Tutorial Going to Teach Me?

* How to take a solution to a ML task within a Jupyter notebook, and map it into two Python modules: one for training a model and one for serving the trained model via a REST API endpoint.
* How to execute the `train` and `deploy` modules (a simple ML pipeline), on a [Kubernetes](https://kubernetes.io/) cluster using [Bodywork](https://bodywork.readthedocs.io/en/latest/).
* How to test the REST API service that has been deployed to Kubernetes.
* How to run the pipeline on a schedule, so that the model is periodically re-trained and then re-deployed, without the manual intervention of an ML engineer.



## Introduction

I’ve written at length on the subject of getting machine learning into production - an area that now falls under Machine Learning Operations (MLOps). MLOps is currently a pressing topic within the field of machine learning engineering. As an example of this, take my [blog post]({filename}k8s-ml-ops.md) on *Deploying Python ML Models with Flask, Docker and Kubernetes*, which is accessed by hundreds of machine learning practitioners every month; or the fact that Thoughtwork’s [essay](https://www.thoughtworks.com/insights/articles/intelligent-enterprise-series-cd4ml) on *Continuous Delivery for ML* has become an essential reference for all machine learning engineers, together with Google’s [paper](https://papers.nips.cc/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html) on the *Hidden Technical Debt in Machine Learning Systems*; and MLOps even has its own entry on [Wikipedia](https://en.wikipedia.org/wiki/MLOps).

### Why is MLOps Getting so Much Attention?

In my opinion, this is because we are at a point where a significant number of organisations have now overcome their data ingestion and engineering problems. They are able to provide their data scientists with the data required to solve business problems using machine learning, only to find that, as Thoughtworks put it,

> “*Getting machine learning applications into production is hard*”

To tackle some of the core complexities of MLOps, many machine learning engineering teams have settled on approaches that are based-upon deploying containerised models, usually as RESTful model-scoring services, to some type of cloud platform. Kubernetes is especially useful for this as I have [written about before]({filename}k8s-ml-ops.md).

### ML Deployment with Bodywork

Running machine learning code in containers has become a common pattern to guarantee reproducibility between what has been developed and what is deployed in production environments.

Most machine learning engineers do not, however, have the time to develop the skills and expertise required to deliver and deploy containerised machine learning systems into production environments. This requires an understanding of how to build container images, how to push build artefacts to image repositories and how to configure a container orchestration platform to use these, to execute batch jobs and deploy services.

Developing and maintaining these deployment pipelines is time-consuming. If there are multiple projects - each requiring re-training and re-deployment - then the management of these pipelines will quickly become a large burden.

This is where Bodywork steps-in - it will deliver your project's Python modules directly from your Git repository into Docker containers and manage their deployment to a Kubernetes cluster. In other words, Bodywork automates the repetitive tasks that most machine learning engineers think of as [DevOps](https://en.wikipedia.org/wiki/DevOps), allowing them to focus their time on what they do best - machine learning.

This post serves as a short tutorial on how to use Bodywork to productionise a common ML pipeline - train-and-deploy. This tutorial refers to files within a Bodywork project hosted on GitHub - see [bodywork-ml-pipeline-project](https://github.com/bodywork-ml/bodywork-ml-pipeline-project).


### Prerequisites

If you want to execute the example code, then you will need:

* to [install](https://bodywork.readthedocs.io/en/latest/installation/) the Bodywork Python package on your local machine.
* access to a Kubernetes cluster - either a single-node on your local machine using [Minikube](https://minikube.sigs.k8s.io/docs/), or as a managed service from a cloud provider, such as [EKS](https://aws.amazon.com/eks) from AWS or [AKS](https://azure.microsoft.com/en-us/services/kubernetes-service/) from Azure.
* [Git](https://git-scm.com) and a basic understanding of how to use it.

Familiarity with basic [Kubernetes concepts](https://kubernetes.io/docs/concepts/) and some exposure to the [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) command-line tool will make life easier, but is not essential. If you're interested, then the introductory article I wrote on [*Deploying Python ML Models with Flask, Docker and Kubernetes*]({filename}k8s-ml-ops.md), is a good place to start.

## The ML Task

The ML problem we have chosen to use for this example, is the classification of iris plants into one of their three sub-species, given their physical dimensions. It uses the [iris plants dataset](https://scikit-learn.org/stable/datasets/index.html#iris-dataset) and is an example of a multi-class classification task.

The Jupyter notebook titled [ml_prototype_work.ipynb](https://github.com/bodywork-ml/bodywork-ml-pipeline-project/blob/master/ml_prototype_work.ipynb) and found in the root of the [project's GitHub repository](https://github.com/bodywork-ml/bodywork-ml-pipeline-project), documents the trivial ML workflow used to arrive at a proposed solution to this task. It trains a Decision Tree classifier and persists the trained model to cloud storage. Take five minutes to read through it.

## The MLOps Task

Now that we have developed a solution to our chosen ML task, how do we get it into production - i.e. how can we split the Jupyter notebook into a 'train-model' stage that persists a trained model to cloud storage, and a separate 'deploy-scoring-service' stage that will load the persisted model and start a web service to expose a model-scoring API?

The Bodywork project for this multi-stage workflow is packaged as a [GitHub repository](https://github.com/bodywork-ml/bodywork-ml-pipeline-project), and is structured as follows,

```s
root/
 |-- stage_1_train_model/
 |--   |-- train_model.py
 |-- stage_2_serve_model/
 |--   |-- serve_model.py
 |-- bodywork.yaml
```

All of the configuration for this deployment is held within the `bodywork.yaml` file, whose contents are reproduced below.

```yaml
version: "1.0"
project:
  name: bodywork-ml-pipeline-project
  docker_image: bodyworkml/bodywork-core:latest
  DAG: stage_1_train_model >> stage_2_scoring_service
stages:
  stage_1_train_model:
    executable_module_path: stage_1_train_model/train_model.py
    requirements:
      - boto3==1.16.15
      - joblib==0.17.0
      - pandas==1.1.4
      - scikit-learn==0.23.2
    cpu_request: 0.5
    memory_request_mb: 100
    batch:
      max_completion_time_seconds: 30
      retries: 2
  stage_2_scoring_service:
    executable_module_path: stage_2_scoring_service/serve_model.py
    requirements:
      - Flask==1.1.2
      - joblib==0.17.0
      - numpy==1.19.4
      - scikit-learn==0.23.2
    cpu_request: 0.25
    memory_request_mb: 100
    service:
      max_startup_time_seconds: 30
      replicas: 2
      port: 5000
      ingress: true
logging:
  log_level: INFO
```

The remainder of this tutorial is concerned with explaining how the configuration within `bodywork.yaml` is used to deploy the pipeline, as defined within the `train_model.py` and `serve_model.py` Python modules.

## Configuring the Batch Stage

The `stages.stage_1_train_model.executable_module_path` points to the executable Python module - `train_model.py` - that defines what will happen when the `stage_1_train_model` (batch) stage is executed, within a pre-built [Bodywork container](https://hub.docker.com/repository/docker/bodyworkml/bodywork-core). This module contains the code required to:

1. download data from an AWS S3 bucket;
2. pre-process the data (e.g. extract labels for supervised learning);
3. train the model and compute performance metrics; and,
4. persist the model to the same AWS S3 bucket that contains the original data.

It can be summarised as,

```python
from datetime import datetime
from urllib.request import urlopen

# other imports
# ...

DATA_URL = ('http://bodywork-ml-pipeline-project.s3.eu-west-2.amazonaws.com'
            '/data/iris_classification_data.csv')

# other constants
# ...


def main() -> None:
    """Main script to be executed."""
    data = download_dataset(DATA_URL)
    features, labels = pre_process_data(data)
    trained_model = train_model(features, labels)
    persist_model(trained_model)


# other functions definitions used in main()
# ...


if __name__ == '__main__':
    main()
```

We recommend that you spend five minutes familiarising yourself with the full contents of [train_model.py](https://github.com/bodywork-ml/bodywork-ml-pipeline-project/blob/master/stage_1_train_model/train_model.py). When Bodywork runs the stage, it will do so in exactly the same way as if you were to run,

```s
$ python train_model.py
```

And so everything defined in `main()` will be executed.

The `stages.stage_1_train_model.requirements` parameter in the `bodywork.yaml` file lists the 3rd party Python packages that will be Pip-installed on the pre-built Bodywork container, as required to run the `train_model.py` module. In this example we have,

```s
boto3==1.16.15
joblib==0.17.0
pandas==1.1.4
scikit-learn==0.23.2
```

* `boto3` - for interacting with AWS;
* `joblib` - for persisting models;
* `pandas` - for manipulating the raw data; and,
* `scikit-learn` - for training the model.

Finally, the remaining parameters in `stages.stage_1_train_model` section of the `bodywork.yaml` file allow us to configure the remaining key parameters for the stage,

```yaml
stage_1_train_model:
  executable_module_path: stage_1_train_model/train_model.py
  requirements:
    - boto3==1.16.15
    - joblib==0.17.0
    - pandas==1.1.4
    - scikit-learn==0.23.2
  cpu_request: 0.5
  memory_request_mb: 100
  batch:
    max_completion_time_seconds: 30
    retries: 2
```

From which it is clear to see that we have specified that this stage is a batch stage (as opposed to a service-deployment), together with an estimate of the CPU and memory resources to request from the Kubernetes cluster, how long to wait and how many times to retry, etc.

## Configuring the Service Stage

The `stages.stage_2_scoring_service.executable_module_path` parameter points to the executable Python module - `serve_model.py` - that defines what will happen when the `stage_2_scoring_service` (service) stage is executed, within a pre-built Bodywork container. This module contains the code required to:

1. load the model trained in `stage_1_train_model` and persisted to cloud storage; and,
2. start a Flask service to score instances (or rows) of data, sent as JSON to a REST API.

We have chosen to use the [Flask](https://flask.palletsprojects.com/en/1.1.x/) framework with which to engineer our REST API server. The use of Flask is **not** a requirement and you are free to use different frameworks - e.g. [FastAPI](https://fastapi.tiangolo.com).

The contents of `serve_model.py` defines the REST API server and can be summarised as,

```python
from urllib.request import urlopen
from typing import Dict

# other imports
# ...

MODEL_URL = ('http://bodywork-ml-pipeline-project.s3.eu-west-2.amazonaws.com/models'
             '/iris_tree_classifier.joblib')

# other constants
# ...

app = Flask(__name__)


@app.route('/iris/v1/score', methods=['POST'])
def score() -> Response:
    """Iris species classification API endpoint"""
    request_data = request.json
    X = make_features_from_request_data(request_data)
    model_output = model_predictions(X)
    response_data = jsonify({**model_output, 'model_info': str(model)})
    return make_response(response_data)


# other functions definitions used in score() and below
# ...


if __name__ == '__main__':
    model = get_model(MODEL_URL)
    print(f'loaded model={model}')
    print(f'starting API server')
    app.run(host='0.0.0.0', port=5000)
```

We recommend that you spend five minutes familiarising yourself with the full contents of [serve_model.py](https://github.com/bodywork-ml/bodywork-ml-pipeline-project/blob/master/stage_2_scoring_service/serve_model.py). When Bodywork runs the stage, it will start the server defined by `app` and expose the `/iris/v1/score` route that is being handled by `score()`. Note, that this process has no scheduled end and the stage will be kept up-and-running until it is re-deployed or [deleted](user_guide.md#deleting-redundant-service-deployments).

The `stages.stage_2_scoring_service.requirements` parameter in the `bodywork.yaml` file lists the 3rd party Python packages that will be Pip-installed on the pre-built Bodywork container, as required to run the `serve_model.py` module. In this example we have,

```s
Flask==1.1.2
joblib==0.17.0
numpy==1.19.4
scikit-learn==0.23.2
```

* `Flask` - the framework upon which the REST API server is built;
* `joblib` - for loading the persisted model;
* `numpy` & `scikit-learn` - for working with the ML model.

Finally, the remaining parameters in `stages.stage_2_scoring_service` section of the `bodywork.yaml` file allow us to configure the remaining key parameters for the stage,

```yaml
stage_2_scoring_service:
  executable_module_path: stage_2_scoring_service/serve_model.py
  requirements:
    - Flask==1.1.2
    - joblib==0.17.0
    - numpy==1.19.4
    - scikit-learn==0.23.2
  cpu_request: 0.25
  memory_request_mb: 100
  service:
    max_startup_time_seconds: 30
    replicas: 2
    port: 5000
    ingress: true
```

From which it is clear to see that we have specified that this stage is a service (deployment) stage (as opposed to a batch stage), together with an estimate of the CPU and memory resources to request from the Kubernetes cluster, how long to wait for the service to start-up and be 'ready', which port to expose, to create a path to the service from an externally-facing ingress controller (if present in the cluster), and how many instances (or replicas) of the server should be created to stand-behind the cluster-service.

## Configuring the Workflow

The `project` section of the `bodywork.yaml` file contains the configuration for the whole workflow - a workflow being a collection of stages, run in a specific order, that can be represented by a Directed Acyclic Graph (or DAG).

```yaml
project:
  name: bodywork-ml-pipeline-project
  docker_image: bodyworkml/bodywork-core:latest
  DAG: stage_1_train_model >> stage_2_scoring_service
```

The most important element is the specification of the workflow DAG, which in this instance is simple and will instruct the Bodywork workflow-controller to train the model and then (if successful) deploy the scoring service.

## Testing the Workflow

Firstly, make sure that the [bodywork](https://pypi.org/project/bodywork/) package has been Pip-installed into a local Python environment that is active. Then, make sure that there is a namespace setup for use by bodywork projects - e.g. `ml-pipeline` - by running the following at the command line,

```s
$ bodywork setup-namespace ml-pipeline
```

Which should result in the following output,

```s
creating namespace=ml-pipeline
creating service-account=bodywork-workflow-controller in namespace=ml-pipeline
creating cluster-role-binding=bodywork-workflow-controller--ml-pipeline
creating service-account=bodywork-jobs-and-deployments in namespace=ml-pipeline
```

Then, the workflow can be tested by running the workflow-controller locally (to orchestrate remote containers on k8s), using,

```s
$ bodywork workflow \
    --namespace=ml-pipeline \
    https://github.com/bodywork-ml/bodywork-ml-pipeline-project \
    master
```

Which will run the workflow defined in the `master` branch of the project's remote GitHub repository, all within the `ml-pipeline` namespace. The logs from the workflow-controller and the containers nested within each constituent stage, will be streamed to the command-line to inform you on the precise state of the workflow, but you can also keep track of the current state of all Kubernetes resources created by the workflow-controller in the `ml-pipeline` namespace, by using the kubectl CLI tool - e.g.,

```s
$ kubectl -n ml-pipeline get all
```

Once the workflow has completed, the scoring service deployed within your cluster will be ready for testing. Service deployments are accessible via HTTP from within the cluster - they are not exposed to the public internet, unless you have [installed an ingress controller](kubernetes.md#configuring-ingress) in your cluster. The simplest way to test a service from your local machine, is by using a local proxy server to enable access to your cluster. This can be achieved by issuing the following command,

```s
$ kubectl proxy
```

Then in a new shell, you can use the curl tool to test the service. For example,

```s
$ curl http://localhost:8001/api/v1/namespaces/ml-pipeline/services/      
  bodywork-ml-pipeline-project--stage-2-scoring-service/proxy/iris/v1/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

If successful, you should get the following response,

```json
{
  "species_prediction":"setosa",
  "probabilities":"setosa=1.0|versicolor=0.0|virginica=0.0",
  "model_info": "DecisionTreeClassifier(class_weight='balanced', random_state=42)"
}
```

If an ingress controller is operational in your cluster, then the service can be tested via the public internet using,

```s
$ curl http://YOUR_CLUSTERS_EXTERNAL_IP/ml-pipeline/ 
  bodywork-ml-pipeline-project--stage-2-scoring-service/iris/v1/score \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

See [here](kubernetes.md#connecting-to-the-cluster) for instruction on how to retrieve `YOUR_CLUSTERS_EXTERNAL_IP`.

## Scheduling the Workflow

If you're happy with the test results, then you can schedule the workflow-controller to operate remotely on the cluster as a Kubernetes cronjob. To setup the the workflow to run every hour, for example, use the following command,

```s
$ bodywork cronjob create \
    --namespace=ml-pipeline \
    --name=ml-pipeline \
    --schedule="* 0 * * *" \
    --git-repo-url=https://github.com/bodywork-ml/bodywork-ml-pipeline-project \
    --git-repo-branch=master \
    --retries=2
```

Each scheduled workflow will attempt to re-run the workflow, end-to-end, as defined by the state of this repository's `master` branch at the time of execution - performing rolling-updates to service-deployments and automatic roll-backs in the event of failure.

To get the execution history for all `ml-pipeline` jobs use,

```s
$ bodywork cronjob history \
    --namespace=ml-pipeline \
    --name=ml-pipeline
```

Which should return output along the lines of,

```s
JOB_NAME                                START_TIME                    COMPLETION_TIME               ACTIVE      SUCCEEDED       FAILED
ml-pipeline-1605214260          2020-11-12 20:51:04+00:00     2020-11-12 20:52:34+00:00               1              0           0
 ```

Then to stream the logs from any given cronjob run (e.g. to debug and/or monitor for errors), use,

```s
$ bodywork cronjob logs \
    --namespace=ml-pipeline \
    --name=ml-pipeline-1605214260
```

## Cleaning Up

To clean-up the deployment in its entirety, delete the namespace using kubectl - e.g. by running,

```s
$ kubectl delete ns ml-pipeline
```

Read the official Bodywork [documentation](https://bodywork.readthedocs.io/en/latest/) or ask a question on the Bodywork [discussion board](https://github.com/bodywork-ml/bodywork-core/discussions).


