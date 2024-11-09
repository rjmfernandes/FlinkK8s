# Flink K8s Operator

- [Flink K8s Operator](#flink-k8s-operator)
  - [Disclaimer](#disclaimer)
  - [References](#references)
  - [Quickstart](#quickstart)
  - [S3 Backed](#s3-backed)
    - [Start Ingress Ready Cluster](#start-ingress-ready-cluster)
    - [Deploy MinIO](#deploy-minio)
    - [Deploy S3 backed Flink with basic job](#deploy-s3-backed-flink-with-basic-job)
  - [SQL Runner](#sql-runner)
  - [Session Deployment](#session-deployment)
    - [Flink SQL Shell](#flink-sql-shell)
    - [Jar Based Job](#jar-based-job)
  - [Persisted Catalogs](#persisted-catalogs)
  - [Cleanup](#cleanup)

## Disclaimer

The code and/or instructions here available are **NOT** intended for production usage. 
It's only meant to serve as an example or reference and does not replace the need to follow actual and official documentation of referenced products.

## References

- https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/try-flink-kubernetes-operator/quick-start/
- https://diogodssantos.medium.com/unlocking-the-power-of-flink-with-kubernetes-operator-simplify-data-management-for-daas-cf0fe0c1485b
- https://github.com/apache/flink-kubernetes-operator/tree/main/examples/flink-sql-runner-example
- https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/custom-resource/overview/ 
- https://github.com/decodableco/examples/tree/main/catalogs/flink-iceberg-hive
- https://github.com/criccomini/hive-metastore-standalone 

## Quickstart

```shell
kind create cluster
```

Install the certificate manager on your Kubernetes cluster to enable adding the webhook component (only needed once per Kubernetes cluster):

```shell
kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

Let's install the flink-kubernetes-operator:

```shell
helm repo add flink-kubernetes-operator https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.9.0/

helm install flink-kubernetes-operator flink-kubernetes-operator/flink-kubernetes-operator
```

Check everything is ready:

```shell
kubectl get pods
```

Let's now deploy the most basic flink setup:

```shell
kubectl create -f https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.8/examples/basic.yaml
```

Check everything is ready:

```shell
kubectl get pods
```

You can check logs:

```shell
kubectl logs -f deploy/basic-example
```

Now if we forward the jobmanager port:

```shell
kubectl port-forward svc/basic-example-rest 8081
```

 We can check our job running (part of the local jar `/opt/flink/examples/streaming/StateMachineExample.jar` defined as job on our yaml https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.8/examples/basic.yaml ):

 http://localhost:8081/#/job/running

Clean up:

```shell
kind delete cluster
```

## S3 Backed 

Let's start again our cluster but now we will use S3 (MinIO) backed storage.

### Start Ingress Ready Cluster

```shell
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

### Deploy MinIO

Let's deploy minio in its namespace minio-dev:

```shell
kubectl create ns minio-dev
kubectl -n minio-dev create -f ./minio.yaml
```

Check everything is ready:

```shell
kubectl get pods -n minio-dev
```

Let's define the rest service for minio and install ingress:

```shell
kubectl -n minio-dev create -f ./minio-rest.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

We wait for ingress to be ready:

```shell
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

We can now add ingress to expose the web dashboard:

```shell
kubectl -n minio-dev create -f ./minio-web.yaml
```

We can now login in MinIO (user: `minioadmin` password: `minioadmin`):

http://localhost/browser

**Create a bucket named `test`.**

### Deploy S3 backed Flink with basic job 

Now we create specific namespaces for our Flink Operator and the jobs and install the flink operator:

```shell
kubectl create ns flink-operator
kubectl create ns flink-jobs
helm repo add flink-kubernetes-operator https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.9.0/
helm -n flink-operator install -f values.yaml flink-kubernetes-operator flink-kubernetes-operator/flink-kubernetes-operator --set webhook.create=false
```

We confirm everything is ready:

```shell
kubectl -n flink-operator get all
```

And we  deploy Flink:

```shell
kubectl create -f ./flink-job.yaml
```

Checking if job manager and task manager are both ready:

```shell
kubectl -n flink-jobs get pods
```

Tail the log like this replacing the pod name accordingly as per your case for job manager:

```shell
kubectl logs -f -n flink-jobs basic-example-1-7cb867698-tq2gn
```

You can check the checkpoints by navigating in the minio dashboard (http://localhost/browser).

And we can visit the Flink dashboard and check our job running: http://localhost/flink-jobs/basic-example-1/#/job/running (ingress url access set as per template defined)

Clean up:

```shell
kind delete cluster
```

## SQL Runner

**Start Ingress Ready Cluster** as before.

**Deploy MinIO** also as before.

Now we will build a flink image containing a jar capable of running the Flink SQL script.

We will use the `flink-sql-runner-example` example from the Apache Flink project and compile it first:

```shell
git clone https://github.com/apache/flink-kubernetes-operator/
cd flink-kubernetes-operator/examples/flink-sql-runner-example/
mvn clean package
```

At this point you could define whatever sql scripts needed inside the folder `flink-kubernetes-operator/examples/flink-sql-runner-example/sql-scripts` and change later the yaml to specify your sql file to be executed. But you would need to recompile it again, of course.

Now we can build our local docker image:

```shell
DOCKER_BUILDKIT=1 docker build . -t flink-sql-runner-example:latest
```

And after that return to our project root:

```shell
cd ../../..
```

Now we install the flink operator but first we rename the cloned repository for not conflicting with it (as before we also create namespaces for both the operator and our job deployments):

```shell
rm -fr cloned-flink-kubernetes-operator
mv flink-kubernetes-operator cloned-flink-kubernetes-operator
helm repo add flink-kubernetes-operator https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.9.0/
kubectl create ns flink-operator
kubectl create ns flink-jobs
helm -n flink-operator install -f values.yaml flink-kubernetes-operator flink-kubernetes-operator/flink-kubernetes-operator --set webhook.create=false
```

Check everything is ready:

```shell
kubectl -n flink-operator get all
```

Now for deploying our Flink sql runner we first load our local image into kind:

```shell
kind load docker-image flink-sql-runner-example:latest
```

Now we can deploy: 

```shell
kubectl create -f ./flink-sql-job.yaml
```

Note that on our yaml we are specifying `sql-scripts/simple.sql` from the example project to be executed. Containing:

```sql
CREATE TABLE orders (
  order_number BIGINT,
  price        DECIMAL(32,2),
  buyer        ROW<first_name STRING, last_name STRING>,
  order_time   TIMESTAMP(3)
) WITH (
  'connector' = 'datagen'
);

CREATE TABLE print_table WITH ('connector' = 'print')
  LIKE orders;

INSERT INTO print_table SELECT * FROM orders;
```

And wait for pods to be ready:

```shell
kubectl -n flink-jobs get pods
```

As before you can check the checkpoints by navigating in the minio dashboard.

You can access the flink dashboard at http://localhost/flink-jobs/basic-example-1/#/overview

You can also check the print outputs of `simple.sql` by checking the log of task manager:

```shell
kubectl logs -f -n flink-jobs basic-example-1-taskmanager-1-1
```

Clean up:

```shell
kind delete cluster
```

## Session Deployment

As before:
- **Start Ingress Ready Cluster** as before.
- **Deploy MinIO** also as before.

We will again create the namespaces and install the operator:

```shell
kubectl create ns flink-operator
kubectl create ns flink-jobs
helm repo add flink-kubernetes-operator https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.9.0/
helm -n flink-operator install -f values.yaml flink-kubernetes-operator flink-kubernetes-operator/flink-kubernetes-operator --set webhook.create=false
```

We confirm everything is ready:

```shell
kubectl -n flink-operator get all
```

And we can install our session cluster (we are not using any locally built docker image now):

```shell
kubectl create -f ./flink-session.yaml
```

Now we are not installing any job together with our cluster. And in here we are expliciting using `mode: standalone`.

Let's confirm everything is ready:

```shell
kubectl -n flink-jobs get pods
```

We should again be able to access Flink dashboard at http://localhost/flink-jobs/basic-example-1/#/overview 

### Flink SQL Shell

Use the pod name for the job manager from last command to open a shell into the pod with something similar to:

```shell
k exec -n flink-jobs -it basic-example-1-57bc4d98d7-bvd9b -- bash
```

And on the shell open the Flink SQL shell:

```shell
./bin/sql-client.sh
```

Now you can execute inside the Flink SQL shell client: 

```sql
CREATE TABLE orders (
  order_number BIGINT,
  price        DECIMAL(32,2),
  buyer        ROW<first_name STRING, last_name STRING>,
  order_time   TIMESTAMP(3)
) WITH (
  'connector' = 'datagen'
);
```

```sql
CREATE TABLE print_table WITH ('connector' = 'print')
  LIKE orders;
```

```sql
INSERT INTO print_table SELECT * FROM orders;
```

List the tables on your default catalog/database;

```sql
SHOW TABLES;
```

Open in another terminal following the same steps another Flink SQL shell in the pod and execute again:

```sql
SHOW TABLES;
```

As you see the tables created by us on first sql session are not available in another session since they only exist in memory for that specific session.

Check the job executing in the Flink dashboard: http://localhost/flink-jobs/basic-example-1/#/job/running

### Jar Based Job

You can also use the Flink dashboard to `Submit New Job`.

First let's copy the same original example jar from inside out job manager pod (replace the pod name biy the one applicable in your case):

```shell
kubectl cp -n flink-jobs basic-example-1-57bc4d98d7-9l5ps:/opt/flink/examples/streaming/StateMachineExample.jar ./StateMachineExample.jar
```

Considering the upload of the jar we will use the port-forward so we start by:

```shell
kubectl port-forward svc/basic-example-1-rest 8081 -n flink-jobs
```

Now use this `StateMachineExample.jar` when submitting the job on `Submit New Job` > `Add New` from the Flink dashboard now at http://localhost:8081/#/submit

Once uploaded we click on the jar and hit `Submit`.

Under `Running Jobs` in the dashboard you should see now the 2 jobs running: one from the SQL shell and the other from the jar.

## Persisted Catalogs

Again:
- **Start Ingress Ready Cluster** as before.
- **Deploy MinIO** also as before.

Now we build our docker image for Hive Metastore:

```shell
cd hms-s3
DOCKER_BUILDKIT=1 docker build . -t hms-s3:latest
cd ..
```

Now for deploying our custom standalone HMS we first load our local image into kind:

```shell
kind load docker-image hms-s3:latest
```

Let's create the namespace and deploy HMS:

```shell
kubectl create ns hms-dev
kubectl -n hms-dev create -f ./hms.yaml
```

Check everything is ready:

```shell
kubectl get pods -n hms-dev
```

Let's define the thrift service for hms:

```shell
kubectl -n hms-dev create -f ./hms-rest.yaml
```

Now we do basically the same to our hive enabled custom flink image:

```shell
cd flink
DOCKER_BUILDKIT=1 docker build . -t flink-hive-s3-custom:latest
cd ..
```

And load our local image into kind:

```shell
kind load docker-image flink-hive-s3-custom:latest
```

As before for Flink Operator:

```shell
kubectl create ns flink-operator
kubectl create ns flink-jobs
helm repo add flink-kubernetes-operator https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.9.0/
helm -n flink-operator install -f values.yaml flink-kubernetes-operator flink-kubernetes-operator/flink-kubernetes-operator --set webhook.create=false
```

Confirm everything is ready:

```shell
kubectl -n flink-operator get all
```

Install our session cluster backed by hms for catalogs:

```shell
kubectl create -f ./flink-session-persisted.yaml
```

Let's check is ready:

```shell
kubectl -n flink-jobs get pods
```

Using the pod name as before to open a shell:

```shell
k exec -n flink-jobs -it basic-example-1-57bc4d98d7-bvd9b -- bash
```

And after open the Flink SQL shell:

```shell
./bin/sql-client.sh
```

And now we execute:

```sql
CREATE CATALOG myhive WITH (
   'type' = 'hive',
   'hive-conf-dir' = './lib'
 );
```

```sql
USE CATALOG myhive;
```

```sql
CREATE TABLE orders (
  order_number BIGINT,
  price        DECIMAL(32,2),
  buyer        ROW<first_name STRING, last_name STRING>,
  order_time   TIMESTAMP(3)
) WITH (
  'connector' = 'datagen'
);
```

```sql
CREATE TABLE print_table WITH ('connector' = 'print')
  LIKE orders;
```

```sql
SHOW TABLES;
```

Now we can open another FLINK SQL shell in another terminal and again:

```sql
CREATE CATALOG myhive WITH (
   'type' = 'hive',
   'hive-conf-dir' = './lib'
 );
```

```sql
USE CATALOG myhive;
```

```sql
SHOW TABLES;
```

As one can see now for the hive catalog we can see table definition are persisted and available for anyone connecting to same Flink catalog.

We can in fact execute form this terminal the statement:

```sql
SELECT * FROM orders;
```

And check the running job at: http://localhost/flink-jobs/basic-example-1/#/job/running

## Cleanup

```shell
rm -fr cloned-flink-kubernetes-operator
rm -fr flink-kubernetes-operator
rm StateMachineExample.jar
kind delete cluster
```