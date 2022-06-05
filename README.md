# Docker Fundamental
Dockerhub: https://hub.docker.com/repository/docker/aclaputra/chatbot-tvlk

```bash
docker images
docker ps
docker ps -a

mkdir test && cd test
```

Spend some time reviewing the https://docs.docker.com/engine/reference/builder/#known-issues-run to understand each line of the Dockerfile.

```bash
docker build -t node-app:0.1 .
docker images
```
create app.js
```bash
docker run -p 4000:80 --name intro-docker node-app:0.1
docker images
```

```bash
curl http://localhost:4000
```
it supposed to show hello world

```bash
docker stop intro-docker && docker rm intro-docker
```
```bash
docker logs [container_id]
```

change app.js then make another images
also used for updating new version
```bash
docker build -t node-app:0.2 .
docker images
```
run it on localhost:8080
```bash
docker run -p 8080:80 --name my-app-2 -d node-app:0.2
docker ps
```
the text will change and the old version is still saved in the docker images can be used it later
```bash
curl http://localhost:8080
curl http://localhost:4000
```
Debug
```bash
docker logs -f [container_id]
```

Get inside container 
```bash
docker exec -it [container_id] bash
ls
exit
```
You can examine a container's metadata in Docker by using Docker inspect:
```bash
docker inspect [container_id]
```
```bash
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [container_id]
```

## publish
check project id
```bash
gcloud config list project
```

```bash
docker tag node-app:0.2 gcr.io/[project-id]/node-app:0.2
docker images

docker push gcr.io/[project-id]/node-app:0.2
```
Check that the image exists in gcr by visiting the image registry in your web browser. You can navigate via the console to Navigation menu > Container Registry and click node-app or visit: http://gcr.io/[project-id]/node-app.

stop and remove all containers
```bash
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
```
You have to remove the child images (of node:lts) before you remove the node image. Replace [project-id].
```bash
docker rmi node-app:0.2 gcr.io/[project-id]/node-app node-app:0.1
docker rmi node:lts
docker rmi $(docker images -aq) # remove remaining images
docker images
```
images will be empty

try to pull it again from google container registry
```bash
docker pull gcr.io/[project-id]/node-app:0.2
docker run -p 4000:80 -d gcr.io/[project-id]/node-app:0.2
curl http://localhost:4000
```
it will run the version 0.2 that we push earlier to the container registry (its like github)

how to upload it to artifact container registry -> https://cloud.google.com/kubernetes-engine/docs/tutorials/hello-app

[>sumber belajar<](https://www.cloudskillsboost.google/focuses/1029?parent=catalog)
