# Deploy SingleStore with Azure Kubernetes Engine

### Prerequisites 

Make sure you have [kubectl](https://kubernetes.io/docs/tasks/tools/) and [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/) installed.

Use the `az login` command to initially login to Azure.

## Creating and connecting to the Kubernetes cluster

We will use the `az aks create` command to create our `sdb-cluster` using the following optional flags:

Note: `--name` and `--resource-group` are required

    --node-count
    --node-vm-size
    --enable-cluster-autoscaler
    --min-count
    --max-count
    --network-plugin
    --network-policy

###### Example command
```
az aks create --name sdb-cluster --resource-group singlestore --node-count 4 --node-vm-size standard_d4s_v3 --enable-cluster-autoscaler --min-count 1 --max-count 12 --network-plugin azure --network-policy calico
```
Note: Azure CNI and Calico are recommended

Run the `az account set --subscription <subscription-id>` and `az aks get-credentials --resource-group singlestore --name sdb-cluster` commands to connect to your Kubernetes cluster.

If you have kubectl correctly installed, you should now be able to run `kubectl get ns` without errors.

## Deploy SingleStore on Kubernetes

### Download the Operator image (optional if not pulling from deployment.yaml)

The `memsql/operator` can be pulled from [Docker Hub](https://hub.docker.com/r/memsql/operator/tags)

###### Example with Docker
```
docker pull memsql/operator:1.2.5-83e8133a
```

### Create the Object Definition files

Download the [object definition files](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files.html) from our site, and save them in an accessible directory. The [rbac.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/rbac-yaml.html) and the [memsql-cluster-crd.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/memsql-cluster-crd-yaml.html) do not require edits.

Edit the [deployment.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/deployment-yaml.html) and reference the image you downloaded, or you can pull it automatically.

#### `deployment.yaml`

###### Example spec
```
    spec:
      serviceAccountName: memsql-operator
      containers:
        - name: memsql-operator
          image: memsql/operator:1.2.5-83e8133a
          imagePullPolicy: IfNotPresent
```

#### `memsql-cluster.yaml`

In order to edit the [memsql-cluster.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/memsql-cluster-yaml.html) you will need your license key from the customer [portal](https://auth.singlestore.com/auth/realms/memsql/protocol/openid-connect/auth?client_id=customer-portal-login&redirect_uri=https%3A%2F%2Fportal.singlestore.com%2F&state=0e422fe0-0db1-45d3-a27d-e9b27c64cd82&response_mode=fragment&response_type=code&scope=openid&nonce=4022881b-27c3-406a-b0e0-ba83cd5d9985) and a hashed version of a secure password for the admin user. 

In order to hash a secure password, we have an example python script:
```
from hashlib import sha1
print("*" + sha1(sha1('secretpass'.encode('utf-8')).digest()).hexdigest().upper())
```
###### How to use the script
> Save the script in a file named `hash_password.py`, replacing `secretpass` with a secure password. Make sure it is executable with `chmod +x hash_password.py`, then run the script with `python3 hash_password.py`. It will print the hashed password to the command line, where you can copy and paste directly to the `memsql-cluster.yaml` file.

Create a namespace for the Kubernetes Objects
```
kubectl create ns singlestore
```
Add the namespace to the object metadata
```
apiVersion: memsql.com/v1alpha1
kind: MemsqlCluster
metadata:
  name: memsql-cluster
  namespace: singlestore
```
Add the `licence_key`, `hashed_password` keeping the quotes, and use the latest `memsql/node` image
```
spec:
  license: <license_key>
  adminHashedPassword: "<hashed_password>"
  nodeImage:
    repository: memsql/node
    tag: latest
```
Change the redundancy level to `2`
```
  redundancyLevel: 2
```
Change the storage class to `default` within the `aggregatorSpec:`
```
storageClass: default
```

The rest of the edits to the `aggregatorSpec` and the `leafSpec` are custom depending on how many aggregators and leafs you want and how many resources you want allocated to them. Save all yaml files to an accessible directory.

### Create the Kubernetes Objects

Create the role-based access control
```
kubectl -n singlestore create -f rbac.yaml
```
Create the memsql custom resource definition
```
kubectl -n singlestore create -f memsql-cluster-crd.yaml
```
Create the Operator deployment
```
kubectl -n singlestore create -f deployment.yaml
```
Now you should be able to run `kubectl -n singlestore get pods` and see that the Operator pod is running.
```
NAME                               READY   STATUS    RESTARTS   AGE
memsql-operator-79874797f4-9t47r   1/1     Running   0          2m50s
```
Create the memsql custom resource
```
kubectl -n singlestore create -f memsql-cluster.yaml
```
Now you can run `kubectl -n singlestore get memsql` to see the custom resource that was created.
```
NAME             AGGREGATORS   LEAVES   REDUNDANCY LEVEL   AGE
memsql-cluster   1             1        2                  30h
```
### Connect to the database

Run `kubectl -n singlestore get services` in order to get the `ddl` endpoint
```
NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
svc-memsql-cluster       ClusterIP      None             <none>          3306/TCP         30h
svc-memsql-cluster-ddl   LoadBalancer   <ip-address>     <ip-address>    3306:30907/TCP   30h
```
The IP address under `EXTERNAL-IP` in the `svc-memsql-cluster-ddl` row is the one that you will use to connect to your database. You will use the `admin` user and the un-hashed password with port `3306` to connect. 

###### Example connection command
```
mysql -h<ip-address> -uadmin -P3306 -p<secretpass>
```
This concludes setting up SingleStore with Azure Kubernetes Engine.
