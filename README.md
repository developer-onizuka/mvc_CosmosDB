# mvc_CosmosDB


# 1. Create Azure Cosmos DB API for MongoDB

- Step1: On the Azure portal menu (https://portal.azure.com),Select Create a resource.
- Step2: Select Databases, and then select Azure Cosmos DB.
- Step3: Select Azure Cosmos DB API for MongoDB.
- Step4: Copy the PRIMARY CONNECTION STRING which starts with "mongodb://".

# 2. Create text file of secret of PRIMARY CONNECTION STRING
What you paste to the text file is just the strings followed "mongodb://".
```
$ cat connection-string.txt 
myfirstcosmosdb:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==@myfirstcosmosdb.mongo.cosmos.azure.com:10255/?ssl=true&retrywrites=false&replicaSet=globaldb&maxIdleTimeMS=120000&appName=@myfirstcosmosdb@
```

# 3. Create secret resouce in kubernetes cluster
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

# 4. Create deployment of "Employee Web app" with 2 repricas connecting Azure CosmosDB
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

