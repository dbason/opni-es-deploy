# opni-es-deploy

## Prepare receiving cluster
1) Install the [NGINX ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/).  For example to deploy on an EKS cluster use the following command:
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
    1) Checkout the opendistro code and package the chart
        ```sh
        cd ..
        git clone https://github.com/opendistro-for-elasticsearch/opendistro-build.git
        helm package opendistro-build/helm
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

## Set up log shipping
1) Install the Rancher logging application
1) Create a secret with the opendistro-es password in it
    ```sh
    kubectl apply -f examples/logging/es-password.yaml
    ```
1) Create a ClusterOutput that points to the hostname of the ingress for the ingest node
    ```sh
    kubectl apply -f examples/logging/cluster-output.yaml
    ```
    N.B. This adds a file buffer to the fluentd Elasticsearch output for resiliency.  It also adds some additional useful options.
1) You can now connect a [Flow/ClusterFlow](https://banzaicloud.com/docs/one-eye/logging-operator/configuration/flow/) to the output you have created