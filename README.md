# Tekton Artifacts for iter8

This project contains artifacts that can be used to create a Tekton pipeline to build a new version of an application and gradually roll it out using **iter8**.

This assumes a basic understanding of [iter8](https://github.com/iter8-tools/docs) and [Tekton](https://github.com/tektoncd/pipeline/tree/master/docs).

## Design Considerations

The following considerations were taken into account in the design of the artifacts:

- **reuse**  - Desire to reuse scripts developed and used for [Progressively rollout a Kubernetes app using iter8](https://github.com/open-toolchain/iter8-toolchain-rollout).

- **automated triggering** - Ability to integrate with the [webhook capability](https://github.com/tektoncd/experimental/tree/master/webhooks-extension) in [Tekton dashboards](https://github.com/tektoncd/dashboard) to trigger automated execution. This requirement [limits](https://github.com/tektoncd/experimental/blob/master/webhooks-extension/docs/Limitations.md#pipeline-limitations) the number of configurable parameters.

- **flexibility** - Avoid assumptions about the application; the same pipeline should work for multiple applications.

## Pipline Overview

At a high level, the pipeline we wish to create contains the following three tasks:

    build -> create experiment -> deploy new version

Once the candidate version is deployed, the iter8 controller will manage the canary deployment.

To simplify and eliminate prerequisite steps, we provide the following additional tasks that enable automated examples:

- deploy bookinfo application
- generate load
- wait for iter8 to complete and delete the unused version

A final pipeline might be something like:

      deploy bookinfo
    /         generate load
    \       /
      build                       deploy new version
            \                   /  
              create experiment 
                                \
                                  wait for completion/delete

Each of these tasks is reviewed in detail below. However, before doing so, some remaining prerequiste steps are identified.

## Prerequisite Steps

### Required Software

- Istio: https://istio.io/docs/setup/
- iter8: https://github.com/iter8-tools/docs/blob/master/doc_files/iter8_install.md
- Tekton: https://github.com/tektoncd/pipeline/blob/master/docs/install.md

If using webhooks, the following additional items are needed:
- Tekton Dashboard: https://github.com/tektoncd/dashboard
- webhooks plugin: https://github.com/tektoncd/experimental/tree/master/webhooks-extension
- knative (version 0.6.0): see documentation for webhooks plugin

### Create Target Namespace

Define a target namespace and enable it for Istio auto-injection:

    NAMESPACE=<target namespace>
    kubectl create namespace $NAMESPACE
    kubectl label namespace $NAMESPACE istio-injection=enabled

### Sample Application

The [bookinfo application](https://istio.io/docs/examples/bookinfo/), a sample application developed for demonstrating features of Istio can be used to demonstrate the pipeline.

The following project has been created from the source of the *reviews* microservice and can be cloned: https://github.com/kalantar/reviews

In the `bookinfo` folder are files that can be used to deploy the bookinfo application. Also a Tekton task has been created that does this.

### Authentication

The build task reads the source code from a GitHub repository, builds a Docker image and pushes it to DockerHub. At execution time, the pods need permission to read GitHub and write to DockerHub. This can be accomplished by defining secrets and associating them with the service account that is used to run the pipeline.

We used a public repository so no GitHub secret is needed. A secret for access to DockerHub can be defined as:

    DOCKER_USERNAME=<your DockerHub username>
    DOCKER_PASSWORD=<your DockerHub password>
    kubectl --namespace ${NAMESPACE} apply --filename - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
      name: dockerhub
      annotations:
        tekton.dev/docker-0: https://index.docker.io
    type: kubernetes.io/basic-auth
    stringData:
      username: ${DOCKER_USERNAME}
      password: ${DOCKER_PASSWORD}
    EOF
  
For additional information on authentication, see the [Tekton Documentation](https://github.com/tektoncd/pipeline/blob/master/docs/auth.md).

### Optional: Create a ServiceAccount

When executing a Tekton pipeline, the `PipelineRun` can be defined to run each task under different service accounts. If using the experimental [github webhook capability](https://github.com/tektoncd/experimental/tree/master/webhooks-extension), the `PipelineRun` created with run all tasks using the same service account. In the subsequent discussion we will asssume the service account is `default`.

### Add Secret (s) to ServiceAccount

You can use any `ServiceAccount`. If you use a non-default account, it will be necessary to specify this in the `PipelineRun` you create to run the pipeline (see below). For simplicity, we show this with the default service account.

    kubectl patch --namespace ${NAMESPACE} \
      serviceaccount default \
      --patch '{"secrets": [{"name": "dockerhub"}]}'

### Give the ServiceAccount the Needed Permissions

The pipeline tasks create iter8 experiments, reads Istio virtual system and destination rules and create kubernetes services and deployments. The service account that runs these tasks must have permission to take these actions.

A `ClusterRole` and `ClusterRoleBinding` can be used to define the necessary permissions and to assign it to the necessary `ServiceAccount`:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: tekton-iter8-role
    rules:
    - apiGroups: [ "networking.istio.io" ]
      resources: [ "destinationrules", "gateways", "virtualservices" ]
      verbs: [ "get", "list", "watch", "create", "update", "patch", "delete" ]
    - apiGroups: [ "iter8.tools" ]
      resources: [ "experiments" ]
      verbs: [ "get", "list", "watch", "create", "update", "patch", "delete" ]
    - apiGroups: [""]
      resources: [ "services" ]
      verbs: [ "get", "list", "watch", "create", "update", "patch", "delete" ]
    - apiGroups: [ "extensions", "apps" ]
      resources: [ "deployments" ]
      verbs: [ "get", "list", "watch", "create", "update", "patch", "delete" ]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: tekton-${NAMESPACE}
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: tekton-iter8-role
    subjects:
    - kind: ServiceAccount
      name: default
      namespace: ${NAMESPACE}

For convenience, these are defined [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/tekton-iter8-role.yaml).

### Optional: Define PipelineResources

The webhook handler will create any needed pipeline resources. However, if executing by hand, it is necessary to create them.

In order to load a GitHub project and write/read a DockerHub image, in a Tekton pipeline, it is necessary to define `PipelineResource` resources. For additional details about `PipelineResource`, see the [Tekton documentation](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md).
Two resources are needed, one for the git project and for the DockerHub image we will build.

You can create the GitHub repo by cloning or forking the [iter8 repo](https://github.com/iter8-tools/bookinfoapp-reviews) The Github resource can be specified as:

    kubectl --namespace ${NAMESPACE} apply --filename - <<EOF    
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: reviews-repo
    spec:
      type: git
      params:
      - name: revision
        value: master
      - name: url
        value: https://github.com/<your github org>/bookinfoapp-reviews
    EOF

The DockerHub image can be specified as:

    kubectl --namespace ${NAMESPACE} apply --filename - <<EOF
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: reviews-image
    spec:
      type: image
      params:
      - name: url
        value: index.docker.io/<your docker namespace>/reviews
    EOF

## Pipeline Tasks

## Task: Deploy Bookinfo Application

The `deploy-bookinfo-task` deploys the bookinfo application with the exception of the *reviews* microservice.

Further, it defined the needed 

It can be applied as:

    kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/tasks/deploy-bookinfo.yaml?token=AAAKR09L-BD0-axwaWobd5_XX90SjzEGks5dpiLiwA%3D%3D

## Task: Build New Version

We build our image and push it to DockerHub using [Kaniko](https://github.com/GoogleContainerTools/kaniko/).
Kaniko both builds and pushes the resulting image to DockerHub. The full Tekton `Task` definition is [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/tasks/build.yaml). It can be applied as:

    kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/tasks/build.yaml?token=AAAKR-I2-xCLr0TtF3lkLn4H8Rkf_drbks5dnj_QwA%3D%3D

## Task: Create Experiment

Iter8 configures Istio to gradually shift traffic from a current version of a Kubernetes application to a new version. It does this over time based on an assessment of the success of the new version. This assessment can be on its own or in comparison to the existing version. The `Experiment` that specifies this rollout is created from a template stored in `iter8/experiment.yaml`. The `Create Experiemnt` task modifies this template to identify the current and next versions and creates the `Experiment`.

The main challenge is to identify the current version. We rely on labels iter8 adds to the `DestinationRule`. These are used to match against the `Deployment` objects to find the current version. If iter8 has never been used, these labels do not exist so we select randomly select one of the matching deployments.

For the new version we use the short commit id of the repo being built.

The full definition of the Tekton `Task` is [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/tasks/create-experiment.yaml). It can be applied as:

   kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/tasks/create-experiment.yaml?token=AAAKR3tY-Q53wGjNC6UQEwR1BCfVcv77ks5dnkJqwA%3D%3D

## Task: Deploy New Version

To deploy an image, we use [kustomize](https://github.com/kubernetes-sigs/kustomize) to create a version specific deployment yaml. This allows us genertate as many resources as are needed; there is no assumption that only a `Deployment` is being created. We assume the kustomize configuration is stored in the source code repository.
The task implements 4 steps:

1. `modify-patch` - modifies a kustomize patch to be version aware
2. `kustomize` - generates the deployment yaml by applying the patch
3. `log-deployment` - logs the generated deployment yaml
4. `apply` - applies the deployment yaml via `kubectl`

The full Tekton `Task` definition is [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/tasks/deploy.yaml). It can be applied as:

    kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/tasks/deploy.yaml?token=AAAKR25UFk7e3LlnIlkzEC8KYsf-UjrNks5dnkAowA%3D%3D

## Task: Generate Load

Iter8 can evaluate the success of a new version if there is load against the system. The load generates meaninful metric data used by iter8 to assess the new version. Since bookinfo is a toy application, we added a task to our pipeline to generate load against the application. The benefit of adding a task instead of doing this manually is that we don't forget to start the load generation.

Once we start load, we face the problem of stopping it when a canary rollout is complete. To accomplish this we add a shared persistent volume between this task and the "wait-completion" task. When the latter identifies a completed rollout, it touches a file on the shared volume. The load-generator watches for this change and terminates the load.

For simplicity we used a volume of the default `StorageClass`. In this example, we rely on dynamic  volume provisioning.

    #kind: PersistentVolume
    #apiVersion: v1
    #metadata:
    #  name: experiment-stop-volume
    #spec:
    #  storageClassName: manual
    #  capacity:
    #    storage: 100Ki
    #  accessModes:
    #    - ReadWriteOnce
    #  hostPath:
    #    path: "/mnt/stop"
    #---
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: experiment-stop-claim
    spec:
      storageClassName: default
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 2Ki

The final Tekton `Task` definition is [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/tasks/generate-load.yaml). It can be applied as:

    kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/tasks/generate-load.yaml?token=AAAKR9oP4ndV3QkLkmOnsQGhkg6c0Tbyks5dnkF3wA%3D%3D

## Task: Wait for completion

A task to test for completion monitors progress. When the canary rollout is complete, it identifies (based on the `status` of the `Experiment`) which deployment (the original or the the new) is being used and deletes the other one to save resources. Finally, it touches a shared file to trigger the termination of load.

The Tekton `Task` definition is [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/tasks/wait-completion.yaml). It can be appkied as:

    kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/tasks/wait-completion.yaml?token=AAAKRwYi55q5g_8sBLRA1qQ5bm-PKAF6ks5dnkG-wA%3D%3D

## Putting it together

A `Pipeline` resource puts the tasks together and allows us to specify dependencies between them. Further, we can map pipeline level paramters to task specific parameters.

We use `runAfter` to create this execution order:

         / -> generate load
    build                      / -> deploy
         \ -> create-experiment 
                               \ -> wait-completion

The completed Tekton `Pipeline` is [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/pipeline.yaml). It can be applied as:

    kubectl --namespace ${NAMESPACE} apply --filename https://raw.github.ibm.com/kalantar/iter8-tekton-blog/master/pipeline.yaml?token=AAAKR5yjd2QiLDqOs67G-GODmwJ_VTNeks5dnka-wA%3D%3D

## Running the Pipeline

### Create `PipelineRun`

Finally, to execute a Tekton pipeline, we create a `PipelineRun`. When created, it manages the execution of the pipeline. We can follow the execution of the pipeline by observing the pods that are created. We can follow the execution of iter8 by observing the creation of the experiment:

    watch kubectl --namespace ${NAMESPACE} experiments.iter8.tools

An example `PipelineRun` is captured [here](https://github.ibm.com/kalantar/iter8-tekton-blog/blob/master/pipelinerun.yaml). It can be applied as:

    kubectl --namespace ${NAMESPACE} apply --filename 

## Conclusions
