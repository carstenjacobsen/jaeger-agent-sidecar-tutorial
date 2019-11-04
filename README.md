# Deploy Jaeger Agent as a Sidecar

## Introduction
In this tutorial, the objective is to setup Jaeger Tracing in Kubernetes, and to build a small Python application, which will implement a simple external API and perform basic tracing.  

### Application
The application is very simple, and the focus is to walk through how to setup tracing. The app is going to have one API endpoint, which is /numbers. When a GET request is sent to the endpoint, a random number is picked, and trivia about that number is returned. The application uses an external API service to provide the trivia.

Jaeger Tracing is used to monitor the duration of the request, and break it down, to see where the time is spent. The goal is to see spans in the Jaeger UI similar to this:

[img ]



### Jaeger Tracing
In distributed tracing, the tracing system doesn’t necessarily sit in the same node as the service being traced. A common practise is to deploy an agent, as a separate container, in the same pod as the service being traced. The method of adding another container to a pod, with the purpose of running an infrastructure service, is referred to as adding a sidecar.

This tutorial shows how to add the Jaeger Agent as a sidecar in Kubernetes, and have it send spans to the Jaeger Collector, deployed to another pod.



## Deploy Jaeger
The first step is to deploy Jaeger, and the recommended way to do this is to install the Jaeger Operator on Kubernetes. This step is also described in the documentation.

First, create a namespace:

```bash
$ kubectl create namespace observability
```

The namespace is used in the deployment files used in this tutorial. The Jaeger operator can be installed using a different namespace, if the deployment files are updated with the namespace.

Next the operator is installed:

```bash
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing_v1_jaeger_crd.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
```

After the Jaeger operator has been deployed, a Jaeger instance can be created. The status of the deployment can be checked with this command:

```bash
$ kubectl get deployment jaeger-operator -n observability

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
jaeger-operator   1         1         1            1           52s
```

When the operator is available, create the instance. First create a file with the configuration, let’s call it jaegerdemo.yaml, and add the following to the file:

apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaegerdemo

Now apply it to create the instance:

$ kubectl apply -f jaegerdemo.yaml

At this point the Jaeger agent, collector and query services are running. Verify the services are running with the get services command:

$ kubectl get services

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
jaegerdemo-agent                ClusterIP   None             <none>        5775/UDP,5778/TCP,6831/UDP,6832/UDP                                                                                                          16m10s
jaegerdemo-collector            ClusterIP   10.111.112.10    <none>        9411/TCP,14250/TCP,14267/TCP,14268/TCP                                                                                                       16m10s
jaegerdemo-collector-headless   ClusterIP   None             <none>        9411/TCP,14250/TCP,14267/TCP,14268/TCP                                                                                                       16d10s
jaegerdemo-query                ClusterIP   10.111.105.202   <none>        16686/TCP                                                                                                                                    16d10s

The agent service is also active, but it will not be used in this tutorial. Instead the agent will be run as a sidecar next to the Python application.

Jaeger is running in a pod created by the operator, the status of the pod can be verified with this command:

$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
jaegerdemo-66b675b846-qhzw4   1/1     Running   0          81s

As a final step in setting up Jaeger, the query service is exposed, so the UI can be accessed from a browser.

$ kubectl expose deployment jaegerdemo --type=NodePort --port 16686 --name=jaeger-expose

Now run this command to see the exposed port numbers:

$ kubectl get services jaeger-expose

NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                AGE
jaeger-expose   NodePort   172.21.232.234   <none>        16686:30902/TCP   87m

The last part of setting up Jaeger, is to get the IP address where the UI service are accessible. The IP address configured for the node can be obtained with this command:

$ kubectl describe nodes | grep External
  ExternalIP:  xxx.xxx.xxx.xxx

Now the Jaeger UI should be able to load from a browser, in this case, navigate to http://xxx.xxx.xxx.xxx:30902

