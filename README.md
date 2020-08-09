# Node (Express) with Kubernetes 
- Nodejs - Express application
- Kubernetes deployment with Helm
- Monitoring - Prometheus and Grafana
- Tracing etc.


### Docker commands
Remove image 
```
docker rmi -f nodeserver-my-app nodeserver-my-tools
```
### Images and Dockerfiles
```
wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/Dockerfile
wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/.dockerignore
docker build -t nodeserver-my-app -f Dockerfile .
docker images
docker run -i  -p 3400:3000 -t nodeserver-my-app
```

### Debug images:
```
wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/Dockerfile-tools
wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/run-dev
wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/run-debug

docker build -t nodeserver-my-tools -f Dockerfile-tools .
docker images

docker run -i -v "$PWD"/package.json:/tmp/package.json -v "$PWD"/node_modules_linux:/tmp/node_modules -w /tmp -t node:10 npm install

docker run -i -p 3400:3000 -v "$PWD"/:/app -v "$PWD"/node_modules_linux:/app/node_modules -t  nodeserver-my-tools /bin/run-dev

docker run -i --expose 9929 -p 9929:9929 -p 3400:3000 -v "$PWD"/:/app -v "$PWD"/node_modules_linux:/app/node_modules -t  nodeserver-my-tools /app/run-debug

```

#### Optimized Run image
```
wget https://raw.githubusercontent.com/CloudNativeJS/docker/master/Dockerfile-run
docker build -t nodeserver-my-run -f Dockerfile-run .
docker run -i  -p 3400:3000 -t nodeserver-my-run
```


### Debug node-js appliction on Chrome:
Use [chrome://insepect](chrome://inspect{:target="_blank"}) to open debugger

### Tag and publish images to Dockerhub:
Tag image with some share-worthy name and push it to Dockerhub - this woudl show up in [Dockerhub, undedr account mahajany](https://hub.docker.com/repository/docker/mahajany/nodeserver-my-run)
```
docker tag nodeserver-my-run production-nodeserver-my-run:1.0.0
docker tag nodeserver-my-run mahajany/nodeserver-my-run:0.0.1
docker push mahajany/nodeserver-my-run:0.0.1
```
Now the image can be used by `docker pull mahajany/nodeserver-my-run`  and be  run with `docker run -i -p 3400:3000 mahajany/nodeserver-my-run`. This will get the latest image, you can specify a version with `docker pull mahajany/nodeserver-my-run:0.0.1`  and be  run with `docker run -i -p 3400:3000 mahajany/nodeserver-my-run:0.0.1`. 

### Kubernetes - liveness and readiness check:
Cloud health and ready
```
npm install --save @cloudnative/health-connect
```
...and some code-change to add __/live__ and __/ready__ endpoints:
http://localhost:3000/live ==>  {"status":"UP","checks":[]}

Add pingcheck:

Success:
{"status":"UP","checks":[{"name":"PingCheck HEAD:example.com:80/","state":"UP","data":{"reason":""}}]}

Failure:
{"status":"DOWN","checks":[{"name":"PingCheck HEAD:example.com:80/","state":"DOWN","data":{"reason":"Failed to ping HEAD:example.com:80/: getaddrinfo ENOTFOUND example.com exampldddde.com:80"}}]}

Liveness check:
http://localhost:3000/live ==>  {"status":"UP","checks":[]}

Customized it - 
{"status":"UP","checks":[{"name":"PingCheck HEAD:example.com:80/","state":"UP","data":{"reason":""}}]}


### Prometheus and Grafana support
```
npm install --save node-gyp
npm install --save appmetrics-prometheus
```
Just add `1var prom = require('appmetrics-prometheus').attach();` and it will expose endpoint `/metrics`! It provides a lot of other stats as well.


## Helm
Get chart from [Cloude Native JS chart example](https://github.com/CloudNativeJS/helm), and update `values.yaml` for image-repository name and tag - `  repository: mahajany/nodeserver-my-run` and `  tag: 0.0.2`. You can change the replica-count as well.

```
helm repo list
helm install noderserver-my-k8 helm/chart/nodeserver
helm status noderserver-my-k8

```

`helm status noderserver-my-k8` gives message like this:
```
NAME: noderserver-my-k8
LAST DEPLOYED: Sat Aug  8 13:59:25 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Congratulations, you have deployed your Node.js Application to Kubernetes using Helm!

To verify your application is running, run the following two commands to set the SAMPLE_NODE_PORT and SAMPPLE_NODE_IP environment variables to the location of your application:

export SAMPLE_NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services nodeserver-service)
export SAMPLE_NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")

And then open your web browser to http://${SAMPLE_NODE_IP}:${SAMPLE_NODE_PORT}" from the command line, eg:

open http://${SAMPLE_NODE_IP}:${SAMPLE_NODE_PORT}
$ echo  http://${SAMPLE_NODE_IP}:${SAMPLE_NODE_PORT}
```
So, you can access application either at http://192.168.65.3:32195 or at localhost:32195.

```
helm uninstall noderserver-my-k8
helm repo list
helm install noderserver-my-k8 helm/chart/nodeserver
helm status noderserver-my-k8
helm upgrade noderserver-my-k8 helm/chart/nodeserver
helm upgrade --install noderserver-my-k8 helm/chart/nodeserver
helm history noderserver-my-k8
helm rollback noderserver-my-k8 1
helm history noderserver-my-k8

```


#### Kubernetes dasbord:

K8 command: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml`

##### Helm:Add kubernetes-dashboard repository
`helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/`
####@  Deploy a Helm Release named "my-release" using the kubernetes-dashboard chart
```
helm install k8-monitor kubernetes-dashboard/kubernetes-dashboard
export POD_NAME=$(kubectl get pods -n default -l "app.kubernetes.io/name=kubernetes-dashboard,app.kubernetes.io/instance=k8-monitor" -o jsonpath="{.items[0].metadata.name}")
echo $POD_NAME
kubectl -n default port-forward $POD_NAME 8443:8443
kubectl proxy
127.0.0.1:8001
```
Get dasboard [here](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login)

##### K8-dasbhoard: get a bearer token:
 `kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')`


#### Prometheus:
Install with helm: `https://hub.helm.sh/charts/stable/prometheus`

helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm install  my-monitor stable/prometheus --namespace promethues
helm install  my-monitor stable/prometheus 


 helm status  my-monitor ---> A lot of info on screen
 
Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  echo $POD_NAME
  kubectl --namespace default port-forward $POD_NAME 9090
  ...now go to [localhost:9090](localhost:9090) for Promethus.

#### Grafana
```
 helm install my-monitor-grafana stable/grafana

 helm install my-monitor-grafana stable/grafana --set adminPassword=password


```  
Import dashboard 1621.


#### Open Tracing
```
npm install --save appmetrics-zipkin
```
...and add line in app.js `var zipkin = require('appmetrics-zipkin')();`. But we need a zipkin server as well for tracing.
`docker run -d -p 9411:9411 openzipkin/zipkin:1`

But Zipkins is not availabel on helm, use jaeger:
```
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger  jaegertracing/jaeger --version 0.33.0

```

## All the credits go to:
#### Tutorials
1. Chris Baily - Docker, Kubernetes and Helm with Node: https://www.linkedin.com/learning/cloud-native-development-with-node-js-docker-and-kubernetes 
2. Express site: https://shapeshed.com/creating-a-basic-site-with-node-and-express/

#### Reference Websites:
http://landscape.cncf.io/
https://www.cloudnativejs.io/   
https://landscape.cncf.io/
https://expressjs.com/
https://hub.helm.sh/
https://hub.helm.sh/charts/stable/prometheus

