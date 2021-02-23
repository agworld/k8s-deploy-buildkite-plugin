# K8s Deploy Buildkite Plugin

A Buildkite plugin that has everything you should (hopefully) need to deploy an application to a Kubernetes cluster.

## Image promotion

Image promotion copies a docker image from one registry to one or more other registries. This is commonly user to promote an image from the staging registry to a production registry, or many production registries in different regions. The command assumes the Buildkite agent has valid AWS credentials for logging into each repository which is normally implemented using a k8s service account with the correct IAM policy assigned.

### Example usage

```YAML
plugins:
    - agworld/k8s-deploy#v1.2.0:
        action: promoteImage
        sourceImage:
            image: "my-awesome-app:${DOCKER_TAG}"
            repositoryUrl: "000000000000.dkr.ecr.ap-southeast-2.amazonaws.com"
            region: "ap-southeast-2"
        destinationImages:
        - image: "my-awesome-app:${DOCKER_TAG}"
          repositoryUrl: "000000000000.dkr.ecr.ap-southeast-2.amazonaws.com"
          region: "ap-southeast-2"
        - image: "my-awesome-app:${DOCKER_TAG}"
          repositoryUrl: "000000000000.dkr.ecr.us-east-1.amazonaws.com"
          region: "us-east-1"
```

## Jobs

Jobs are used to run custom commmands from your chosen container(s) with the output being piped into the Buildkite logs. This is typically for running migrations or compiling & uploading assets to thier correct location. 

Jobs require a JSON template of a standard Kubernetes Job and that the templates are stored on S3. The command assumes that the Buildkite agent has valid credentials for downloading from the bucket storing the template and that a Kubernetes service account exists with permissions to run the job.

Environment variables are used to import variables from the build pipeline like the image to use, both of the below variables should be used in the template.

| Variable      | Description |
| ----------- | ----------- |
| $JOB_IMAGE      | The image specified in the jobImage buildkite variable       |
| $JOB_NAME   | A unique name for the job generated by the plugin        |

### Example usage

```YAML
plugins:
    - agworld/k8s-deploy#v1.2.0:
        action: runJob
        jobTemplateUrl: "s3://s3-bucket-name/jobs/awesome-migrate-job.json"
        jobImage: "000000000000.dkr.ecr.ap-southeast-2.amazonaws.com/my-awesome-app:${DOCKER_TAG}"
```

## Deployments

Deployments are used to update images and tags on one or more deployments running on a Kubernetes cluster. When using multiple deployments, all deployments will be done in parallel and if one fails they will all roll back to the previous deployment revision. 

`patches` are used to update metadata on the deployment for things like version numbers. After patching each deployments the plugin annotates the change with the date, who unblocked the deploy, the last commit hash and commit message (if any).

### Example usage

```YAML
plugins:
    - agworld/k8s-deploy#v1.2.0:
        action: updateImage
        deployments:
        - name: my-awesome-app
            namespace: default
            patches:
            - op: "replace"
              path: "/metadata/labels/tags.datadoghq.com~1version"
              value: "${DOCKER_TAG}"
            - op: "replace"
              path: "/spec/template/metadata/labels/tags.datadoghq.com~1version"
              value: "${DOCKER_TAG}"
            containers:
            - containerName: app
              image: "000000000000.dkr.ecr.ap-southeast-2.amazonaws.com/my-awesome-app:${DOCKER_TAG}"

        - name: my-awesome-app-background-workler
            namespace: default
            patches:
            - op: "replace"
              path: "/metadata/labels/tags.datadoghq.com~1version"
              value: "${DOCKER_TAG}"
            - op: "replace"
              path: "/spec/template/metadata/labels/tags.datadoghq.com~1version"
              value: "${DOCKER_TAG}"
            containers:
            - containerName: app-background
              image: "000000000000.dkr.ecr.ap-southeast-2.amazonaws.com/my-awesome-app:${DOCKER_TAG}"
```


