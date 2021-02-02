# limitation
- Lambda functions packaged as container images do not support adding Lambda layers to the function configuration.

# impelment dependency in custom container
- When you include one or more layers in a function, during initialization, the contents of each layer are extracted in order to the /opt directory in the function execution environment.
- Lambda searches the /opt/extensions directory and starts initializing any extensions found. Extensions must be executable as binaries or scripts. As the function code directory is read-only, extensions cannot modify function code. Lambda layers and extensions are just files copied into specific file paths in the execution environment during the function initialization. The files are read-only in the execution environment.

# Using Lambda layers in container images, not consider non-official image
There are a number of ways to use container image layering to add the functionality of Lambda layers to your Lambda function container images.

## import layer from container
### Build a container image contain Lambda layer
**dockerfile**
```
FROM python:3.8-alpine AS installer
#Layer Code
COPY extensionssrc /opt/
COPY extensionssrc/requirements.txt /opt/
RUN pip install -r /opt/requirements.txt -t /opt/extensions/lib

FROM scratch AS base
WORKDIR /opt/extensions
COPY --from=installer /opt/extensions .
```

**login/tag/push to ECR**
```shell
docker build -t log-extension-image:latest  .
docker tag log-extension-image:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-image:latest
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-image:latest
```

### Use a container image version include Lambda layer
published container images must have the equivalent files located in the /opt directory. An image containing an extension must include the files in the /opt/extensions directory.

**dockerfile**
```
FROM public.ecr.aws/myrepo/shared-lib-layer:1 AS shared-lib-layer
# Layer code
WORKDIR /opt
COPY --from=shared-lib-layer /opt/ .

FROM aws-partner/extensions-layer:1 as extensions-layer
# Extension  code
WORKDIR /opt/extensions
COPY --from=extensions-layer /opt/extensions/ 
```

## import layer from uploaded layer by curl
### Copy the contents of a Lambda layer into a container image

**dockerfile**
```
FROM alpine:latest as layer-copy # Create a build stage to copy the files from S3 using credentials

ARG AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-"us-east-1"}
ARG AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID:-""}
ARG AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY:-""}
ENV AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
ENV AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
ENV AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

RUN apk add aws-cli curl unzip

RUN mkdir -p /opt

RUN curl $(aws lambda get-layer-version-by-arn --arn arn:aws:lambda:us-east-1:1234567890123:layer:shared-lib-layer:1 --query 'Content.Location' --output text) --output layer.zip
RUN unzip layer.zip -d /opt
RUN rm layer.zip

RUN curl $(aws lambda get-layer-version-by-arn --arn arn:aws:lambda:us-east-1:987654321987:extensions-layer:1 --query 'Content.Location' --output text) --output layer.zip
RUN unzip layer.zip -d /opt
RUN rm layer.zip

FROM scratch # Start second stage from blank image to squash all previous history, including credentials.
WORKDIR /opt
COPY --from=layer-copy /opt .
```

**build container that include lambda layer**
```shell
docker build . -t layer-image1:latest \
--build-arg AWS_DEFAULT_REGION=cn-northwest-1 \
--build-arg AWS_ACCESS_KEY_ID=xxxx \
--build-arg AWS_SECRET_ACCESS_KEY=xxxx
```

**check credentials not leaked**
```shell
docker history layer-image1:latest
```

## install layer directly in docker build

**dockerfile for extension image, can be merged into one**
```
FROM python:3.8-alpine AS installer
#Layer Code
COPY extensionssrc /opt/
COPY extensionssrc/requirements.txt /opt/
RUN pip install -r /opt/requirements.txt -t /opt/extensions/lib

FROM scratch AS base
WORKDIR /opt/extensions
COPY --from=installer /opt/extensions .
```

**login/tag/push to ECR, for extension image only**
```shell
docker build -t log-extension-image:latest  .
docker tag log-extension-image:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-image:latest
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-image:latest
```

**dockerfile for final image, can be merged into one**
```
FROM 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-image:latest AS layer
FROM public.ecr.aws/lambda/python:3.8
# Layer code
WORKDIR /opt
COPY --from=layer /opt/ .
# Function code
WORKDIR /var/task
COPY app.py .
CMD ["app.lambda_handler"]
```

**login/tag/push to ECR, for final function this time**
docker build -t log-extension-function:latest  .
docker tag log-extension-function:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-function:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-function:latest

**create lambda function from such image**
```shell
aws lambda create-function --region us-east-1  --function-name log-extension-function \
--package-type Image --code ImageUri=123456789012.dkr.ecr.us-east-1.amazonaws.com/log-extension-function:latest \
--role "arn:aws:iam:: 123456789012:role/lambda-role" \
--environment  "Variables": {"S3_BUCKET_NAME": "s3-logs-extension-demo-logextensionsbucket-us-east-1"}
```

# Using custom runtimes with container images
With .zip archive functions, custom runtimes are added using Lambda layers. With container images, you no longer need to copy in Lambda layer code for custom runtimes.

You can build your own custom runtime images starting with AWS provided base images for custom runtimes. You can add your preferred runtime, dependencies, and code to these images. To communicate with Lambda, the image must implement the Lambda Runtime API. We provide Lambda runtime interface clients for all supported runtimes, or you can implement your own for additional runtimes.


# reference
https://aws.amazon.com/blogs/compute/working-with-lambda-layers-and-extensions-in-container-images/
