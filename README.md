# helm-ghost
What is Helm Chart ?
Helm is the ”The Kubernetes package manager”. It is a command-line tool that enables you to create and use so-called Helm Charts.
A Helm Chart is a collection of templates and settings that describe a set of Kubernetes resources. Its power spans from managing a single node definition to a highly scalable multi-node cluster.
The architecture of Helm has changed over the last years. The current version of Helm communicates directly to your Kubernetes cluster via Rest. Tiller was removed in Helm 3.
Helm itself is stateful. When a Helm Chart gets installed, the defined resources are getting deployed and meta-information is stored in Kubernetes secrets.
How to Start a Ghost Blog Using Docker
Ghost is a free and open source blogging platform written in JavaScript , designed to simplify the process of online publishing for individual bloggers as well as online publications.
Part I: Launch ghost by using only docker
docker run --rm -p 2368:2368 --name my-ghost ghost

The blog is available at: http://localhost:2368.

Let’s stop the instance to be able to launch another one using Kubernetes:
$ docker rm -f my-ghost
  my-ghost
Part II: Now, we want to deploy the ghost blog with 2 instances in our Kubernetes cluster.
Create the new directory called applicaion
$ mkdir application
$ cd application
Let’s set up a plain deployment first:
# file 'application/deployment.yaml'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost-app
spec:
  selector:
    matchLabels:
      app: ghost-app
  replicas: 2
  template:
    metadata:
      labels:
        app: ghost-app
    spec:
      containers:
        - name: ghost-app
          image: ghost

          ports:
            - containerPort: 2368
and put a load balancer before it to be able to access our container and to distribute the traffic to both containers:
# file 'application/service.yaml'

apiVersion: v1
kind: Service
metadata:
  name: my-service-for-ghost-app
spec:
  type: LoadBalancer
  selector:
    app: ghost-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 2368
We can now deploy both resources using kubectl:
$ kubectl apply -f deployment.yaml -f service.yaml

The ghost application is now available via http://localhost:32344.

Let’s again stop the application:
$ kubectl delete -f deployment.yaml -f service.yaml

So far it works good, it works with plain Kubernetes. But what if we need different settings for different environments?
Imagine that we want to deploy it to multiple data centers in different stages (non-prod, prod). You will end up duplicating your Kubernetes files over and over again. It will be hell in terms of maintenance. Instead of scripting a lot, we can leverage Helm.
Part III: Let’s create a new Helm Chart from scratch:
$ helm create my-ghost-app
Helm created a bunch of files for you that are usually important for a production-ready service in Kubernetes. To concentrate on the most important parts, we can remove a lot of the created files. Let’s go through the only required files for this example.
We need a project file that is called Chart.yaml:
$ Chart.yaml
apiVersion: v2
name: my-ghost-app
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: 1.16.0
The deployment template file:
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost-app
spec:
  selector:
    matchLabels:
      app: ghost-app
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      labels:
        app: ghost-app
    spec:
      containers:
        - name: ghost-app
          image: ghost
          ports:
            - containerPort: 2368
          env:
            - name: url
              {{- if .Values.prodUrlSchema }}
              value: http://{{ .Values.baseUrl }}
              {{- else }}
              value: http://{{ .Values.datacenter }}.non-prod.{{ .Values.baseUrl }}
              {{- end }}
It looks very similar to our plain Kubernetes file. Here, you can see different placeholders for the replica count, and an if-else condition for the environment variable called url. In the following files, we will see all the values defined.
The service template file:
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-for-my-webapp
spec:
  type: LoadBalancer
  selector:
    app: ghost-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 2368
Our Service configuration is completely static.
The values for the templates are the last missing parts of our Helm Chart. Most importantly, there is a default values file required called values.yaml:
# values.yaml
replicaCount: 1
prodUrlSchema: false
datacenter: us-east
baseUrl: myapp.org
A Helm Chart should be able to run just by using the default values. Before you proceed, make sure that you have deleted:
my-ghost-app/templates/tests/test-connection.yaml
my-ghost-app/templates/serviceaccount.yaml
my-ghost-app/templates/ingress.yaml
my-ghost-app/templates/hpa.yaml
my-ghost-app/templates/NOTES.txt.

We can get the final output that would be sent to Kubernetes by executing a “dry-run”:
$ helm template --debug my-ghost-app
install.go:159: [debug] Original chart version: ""
install.go:176: [debug] CHART PATH: /helm/my-ghost-app
---
# Source: my-ghost-app/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-for-my-webapp
spec:
  type: LoadBalancer
  selector:
    app: my-example-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 2368
---
# Source: my-ghost-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost-app
spec:
  selector:
    matchLabels:
      app: ghost-app
  replicas: 1
  template:
    metadata:
      labels:
        app: ghost-app
    spec:
      containers:
        - name: ghost-app
          image: ghost
          ports:
            - containerPort: 2368
          env:
            - name: url
              value: us-east.non-prod.myapp.org
Helm inserted all the values and also set url to us-east.non-prod.myapp.org because in the values.yaml, prodUrlSchema is set to false and the datacenter is set to us-east.
To get some more flexibility, we can define some override value files. Let’s define one for each datacenter:
# values.us-east.yaml
datacenter: us-east
# values.us-west.yaml
datacenter: us-west
and one for each stage:
# values.nonprod.yaml
replicaCount: 1
prodUrlSchema: false
# values.prod.yaml
replicaCount: 3
prodUrlSchema: true
We can now use Helm to combine them as we want and check the result again:
$ helm template --debug my-ghost-app -f my-ghost-app/values.nonprod.yaml  -f my-ghost-app/values.us-east.yaml 
install.go:159: [debug] Original chart version: ""
install.go:176: [debug] CHART PATH: /helm/my-ghost-app
---
# Source: my-ghost-app/templates/service.yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-for-my-webapp
spec:
  type: LoadBalancer
  selector:
    app: my-example-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 2368
---
# Source: my-ghost-app/templates/deployment.yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost-app
spec:
  selector:
    matchLabels:
      app: ghost-app
  replicas: 1
  template:
    metadata:
      labels:
        app: ghost-app
    spec:
      containers:
        - name: ghost-app
          image: ghost
          ports:
            - containerPort: 2368
          env:
            - name: url
              value: http://us-east.non-prod.myapp.org
And for sure, we can do a final deployment:
$ helm install -f my-ghost-app/values.prod.yaml my-ghost-prod ./my-ghost-app/
NAME: my-ghost-prod
LAST DEPLOYED: Mon Dec 21 00:09:17 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
As before, our ghost blog is available via http://localhost.

We can delete this deployment and deploy the application with us-east and non prod settings like this:
$ helm delete my-ghost-prod                                                 
  release "my-ghost-prod" uninstalled
$ helm install -f my-ghost-app/values.nonprod.yaml -f my-ghost-app/values.us-east.yaml my-ghost-nonprod ./my-ghost-app
We finally clean up our Kubernetes deployment via Helm:
$ helm delete my-ghost-nonprod
Conclusion
The power of a great templating engine and the possibility of executing releases, upgrades, and rollbacks makes Helm great. On top of that comes the publicly available Helm Chart Hub that contains thousands of production-ready templates. This makes Helm a must-have tool in your toolbox if you work with Kubernetes on a bigger scale!
