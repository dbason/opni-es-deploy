# opni-es-deploy

## Prepare receiving cluster
1) (Optional - Ignore this if using a LoadBalancer service) Install the [NGINX ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/).  For example to deploy on an EKS cluster use the following command:
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/aws/deploy.yaml
    ```
1) (Optional) [Install cert-manager](https://cert-manager.io/docs/installation/kubernetes/)
    1) E.g to install with Helm:
        ```sh
        kubectl create namespace cert-manager
        helm repo add jetstack https://charts.jetstack.io
        helm repo update
        kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.crds.yaml
        helm install \
            cert-manager jetstack/cert-manager \
            --namespace cert-manager \
            --version v1.3.1
        ```
    1) Create a cluster-issuer for cert-manager
        ```sh
        kubectl apply -f examples/cert-manager/cluster-issuer.yaml
        ```
1) Install opendistro-es from the [helm chart](https://github.com/opendistro-for-elasticsearch/opendistro-build/tree/main/helm)
    1) Edit the file in `examples/opendistro-es/values.yaml`.  Uncomment either the LoadBalancer service or ingress blocks for the kibana and elasticsearch.client sections.  If you are using an ingress you will also need to change the hostnames for the kibana and elasticsearch client endpoints.  You will need to have DNS entries pointing those hostnames to the cluster IP addresses.
    1) Checkout the opendistro code and package the chart
        ```sh
        cd ..
        git clone https://github.com/opendistro-for-elasticsearch/opendistro-build.git
        helm package opendistro-build/helm/opendistro-es
        ```
    1) Create a release of the opendistro-es chart
        ```sh
        kubectl create namespace opendistro-es
        helm install \
            opendistro-es opendistro-es-1.13.2.tgz \
            --namespace opendistro-es \
            -f opni-es-deploy/examples/opendistro-es/values.yaml
        ```
        N.B. The included values will create an opendistro deploy with 1 master, 2 workers, and 1 ingres/client node.  All of the nodes have dedicated memory that is double the Java settings.  The data nodes have 4Gi of memory (2g in the Java settings) and 25Gi of allocated storage.
        The values also create ingresses for Kibana and the ingest node.
    1) If you are using LoadBalancer services you will need to get the hostname for the elasticsearch cluster to ship logs to.
        ```sh
        kubectl get svc -n opendistro-es opendistro-es-client-service --output jsonpath='{.status.loadBalancer.ingress[0].hostname}'
        ```

## Set up log shipping in the simulation cluster
1) Install the Rancher logging application in the cluster you want to ship logs
1) Create a secret with the opendistro-es password in it
    ```sh
    kubectl apply -f examples/logging/es-password.yaml
    ```
1) Modify the examples/logging/cluster-output.yaml file to point to the Elasticsearch address.  You will need to modify the spec.elasticsearch.host field.
1) Create a ClusterOutput that points to the hostname of the ingress for the ingest node
    ```sh
    kubectl apply -f examples/logging/cluster-output.yaml
    ```
    N.B. This adds a file buffer to the fluentd Elasticsearch output for resiliency.  It also adds some additional useful options.
1) You can now connect a [Flow/ClusterFlow](https://banzaicloud.com/docs/one-eye/logging-operator/configuration/flow/) to the output you have created