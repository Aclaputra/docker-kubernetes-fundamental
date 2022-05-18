## setup and requirements
In the cloud shell environment type the following command to set the zone:
```bash
gcloud config set compute/zone us-central1-b
```
After you set the zone, start up a cluster for use in this lab:
```bash
gcloud container clusters create io
```
## get the samples code
Clone the GitHub repository from the Cloud Shell command line:
```bash
gsutil cp -r gs://spls/gsp021/* .
```
Change into the directory needed for this lab:
```bash
cd orchestrate-with-kubernetes/kubernetes
```
List the files to see what you're working with:
```bash
ls
```

## quick kubernetes demo
The easiest way to get started with Kubernetes is to use the kubectl create command. Use it to launch a single instance of the nginx container:
```bash
kubectl create deployment nginx --image=nginx:1.10.0
```
In Kubernetes, all containers run in a pod. Use the kubectl get pods command to view the running nginx container:
```bash
kubectl get pods
```
```bash
kubectl expose deployment nginx --port 80 --type LoadBalancer
```
List our services now using the kubectl get services command:
```bash
kubectl get services
```
Add the External IP to this command to hit the Nginx container remotely:
```bash
curl http://<External IP>:80
```
## pods
At the core of Kubernetes is the [Pod](https://kubernetes.io/docs/concepts/workloads/pods/).

Pods represent and hold a collection of one or more containers. Generally, if you have multiple containers with a hard dependency on each other, you package the containers inside a single pod.

## creating pods
```bash
cat pods/monolith.yaml
```
output :
```bash
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
```
Create the monolith pod using kubectl:
```bash
kubectl create -f pods/monolith.yaml
```
Examine your pods. Use the kubectl get pods command to list all pods running in the default namespace:
```bash
kubectl get pods
```
Once the pod is running, use kubectl describe command to get more information about the monolith pod:
```bash
kubectl describe pods monolith
```

## interacting with pods
In the 2nd terminal, run this command to set up port-forwarding:
```bash
kubectl port-forward monolith 10080:80
```
Now in the 1st terminal start talking to your pod using curl:
```bash
curl http://127.0.0.1:10080
```
Now use the curl command to see what happens when you hit a secure endpoint:
```bash
curl http://127.0.0.1:10080/secure
```
Try logging in to get an auth token back from the monolith:
```bash
curl -u user http://127.0.0.1:10080/login
```
At the login prompt, use the super-secret password "password" to login.

Logging in caused a JWT token to print out. Since Cloud Shell does not handle copying long strings well, create an environment variable for the token.
```bash
TOKEN=$(curl http://127.0.0.1:10080/login -u user|jq -r '.token')
```
Enter the super-secret password "password" again when prompted for the host password.

Use this command to copy and then use the token to hit the secure endpoint with curl:
```bash
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure
```
At this point you should get a response back from our application, letting us know everything is right in the world again.

Use the kubectl logs command to view the logs for the monolith Pod.
```bash
kubectl logs monolith
```
Open a 3rd terminal and use the -f flag to get a stream of the logs happening in real-time:
```bash
kubectl logs -f monolith
```
Now if you use curl in the 1st terminal to interact with the monolith, you can see the logs updating (in the 3rd terminal):
```bash
curl http://127.0.0.1:10080
```
Use the kubectl exec command to run an interactive shell inside the Monolith Pod. This can come in handy when you want to troubleshoot from within a container:
```bash
kubectl exec monolith --stdin --tty -c monolith -- /bin/sh
```
For example, once you have a shell into the monolith container you can test external connectivity using the ping command:
```bash
ping -c 3 google.com
```
Be sure to log out when you're done with this interactive shell.
```bash
exit
```
## services
Pods aren't meant to be persistent. They can be stopped or started for many reasons - like failed liveness or readiness checks - and this leads to a problem:

What happens if you want to communicate with a set of Pods? When they get restarted they might have a different IP address.

That's where [Services](https://kubernetes.io/docs/concepts/services-networking/service/) come in. Services provide stable endpoints for Pods.

## creating a service
Before you can create our services, first create a secure pod that can handle https traffic.

If you've changed directories, make sure you return to the ~/orchestrate-with-kubernetes/kubernetes directory:
```bash
cd ~/orchestrate-with-kubernetes/kubernetes
```
Explore the monolith service configuration file:
```bash
cat pods/secure-monolith.yaml
```
Create the secure-monolith pods and their configuration data:
```bash
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf
kubectl create -f pods/secure-monolith.yaml
```
Now that you have a secure pod, it's time to expose the secure-monolith Pod externally.To do that, create a Kubernetes service.

Explore the monolith service configuration file:
```bash
cat services/monolith.yaml
```
output :
```bash
kind: Service
apiVersion: v1
metadata:
  name: "monolith"
spec:
  selector:
    app: "monolith"
    secure: "enabled"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443
      nodePort: 31000
  type: NodePort
```
Things to note:

There's a selector which is used to automatically find and expose any pods with the labels app: monolith and secure: enabled.
Now you have to expose the nodeport here because this is how you'll forward external traffic from port 31000 to nginx (on port 443).
Use the kubectl create command to create the monolith service from the monolith service configuration file:
```bash
kubectl create -f services/monolith.yaml
```
Use the gcloud compute firewall-rules command to allow traffic to the monolith service on the exposed nodeport:
```bash
gcloud compute firewall-rules create allow-monolith-nodeport \
  --allow=tcp:31000
```
Now that everything is set up you should be able to hit the secure-monolith service from outside the cluster without using port forwarding.

First, get an external IP address for one of the nodes.
```bash
gcloud compute instances list
```
Now try hitting the secure-monolith service using curl:
```bash
curl -k https://<EXTERNAL_IP>:31000
```
Uh oh! That timed out. What's going wrong?

It's time for a quick knowledge check.Use the following commands to answer the questions below.
```bash
kubectl get services monolith

kubectl describe services monolith
```
Questions:

Why are you unable to get a response from the monolith service?
How many endpoints does the monolith service have?
What labels must a Pod have to be picked up by the monolith service?
Hint: it has to do with labels. You'll fix the issue in the next section.

## adding label to pods
Currently the monolith service does not have endpoints. One way to troubleshoot an issue like this is to use the kubectl get pods command with a label query.

You can see that you have quite a few pods running with the monolith label.
```bash
kubectl get pods -l "app=monolith"
```
But what about "app=monolith" and "secure=enabled"?
```bash
kubectl get pods -l "app=monolith,secure=enabled"
```
Use the kubectl label command to add the missing secure=enabled label to the secure-monolith Pod. Afterwards, you can check and see that your labels have been updated.
```bash
kubectl label pods secure-monolith 'secure=enabled'
kubectl get pods secure-monolith --show-labels
```
Now that your pods are correctly labeled, view the list of endpoints on the monolith service:
```bash
kubectl describe services monolith | grep Endpoints
```
Test this out by hitting one of our nodes again.
```bash
gcloud compute instances list
curl -k https://<EXTERNAL_IP>:31000
```
Bam! Houston, we have contact.
## deploying applications with kubernetes
The goal of this lab is to get you ready for scaling and managing containers in production. That's where [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#what-is-a-deployment) come in. Deployments are a declarative way to ensure that the number of Pods running is equal to the desired number of Pods, specified by the user.

## creating deployments
Get started by examining the auth deployment configuration file.
```bash
cat deployments/auth.yaml
```
The deployment is creating 1 replica, and you're using version 2.0.0 of the auth container.

When you run the kubectl create command to create the auth deployment it will make one pod that conforms to the data in the Deployment manifest. This means you can scale the number of Pods by changing the number specified in the Replicas field.

Anyway, go ahead and create your deployment object:
```bash
kubectl create -f deployments/auth.yaml
```
It's time to create a service for your auth deployment. Use the kubectl create command to create the auth service:
```bash
kubectl create -f services/auth.yaml
```
Now do the same thing to create and expose the hello deployment:
```bash
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
```
```bash
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
```
Interact with the frontend by grabbing its External IP and then curling to it:
```bash
kubectl get services frontend
```
Note:It might take a minute for the external IP address to be generated. Run the above command again if the EXTERNAL-IP column status is pending.
```bash
curl -k https://<EXTERNAL-IP>
```