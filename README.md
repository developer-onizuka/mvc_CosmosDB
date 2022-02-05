# mvc_CosmosDB


# 1. Create Azure Cosmos DB API for MongoDB

- Step1: On the Azure portal menu (https://portal.azure.com), Select Create a resource.
- Step2: Select Databases, and then select Azure Cosmos DB.
- Step3: Select Azure Cosmos DB API for MongoDB.
- Step4: Copy the PRIMARY CONNECTION STRING which starts with "mongodb://".

![cosmosdb1.png](https://github.com/developer-onizuka/mvc_CosmosDB/blob/main/cosmosdb1.png)

# 2. Make the secret file of PRIMARY CONNECTION STRING to access the CosmosDB you created
What you paste to the text file is just the strings followed "mongodb://".
```
$ vi connection-string.txt 
myfirstcosmosdb:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==@myfirstcosmosdb.mongo.cosmos.azure.com:10255/?ssl=true&retrywrites=false&replicaSet=globaldb&maxIdleTimeMS=120000&appName=@myfirstcosmosdb@
```

# 3. Create a secret resouce in kubernetes cluster
You can remove the file written secret after creating secret resource in kubernetes cluseter, immediately.
```
$ kubectl create secret generic connectionstring --from-file=secretenv=./connection-string.txt
$ rm -f connection-string.txt
$ kubectl describe secrets connectionstring 
Name:         connectionstring
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
secretenv:  269 bytes
```

# 4. Create IngressGateway for this purpose
```
$ git clone https://github.com/developer-onizuka/mvc_CosmosDB
$ cd mvc_CosmosDB
$ istioctl install -y -f azure-gateway.yaml 
✔ Ingress gateways installed                                                                                                                                      
✔ Installation complete                                                                                                                                           

$ kubectl get pods -n istio-system
NAME                                     READY   STATUS    RESTARTS      AGE
grafana-5fb899f96-67d5f                  1/1     Running   1 (8d ago)    21d
istio-azuregateway-67d6d7d84f-fznfr      1/1     Running   0             27s
istio-eastwestgateway-7c8db99c44-fjbfd   1/1     Running   1 (14d ago)   21d
istio-ingressgateway-779cd8d9fb-6bxns    1/1     Running   1 (14d ago)   21d
istiod-7d5ddd8fcf-6bcq9                  1/1     Running   1 (8d ago)    21d
jaeger-d7849fb76-hsbqq                   1/1     Running   1 (8d ago)    21d
kiali-c9d6f75d5-vjvv5                    1/1     Running   1 (14d ago)   21d
prometheus-d7df8c957-9tkwd               2/2     Running   2 (8d ago)    21d

$ kubectl get services -n istio-system
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                           AGE
grafana                 ClusterIP      10.103.59.87     <none>           3000/TCP                                                          21d
istio-azuregateway      LoadBalancer   10.103.187.59    192.168.33.223   15021:30238/TCP,443:32535/TCP,80:31476/TCP                        37s
istio-eastwestgateway   LoadBalancer   10.109.178.196   192.168.33.221   15021:30600/TCP,15443:31534/TCP,15012:31242/TCP,15017:30426/TCP   21d
istio-ingressgateway    LoadBalancer   10.110.212.70    192.168.33.220   15021:31932/TCP,80:30217/TCP,443:31930/TCP                        21d
istiod                  ClusterIP      10.111.13.175    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                             21d
jaeger-collector        ClusterIP      10.101.9.249     <none>           14268/TCP,14250/TCP,9411/TCP                                      21d
kiali                   LoadBalancer   10.107.184.95    192.168.33.222   20001:32532/TCP,9090:32092/TCP                                    21d
prometheus              ClusterIP      10.109.69.225    <none>           9090/TCP                                                          21d
tracing                 ClusterIP      10.111.142.142   <none>           80/TCP,16685/TCP                                                  21d
zipkin                  ClusterIP      10.101.75.155    <none>           9411/TCP                                                          21d

```

# 5. Create deployment of "Employee Web app" with 2 repricas connecting Azure CosmosDB
```
$ kubectl apply -f employee-azure-cosmosdb.yaml 
service/employee-azure-svc created
deployment.apps/employee-azure created
```
The environment of MONGO is for setting a connection string, but in this case it is a secret of PRIMARY CONNECTION STRING, especially as like below:
```
        env:
        - name: MONGO
          #value: 'mongo-0'
          #value: 'mongo-0:27017,mongo-1:27017,mongo-2:27017/?replicaSet=myReplicaSet'
          #value: 192.168.33.30:27017,192.168.33.31:27017,192.168.33.32:27017/?replicaSet=myReplicaSet
          valueFrom:
            secretKeyRef:
              name: connectionstring
              key: secretenv
```

# 5. Create Ingress Gateway for accessing from outside of the Cluster
```
$ kubectl apply -f ingress-gateway.yaml 
gateway.networking.istio.io/employee-gateway created
```

# 6. Create Nginx's config files and Configmap
```
$ kubectl create configmap nginx-azure-config --from-file=default-azure.conf 
configmap/nginx-azure-config created
```

# 7. Create depolyment of Nginx with 2 repricas
```
$ kubectl apply -f nginx-azure.yaml 
virtualservice.networking.istio.io/nginx-azure-vsvc created
service/nginx-azure-svc created
deployment.apps/nginx-azure created
```

# 8. Check if all of pods are available
```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
employee-azure-57cf4cfdd4-4r6qw   2/2     Running   0          20s
employee-azure-57cf4cfdd4-7l8jf   2/2     Running   0          20s
nginx-azure-587c6698b9-k6chb      2/2     Running   0          5s
nginx-azure-587c6698b9-qzpv4      2/2     Running   0          5s
```

# 9. Check if which services are invoked
```
$ kubectl get services -o wide
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE   SELECTOR
employee-azure-svc   ClusterIP   10.104.95.102    <none>        5001/TCP,5000/TCP   96s   app=employee-azure
kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP             18d   <none>
nginx-azure-svc      ClusterIP   10.102.208.104   <none>        8080/TCP            81s   app=nginx-azure
```

# 10. Let's Access to it
Find the IP address of Istio-ingressgateway. In this case, it is 192.168.33.220.
```
$ kubectl get services -n istio-system 
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                           AGE
grafana                 ClusterIP      10.103.59.87     <none>           3000/TCP                                                          18d
istio-eastwestgateway   LoadBalancer   10.109.178.196   192.168.33.221   15021:30600/TCP,15443:31534/TCP,15012:31242/TCP,15017:30426/TCP   18d
istio-ingressgateway    LoadBalancer   10.110.212.70    192.168.33.220   15021:31932/TCP,80:30217/TCP,443:31930/TCP                        18d
istiod                  ClusterIP      10.111.13.175    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                             18d
jaeger-collector        ClusterIP      10.101.9.249     <none>           14268/TCP,14250/TCP,9411/TCP                                      18d
kiali                   LoadBalancer   10.107.184.95    192.168.33.222   20001:32532/TCP,9090:32092/TCP                                    18d
prometheus              ClusterIP      10.109.69.225    <none>           9090/TCP                                                          18d
tracing                 ClusterIP      10.111.142.142   <none>           80/TCP,16685/TCP                                                  18d
zipkin                  ClusterIP      10.101.75.155    <none>           9411/TCP                                                          18d
```

![cosmosdb4.png](https://github.com/developer-onizuka/mvc_CosmosDB/blob/main/cosmosdb4.png)

![cosmosdb5.png](https://github.com/developer-onizuka/mvc_CosmosDB/blob/main/cosmosdb5.png)

![cosmosdb6.png](https://github.com/developer-onizuka/mvc_CosmosDB/blob/main/cosmosdb6.png)
