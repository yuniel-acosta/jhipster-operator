# JHipster Operator Getting Started Guide
On this step by step guide we will see how to install and work with the JHipster Kubernetes Operator

First we will create an application, run it in Kubernetes KIND (local, if you don't have a cluster) and then we will deploy the 
JHipster Kubernetes Operator to teach Kubernetes about JHipster's Apps.

The first that we will do, is to generate an Application form a JHipster JDL DSL, which describes a JHipster Application using the MicroServices approach. This application is composed by:
- A Gateway that also hosts the User Internface
- Invoice Service (which hosts three entities: Shipment, Order, Invoice)
- Review Service (which doesn't have any entity but will be in charge of reviewing the Invoice Service Entities)

You can find the app.jdl file in this repository.

# Generating the JHipster Application
In order to create the source code for this application we need to import our JDL file by calling:
> jhipster import-jdl app.jdl

By running this command we will generate 3 maven projects:
1) gateway
2) invoice
3) review

Some details about these services: 
- Gateway will run by default in port 8080
- Invoice and Review will use 8081

# Running your application in Kubernertes

In order to run your application (microservices) in Kubernetes you need to generate Kubernetes Manifests, which are basically files that describe how Kubernetes will deploy and manage your services. 

In general to work with JHipster into Kubernetes you will follow these steps:

![]


## Creating K8s Manifests

Today, there are three ways in which you can create the K8s manifests: 
1) You create your own manifests
2) You can use jhipster kubernetes generator
3) You can use jhipster kubernetes-helm generator

Let's use the second option for simplicity (but I encourage you to try the HELM approach as well):
> mkdir kubernetes/
> cd kubernetes/
> jhipster kubernetes


Then choose: 
1) Microservice application + Enter
2) root directory ../ + Enter
3) select the three services (gateway, invoice, review) with space and the arrows and then Enter
4) Monitoring -> No
5) Admin password (admin) Enter
6) namespace: I will use "jhipster" + Enter
7) name for the docker repository name (if you are planning to push your images to hub.docker.com) you should use your user name, I will use a local docker registry, so just Enter
8) docker push (default value) + Enter
9) No Istio for now
10) Service Type for your edge services (in this case the gateway), I will choose LoadBalancer + Enter


 After this all the Kubernetes Manifest to run your services are generated. The only missing step is to generate your docker images that are going to be used by these manifest to run your services in the cluster. 

 The output of the generation shows how to create these docker images for each service. Everytime that you make a change in the service, you should generate a new docker image to contain those changes:

```
 To generate the missing Docker image(s), please run:
  ./mvnw -Pprod verify jib:dockerBuild in /gateway
  ./mvnw -Pprod verify jib:dockerBuild in /invoice
  ./mvnw -Pprod verify jib:dockerBuild in /review
```

You will also see in the output:
  ```
  WARNING! You will need to push your image to a registry. If you have not done so, use the following commands to tag and push the images:
  docker push gateway
  docker push invoice
  docker push review
  ```
This last step (of pushing the docker images) is only required if the Cluster is remote and you have no other way to make the images available to the cluster.

Now all you need is a cluster to run these services, welcome Kubernetes KIND. 

