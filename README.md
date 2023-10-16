# k8s-postgres-cluster


### Pre-requisites

- [docker](https://docs.docker.com/get-docker/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [psql](https://www.postgresql.org/download/)
- [postgres-docker](https://hub.docker.com/_/postgres)

### Running minikube in local machine for Mac M1

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
minikube start --driver=docker
```

### Folder structure

<pre>

|-- postgres-namespace.yml          # Contains the namespace for the postgres cluster
|-- postgres-master-configmap.yml   # Contains the configmap for the master postgres
|-- postgres-master-statefulset.yml # Contains the statefulset for the master postgres
|-- postgres-master-svc.yml         # Contains the service for the master postgres
|-- postgres-slave-configmap.yml    # Contains the configmap for the slave postgres
|-- postgres-slave-statefulset.yml  # Contains the statefulset for the slave postgres
|-- postgres-slave-svc.yml          # Contains the service for the slave postgres
|-- README.md                       # Contains the instructions to run the cluster
</pre>


### Commands to run

```bash
kubectl apply -f postgres-namespace.yml

kubectl apply -f postgres-master-configmap.yml
kubectl apply -f postgres-master-statefulset.yml
kubectl apply -f postgres-master-svc.yml


# Wait for the master to be up and running
# This might take a couple of minutes when running for the first time
kubectl get pods -n postgres-ns

# Once the master is up and running, run the below commands
kubectl apply -f postgres-slave-configmap.yml
kubectl apply -f postgres-slave-statefulset.yml
kubectl apply -f postgres-slave-svc.yml

# Sometimes the slave might not be up and running. Run the below command to check the status of the pods
# It might go into Error or CrashLoopBackOff state. It will automatically restart and come back to Running state
kubectl get pods -n postgres-ns
```

### Check the status of the cluster
```bash
# This should return atleast one row
kubectl exec -it postgres-master-0 -n postgres-ns -- psql -U postgres -d postgres -c "select * from pg_stat_replication;"

# Create a table in master and check if it is replicated to slave
kubectl exec -it postgres-master-0 -n postgres-ns -- psql -U postgres -d postgres -c "CREATE TABLE test (id serial PRIMARY KEY,name varchar(255) NOT NULL,age int NOT NULL);"

# insert some data
kubectl exec -it postgres-master-0 -n postgres-ns -- psql -U postgres -d postgres -c "INSERT INTO test (name, age) VALUES ('test', 10);"

# Check if the data is replicated to slave
kubectl exec -it postgres-slave-0 -n postgres-ns -- psql -U postgres -d postgres -c "SELECT * FROM test;"
kubectl exec -it postgres-slave-1 -n postgres-ns -- psql -U postgres -d postgres -c "SELECT * FROM test;"

```

### Check the logs of the cluster
```bash
kubectl logs -f postgres-master-0 -n postgres-ns
kubectl logs -f postgres-slave-0 -n postgres-ns
kubectl logs -f postgres-slave-1 -n postgres-ns
```

### Connect to the postgres cluster
```bash
kubectl exec -it postgres-master-0 -n postgres-ns -- psql -U postgres
kubectl exec -it postgres-slave-0 -n postgre-ns -- psql -U postgres
kubectl exec -it postgres-slave-1 -n postgres-ns -- psql -U postgres
```
OR
```
# Expose the service and connect to it

# Master
minikube service postgres-master-service -n postgres-ns --url

# Slave
minikube service postgres-slave-service -n postgres-ns --url

# Connect to the url using psql, password is 'randompassword'. Same is mentioned under key POSTGRES_PASSWORD in postgres-master-statefulset.yml
psql -h <url> -p <port> -U postgres
```


### Delete the cluster
```bash
kubectl delete ns postgres-ns
```

### Caveats

- Passwords are hardcoded in the configmap. This is not a good practice. In real world, we should use secrets to store the passwords.
- Services are exposed using ClusterIP. Not using headless service intentionally. This is to be able to load balance the requests to the slave.