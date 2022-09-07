# Telepresence with Amazon EKS and ReleaseHub

This project demonstrates how to use [Telepresence.io](https://www.telepresence.io/docs/latest/quick-start/) to intercept traffic sent to a [ReleaseHub](https://releasehub.com)-managed Kubernetes service on Amazon EKS in your AWS account and reroute it to your local developer machine.

## Overview

### ReleaseHub

[ReleaseHub](https://releasehub.com) _environments-as-a-service_ (EaaS) offloads the heavy lifting of cloud infrastructure and service configuration by translating files like `docker-compose.yaml` or `package.json` into _complete_, full-stack ephemeral cloud environments in your own cloud accounts. Users may also package virtually any existing code (e.g. infra-as-code, test automation, or database migrations) into a ReleaseHub template.

### Telepresence (and Ambassador)

[Telepresence](https://www.telepresence.io/) is an open-source project developed by [Ambassador Labs](https://www.getambassador.io/) that allows a developer to temporarily redirect traffic from a Kubernetes container service to their own local machine. Telepresence has a wide range of configuration options and use cases, and their documentation is the best way to learn about it's full capabilities.

[Ambassador](https://www.getambassador.io/products/telepresence/) is a commercial extension from Ambassador Labs provides useful additional capabilities on top of Telepresence, like a team-centric UI to manage services & connections, the ability for a developer to generate preview URLs to their loca service to share with their team, and more.



### ReleaseHub & Telepresence

The diagram below shows a simplfied version of what ReleaseHub could provide just by supplying a Docker Compose file listing a two-tier container application environment:

<img src="images/with-telepresence.png" width="500">

## Prerequisites

* ReleaseHub user has the `owner` role within the account you intend to use
* ReleaseHub account is on a full-access plan (paid or temporary full-access trial
* Your account has an Amazon EKS cluster created using ReleaseHub
* On your local developer machine:
  
  * [kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
  * [Helm CLI](https://helm.sh/docs/intro/install/)
  * Optional: [yq](https://github.com/mikefarah/yq/#install) (for parsing ReleaseHub CLI output)

## Quickstart

This quickstart is written with ReleaseHub users in mind, though most is still applicable to the broader community.

1. [**Install the Telepresence CLI & daemon** on your local machine.](https://www.telepresence.io/docs/latest/install).
  
2. **Install the Telepresence Traffic Manager** into your EKS cluster with:

    ```sh
    release clusters exec -- telepresence helm install
    ```

    This will install the Traffic Manager into it's own `ambassador` namespace, and this Traffic Manager will proxy traffic between the Telepresence daemon on your local machine and the Telepresence agent sidecar container added to your Kubernetes pods when you later use the Telepresence CLI to intercept a service or otherwise connect to a namespace.

    [Telepresence Traffic Manager](https://www.telepresence.io/docs/latest/reference/architecture) installed into your ReleaseHub EKS cluster.

3. **Create or select a service to intercept:** From within the [ReleaseHub Web Console](https://releasehub.com) or by using the ReleaseHub CLI ([example])(#browse_with_cli)), identify or create an environment on a ReleaseHub-managed EKS cluster with a container service that you want to intercept and write down the environment's `namespace` and service's `name`.

    If none of your ReleaseHub Application Templates define a service that makes sense for traffic interception, you can fork one of the projects in GitHub at [https://github.com/awesome-release](https://github.com/awesome-release) and transform into a new Application & Environment. We recommend using an example that contains a frontend service with public ingress, such as [awesome-release/angular](https://github.com/awesome-release/angular).

4. **Remove the resource quota on your environment namespace:** ReleaseHub places a memory resource quota on each environment's Kubernetes namespace with a dynamically-generated value that matches the the aggregate memory requested by services in your Application Template. When a memory quota is present, Kubernetes will reject containers that do not explicitly set container-level memory limits. However, when Telepresence intercepts traffic, it adds a traffix proxy container to your namespace that does not contain a memory limit.

    Remove the default memory quota on your ReleaseHub environment's namespace:
      
      ```sh
      release clusters exec -- kubectl delete resourcequota quota --namespace YOUR_NAMESPACE
      ```

1. **Start a local service to intercept traffic:** On your local machine, start and leave running a service that you want to use as the target for the traffic intercepted from your remote environment service.

    Or, if you just want to quickly test Telepresence with a local service, you can simply run an HTTP or TCP echo server with one of the commands below:

    **HTTP Echo Server:**

    ```sh
    # HTTP echo server:
    docker run -p 8080:5678 hashicorp/http-echo -text="Hello, I am your local service intercepting traffic from your remote environment."
    ```

    **TCP Echo Server:**

    ```sh
    # TCP echo server:
    docker run -d --name echo -p 8080:9999 abrarov/tcp-echo --port 9999 --inactivity-timeout 300
    ```

1. **Intercept remote EKS service traffic:** With your remote service identified and your local service running, you are now ready to intercept traffic with Telepresence.

    ```sh
    export TELEPRESENCE_LOCAL_PORT=8080
    export TELEPRESENCE_SERVICE_NAME=frontend
    export TELEPRESENCE_NAMESPACE=multifeature-demo-main
    export RELEASE_CLUSTER=hubofhubs-release-us-west-2

    release clusters exec --cluster $RELEASE_CLUSTER -- \
        telepresence intercept \
        --preview-url=false \
        --http-header=all \
        -p $TELEPRESENCE_LOCAL_PORT \
        -n $TELEPRESENCE_NAMESPACE \
        $TELEPRESENCE_SERVICE_NAME
    ```

## Appendix

### Using ReleaseHub CLI to browse Applications and Environments

As an alternative to the ReleaseHub Web Console, you can use the ReleaseHub CLI to explore available Applications, environments, and more. Examples are below:

```sh
# Find an application
release apps list

# Find running environment(s) for a given app ID:
release environments list --app <APP_ID>

# Return the "services:" section of an existing ReleaseHub environment
release config-get --app <APP_ID> --environment <ENV_ID> | yq '.services'
```

## Troubleshooting

### No kind "ExecCredential" is registered for version "client.authentication.k8s.io/v1alpha1"

Kubernetes deprecated the alpha version of an API used for cluster user authentication, `client.authentication.k8s.io/v1alpha1`, in `kubeconfig` with K8s version 1.24. If you have an old version of `kubectl` installed, you may receive an error like this when attempting to run commands line `telepresence helm install` or `release clusters exec kubectl get svc`:

```
Unable to connect to the server: getting credentials: decoding stdout: no kind "ExecCredential" is registered for version "client.authentication.k8s.io/v1alpha1" in scheme "pkg/client/auth/exec/exec.go:62"
```

Upgrading to the latest version of kubectl should fix this.

If you are using AWS IAM helpers in your `~/.kube/config` to access your Amazon EKS cluster using IAM access keys or an IAM role from your local AWS CLI profile, the AWS CLI is used under-the-hood to retrieve temporary credentials from EKS. Old versions of the CLI will also have the same problem ([PR 6476](https://github.com/aws/aws-cli/pull/6476)) as kubectl, above, and should be upgraded.
