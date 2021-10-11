# confluent-for-k8-hybrid-monitoring
This repo is to deploy a self-managed connect cluster, KSQLDB and control center backed confluent cloud and configuring centralized monitoring.

Please note that this not production ready repo, pick up the assets (grafana dashboards, prometheus config) and integrate with your k8 deployment.

1.	Create namespace “confluent” in kubernetes cluster ( all the yaml files in this repo using “confluent” namespace for deployment)
              
     `kubectl create namespace confluent`
2.	Set “confluent” namespace as default-context to run the deployment

    `kubectl config set-context --current  --namespace=confluent`

3.	Set the confluent-hybrid director for the directory you downloaded the repo

   `export CONFLUENT_HYBRID=<downloaded directory>/ccloud-integration`

4.	Set up the Helm Chart

    `helm repo add confluentinc https://packages.confluent.io/helm`
    
5.	Install Confluent For Kubernetes using Helm:

   `helm upgrade --install operator confluentinc/confluent-for-kubernetes`
   
6.	Check that the Confluent For Kubernetes pod comes up and is running:

kubectl get pods
7.	Generate a CA pair to use in this tutorial(this tutorial uses self-signed certificates):
`openssl genrsa -out $CONFLUENT_HYBRID/ca-key.pem 2048
openssl req -new -key $CONFLUENT_HYBRID/ca-key.pem -x509 \
  -days 1000 \
  -out $CONFLUENT_HYBRID/ca.pem \
  -subj "/C=US/ST=CA/L=MountainView/O=Confluent/OU=Operator/CN=TestCA"`
  
8.	Create a Kubernetes secret for inter-component TLS:

`kubectl create secret tls ca-pair-sslcerts \
  --cert=$CONFLUENT_HYBRID/ca.pem \
  --key=$CONFLUENT_HYBRID/ca-key.pem`
  
9.	Provide authentication credentials
Confluent Cloud provides you an API key for both Kafka and Schema Registry. Configure Confluent For Kubernetes to use the API key when setting up Connect and ksqlDB to connect.
Create a Kubernetes secret object for Confluent Cloud Kafka access. This secret object contains file based properties. These files are in the format that each respective Confluent component requires for authentication credentials.
`kubectl create secret generic cloud-plain \
--from-file=plain.txt=$CONFLUENT_HYBRID/creds-client-kafka-sasl-user.txt
kubectl create secret generic cloud-sr-access \
--from-file=basic.txt=$CONFLUENT_HYBRID/creds-schemaRegistry-user.txt`

`kubectl create secret generic control-center-user \
--from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt`

10.	Deploy confluent platform
Edit the confluent-platform.yaml deployment YAML, and add your respective Confluent Cloud URLs in the following places:
•	<cloudSR_url>
•	<cloudKafka_url>
11.	 Validate
 Set up port forwarding to Control Center web UI from local machine:

 `kubectl port-forward controlcenter-0 9021:9021`

Browse to Control Center and log in as the admin user with the Developer1 password:

https://localhost:9021

## Monitoring
Confluent cloud cluster metrics set up:

1.	Create secret file for ccloud metrics api
    
    'export CCLOUD_CLUSTER=<<confluent cloud cluster name>>
     export CCLOUD_API_KEY=<<confluent cloud api key>>
     export CCLOUD_API_SECRET=<<confluent cloud api secret>>'
     
 Note: Service account associated with APIKey should have metrics viewer role to pull the metrics from confluent cloud. Refer this to grant metrics viewer role access.
 
 Run the below command to create ccloud exporter secret file

 `kubectl create secret generic $CCLOUD_CLUSTER-vars --from-literal=CCLOUD_API_KEY=${CCLOUD_API_KEY} --from-literal=CCLOUD_API_SECRET=${CCLOUD_API_SECRET} --from-literal=CCLOUD_CLUSTER=${CCLOUD_CLUSTER}`


2.	Replace ccloud cluster id in deployment yaml file

    `sed s/%CCLOUD_CLUSTER%/$CCLOUD_CLUSTER/g \
    $CONFLUENT_HYBRID/monitoring/ccloud_exporter.yaml > $CONFLUENT_HYBRID/monitoring/ccloud_exporter-$CCLOUD_CLUSTER.yaml`

3.	Deploy ccloud_exporter

    `kubectl apply -f $CONFLUENT_HYBRID/monitoring/ccloud_exporter-$CCLOUD_CLUSTER.yaml`
    
kafka-lag exporter setup(monitoring consumer lag):

kafka lag exporter is an open source utility to monitor the consumer lag and publish metrics in open metrics format(promethues specific).

1.  Update the $CONFLUENT_HYBRID/monitoring/config/application.conf with confluent cloud bootstrap url, api key and secret (refer to the $CONFLUENT_HYBRID/monitoring/config/samples/application_sample.conf)
2.  Create configmap with monitoring configuration

    `kubectl create configmap monitor-config --from-file=$CONFLUENT_HYBRID/monitoring/config`

3.  Deploy kafka lag exporter

    `kubectl apply -f $CONFLUENT_HYBRID/monitoring/kafkalag.yaml`

Deploy node exporter( monitor node level metrics):

   `kubectl apply -f $CONFLUENT_HYBRID/monitoring/node-exporter.yaml`
   

Prometheus and Grafana set up:
   
1.  Create configmaps with Prometheus and Grafana dashboard files (data source and dashboards)

    `kubectl create configmap grafana-datasource --from-file=$CONFLUENT_HYBRID/monitoring/config/grafana/datasources

     kubectl create configmap dashboard-config --from-file=$CONFLUENT_HYBRID/monitoring/config/grafana/dashboards`

2.  Deploy Prometheus and Grafana

    `kubectl apply -f $CONFLUENT_HYBRID/monitoring/prometheus-grafana.yaml`

3. Grafana port forward to localhost

   `kubectl get pods`

Take the pod name starts with “prometheus-grafana-dp” and apply in the below command

   `kubectl port-forward <<Prometheus Grafana pod>> 3000:3000`

Access Grafana from your machine

http://localhost:3000

user:admin

password: admin


## Self-managed conenctors
![connectors monitoring_dashboards](/images/connector.jpg)

## Self-managed KSQLDB

![ksqldb Monitoring Dashboard](/images/ksqldb.jpg)

## Confluent cloud

![ccloud_monitoring_dashboard](/images/ccloud.jpg)


## Kafka consumer lag

![Kafka consumer lag_Dashboard](/images/kafka-lag.png)