2. Create Python Application
There are two steps to creating the Python application for this tutorial. First the Python application itself is created, and then it’s containerized so it easily can be deployed in the Kubernetes environment, with Jaeger Agent as a sidecar.

Create Python application
The application is very simple in this tutorial, and the focus is of course on how to setup tracing. The app is going to have one API endpoint, which is /numbers. When a GET request is sent to the endpoint, a random number is picked, and trivia about that number is returned. The application uses an external API service to provide the jokes.

Jaeger Tracing is used to monitor the duration of the request, and break it down, to see where the time is spent.
Application Endpoint
First the endpoint is created by using Flask. The class Flask is imported from the flask-package, and then used to create an application object as an instance of the class.

The endpoint, in this case /jokes, is defined with the @app.route decorator. The decorator takes a function as a parameter, and the functionality of the application will reside in this function.

Finally app.run() is called, with the host IP and port as parameters.

from flask import Flask

app = Flask(__name__)

@app.route('/numbers)
def request_numberinfo():
  # Do something here

if __name__ == "__main__":
  app.run(host="0.0.0.0", port=int("5000"))

Adding functionality
As previously described, the functionality of this simple application is to pick a random number, and return trivia about that number. For this purpose the free Numbers API service used. This API can return trivia based on a number:

$ curl http://numbersapi.com/42

42 is the number of kilometers in a marathon.

The Python HTTP library requests is used to execute the API request, and the implementation is done with a few lines of code:

import requests

# Generate a random number, and construct the API URL
id = random.randint(0, 100)
api_url= "http://numbersapi.com/" + str(id)

# Send HTTP GET request to API URL
r = requests.get(api_url)
# Get the content of the response
response = r.content

return response

Setup Jaeger Client
Jaeger provides a Python client, which makes it quick and easy to configure and implement tracing in Python applications. Information about the client is available on the Jaeger Tracing website and the client’s repository on GitHub.
Configure the client
The sampling type and rate are defined in the configuration, and the service name is defined in the client configuration as well. The Jaeger agent is running in the same pod as the application, so it’s not necessary to configure the agent’s IP and port, they are configured automatically.

The client supports different sampling types (see documentation), and in this tutorial’s application, every trace is sampled. The sampling type constant is used, and this type accepts the parameter 1 and 0 (sample all or no traces).

The code for configuring the client, and initializing the Jaeger tracer instance, looks like this:

from jaeger_client import Config

def init_tracer():
  config = Config(
    config={
      'sampler': {
        'type': 'const',
        'param': 1,
      }
    },
    service_name='pythonapp'
  )
  return config.initialize_tracer()

tracer = init_tracer()

# trace something

tracer.close()

Define spans
As a final step in creating the Python application, the tracing spans are created by using the tracer instance’s start_span() function. As illustrated in the beginning of the tutorial, the objective is to trace the external HTTP request, returning the response and the overall time spent executing the service.


To achieve this, a parent span is created, and to child spans are created. The parent span measures the total execution time of the service, and the child spans measure time spent on fetching data from an external API, and manipulating the response, which then is returned.

# start parent span
with tracer.start_span('HTTP GET: /numbers') as parent_span:
  id = random.randint(0, 100)
  api_url= "http://numbersapi.com/" + str(id)

  # start child span for fetching external data
  with tracer.start_span('API Request', child_of=parent_span) as child_span_1:  
    r = requests.get(api_url)

  # start child span for manipulating and returning data
  with tracer.start_span('Return Value', child_of=parent_span) as child_span_2:
    response = r.content
    return response

Result
The application is now complete, and here everything is put together:

pythonapp.py
from flask import Flask
import requests
from jaeger_client import Config
import time
import random

app = Flask(__name__)

def init_tracer():
  config = Config(
    config={
      'sampler': {
        'type': 'const',
        'param': 1,
      }
    },
    service_name='pythonapp'
  )
  return config.initialize_tracer()

tracer = init_tracer()

@app.route('/numbers')
def request_number():

  # start parent span
  with tracer.start_span('HTTP GET: /numbers') as parent_span:
    # Generate a random number, and construct the API URL
    id = random.randint(0, 100)
    api_url= "http://numbersapi.com/" + str(id)
    time.sleep(0.05)

    # start child span for fetching external data
    with tracer.start_span('API Request', child_of=parent_span) as child_span_1:
      # Send HTTP GET request to API URL
      r = requests.get(api_url)

    # start child span for manipulating and returning data
    with tracer.start_span('Return Value', child_of=parent_span) as child_span_2:
      # Manipulate response and return
      response = r.content
      time.sleep(0.05)
      return response

  tracer.close()


if __name__ == "__main__":
  app.run(host="0.0.0.0", port=int("5000"))

3. Containerize with Docker
The Python application is containerized with Docker, to make it easy to deploy in Kubernetes. Docker needs two files, besides the application file, and that’s a requirements.txt file and a Dockerfile.
requirements.txt
This file tells Docker which libraries to include, besides the standard libraries. The Python application depends on three libraries, including the Jaeger client, so these are listed in the text file:

requirements.txt
flask
requests
jaeger_client

Dockerfile
The Dockerfile (no extension) contains instructions to build the Docker image. The commands can be run manually, but with the Dockerfile, the image can be built with just one command, the docker build command.

The commands needed to build the Python application image are the following:

Dockerfile
FROM python:3.8.0-slim-buster
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 5000
CMD python ./numberapp.py

The image will be based off a Python image, using the slim version of Debian Buster. This is a very compact image (60MB), yet it has everything needed to run this application.

The dependencies listed in the requirements.txt file will be installed, and port 5000 will be exposed, so the application’s endpoint can be reached outside the container. The last line of the Dockerfile tells the container what to execute at startup.
Build and Push image
After creating the Python application file, the requirements.txt file and the Dockerfile, the image can be built and pushed to Docker Hub.

To build the image, use this command:

$ docker build -t <your-username>/pythonapp:latest .

After building the image, push it the Docker Hub:


$ docker push <your-username>/pythonapp:latest

The image can now be used to deploy the application on Kubernetes.

Test the image
The image can be tested with the docker run command:

$ docker run -p5000:5000 <your-username>/pythonapp:latest

Navigating to http://localhost:5000/numbers in a browser, or running the command curl http://localhost:5000/numbers in a terminal, will output trivia about a random number.

$ curl http://localhost:5000/numbers

9 is the number of innings in a regulation, non-tied game of baseball.

4. Deploy Application w/Sidecar
The Python application is already prepared to send traces to a Jaeger agent, since Jaeger’s Python client automatically will find the agent, if it’s present. What’s needed in this step, is to deploy both the application and the agent (as a sidecar) in the same pod.

The Kubernetes Deployment YAML file is defining the containers and configurations, and is all needed to deploy the application and agent sidecar in the same pod.
General information
First define the version, kind and metadata.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: pythonapp-deployment
  namespace: default

The namespace default was created while setting up Jaeger Tracing.
Deployment Spec
The spec object is used to define which containers to deploy and their configurations. The label selector groups and identifies the objects in this deployment.   


spec:
  selector:
    matchLabels:
      app: pythonapp
  template:
    metadata:
      labels:
        app: pythonapp
    spec:
      containers:
        # add containers

Containers
Two containers will be deployed with this YAML file, the Python application container and the Jaeger Agent container. Each container is specified as a template in the spec object.
Python application
The image, which was pushed to Docker Hub in the previous step of the tutorial, is pulled during deployment. The only configuration necessary, is to expose port 5000, which is done by defining a container port.

- name: pythonapp
  image: <your-username>/pythonapp:latest
  ports:
  - containerPort: 5000
Jaeger Agent
A Jaeger Agent image is also available from the Docker Hub, so this container can be deployed similar to the Python application. In addition to exposing the ports, the Jaeger Collector host IP and port number is passed on to the agent container as an argument. The agent needs to know where the collector is located, since it’s running in a different pod.

The collector’s IP address can be retrieved by describing the collector service with kubectl:

$ kubectl describe services jaegerdemo-collector | grep IP

Type:                  ClusterIP
IP:                    10.111.112.10

The Jaeger Collector can receive spans using the TChannel protocol on port 14267, so that port is used.

- name: jaeger-agent
  image: jaegertracing/jaeger-agent:latest
  ports:
    - containerPort: 5775
      protocol: UDP
    - containerPort: 5778
      protocol: TCP
    - containerPort: 6831
      protocol: UDP
    - containerPort: 6832
      protocol: UDP
  args: ["--collector.host-port=10.111.112.10:14267"]
The complete YAML file
Now all parts of the YAML deployment file are done, and the file looks like this:

python-app-jaeger-agent.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pythonapp
  namespace: default
spec:
  selector:
    matchLabels:
      app: pythonapp
  template:
    metadata:
      labels:
        app: pythonapp
    spec:
      containers:
      - name: pythonapp
        image: <your-username>/pythonapp:latest
        ports:
        - containerPort: 5000
      - name: jaeger-agent
        image: jaegertracing/jaeger-agent:latest
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        - containerPort: 6831
          protocol: UDP
        - containerPort: 6832
          protocol: UDP
        args: ["--collector.host-port=10.111.112.10:14267"]

Deploy
The Python application, and Jaeger Agent sidecar, can now be deployed on the same Kubernetes cluster as Jaeger Tracing. This is done using kubectl create:

$ kubectl create -f python-app-jaeger-agent.yml

Verify the creation of the pod, it will be called pythonapp-xxxxxxxxxx-xxxxx:

$ kubectl get pods

NAME                         READY     STATUS    RESTARTS   AGE
pythonapp-6495499cf4-tzhz5   2/2       Running   0          1m
jaegerdemo-695765785-6pd8m   1/1       Running   0          55m

Expose application
The final step is to expose the Python application, so the endpoint can be accessed outside the cluster. The Python application will be exposed, but since the agent only receives spans from the application inside the pod, there’s no need to expose the agent’s ports.

$ kubectl expose deployment/pythonapp --type="NodePort" --name=pythonapp-expose --port 5000

Get the public port number with kubectl get services:

$ kubectl get services | grep pythonapp

NAME        TYPE        CLUSTER-IP       EXTERNAL-IP    PORT(S)            AGE
pythonapp   NodePort    10.97.159.86     <none>         5000:32519/TCP     28m

5. Test
The application and Jaeger Tracing is now fully deployed, and ready to test. The test is very basic, navigate to the endpoint in a browser, and then see the trace in Jaeger’s UI in a browser.

First, navigate to endpoint in the browser. The IP address is the same as retrieved while setting up Jaeger. The port number is the exposed port number, and not the application’s internal port number. The actual IP and port number will be different from environment to environment, for this application it will be http://xxx.xxx.xxx.xxx:32519/numbers




Now go to the Jaeger Tracing UI, to see the trace of the request. Navigate to the same IP address as used in the endpoint, but use the exposed Jaeger UI port. In this

http://xxx.xxx.xxx.xxx:30902

In the Search menu  on the left side of the screen, choose pythonapp as the service, and click the Find Traces button. The default lookback is the last hour, and the Jaeger UI will now display traces captured the past hour.



To see the spans, click one of the results.


The spans shows the overall HTTP GET request span, and the span of the external API call, and the span which returns the response from the API call.

This concludes this minimal test. It shows the application is sending the spans to the sidecar agent, the agent is sending the spans to the collector, which resides in a different Kubernetes pod, and the result is viewable in the Jaeger UI.

Conclusion
sdf
