# Deploy and scale an XAF Blazor Server app to serve hundreds of users with Azure Kubernetes Service

Follow the instruction in this example to deploy an XAF Blazor application to a Kubernetes cluster with horizontal autoscaling. We tested the application in two types of clusters: locally-run [K3s](https://k3s.io/) and [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service/). The maximum pod replica number (20) allowed around 300 concurrent users. An AKS cluster needs two nodes (B4ms machines: 4 Cores, 16 GB RAM) to operate with such a number of pod replicas and the same load.

This repository contains the following useful resources: 

* a `Dockerfile` that helps you publish an app to a Linux container, and a version for a Windows container (`Dockerfile.win`)
* *.yaml files that help you deploy an app to a Kubernetes cluster with a Microsoft SQL Server database engine container, Horizontal Pod Autoscaler, and Ingress 

The following diagram illustrates the cluster architecture:

![Cluster diagram](/images/cluster-diagram.png)

## Get Started

### 1. Install Docker Engine or Docker Desktop

Visit [docker.com](https://www.docker.com/) for downloads and additional information.

### 2. Clone this repository

### 3. Build a Docker image 

Use [BuildKit](https://docs.docker.com/develop/develop-images/build_enhancements/#new-docker-build-secret-information) to build a Docker image. The `--secret` flag helps you safely pass a NuGet source URL:

```
DOCKER_BUILDKIT=1 docker build -t your_docker_hub_id/xaf-container-example --secret id=dxnuget,src=<( echo your_devexpress_nuget_source_url ) .
```

The following command runs a container with the image you built:

```
docker run --network="host" -e CONNECTION_STRING=MSSQLConnectionString your_docker_hub_id/xaf-container-example:latest
```

**Note**: Example in this repository requires that you pass a CONNECTION_STRING environment variable. This variable specifies the connection string name (defined in `appsetting.json`) to be used in the container.

If your XAF Blazor application's database is live and doesn't require updates, then the application is ready for use at `http://localhost/`. If the database doesn't exist or requires an update based on your latest data model and XAF modules, then you will see a database version mismatch error in the console. To resolve the error, force a database update. Launch another application instance in the running container. Run the following command to find the container's ID first:

```
docker ps
```

Once you obtain the container ID, execute the following command to force the update:

```
docker exec your_container_id dotnet XAFContainerExample.Blazor.Server.dll --updateDatabase --forceUpdate --silent
```

### 4. Store the image in the Docket Hub 

Log in with your Docker credentials: 

```
docker login
```

Push the image to your Docker Hub:

```
docker push your_docker_hub_id/xaf-container-example:latest
```

**Note**: You can pull already-built docker images from your own Docker hub. To accomplish this, change image names in the following files: `app-depl.yaml` and `docker-compose.yml` (optional). 

### 5. (Optional) Use Docker Compose to run a multi-container application 

You can use [Docker Compose](https://docs.docker.com/compose/) to run a multi-container application. The `docker-compose.yml` file describes how the application and Microsoft SQL Server containers interact. Run the following command to launch:

```
docker-compose up
```

The application should be available at `http://localhost/`.

### 6. Run a terminal 

Open a terminal on the machine that runs Kubernetes. You can use any Kubernetes version: Azure Kubernetes Service (AKS), Google Kubernetes Engine (GKE), locally installed lightweight [K3s](https://k3s.io/), [minikube](https://minikube.sigs.k8s.io/docs/), or others. As this article already mentioned, we tested this example with AKS and locally-installed K3s.

### 7. Create a storage for the database 

Apply a [Persistent Volume Claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) definition to create a storage for the database:

```
kubectl apply -f ./K8S/local-pvc.yaml
```

### 8. Deploy the database

Store the database password into a secret:

```
kubectl create secret generic mssql --from-literal=SA_PASSWORD="Qwerty1_"
```

Apply the manifest: create a Microsoft SQL Server deployment with its ClusterIP Service.

```
kubectl apply -f ./K8S/mssql-app-depl.yaml
```

### 9. Deploy the application

Create an application deployment with its ClusterIP Service. 

Open the [app-depl.yaml](/K8S/app-depl.yaml) file and change the `devexpress` Docker Hub id to yours (or leave it as is to pull the image from the DevExpress repository). Apply the deployment manifest:

```
kubectl apply -f ./K8S/app-depl.yaml
```

**Note**: To update the database, you can use the following technique. First, find a pod with the running application:

```
kubectl get pods
```

The command output may look like this:

```
NAME                         READY   STATUS    RESTARTS     AGE
app-depl-f487bdcfd-mxnrz     1/1     Running   0            75m
mssql-depl-c47fdc8c7-5x5m7   1/1     Running   1 (2s ago)   5s
```

In the example above, the application pod's name is `app-depl-f487bdcfd-mxnrz`. Use this name to run another application instance in database update mode:

```
kubectl exec -it %pod_name% -- dotnet XAFContainerExample.Blazor.Server.dll --updateDatabase --forceUpdate --silent
```

### 10. Configure Ingress 

In this step, you will accomplish the following: 

* Configure [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to make the application accessible from outside the cluster 
* Set up [Sticky Sessions](https://docs.microsoft.com/en-us/aspnet/core/blazor/host-and-deploy/server?view=aspnetcore-6.0#kubernetes)

Before you proceed, install Ingress NGINX Controller if you haven't done so already. For example, visit the following URL for K3s setup instructions: https://docs.rancherdesktop.io/how-to-guides/setup-NGINX-Ingress-Controller/. 

Apply the Ingress definition:

```
kubectl apply -f ./K8S/ingress-srv.yaml
```

Wait for a couple of minutes and check that the application is accessible from outside the cluster:

```
kubectl get ingress
```

The output may look like this:

```
NAME          CLASS   HOSTS   ADDRESS       PORTS   AGE
ingress-srv   nginx   *       <your-ip>     80      5d21h
```

Try to open the starting page in the browser. Use the following URL: `http://<your-ip>/`.

### 11. Enable Horizontal Scaling 

The application is now running in a single [Pod](https://kubernetes.io/docs/concepts/workloads/pods/). To scale the app horizontally, you can use [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/):

```
kubectl apply -f ./K8S/app-hpa.yaml
```

You can now run the following command to see active pods and their CPU load:

```
kubectl get hpa
```

```
user@ubuntu-k8s:~/Work/xaf-blazor-app-load-testing-example$ kubectl get hpa
NAME      REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app-hpa   Deployment/app-depl   13%/50%   1         15        7          54m
```

## Implementation Details

### Build a Docker image for an XAF Blazor application 

This solution contains a `Dockerfile` example based on [microsoft-dotnet](https://hub.docker.com/_/microsoft-dotnet) images.

```
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
RUN --mount=type=secret,id=dxnuget dotnet nuget add source $(cat /run/secrets/dxnuget) -n devexpress-nuget
COPY ["XAFContainerExample.Blazor.Server/XAFContainerExample.Blazor.Server.csproj", "XAFContainerExample.Blazor.Server/"]
COPY ["XAFContainerExample.Module/XAFContainerExample.Module.csproj", "XAFContainerExample.Module/"]
RUN dotnet restore "XAFContainerExample.Blazor.Server/XAFContainerExample.Blazor.Server.csproj"
COPY . .
WORKDIR "/src/XAFContainerExample.Blazor.Server"
RUN dotnet build "XAFContainerExample.Blazor.Server.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "XAFContainerExample.Blazor.Server.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "XAFContainerExample.Blazor.Server.dll"]
```

You can also generate such a file in Visual Studio. Right-click the **YourApp.Blazor.Server** project and select **Add | Docker Support**. Note that you need to move the created `Dockerfile` up to the root solution folder.

![Docker support](/images/docker-support.png)

To restore NuGet packages correctly, pass the DevExpress NuGet source URL as a secret (see [BuildKit](https://docs.docker.com/develop/develop-images/build_enhancements/#new-docker-build-secret-information) documentation). 

Refer to [Docker reference](https://docs.docker.com/engine/reference/builder/) for additional information on command syntax.

### Run an XAF Blazor application in a Kubernetes cluster

This section describes all the specifications located in the `K8S` folder. These specifications are sufficient to deploy and run an XAF Blazor application with load balancing and autoscaling.

#### 1. Database deployment

Database deployment requires a storage. 

The [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) subsystem implements API that abstracts details of storage allocation from storage consumption. A **PersistentVolumeClaim** (PVC) is a request for storage. The sample below is a specification for a simple PVC ([local-pvc.yaml](/K8S/local-pvc.yaml)), sufficient for a locally-run cluster, such as [k3s](https://k3s.io/):

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mssql-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

The [mssql-app-depl.yaml](/K8S/mssql-app-depl.yaml) file contains specifications for database engine [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#writing-a-deployment-spec). The deployment procedure accomplishes two tasks: 

- Runs Microsoft SQL Server in a container
- Runs ClusterIP service to [expose an app](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/) on an internal IP in the cluster. (The database server is only reachable inside the cluster.)

#### 2. Application deployment

The [app-depl.yaml](/K8S/app-depl.yaml) file describes application deployment parameters, including a ClusterIP service:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xafcontainerexample
  template:
    metadata:
      labels:
        app: xafcontainerexample
    spec:
      containers:
      - name: xafcontainerexample
        image: devexpress/xaf-container-example:latest
        imagePullPolicy: Never
        env:
          - name: CONNECTION_STRING
            value: K8sMSSQLConnectionString
        resources:
          requests:
            cpu: 400m
            memory: 500Mi
          limits:
            cpu: 800m
            memory: 1Gi
```

The file specifies the pre-built image, additional environment variables (such as CONNECTION_STRING), and hardware resources that the cluster should reserve for this container.

#### 3. Ingress

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource. Visit the following webpage from Kubernetes documentation to learn more: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).  

Blazor Server applications use long-living WebSocket to communicate between browser and server. This means that you need to enable [Sticky Sessions](https://docs.microsoft.com/en-us/aspnet/core/blazor/host-and-deploy/server?view=aspnetcore-6.0#kubernetes) to maintain the connection to a Pod during the entire application run. 

The [ingress definition](/K8S/ingress-srv.yaml) example in this repository works with Kubernetes version 1.19 and later. The cluster must run an [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

#### 4. Horizontal Pod Autoscaler.

The [app-hpa.yaml](/K8S/app-hpa.yaml) manifest defines a HorizontalPodAutoscaler (HPA) that adjusts the number of running pod replicas according to the specified metrics.

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
 name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-depl
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

This example can scale pod replicas from 1 (`minReplicas`) up to 20 (`maxReplicas`) based on CPU utilization. Refer the [HPA documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details) to learn more.

### Use Docker Compose to run a multi-container application

If you don't want to scale the app automatically and set up Kubernetes, consider **Docker Compose**. 

The `docker-compose.yml` file contains definitions for two containers. The first uses the `xafcontainerexample` image described above. The second runs a Microsoft SQL Server and allows access to it from the first container.

```
version: "3.9"
services:
    web:
        image: "devexpress/xaf-container-example:latest"
        ports:
          - "80:80"
        environment:
          - CONNECTION_STRING=DockerMsSqlConnectionString
            
    db:
        image: "mcr.microsoft.com/mssql/server"
        environment:
            SA_PASSWORD: "Qwerty1_"
            ACCEPT_EULA: "Y"
        ports:
          - "1433:1433"
```

The application from the first contaier can use the following connection string to access the database:

```
Pooling=false;Data Source=db;Initial Catalog=XAFContainerExample;User Id=SA;Password=<your_strong_password>
```

Refer the [Compose specification](https://docs.docker.com/compose/compose-file/) webpage for better understanding of the Compose file format.

## FAQ / Troubleshooting and limitations

### 1. Will my own XAF Blazor app work with 100, 200, 300, or more concurrent users with the same performance and using the same hardware/software?
We neither provide a universal calculator for web server hardware/software requirements nor it is possible to give a universal answer without testing (and without misleading people). Everything depends on the complexity of a specific application and implemented behavior, which is different for each developer, application type, environment and even tested use-case scenarios. For instance, the number of persistent classes and their fields, Controllers, Application Model customizations, whether end-users export or import large amounts of data every minute, generate complex reports or do other processor and memory intensive operations, which directly affect the web server response time. For more information, review [XAF ASP.NET WebForms or Blazor Server UI for SaaS with 1000 users](https://supportcenter.devexpress.com/ticket/details/t585727/xaf-asp-net-webforms-or-blazor-server-ui-for-saas-with-1000-users) and [XAF ASP.NET Web Forms application deployment and load testing considerations](https://supportcenter.devexpress.com/ticket/details/s36497/xaf-asp-net-web-forms-application-deployment-and-load-testing-considerations).

Even with this horizontal scaling, we recommend that you carefully test your own and unique XAF Web apps under conditions close to your real production environment to measure the actual performance over time or based on user load. For this, you may be interested in our GitHub example - [XAF Blazor load testing on Linux and MySql using Puppeteer and GitHub Actions](https://github.com/DevExpress/xaf-blazor-app-load-testing-example).

### 2. Will you consult us if we experience configuration or deployment issues with Docker, Kubernetes, Windows or Linux-based servers, hosting providers, etc.?
I am afraid we cannot assist you on this, because this question (deployment) is not specific to DevExpress (XAF Blazor), and you can do whatever Microsoft ASP.NET Core Blazor Server can handle (refer to [Microsoft Docs](https://docs.microsoft.com/en-us/aspnet/core/blazor/host-and-deploy/server) | [Deployment Recommendations for XAF Blazor UI](https://docs.devexpress.com/eXpressAppFramework/403362/deployment/deployment-recommendations-blazor)).
We also we do not administer web servers, hosting environments for customers; we do not consult on various server and operating system configurations as part of our support services. For more information, please review the Prerequisites and Technical Support Scope sections at https://www.devexpress.com/products/net/application_framework/xaf-considerations-for-newcomers.xml. To troubleshoot issues, we recommend that you first make sure that this deployment scenario works for you without XAF with a pure Blazor Server app (with the same database and XPO or EF Core for data access) - XAF Blazor should work too once you get your non-XAF Blazor Server app up and running.

### 3. How do I avoid a "Connection Error" message in the web browser for some users when the HPA scales down replicas?

This problem is caused by a Sticky Session. The browser communicates with only one particular server all the time a page is open. When a pod replica is terminated, the connection is lost. You can implement a workaround - refresh the browser if the app loses server connection. You can find an example in the following file: [_Host.cshtml](/XAFContainerExample.Blazor.Server/Pages/_Host.cshtml). 

A Pod can also finish processes gracefully upon termination. You can set the [terminationGracePeriodSeconds](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#lifecycle) option (30 seconds by default) to specify a delay, in seconds, after the container receives the termination signal and before it forcibly halts the process. You may also consider [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) (such as `preStop`) to manage application instance state before pod termination.

### 4. How do I get Ingress working on K3s Kubernetes distribution (The application web page cannot be reached outside the cluster)?

Check the `ingress-nginx-controller` service:

```
kubectl get svc -n ingress-nginx
```

The output may look like this:

```
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller-admission   ClusterIP      10.43.168.32   <none>          443/TCP                      14m
ingress-nginx-controller             LoadBalancer   10.43.132.9    <pending>       80:31075/TCP,443:32734/TCP   14m
```

See if the `ingress-nginx-controller` service always displays 'pending' under **External IP**. If that is the case, you probably experience the following issue: https://github.com/rancher/k3os/issues/208. Try a workaround from the following comment: https://github.com/rancher/k3os/issues/208#issuecomment-599087377

```
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer", "externalIPs":["your-external-ip"]}}'
```

### 5. How do I build a Docker image for a Windows container (Docker BuildKit supports only Linux containers)?

If you need to build an image for a Windows container, use the following workaround to avoid passing DevExpress NuGet source insecurely. 

Build the application on the local machine and put the app into an image.

```
dotnet publish ./XAFContainerExample.Blazor.Server/XAFContainerExample.Blazor.Server.csproj -c Release -o ./app
```

Change the container type in the running Docker instance. Right-click the System Tray's Docker icon and choose **Switch to Windows containers...** To build an image with a custom Dockerfile name, use the `-f ` flag:

```
docker build -f Dockerfile.win -t <your_docker_hub_id>/xaf-container-example:win .
```

You cannot run the container on Windows in the same manner as described in the "Getting Started" section. The `--network="host"` mode is only supported on Docker for Linux. Use the [host.docker.internal](https://docs.docker.com/desktop/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host) as the hostname instead of `localhost`.

```
docker run -p 80:80 -e CONNECTION_STRING=DockerMSSQLConnectionString your_docker_hub_id/xaf-container-example:latest
``
