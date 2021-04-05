# EKM Argo CD applications

---

This repository is used to manage the applications that run within EKM's Kubernetes clusters. Currently there is only one configured, however, it is possible that this number will increase in the future.

## Getting started

In order to get the most out of this repository you should at least have some knowledge of the technologies that surround it. You may find [this document](https://docs.google.com/document/d/1OfWjJ8VW38qqhmhrPOSnRheEH4M7GCouDTIOmvaRTrc/edit) and its list of resources helpful in getting started.

You will also find it necessary to set up a user for Argo CD. Argo CD users are currently managed by [Jack Caldwell](https://github.com/jackcaldwell). If you wish to be added to the list of users then you can get in contact with him (me) via Workplace / Email / Slack / Discord.

Once you have an account set up you will be able to log in to the Argo CD UI at [https://argo-cd.ekmsvc.com](https://argo-cd.ekmsvc.com).

After logging into your account you should be greeted by a screen that looks similar to the following.

![Argo CD Home](https://i.ibb.co/CtDRsWD/argo-cd-home.png)

From this page you will be able to view the status of each application and whether or not the state of the cluster is currently in sync with the desired state that is defined within this repository.

## Architecture

Argo CD is quite flexible and doesn't really hold too many opinions about how CI / CD should be managed for your application. It's main purpose is to ensure that the resources currently running within the cluster accurately reflect those defined within this repository. This means that to create new applications all that you need to do is to add files to this repository that describe the resources that are needed to run your application (deployments, services, ingresses etc.) If you want to learn more about Argo CD specifically I would recommend their [documentation](https://argoproj.github.io/argo-cd/) as a starting point.

![Argo CD Architecture](https://argoproj.github.io/argo-cd/assets/argocd_architecture.png)

The image above is taken from this documentation and I think it does quite a good job of illustrating this architecture. The process for making changes to what is running in the Kubernetes looks something like this.

- A commit is pushed to the default branch of this repository. This could be a change in the image tag of an application that is running is certain environment, a whole new application is added, or a Prometheus `ServiceMonitor` has been added (or pretty much anything inbetween).
- Argo CD looks at the updated version of the default branch and compares it to what is currently in the cluster.
- Argo CD executes any changes that need to be made.

This declarative way of managing our applications is nice because it means that we don't need to worry about _how_ something is deployed and only _what_ is deployed.

## Repository structure overview

There are currently two main `AppProject` resources defined within this repository - `Staging` and `Production`. These projects are defined in [project.yaml](https://github.com/ekmsystems/argocd-apps/blob/master/project.yaml). There is nothing particularly special about these projects and they primarily provide a way for us to categorise running versions of our application.

Within [apps.yaml](https://github.com/ekmsystems/argocd-apps/blob/master/apps.yaml) a single application is defined for each of these projects (`Staging` / `Production`). These applications point to the [production](https://github.com/ekmsystems/argocd-apps/tree/master/production) and [staging](https://github.com/ekmsystems/argocd-apps/tree/master/staging) folders of the application respectively. This pattern may seem quite strange at first as you have files within the repository referencing folders within the same repository. However, this pattern allows you to introduce new applications to the cluster by simply adding new manifest files (rather than clicking through the UI / executing CLI commands to add new applications). This makes our infrastructure more reproducible.

In each of these folders `staging/` & `production/` you can see an `Application` resource for each application that we want to manage with Argo CD. `Application` is a [custom resource definition (CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) defined by Argo CD to represent the concept of an application that it manages. The most important of these `Application` resources is the `source` property of the `spec`. This allows you to define the repository and path where the Kubernetes resources are defined. It is possible to reference a repository outside of this one. However, it is recommended that each application has its own folder within this repository with [Kustomize](https://kustomize.io/) overlays for the different application environments.

> Note: Kustomize is basically just a tool that we use to prevent duplicated resources across environments. E.g. It is likely that an application will have similar resources for production / staging (they both have similar deployments, services etc.). Kustomize allows to define 'overlays' on top of a base definition to minimize duplication. It would be perfectly reasonable to have a production / staging within an application folder that contains the plain Kubernetes manifest files.

## Creating a new application (the opinionated way)

In this section I will describe how to set up a new application to be deployed within the cluster. I will do my best to describe what each piece does along the way. There are definitely alternatives to the steps that I outline above and you should feel free to deviate from the path I set out if you feel that the steps don't completely fit your use case. However, I would hope that for the vast majority of applications something similar to what is outlined below would be suitable. I will use the demo `go-todo` application as an example for each of these steps.

### Add a new ECR registry for your application

The first thing that you'll need to do is create a new ECR registry for your application. Its possible to do this using the AWS UI. However, I'd much prefer it if you use our [ekm-aws-terraform](https://github.com/ekmsystems/ekm-aws-terraform) repository. This is the repository that should really be used for any AWS resources going forward, but especially if they are related to our Kubernetes in any way.

- Clone the [repository](https://github.com/ekmsystems/ekm-aws-terraform) - `git clone https://github.com/ekmsystems/ekm-aws-terraform.git`
- Add a new resource to the `ecr.tf`. It should look similar to the following:

```
resource "aws_ecr_repository" "orders-api" {
  name                 = "orders-api"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

This describes your ECR repository.
**NOTE**: Make sure the repository name matches the application name that you intend to use throughout.

- Create a new branch for your changes.
- Push the changes and create a pull request into the default branch (`main`).
- This will trigger a GitHub action that performs a GitHub action that will describe the changes that will happen if your changes are merged. These changes should include the creation of a new ECR repository for you container images.
- If everything looks as you'd expect then you can merge your changes and they will be applied by another GitHub action. This will create your specified resources.
- If you are sceptical, you can then use the AWS UI to check that your resources have actually been created.

### Add the application file for each of your environments

This should be done for each of the environments that you would like your application to be deployed to. E.g [production](https://github.com/ekmsystems/argocd-apps/tree/master/production) and [staging](https://github.com/ekmsystems/argocd-apps/tree/master/staging).

#### Example `Application` manifect

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-todo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    path: go-todo/overlays/production
    repoURL: git@github.com:ekmsystems/argocd-apps.git
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

You should update the application name and source, along with the project and namespace depending on your environment.

You can see that the `source` points to a folder within this repository. It is necessary for you to create a folder and do the same for your new application. This folder is where the resource files specific to your application will be added.

Within your application's new folder you can define a `base` folder (this is where the resources that are shared between environments) is defined and an `overlays` folder (this is where resources that are environment specific will be defined).

Add a `service.yaml` to your `base` folder.

```
apiVersion: v1
kind: Service
metadata:
  name: go-todo
  labels:
    app: go-todo
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: go-todo
```

Update the name / labels.app / selector.app fields to the name of your application. It is also likely that you will want to update the `port` / `targetPort` for your application. This is dependent on the port that is exposed within your applications `Dockerfile` (it is common that this will be port 80, expecially for .NET Core applications). A service within Kubernetes is a way for you to expose an application running on a set of pods as a network service. As pods ephemeral (i.e. can be destroyed at any time) we can't reliably use a pod's IP to make requests to it. Services wrap around a set of pods and ensure that an application is consistently accessible. More information about services can be found [here](https://kubernetes.io/docs/concepts/services-networking/service/).

Add a `kustomization.yaml` file to your `base` folder. This file is simply used to gather up the resources that you want to be included as a base for your application. Our example contains only the `service.yaml` file.

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - service.yaml
```

Now we need to define each of the overlays for the production / staging environments. I will only describe the production files here, but the same applies for both. Its possible that you would want to use a `HorizontalPodAutoscaler` in place of the deployment for a production environment. This would allow you to automatically create new instances of your application in times of high traffic. However, I need to investigate this topic a bit more before recommending a specific way to do it.

Add your `deployment.yaml` file to the environment overlay.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-todo
  labels:
    app: go-todo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: go-todo
  template:
    metadata:
      labels:
        app: go-todo
    spec:
      containers:
        - name: go-todo
          image: '328373003584.dkr.ecr.eu-west-2.amazonaws.com/go-todo:74219bccf809027b1641c5dc0b05f7add6031aa2'
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
          resources:
            limits:
              cpu: 100m
              memory: 256Mi
```

This resource includes metadata about the application (name / labels etc), but most importantly it includes information about the application that should be run and _how_ it should be run.

You can define the `image` that should be run for each container, the resource limits / requested amounts, the health check endpoint for the container, the number of replicas of the application that should be running and more. More information can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Add your `ingress.yaml` file to the environment overlay.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-todo
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    # *.staging.ekmsvc.com certificate ARN
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-west-2:328373003584:certificate/b7abafe3-00e1-4845-b765-34108f65225f
spec:
  rules:
    - http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: go-todo
                port:
                  number: 8080
      host: todo.ekmsvc.com
```

This file defines how your application is exposed to the internet. We make use of some built in AWS functionality that allows us to automatically create application load balancers for our application. We can also define the the SSL certificate to use for the load balancer. This allows us to perform TLS termination at the load balancer level. (Note: We could do TLS termination at a lower level within the cluster, but this is the simplest solution that I've found so far.)

It is also worth noting that different certificates have been provisioned for the domains _.ekmsvc.com and _.staging.ekmsvc.com. If you want to deploy to another domain then additional certificates may need to be provisioned. You will also need to make sure that you specify that correct certificate for the host that you are deploying to.

### Accessing your application

Once you have followed these steps you can make a commit to the default branch of this repository. After doing so, once Argo CD has detected the changes, the cluster should be synced to reflect your changes. This should be everything that you need to actually have your application running and accessible from the cluster. You should be able to visit the load balancer management section with AWS EC2 and see that a load balancer has been created for you application. If you have created a staging / production environment then you should actually see two. [Link to AWS EC2 load balancer management for EU West 2 region](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#LoadBalancers:sort=loadBalancerName)

![AWS load balancer](https://i.ibb.co/1L3bnvj/aws-alb.png)

Hopefully you have included a health check endpoint or something similar within your application that you can access publically to ensure that it is running as expected. If you have you should be able to make a request to the DNS name given to your load balancer at the endpoint that your public endpoint lives at and see that it is running as expected.

We can see that the staging version of our demo application is running at

`k8s-staging-gotodost-609fb1d53f-248881577.eu-west-2.elb.amazonaws.com`

So we can make a request to `/health` and (hopefully) see that it is running as expected.

### Setting up DNS for your application

Using the AWS Route 53 service setting up DNS for our application is fairly simple.

- Go to the Route 53 [dashboard](https://console.aws.amazon.com/route53/v2/home#Dashboard).
- Click '[Hosted Zone](https://console.aws.amazon.com/route53/v2/hostedzones#)'
- Click '[ekmsvc.com](https://console.aws.amazon.com/route53/v2/hostedzones#ListRecordSets/Z04056062U968WUR9UZ3E)' - or whichever domain you would like to connect to the load balancer.
- Click 'Create record'

You should see something similar to this but without the fields filled in.
![Route 53](https://i.ibb.co/Bc6jCp4/route-53.png)

- For 'Record name' choose the prefix that you would like for you service.

- Click the 'Alias' switch.
- Choose 'Alias to Application and Classic Load Balancer'.
- Choose the region where your load balancer lives (most likely eu-west-2).
- Select your load balancer from the dropdown list.
- Click 'Create records'

This should set up the DNS for your application. You probably want to repeat this for each environment.

### CI / CD

At this point we have covered how to actually get a running application within the cluster. However, we are missing a deployment story. Any changes that we make would require manually pushing to a docker registry then updating the image's tag within the deployment for the environment that we want to update. We want this to be automated so how do we achieve this? This could be done with GitHub actions or any similar tool that would allow us push an image to a docker registry and make changes to a repository. However, for this example we are going to use a combination of [Argo Events](https://argoproj.github.io/argo-events/) and [Argo Workflows](https://argoproj.github.io/argo-workflows/).

#### Setting up an event source

The first thing that we need to do is add an event source for your repository. We would like it whenever we make a push to our application's repository something happens to deploy a new version of our application.

Add a file for your event source to [`/argo-events/overlays/production`](https://github.com/ekmsystems/argocd-apps/tree/master/argo-events/overlays/production) of this repository and define your event source resource.

```
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: go-todo
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  github:
    go-todo:
      owner: jackcaldwell
      repository: go-todo
      webhook:
        endpoint: /go-todo
        port: "12000"
        method: POST
        url: https://webhook.ekmsvc.com
      events:
        - "push"
      apiToken:
        name: github-access
        key: token
      insecure: false
      active: true
      contentType: json
```

This resource tells Argo Events to create an event source for our application repository. When this resource gets created in the cluster it will also automatically set up the webhook to make requests to our instance of Argo Events running within the cluster. There is a webhook listener set up at `https://webhook.ekmsvc.com` which handles webhook requests to Argo Events.

You will also need to add entry to the Argo Events resource ingress to let it know that requests to `/{your-app}` should go the event source service for the event source that you just set up. That resource can be found [here](https://github.com/ekmsystems/argocd-apps/blob/master/argo-events/overlays/production/event-sources.yaml) and should look similar to this.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: github
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    # *.ekmsvc.com certificate ARN
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-west-2:328373003584:certificate/b7abafe3-00e1-4845-b765-34108f65225f
spec:
  rules:
    - http:
        paths:
          - path: /orders-api
            pathType: Prefix
            backend:
              service:
                name: orders-api-eventsource-svc
                port:
                  number: 12000
          - path: /go-todo
            pathType: Prefix
            backend:
              service:
                name: go-todo-eventsource-svc
                port:
                  number: 12000
      host: webhook.ekmsvc.com
```

An additional path will need to be added for your application.

Once we have done this Argo Events will be listening for push events from your repository but nothing will be happening once it receives them.

### Adding a sensor for your webhook event source

To define what we actually want to happen when we receive an event we use a `Sensor`. These are defined in [`/argo-events/overlays/production`](https://github.com/ekmsystems/argocd-apps/blob/master/argo-events/overlays/production/sensors.yaml). Our example looks like this.

```
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: go-todo-webhook
spec:
  template:
    serviceAccountName: argo-events-sa
  dependencies:
    - name: todo
      eventSourceName: go-todo
      eventName: go-todo
      filters:
        data:
          - path: body.X-GitHub-Event
            type: string
            value:
              - push
          - path: body.ref
            type: string
            value:
              - refs/heads/main
  triggers:
    - template:
        name: trigger
        argoWorkflow:
          group: argoproj.io
          version: v1alpha1
          resource: workflows
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: go-todo-
                namespace: workflows
              spec:
                entrypoint: build
                serviceAccountName: workflow
                volumes:
                  - name: docker-config
                    configMap:
                      name: docker-config
                  - name: github-access
                    secret:
                      secretName: github-access
                      items:
                        - key: token
                          path: token
                        - key: user
                          path: user
                        - key: email
                          path: email
                templates:
                  - name: build
                    dag:
                      tasks:
                        - name: build
                          templateRef:
                            name: container-image
                            template: build-kaniko-git
                            clusterScope: true
                          arguments:
                            parameters:
                              - name: repo_url
                                value: ""
                              - name: repo_ref
                                value: ""
                              - name: repo_commit_id
                                value: ""
                              - name: container_image
                                value: 328373003584.dkr.ecr.eu-west-2.amazonaws.com/go-todo
                              - name: container_tag
                                value: ""
                        - name: promote-staging
                          templateRef:
                            name: promote
                            template: promote
                            clusterScope: true
                          arguments:
                            parameters:
                              - name: environment
                                value: staging
                              - name: image_owner
                                value: 328373003584.dkr.ecr.eu-west-2.amazonaws.com
                              - name: image_name
                                value: go-todo
                              - name: tag
                                value: ""
                              - name: application_path
                                value: go-todo
                          dependencies:
                            - build
                        - name: request-promote-production
                          templateRef:
                            name: request-promote
                            template: request-promote
                            clusterScope: true
                          arguments:
                            parameters:
                              - name: environment
                                value: production
                              - name: image_owner
                                value: 328373003584.dkr.ecr.eu-west-2.amazonaws.com
                              - name: image_name
                                value: go-todo
                              - name: tag
                                value: ""
                              - name: application_path
                                value: go-todo
                          dependencies:
                            - promote-staging
          parameters:
            - src:
                dependencyName: todo
                dataKey: body.repository.git_url
              dest: spec.templates.0.dag.tasks.0.arguments.parameters.0.value
            - src:
                dependencyName: todo
                dataKey: body.ref
              dest: spec.templates.0.dag.tasks.0.arguments.parameters.1.value
            - src:
                dependencyName: todo
                dataKey: body.after
              dest: spec.templates.0.dag.tasks.0.arguments.parameters.2.value
            - src:
                dependencyName: todo
                dataKey: body.after
              dest: spec.templates.0.dag.tasks.0.arguments.parameters.4.value
            - src:
                dependencyName: todo
                dataKey: body.after
              dest: spec.templates.0.dag.tasks.1.arguments.parameters.3.value
            - src:
                dependencyName: todo
                dataKey: body.after
              dest: spec.templates.0.dag.tasks.2.arguments.parameters.3.value
```

This tells Argo Events to listen for events from our event source (configured in the `dependencies` section), filter based on the branch that is pushed (configured in the `filters` section), and execute a series of steps (configured in the `templates` section). Certain parameters are also populated dynamically (such as commit tag) using data from the webhook (configured in the `parameters` section). Each step in the workflow uses a template defined the [`/argo-workflows/base/templates.yaml`](https://github.com/ekmsystems/argocd-apps/blob/master/argo-workflows/base/templates.yaml)

Our workflow has 3 steps.

1. `build` uses template `build-kaniko-git`

This step takes paramaters that describe where to access the Git repository from and where to push the resulting image too. It makes use of a container tool from Google called [Kaniko](https://github.com/GoogleContainerTools/kaniko). What this does is quite simple. It clones the repo, builds an image based on the `Dockerfile`, and pushes it to a registry.

2. `promote` uses template `promote`

This step clones this repository and updates the image tag for the application. It then pushes the changes directly to the master branch. Once Argo CD detects these changes then the new version of the application will be deployed to the cluster. In our example this step only makes changes to the staging environment.

3. `request-promote-production` uses template `request-promote`

This step is very similar to the normal promotion except it only requests a promotion by creating a new branch and requesting a merge into the default branch of this repository.

## Additional sections to cover

- Monitoring (Prometheus & Service monitors)
- Secret management