## Kubernetes KIND
[KIND](https://github.com/kubernetes-sigs/kind) stands for Kubernetes in Docker and based on their own description:
```
kind is a tool for running local Kubernetes clusters using Docker container "nodes".
kind is primarily designed for testing Kubernetes 1.11+, initially targeting the conformance tests.
```

This is a great tool because it allows us to run a multi node cluster only depending on Docker. We don't need any VM tooling such as Vagrant used in projects like Minikube for example. Also the fact that KIND is targeting conformance tests, means that we have an hermetic environment where we can run our tests against a cluster that looks (from the API and topology prespective) pretty much as our production environments. 

KIND allows us to create and delete clusters in an automated fashion, then we can automate deploying our services into it. This becomes really important when we have services or Operators which needs to interact with the cluster, due we don't want to mock those interactions, we want to test the real cluster behaviour for our components. 

In order to install KIND please follow the instrutions in the Github repo: [KIND](https://github.com/kubernetes-sigs/kind

Once KIND is installed just run:
> kind create cluster

Once the cluster is up, you can try it out with 
> kubectl get pods

This should return 
```
No resources found.
```

## Deploying our application to KIND Cluster
Before moving forward we need to generate the docker images for our services, let's run what was suggested by the "jhipster kubernetes"
```
 To generate the missing Docker image(s), please run:
  ./mvnw -Pprod verify jib:dockerBuild in /gateway
  ./mvnw -Pprod verify jib:dockerBuild in /invoice
  ./mvnw -Pprod verify jib:dockerBuild in /review
```

Once this is done, we now should have 3 docker images loaded in our local Docker Deamon. Now we need to make these images available to KIND.
Which we can do with:

```
> kind load docker-image gateway
> kind load docker-image invoice
> kind load docker-image review
```

Once this is done, we are ready to run
```
> bash kubectl-apply.sh
```
Before running the services, we need to adapt some of the Manifests to make sure that the Docker Images are not downloaded from a remote docker registry. **This is KIND specific.**
We need to change the following files inside the kubernetes (manifest directory, generated by jhipster kubernetes):
- gateway/gateway-deployment.yml
- invoice/invoice-deployment.yml
- review/review-deployment.yml
You need to locate the section:
```
  containers:
    - name: gateway-app
      image: gateway
```
add
```
imagePullPolicy: Never
```

like: 
```
  containers:
    - name: gateway-app
      image: gateway
      imagePullPolicy: Never
```

Also suggested by the generator to deploy all the services into K8s. 
```
salaboy@kubernetes> bash kubectl-apply.sh
namespace/jhipster created
configmap/application-config created
secret/registry-secret created
service/jhipster-registry created
statefulset.apps/jhipster-registry created
deployment.apps/gateway created
deployment.apps/gateway-mysql created
service/gateway-mysql created
service/gateway created
deployment.apps/invoice created
deployment.apps/invoice-mysql created
service/invoice-mysql created
service/invoice created
deployment.apps/review created
deployment.apps/review-mysql created
service/review-mysql created
service/review created

```

Notice that this created in the first line a new namespace called "jhipster" as it was instructred in the generator wizard.
Also notice that we are creating 3 pods which will run MySQL, and KIND will need to fetch from Docker Hub the MySQL docker image the first time that you run these commands.

In order to change the current namespace (so we can get resources without specifying the namespace everytime) we can switch to the jhipster one by running:
```
kubectl config set-context $(kubectl config current-context) --namespace=jhipster
```

Once this is done we can check that our services are running:
```
> k get pods
```

Then to access the service, because we are running 
```
kubectl port-forward svc/gateway 8080:8080 -n jhipster
```

# Operator Time! (JHipster Operator)!

Add Image for K8s Jhipster Operator
![]()

## Building the Operator 

You can skip this section because a docker image is already provided in Docker Hub at **salaboy/jhipster-operator**

If you want to build the JHipster Operator from the Source Code you can clone the project from GitHub: http://github.com/salaboy/jhipster-operator

```
git clone http://github.com/salaboy/jhipster-operator
cd jhipster-operator/
mvn clean install
```

Once the project is built, we can build the Docker Image from it
```
docker build -t jhipster-operator .
```

Once we have the docker image ready we can share that with our KIND cluster

```
kind load docker-image jhipster-operator
```

## Deploying the JHipster K8s Operator
In order to deploy the JHipster Operator you can run:
```
cd kubernetes/
bash deploy.sh 
```

This script just apply all the manifests inside the **deploy/** and **deploy/crds** directory. 

There are three things happening here: 
- Security: create Role, RoleBinding and Service Account. This is required to grant access to our Operator to the Kubernetes APIs from inside a POD. This means that now the Operator has access to read and write Kubernetes Resources.  (Inside the deploy/ directory: cluster-role-binding.yaml, cluster-role.yaml and service-account.yaml)
- Deploy Custom Resource Definitions: these are JHipster Specific types that now Kubernetes understand. I have defined 4 CRDs: Application, Module, Gateway and Registry. These definitions can be located inside the **crds** directory. (all resources inside the deploy/crds/ directory)
- Create the actual deployment that will use the security resources to operate our Custom Resource Definitions and how they relate to Kubernetes Native concepts. (inside the deploy/ directory: deployment.yaml and service.yaml)


Doing now:
```
kubectl get pods
```

Should return our new **jhipster-operator-<hash>** pod, plus all the other pods from our application. 
we can tail the logs from the operator:
```
kubectl logs -f jhipster-operator-<hash>
```

We should be able to see something like this in the logs:
```
tech.jhipster.operator.MyApplication     : + --------------------- RECONCILE LOOP -------------------- + 
tech.jhipster.operator.AppsOperator      : > No Healthy Apps found.
tech.jhipster.operator.MyApplication     : + --------------------- END RECONCILE  -------------------- +
```


## Interacting with the Operator
So now we are in a state where there is a JHipster Application (the one that we created with the app.jdl file) running in our cluster and the Operator running. In order to link the two worlds we can notify the Operator that now it needs to manage this application.
We can do that by sending the Operator the app.jdl file which contains the structure, so the Operator can validate that this application is up and running and make Kubernetes aware of this application specific concepts. 


