# Kubernetes workshop

This workshop guides through Google Kubernetes, Docker and Jenkins CI

Things you might need:
### VI cheat sheet

VI cheat sheet can be found here: http://vim.rtorr.com/

## Activity 0: Setup Kubernetes

Setup and teardown scripts has been placed into ```cluster/kube-up.sh``` and ```cluster/kube-down.sh``` directories correspondingly

### Connecting to Server

This what can help you!

```
chmod 600 ~/.ssh/javaday.pem
ssh -i ~/.ssh/javaday.pem ubuntu@123.123.123.123 -L8080:localhost:8080 -4
```

### Connected?

To start kubernetes just run following command:
```
$ git  clone https://github.com/akranga/kube-workshop.git kube-workshop
$ cd kube-workshop
$ bash cluster/kube-up.sh
$ kubectl config set-cluster http://localhost:8080
```

To check instalation run:
```
$ kubectl get nodes

NAME        LABELS                             STATUS
127.0.0.1   kubernetes.io/hostname=127.0.0.1   Ready
```

Next step is to clone our micro service
```
$ git clone https://github.com/akranga/chucknorris.git
cd chucknorris
```

## Activity 1: Compiling with Docker

ChuckNorris is a Spring boot micro service with one controller. As the first step we will need to compile it

However our environmetn is not ready. We simply (I don't) do not have Java Development Kit of right version. We will use Java docker container to compile our app and produce distribution JAR file

For the first time we will do it manually:

Simply run command:
```
$ docker run --rm -i -t --volume=$(pwd):/app -w /app java:jdk bash
```

This will download Docker container with latest Open JDK (8); mount current directory and let you inside

Now we can run compilation with gradle
```
root@477c94708216:/app$ ./gradlew assemble

:processResources
:classes
:jar
:findMainClass
:startScripts
:distTar
:distZip
:bootRepackage
:assemble

BUILD SUCCESSFUL

Total time: 19.007 secs
```

Now you can exit container and take a look at the compilation result
```
root@477c94708216:/app# exit
$ ls build/libs/
chnorr-0.1.0.jar
```

Let's do the same compilation as oneliner
```
$ docker run --rm -i -t --volume=$(pwd):/app -w /app java:jdk ./gradlew clean assemble

:processResources
:classes
:jar
:findMainClass
:startScripts
:distTar
:distZip
:bootRepackage
:assemble

BUILD SUCCESSFUL

Total time: 17.534 secs
```

How about to compile and run it? This directory has a Dockerfile. Which where we will put our compiled JAR file and create a container image our of it. Once we have an image then we can run it

Run following command:
```
$ docker build -t akranga/chucknorris .

Sending build context to Docker daemon 37.28 MB
Step 0 : FROM java:jdk
 ---> 7547e52aac4b
Step 1 : ENV APP_VERSION 0.1.0
 ---> Running in ac77614ee44a
...
Step 4 : ENTRYPOINT java -jar /app.jar
 ---> Running in 0991ddf114cf
 ---> c5c729af0559
Removing intermediate container 0991ddf114cf
Successfully built c5c729af0559

$ docker run -d -p 8000:8080 --name=chuck akranga/chucknorris
$ curl localhost:8000

Chuck Norris can read all encrypted data, because nothing can hide from Chuck Norris.
```

Now let's tear down our application
```
$ docker ps

CONTAINER ID        IMAGE                                       COMMAND             
865a6151d7ed        akranga/chucknorris                         "java -jar /app.jar"

$ docker stop 865a6151d7ed
$ docker rm 865a6151d7ed
```

## Activity 2: manipulatipns with Kubernetes

Now let's run our Chuck Norris app as part of kubernetes

Before we start, let's do sanity check
```
$ kubectl get nodes
NAME        LABELS                             STATUS
127.0.0.1   kubernetes.io/hostname=127.0.0.1   Ready
```

Now we are ready to start our Chuck:
```
$ kubectl run chuck --image=akranga/chucknorris --port=8080

CONTROLLER   CONTAINER(S)   IMAGE(S)              SELECTOR    REPLICAS
chuck        chuck          akranga/chucknorris   run=chuck   1

$ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
chuck-bbbj5            1/1       Running   0          3m
```

Next step is to expose our Chuck norris to the world
```
$ kubectl expose rc chuck --container-port=8080 --port=8000

NAME      LABELS      SELECTOR    IP(S)     PORT(S)
chuck     run=chuck   run=chuck             8000/TCP

$ kubectl describe service chuck

Name:			chuck
Labels:			run=chuck
Selector:		run=chuck
...
IP:			    10.0.0.93
Port:			<unnamed>	8080/TCP
Endpoints:		172.17.0.14:8080

$ curl 172.17.0.14:8080

Chuck Norris can solve the Towers of Hanoi in one move.
```

Next step is to scale our application to 5 replicas
```
$ kubectl scale --current-replicas=1 --replicas=3 rc chuck

scaled

$ kubectl get pods

NAME                   READY     STATUS    RESTARTS   AGE
chuck-2z97v            1/1       Running   0          30s
chuck-bbbj5            1/1       Running   0          16m
chuck-ow5s0            1/1       Running   0          30s

$ kubectl describe service chuck

Name:			chuck
...
Port:			<unnamed>	8080/TCP
Endpoints:		172.17.0.14:8080,172.17.0.15:8080,172.17.0.16:8080

$ kubectl delete rc chuck

replicationcontrollers/chuck

$ kubectl delete service chuck

services/chuck
```

# Activity 2: Kubernetes manipulations via manifests

Commands are good, however we can operate with kubernetes via files. This will give us possiblity to store our configuration in the SCM and make it part of our applicaiton. So it could evolve together.

Take a look in the file stores ```src/main/infra/*```. You will see RC and SCV yaml manifests. One goes to replicaiton controller declaration and another goes to Service.

To start you can run following command:
```
$ kubectl create -f src/main/infra/chucknorris-rc.yml

replicationcontrollers/chuck

$ kubectl create -f src/main/infra/chucknorris-svc.yml

services/chuck

$ kubectl describe service chuck
Name:			chuck
Namespace:		default
Labels:			name=chuck
Selector:		app=chuck,version=0.1.0
Type:			ClusterIP
IP:			10.0.0.97
Port:			<unnamed>	80/TCP
Endpoints:		172.17.0.2:8080

$ curl 172.17.0.2
"It works on my machine" always holds true for Chuck Norris.
```
### Accessing services via service proxy 

Remember? Kubernetes has service proxy. We can access our services there as well

Presuming kubernetes has been avaible at ```http://localhost:8080``` (if you are keeping SSH connection with port mapping to localhost, otherwise chage ```localhost``` to public ip of kubernetes master).

Open your web browser and enter following enter following URL: ```http://localhost:8080/api/v1/proxy/namespaces/default/services/chuck/```

Then you should get something like the following: 
![alt text](https://raw.githubusercontent.com/akranga/kube-workshop/master/docs/chuck-browser.png "Chuck in your browser")

# Activity 3: Starting Jenkins Master

Instead of building the applications we will build container with application inside and then schedule it for Kubernetes.

For nice CI/CD experience we will need. 

```
- Private Docker Registy: To store containers that we are building. Private registry should be in the same network. This will reduce network latency for transferring containers
- Jenkins Master: This container will have Jenkins CI master with some nice plugins, including 'workflow' that allows you to write jenkins CD/CD job in Groovy DSL
- Jenkins java slave: Docker container that connects via swarm plugin to Jenkins master. It has JDK and other Java build tools 
- Jenkins docker slave: Acts after java app has beeen built and unit tested. It is building Docker image out of it. WARNING: requires priviledged mode 
- Jenkins kubernetes slave: interacts with kuberneetes instance to run container as kubernetes service
```

## Creating Jenkins Master

Jenkins Master replication controller and service availale here:

```
$ kubectl create -f jenkins/jenkins-master-rc.yml

replicationcontrollers/jenkins

$ kubectl create -f jenkins/jenkins-master-svc.yml

services/jenkins
```

It will take few minutes for Jenkins to warm up... Be patient! You can check status by executing ```$ kubectl get pods```

Once it is running you should be able to connect by mapping ports from remote host to local via SSH (if you running remote host in the cloud)

To do so, you need to capture IP and ports by executing following command:

```
$ kubectl describe service jenkins

Name:			jenkins
...
Port:			web	80/TCP
Endpoints:		172.17.0.2:8080
Port:			web8080	8080/TCP
Endpoints:		172.17.0.2:8080
Port:			swarm	50000/TCP
Endpoints:		172.17.0.2:50000
```

We only need endpoint 'web' or 'web8080' both actually mapping same port of the container. Now disconnect your SSH. You will need to add
```-L8081:172.17.0.2:8080``` (where 172.17.0.2 is the IP of the endpoint. It might be different for you!).

Then you should be able to point with your browser by entering `http://localhost:8081` in addressbar

NOTE: You cannot navigate Jeniins UI proxied with Kubernetes service proxy. Jenkins doesn't like it. As a solution for this you need to use an external load balancer. However for this workshop we will just call Jenkins service endpoint directly via port mapping.

![alt text](https://raw.githubusercontent.com/akranga/kube-workshop/master/docs/jenkins-1.png "Jenkins CI Server")

So far so good! However we are not ready yet. Problem is that pods are short-living beasts. What if kubernetes will decide to restart our Jenkins pod?. Remember? Container dies with it's data. Sounds like we need some persistent volumes

# Activiy 4: Operations with Persistent volumes

Here is the contract. Persistent volumes have been configured by Operator, while POD and RC are typically responsibility of the DEV. We need to put between persistent volume (PV) and POD an abstraction layer called in Kubernetes as Persistent Volume Claim. It will give possiblity to the PODs claim storage without knowing specifics storage provisioning details

Persistent volumes has been configuration can be executed by running command:
```
$ kubectl create -f jenkins/jenkins-pv.yml
persistentvolumes/jenkins-data-vol1
persistentvolumes/jenkins-data-vol2
ubuntu@ip-172-31-8-2:~/kube-workshop$ kubectl get pv
NAME                LABELS                   CAPACITY      ACCESSMODES   STATUS      CLAIM     REASON
jenkins-data-vol1   name=jenkins-workspace   21474836480   RWO           Available
jenkins-data-vol2   name=jenkins-jobs        10737418240   RWO           Available
```

It marks local directories `/data/vol-01` and `/data/vol-02` as persistent volumes for Kubernetes pods. You can map it directly to pod, however it is more nice to map it via so called `Persistent Volume Claims (pvc)` that abstracts pods from concrete persistent volume implementations (hostDir, AWS EBS volume, nfs etc). Pod can request Persistent Volume from kubernetes by claiming it by capacity, read-write strategy etc.

Persistent volumes claims can be created by running following command:
```
$ kubectl create -f jenkins/jenkins-pvc.yml

persistentvolumeclaims/jenkins-workspace
persistentvolumeclaims/jenkins-jobs

$ kubectl get pvc
NAME                LABELS    STATUS    VOLUME
jenkins-jobs        map[]     Bound     jenkins-data-vol2
jenkins-workspace   map[]     Bound     jenkins-data-vol1
```

You can see Claims has been bound to the Persistent Volumes

### Back to Jenkins

Now our goal is to run Jenkins POD with respect to storage. You need to stop RC and POD of "stateless" Jenkins and run with volumes.

```
$ kubectl stop -f jenkins/jenkins-master-rc.yml
replicationcontrollers/jenkins

$ kubectl create -f jenkins/jenkins-master-rc-with-volumes.yml
replicationcontrollers/jenkins

$ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
jenkins-mlmtm          1/1       Running   0          40s
```

Second POD file has one difference: it uses PVC
```yaml
spec:
    ...
    volumeMounts:
    - name: jenkins-workspace
      mountPath: /var/jenkins_home/workspace
      readOnly: false
    - name: jenkins-jobs
      mountPath: /var/jenkins_home/jobs
      readOnly: false
  volumes:
  - name: jenkins-jobs
    persistentVolumeClaim:
      claimName: jenkins-jobs
  - name: jenkins-workspace
    persistentVolumeClaim:
      claimName: jenkins-workspace
```

We have two volumes one to store Jenkins jobs and another to store workspaces. This is what we don't want to loose. Other Jenkins configuration (plugins, tools etc) needs to be baked inside of container.

### Add a simple Jenkins Workflow

Now let's create a simple job! Click to create Jobs link (as follows on the screenshot).
![alt text](https://raw.githubusercontent.com/akranga/kube-workshop/master/docs/jenkins-2.png "Jenkins CI")


Give a name to the new job: let's say: ```hello-chuck```. And select a type ```Workflow```
![alt text](https://raw.githubusercontent.com/akranga/kube-workshop/master/docs/jenkins-3.png "Jenkins CI")

Add following script:
```groovy

node("master") {
  git 'https://github.com/akranga/chucknorris'
  echo 'Done!!!"
}

```

And... run the build!

# Activity 5: Operations with Jenkins

We setup our Jenkins Master and configured persistent volumes, As you noticed we are able to do git pull but we do not able to  compile it. This is because Jenkins Master doesn't have JDK tool installed. We can install it (that's an option); however this will make our Jenkins Master more majical that violates microservice nature of Docker.

Instead of running compilation on Jenkins Master we will prepare set of Dockerized Slaves. 

* Java Builder: has tool JDK. Can run, compile and do test-suits.
* Docker Builder: will create build docker container
* Kubernetes Builder: will manipulate with kubernetes API. Schedule PODs and Services

Let's run it:

```
$ kubectl create -f jenkins/java-slave-rc.yml
$ kubectl create -f jenkins/docker-slave-rc.yml
$ kubectl create -f jenkins/kubernetes-slave-rc.yml
```

### How does it works?

We have a challenge
