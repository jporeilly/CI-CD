## <font color='red'>4.1 Tekton</font>
Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

In this lab we're going to:
* install Tekton
* install Tekton CLI
* install Tekton Dashboard on Kubernetes

* run the application tests inside the cloned git repository
* build a Docker image for our Go application and push it to DockerHub

**The second part requires a Docker Hub account.**

---

#### <font color='red'>IMPORTANT:</font> 
<strong>Please ensure you start with a clean environment. 
If you have previously run minikube, you will need to delete the existing instance.</strong>

to stop  minikube:
```
minikube stop
```
to delete  minikube:
```
minikube delete
```

the next step is to create Kubernetes cluster: 
* install minikube

start minikube:
```
minikube start
```
start tunnel:
```
minikube tunnel
```

---

#### <font color='red'>4.3.1 Install Tekton + CLI</font>


install tekton pipeline:
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```
verify installation:
```
kubectl get pods --namespace tekton-pipelines
```

**Tekton CLI**
install Tekton CLI:
```
curl -LO https://github.com/tektoncd/cli/releases/download/v0.17.2/tkn_0.17.2_Linux_x86_64.tar.gz
sudo tar xvzf tkn_0.17.2_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
```
To run a CI/CD workflow, you need to provide Tekton a Persistent Volume for storage purposes. Tekton requests a volume of 5Gi with the default storage class by default. 

check available persistent volumes and storage classes:
```
kubectl get pv
kubectl get storageclasses
```

**Tekton Dashboard**

install Tekton Dashboard on a Kubernetes cluster:
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml
```
access the Dashboard is using kubectl port-forward:
```
kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
```

  > view dashboard: http://localhost:9097

---

#### <font color='red'>4.3.2 Tekton Piepline</font>
In our first tekton pipeline a Go application simply prints the sum of two integers.
* run the application tests inside the cloned git repository

The required resource files can be found at:

  > Tekton demo repository: http://github.com/jporeilly/Tekton-demo.git

create a file called 01-task-test.yaml with the following content:
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test
spec:
  resources:
    inputs:
      - name: repo
        type: git
  steps:
    - name: run-test
      image: golang:1.14-alpine
      workingDir: /workspace/repo/src
      command: ["go"]
      args: ["test"]`
```
The resources: block defines the inputs that our task needs to execute its steps. Our step (name: run-test) needs the cloned tekton-demo git repository as an input and we can create this input with a PipelineResource.  

The git resource type will use git to clone the repo into the /workspace/$input_name directory everytime the Task is run. Since our input is named repo the code will be cloned to /workspace/repo. If our input would be named foobar it would be cloned into /workspace/foobar.

The next block in our Task (steps:) specifies the command to execute and the Docker image in which to run that command. We're going to use the golang Docker image as it already has Go installed.

For the go test command to run we need to change the directory. By default the command will run in the /workspace/repo directory but in our tekton-demo repo the Go application is in the src directory. We do this by setting workingDir: /workspace/repo/src.

Next we specify the command to run (go test) but note that the command (go) and args (test) need to be defined separately in the YAML file.

create a file called 02-pipelineresource.yaml:
```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: tekton-example
spec:
  type: git
  params:
    - name: url
      value: https://github.com/jporeilly/tekton-demo
    - name: revision
      value: master
```

apply the Task and the PipelineResource with kubectl:
```
kubectl apply -f 01-task-test.yaml
kubectl apply -f 02-pipelineresource.yaml
```
To run our Task we have to create a TaskRun that references the previously created Task and passes in all required inputs (PipelineResource).

create a file called 03-taskrun.yaml with the following content:
```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: testrun
spec:
  taskRef:
    name: test
  resources:
    inputs:
      - name: repo
        resourceRef:
          name: tekton-example
```
This will take our Task (taskRef is a reference to our previously created task name: test) with our tekton-demo git repo as an input (resourceRef is a reference to our PipelineResource name:: tekton-example) and execute it.

