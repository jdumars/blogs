# Zero-downtime Deployment in Kubernetes with Jenkins

"How do I do zero-downtime deployment" is a frequently asked question ever since we added
the [Kubernetes Continuous Deploy](https://aka.ms/azjenkinsk8s) and
the [Azure Container Service](https://aka.ms/azjenkinsacs) plugins in the Jenkins update center.
So we created a quickstart template on Azure to demonstration how we can do that. 
Although it's based on Azure, the concept could be applied to general Kubernetes cluster.

## Rolling Update

Kubernetes supports `RollingUpdate` strategy that replaces old pods with new ones gradually while
serving the clients without downtime. To perform RollingUpdate deployment,
1. set `.spec.strategy.type` to `RollingUpdate`, which is the default value.
1. set `.spec.strategy.rollingUpdate.maxUnavailable` and `.spec.strategy.rollingUpdate.maxSurge` to some reasonable value.
   * `maxUnavailable`: the maximum number of Pods that can be unavailable during the update process. Can be absolute
      number or percentage of the `replicas` count; default is 25%.
   * `maxSurge`: the maximum number of Pods that can be created over the desired number of Pods. Can be absolute number
      or percentage of the `replicas` count; default is 25%.
1. configure the `readinessProbe` for your service container to help Kubernetes determine the state of the Pods.
   Kubernetes will only route the client traffic to the Pods with healthy liveness probe.

We use deployment of the official Tomcat image to demonstrate this:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: tomcat-deployment-rolling-update
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: tomcat
        role: rolling-update
    spec:
      containers:
      - name: tomcat-container
        image: tomcat:${TOMCAT_VERSION}
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 50%
```

If the Tomcat running in the current deployments are version 7, and we can replace `${TOMCAT_VERSION}` with 8 and apply
this to the Kubernetes cluster. With the [Kubernetes Continuous Deploy](https://aka.ms/azjenkinsk8s) or
the [Azure Container Service](https://aka.ms/azjenkinsacs) plugin, the value can be fetched from environment
variable which eases the deployment process.

Under the scene, Kubernetes manages the update like the following:

![Kubernetes Rolling Update](img/k8s-rolling.png)

1. Initially, all pods are running Tomcat 7 and the frontend Service routes the traffic to these pods.
1. During the rolling update, Kubernetes takes down some Tomcat 7 pods and create some new Tomcat 8 pods. It ensures:

   * at most `maxUnavailable` pods in the desired Pods can be unavailable, that is, at least 
      (`replicas` - `maxUnavailable`) pods should be serving the client traffic, which is `2-1=1` in our case.
   * at most `maxSurge` more pods can be created during the update process, that is `2*50%=1` in our case.

   One Tomcat 7 Pod is taken down, and one Tomcat 8 Pod is created. Kubernetes will not route the traffic to any of them
   because their readiness probe is not successful.
1. When the new Tomcat 8 Pod is ready as determined by the readiness probe, Kubernetes will start routing the traffic to it.
   This means during the update process, user may see both the old service and the new service.
1. The rolling updates continues by taking down Tomcat 7 Pods and create Tomcat 8 pods, and route the traffic to the ready
   Pods.
1. Finally, all Pods are on Tomcat 8.

Rolling Update ensures we always have some **Ready** backend Pods serving the client requests, so there's no service
downtime. However, some extra care is required:

1. During the update, both the old Pods and new Pods may serve the requests. Without well defined session affinity in the
   Service layer, a user may be routed to the new Pods and later back to the old Pods.

   This also requires us to maintain well defined forward & backward compatibility on both data and API, which is difficult
   generally.
1. It may take a long time before a Pod is ready for traffic after it is started. There may be a long time window where the
   traffic is served with less backend Pods than usual. Generally, this should not be a problem as we tend to do the production
   upgrade when the service is not busy. But this will also extend the time window for issue 1.
1. We cannot do comprehensive tests for the new Pods being created. Moving the products from dev / QA environment to production
   environment is always a big risk of breaking the current production functionality. The readiness probe can do some work to
   check the readiness, however, it should be a lightweight task that can be run periodically and not suitable to be used 
   as an entry point to start the complete tests.

## Blue/green Deployment

*Blue/green deployment quoted from [TechTarget](http://searchitoperations.techtarget.com/definition/blue-green-deployment)*
> A blue/green deployment is a change management strategy for releasing software code. Blue/green deployments,
> which may also be referred to as A/B deployments require two identical hardware environments that are 
> configured exactly the same way. While one environment is active and serving end users, the other environment remains idle.

Container technology offers stand-alone environment to run the desired service, which makes it super easy to create identical
environments as required in the blue/green deployment. The loosely coupled Services - ReplicaSets, and the label/selector based
service routing in Kubernetes makes it easy to switch between different backend environments. With these techniques, the blue/green
deployment in Kubernetes can be done as followed:

1. Before the deployment, the infrastructure is prepared as followed:
   1. Prepare the blue deployment and green deployment with `TOMCAT_VERSION=7` and `TARGET_ROLE` set to `blue` or
      `green` respectively.

      ```yaml
      apiVersion: extensions/v1beta1
      kind: Deployment
      metadata:
        name: tomcat-deployment-${TARGET_ROLE}
      spec:
        replicas: 2
        template:
          metadata:
            labels:
              app: tomcat
              role: ${TARGET_ROLE}
          spec:
            containers:
            - name: tomcat-container
              image: tomcat:${TOMCAT_VERSION}
              ports:
              - containerPort: 8080
              readinessProbe:
                httpGet:
                  path: /
                  port: 8080
      ```
   1. Prepare the public service endpoint, which initially routes to one of the backend environment, say `TARGET_ROLE=blue`.

      ```yaml
      kind: Service
      apiVersion: v1
      metadata:
        name: tomcat-service
        labels:
          app: tomcat
          role: ${TARGET_ROLE}
          env: prod
      spec:
        type: LoadBalancer
        selector:
          app: tomcat
          role: ${TARGET_ROLE}
        ports:
          - port: 80
            targetPort: 8080
      ```
   1. Optionally, prepare some test endpoint so that we can visit the backend environments for testing. They are
      similar to the public service endpoint, but they are intended to be accessed internally by the dev/ops team
      only.

      ```yaml
      kind: Service
      apiVersion: v1
      metadata:
        name: tomcat-test-${TARGET_ROLE}
        labels:
          app: tomcat
          role: test-${TARGET_ROLE}
      spec:
        type: LoadBalancer
        selector:
          app: tomcat
          role: ${TARGET_ROLE}
        ports:
          - port: 80
            targetPort: 8080
      ```
1. Update the application in the inactive environment, say `green` environment. Set `TARGET_ROLE=green` and 
   `TOMCAT_VERSION=8` in the deployment config to update the `green` environment.
1. Test the deployment via the `tomcat-test-green` test endpoint to ensure the `green` environment is ready to serve client
   traffic.
1. Switch the frontend Service routing to the `green` environment by updating the Service config with `TARGET_ROLE=green`.
1. Do some more test on the public endpoint to ensure it is working properly.
1. Now the `blue` environment is idle and we can:
   * leave it with the old application so that we can roll back if there's issue with the new application
   * update it to make it a hot backup of the active environment
   * reduce its replication count to save the occupied resources

![Kubernetes Blue/green](img/k8s-blue-green.png)

As compared to Rolling Update, the blue/green update

* Does not rely on the update strategy of a specific backend environment, either `RollingUpdate` or `Recreate` will do.
* The public service is either routed to the old applications, or new applications, but never both at the same time.
* The time it takes for the new Pods to be ready does not affect the public service quality, as the traffic will only be
   routed to the new Pods when all of them are tested to be ready.
* We can do comprehensive tests on the new environment before it serves the public traffic. Just keep in mind this is in
   production and the tests should not polute the applicaiton data.

## Jenkins Automation

Jenkins provides easy to setup workflow to automate your deployments. With the [Pipeline](https://jenkins.io/doc/book/pipeline/)
support, it is flexible to build the zero-downtime deployment workflow, and visualize the deployment steps.

To facilitate the deployment process of the Kubernetes resources, we published the [Kubernetes Continuous Deploy](https://aka.ms/azjenkinsk8s) and
the [Azure Container Service](https://aka.ms/azjenkinsacs) plugins built based on the [kubernetes-client](https://github.com/fabric8io/kubernetes-client).
You can deploy the resource to Azure Kubernetes Service (AKS) or the general Kubernetes cluster without the need of `kubectl`,
and it supports variable substitution in the resource config so that you can deploy environment specific resources to the clusters
without updating the resource config.

We created a Jenkins Pipeline to demonstrate the blue/green deployment to AKS. The flow is like the following:

![AKS Blue/green Deployment Pipeline](img/aks-blue-green-pipeline.png)

1. **Pre-clean**: clean workspace.
1. **SCM**: pulling code from the source control management system.
1. **Prepare Image**: prepare the application docker images and upload them to some Docker repository.
1. **Check Env**: determine the active & inactive environment, which drives the following deployment.
1. **Deploy**: deploy the new application resource configuration to the inactive environment. With the **Azure Container Service**
   plugin, this can be done with:

   ```groovy
   acsDeploy azureCredentialsId: 'stored-azure-credentials-id',
             configFilePaths: "glob/path/to/*/resource-config-*.yml",
             containerService: "aks-name | AKS",
             resourceGroupName: "resource-group-name",
             enableConfigSubstitution: true
   ```
1. **Verify Staged**: verify the deployment to the inactive environment to ensure it is working properly. Again, note this
   is in production environment, be aware not to pollute the application data during the tests.
1. **Confirm**: Optionally, send email notifications for manual user approval to proceed the actual environment switch.
1. **Switch**: Switch the frontend service endpoint routing to the inactive environment. This is just another service deployment
   to the AKS Kubernetes cluster.
1. **Verify Prod**: verify the frontend service endpoint is working properly with the new environment.
1. **Post-clean**: do some post clean on the temporary files.

For the Rolling Update, just deploy the deployment configuration to the Kubernetes cluster, which is just a straight forward,
single step.

## Put It All Together

We built a quickstart template on Azure to demonstrate how we can do the zero-downtime deployment to AKS (Kubernetes) with
Jenkins. Go to [Jenkins Blue-Green Deployment on Kubernetes](https://aka.ms/azjenkinsk8sqs)
and click the button **Deploy to Azure** to get the working demo. This template will provision:

* An AKS cluster, with the following resources:
   * Two similar deployments representing the environment blue and green. Both are initially setup with the `tomcat:7` image.
   * Two test endpoint services (`tomcat-test-blue` and `tomcat-test-green`), which are connected to the corresponding
      deployments, and can be use to test if the deployments are ready for production use.
   * A production service endpoint (`tomcat-service`) which represents the public endpoint that the users will access. 
      Initially it is routing to the blue environment.
* A Jenkins master running on a Ubuntu 16.04 VM, with the Azure service principal credentials configured. The Jenkins instance has two sample jobs:
   * **AKS Kubernetes Rolling Update Deployment** pipeline to demonstrate the Rolling Update deployment to AKS.
   * **AKS Kubernetes Blue/green Deployment** pipeline to demonstrate the blue/green deployment to AKS.

      We didn't include the email confirmation step in the quickstart template. To add that, you need to configure the email SMTP
      server details in the Jenkins system configuration, and then add a Pipeline stage before `Switch`:

      ```groovy
      stage('Confirm') {
          mail (to: 'to@example.com',
              subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
              body: "Please go to ${env.BUILD_URL}.")
          input 'Ready to go?'
      }
      ```

Follow the [Steps](https://github.com/Azure/azure-quickstart-templates/tree/master/301-jenkins-aks-zero-downtime-deployment#steps)
to setup the resources and you can try it out by start the Jenkins build jobs.
