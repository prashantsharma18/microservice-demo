## Step-01: Provision Kubernetes cluster

## Step-02: Download Istio
1. Go to the Istio release page to download the installation file for your OS, or download and extract the latest release automatically (Linux or macOS):
    curl -L https://istio.io/downloadIstio | sh -

You can also use specific version or override the processor architecture using below parameters:
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.14.0 TARGET_ARCH=x86_64 sh -

2. Move to the Istio package directory. For example, if the package is istio-1.14.0:
    cd istio-1.14.0

    Add the istioctl client to your path (Linux or macOS):
    export PATH=$PWD/bin:$PATH

## Step-03: Install Istio
1. For this installation, we use the demo configuration profile. It’s selected to have a good set of defaults for testing.
    istioctl install --set profile=demo -y

    There are other profiles that you can choose from https://istio.io/latest/docs/setup/additional-setup/config-profiles/

2. Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later
    kubectl label namespace default istio-injection=enabled

## Step-03: Deploy sample application (using manifest files)
1. Move to the directory where application manifests are present and apply it in the environment
   cd release
   kubectl apply -f release/kubernetes-manifests.yaml

   Fire below commands to see the services are installed properly and pods are in Running state
   kubectl get services
   kubectl get pods


## Step-04: Open the application to outside traffic
The application is deployed but not accessible from the outside. To make it accessible, you need to create an Istio Ingress Gateway, which maps a path to a route at the edge of your mesh

1. Associate this application with the Istio gateway:
    kubectl apply -f release/istio-manifests.yaml

2. Ensure that there are no issues with the configuration:
    istioctl analyze
    ✔ No validation issues found when analyzing namespace: default.

3. Execute the following command to determine if your Kubernetes cluster is running in an environment that supports external load balancers:
    kubectl get svc istio-ingressgateway -n istio-system

    NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                      AGE
    istio-ingressgateway   LoadBalancer   172.21.109.129   130.211.10.121  80:31380/TCP,443:31390/TCP,31400:31400/TCP   17h

    If the EXTERNAL-IP value is set, your environment has an external load balancer that you can use for the ingress gateway. If the EXTERNAL-IP value is <none> (or perpetually <pending>), your environment does not provide an external load balancer for the ingress gateway. In this case, you can access the gateway using the service’s node port.

    Set the ingress IP and ports:
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

    In certain environments, the load balancer may be exposed using a host name, instead of an IP address so use below command.
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

    Set gateway url:
    export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

    Ensure an IP address and port were successfully assigned to the environment variable:
    echo "$GATEWAY_URL"
    192.168.99.100:32194

    Paste the output from the previous command into your web browser and confirm that the application page is displayed


## Step-05: Install Kiali, grafana, Prometheus & Jaeger
1. Go to istio-1.14.0 folder which we downloaded in Step 02. Install Kiali and the other addons and wait for them to be deployed.
    $ kubectl apply -f samples/addons
    $ kubectl rollout status deployment/kiali -n istio-system
    Waiting for deployment "kiali" rollout to finish: 0 of 1 updated replicas are available...
    deployment "kiali" successfully rolled out

    If there are errors trying to install the addons, try running the command again. There may be some timing issues which will be resolved when the command is run again.

2. Access the Kiali dashboard. Below command opens the dashboard in the browser
    istioctl dashboard kiali

3. grafana & prometheus are on ClusterIP so they can't be accessed directly from you local machine (or internet). You should identify the pod and port numner    
   to use port-forwarding technique to open it from your local machine.

   Run below command which shows are all the services provisioned in instio-system namespace. Identify the port for grafana 
   kubectl get services -n istio-system

   Run below command which shows are all the pods provisioned in instio-system namespace. Identify the pod for grafana
   kubectl get pods -n istio-system

   Below command will perform port forwarding and you can access it from local machine on "localhost:3000"
   kubectl -n istio-system port-forward grafana-84ffc4bb9-ww8k5 3000:3000

4. Import Grafana dashboard which can provide kubernetes monitoring dashboard. 
    In grafana, click on plus symbol on left and select "Import"
    In "Import via grafana.com", input dashboard no "12740" (https://grafana.com/grafana/dashboards/12740-kubernetes-monitoring) and click Load
    In the prometheus datasource field select the existing datasource "Prometheus (default)

## Step-06: Install Chaos-mesh
1. Before installing Chaos Mesh, make sure that you have installed Helm in your environment.
    helm version

2. Add the Chaos Mesh repository to the Helm repository
    helm repo add chaos-mesh https://charts.chaos-mesh.org

3. Create the namespace to install Chaos Mesh
    kubectl create ns chaos-testing

4. Install Chaos Mesh
    helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-testing --version 2.2.0

5. Verify Installation. You should see below pods running
    kubectl get po -n chaos-testing

    NAME                                        READY   STATUS    RESTARTS   AGE
    chaos-controller-manager-69fd5c46c8-xlqpc   3/3     Running   0          2d5h
    chaos-daemon-jb8xh                          1/1     Running   0          2d5h
    chaos-dashboard-98c4c5f97-tx5ds             1/1     Running   0          2d5h

   


6. Port forwading to access Chaos Mesh dashboard. By default Chaos Mesh is installed on Nodeport which is not publicly accessible. 
    Identify the pod and the port using below commands.
    kubectl get services -n chaos-testing
    kubectl get pods -n chaos-testing

    Below command will perform port forwarding and you can access it from local machine on "localhost:2333"
    kubectl -n chaos-testing port-forward chaos-dashboard-84ffc4bb9-wmkdp 2333:2333

7. Create a admin user and cluster role binding
    kubectl apply -f ./AdminCreation

    Get the secret token to login
    kubectl describe secrets account-cluster-manager-hcfts