Apply the file with kubectl and then check the Pods and TaskRun resources. The Pod will go through the Init:0/2 and PodInitializing status and then succeed:
```
kubectl apply -f 03-taskrun.yaml
```
check Pods:
```
kgpo
```
check taskrun:
```
kg taskrun
```
To see the output of the containers we can run the following command. Make sure to replace testrun-pod-pds5z with the the Pod name from the output above (it will be different for each run).
```
kubectl logs testrun-pod-pds5z --all-containers
```
Our tests passed and our task succeeded. Next we will use the Tekton CLI to see how we can make this whole process easier.

Instead of manually writing a TaskRun manifest we can run the following command which takes our Task (named test), generates a TaskRun (with a random name) and shows its logs:
```
tkn task start test --inputresource repo=tekton-example --showlog
```

---

#### <font color='red'>4.3.3 Tekton Piepline - Docker</font>
In our second tekton pipeline a Go application simply prints the sum of two integers.
* build a Docker image for our Go application and push it to DockerHub

**You will need a Docker Hub account**

To build and push our Docker image we use Kaniko, which can build Docker images inside a Kubernetes cluster without depending on a Docker daemon.

Kaniko will build and push the image in the same command. This means before running our task we need to set up credentials for DockerHub so that the docker image can be pushed to the registry.

The credentials are saved in a Kubernetes Secret. 

create a file named 04-secret.yaml with the following content and replace myusername and mypassword with your DockerHub credentials:
```
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass
  annotations:
    tekton.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
stringData:
    username: [myusername]
    password: [mypassword]
```
Note: the tekton.dev/docker-0 annotation in the metadata which tells Tekton the Docker registry these credentials belong to.

WARNING: This will write the unencypted credentials to /tekton/home.docker

Next we create a ServiceAccount that uses the basic-user-pass Secret. 

create a file named 05-serviceaccount.yaml with the following content:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: basic-user-pass
```
apply both files with kubectl:
```
kubectl apply -f 04-secret.yaml
kubectl apply -f 05-serviceaccount.yaml
```
Now that the credentials are set up we can continue by creating the Task that will build and push the Docker image.

create a file called 06-task-build-push.yaml with the following content:
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-and-push
spec:
  resources:
    inputs:
      - name: repo
        type: git
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v1.3.0
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
      command:
        - /kaniko/executor
        - --dockerfile=Dockerfile
        - --context=/workspace/repo/src
        - --destination=jporeilly/tekton-test:v1
```
Similarly to the first task this task takes a git repo as an input (the input name is repo) and consists of only a single step since Kaniko builds and pushes the image in the same command.

Make sure to create a DockerHub repository and replace jporeilly/tekton-test with your repository name. In this example it will always tag and push the image with the v1 tag.

Tekton has support for parameters to avoid hardcoding values like this. However to keep this tutorial simple I've left them out...  :)

The DOCKER_CONFIG env var is required for Kaniko to be able to find the Docker credentials.

apply the file with kubectl:
```
kubectl apply -f 06-task-build-push.yaml
```
There are two ways we can test this Task, either by manually creating a TaskRun definition and then applying it with kubectl or by using the Tekton CLI (tkn).

To run the Task with kubectl we create a TaskRun that looks identical to the previous with the exception that we now specify a ServiceAccount (serviceAccountName) to use when executing the Task.

create a file named 07-taskrun-build-push.yaml with the following content:
```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-and-push
spec:
  serviceAccountName: build-bot
  taskRef:
    name: build-and-push
  resources:
    inputs:
      - name: repo
        resourceRef:
          name: tekton-example
```
apply the task and check the log of the Pod by listing all Pods that start with the Task name build-and-push:
```
kubectl apply -f 07-taskrun-build-push.yaml
```
check Pods:
```
kubectl get pods | grep build-and-push
```

To see the output of the containers we can run the following command. Make sure to replace build-and-push-pod-c698q with the the Pod name from the output above (it will be different for each run).
```
kubectl logs --all-containers build-and-push-pod-c698q --follow
```
the task executed without problems and we can now pull/run our Docker image:
```
docker run [docker-hub-username]/tekton-test:latest
```
Running the Task with the Tekton CLI is more convenient. With a single command it generates a TaskRun manifest from the Task definition, applies it, and follows the logs.

```
tkn task start build-and-push --inputresource repo=tekton-example --serviceaccount build-bot --showlog
```






clean up:
```
delete-local-cluster.sh
```

---