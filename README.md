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

# 3. Create secret as a k8s resouce
```
$ kubectl create secret generic connectionstring --from-file=secretenv=./connection-string.txt
```

