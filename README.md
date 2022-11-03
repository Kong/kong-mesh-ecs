# Kong Mesh on ECS + Fargate

[![Nightly Kong Mesh on ECS](https://github.com/Kong/kong-mesh-ecs/actions/workflows/nightly.yaml/badge.svg)](https://github.com/Kong/kong-mesh-ecs/actions/workflows/nightly.yaml)

This repository provides some example CloudFormation templates for running
Kong Mesh on ECS + Fargate.

It provisions all the necessary AWS infrastructure for
running a standalone Kong Mesh zone with a postgres backend (on AWS Aurora)
and runs the [Kuma counter demo](https://github.com/kumahq/kuma-counter-demo).

## Deploying

The example deployment consists of CloudFormation stacks for setting up the mesh:

- VPC & ECS cluster stack
- Kong Mesh CP stack

and two stacks for launching the demo.

### Workload identity

The `kuma-dp` container will use the identity of the ECS task to
authenticate with the Kuma control plane.

To enable this functionality, note that
[we set the following `kuma-cp` options via environment variables](./deploy/controlplane.yaml#L334-L337):

```yaml
- Name: KUMA_DP_SERVER_AUTH_TYPE
  Value: aws-iam
- Name: KUMA_DP_SERVER_AUTH_USE_TOKEN_PATH
  Value: "true"
```

and we [add the following option to the `kuma-dp` container command](./deploy/counter-demo/demo-app.yaml#L251):

```yaml
                - --auth-type=aws
```

In these examples, the [ECS task IAM role has the `kuma.io/service` tag set](./deploy/counter-demo/demo-app.yaml#L126-L128)
to the name of the service the workload is running under:

```yaml
      Tags:
        - Key: kuma.io/service
          Value: !FindInMap [Config, Workload, Name]
```

### Setup

You'll need to have the [Kong Mesh CLI (`kumactl`)
installed](https://docs.konghq.com/mesh/1.5.x/install/) as well as the [AWS
CLI](https://aws.amazon.com/cli/) setup on the machine you're deploying from.

Check the [example IAM policy in this repo](./policy.json) for permissions
sufficient to deploy everything in this repository.

### VPC

The VPC stack sets up our VPC, adds subnets, sets up routing and private DNS and creates a load balancer.
It also provisions the ECS cluster and corresponding IAM roles.

```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-vpc \
    --template-file deploy/vpc.yaml
```

### Control Plane

The Kong Mesh CP stack launches Kong Mesh in standalone mode with an Aurora backend, fronted by an AWS Network Load Balancer.

#### License

The first step is to add your Kong Mesh license to AWS Secrets Manager. Assuming
your license file is at `license.json`:

```
LICENSE_SECRET=$(
  aws secretsmanager create-secret \
      --name ecs-demo/KongMeshLicense --description "Secret containing Kong Mesh license" \
      --secret-string file://license.json \
    | jq -r .ARN)
```

#### TLS

We need to provision TLS certificates for the control plane to use for both external
traffic (port `5682`) and proxy to control plane traffic (port `5678`), both of
which are protected by TLS.

In a production scenario, you'd have a static domain name to point to the load balancer
and a PKI or AWS Certificate Manager for managing TLS certificates.

In this walkthrough, we'll use the DNS name provisioned for the load balancer by AWS and use
`kumactl` to generate some TLS certificates.

##### CP address

The load balancer's DNS name is exported from our VPC stack and the HTTPS (`5682`) endpoints
are exposed:

```
CP_ADDR=$(aws cloudformation describe-stacks --stack-name ecs-demo-vpc \
  | jq -r '.Stacks[0].Outputs[] | select(.OutputKey == "ExternalCPAddress") | .OutputValue')
```

##### Certificates

`kumactl` provides a utility command for generating certificates. The
certificates will have two SANs. One is the DNS name of our load balancer and
the other is the internally-routable, static name we provision via ECS Service
Discovery for our data planes.

```
kumactl generate tls-certificate --type=server --hostname ${CP_ADDR} --hostname controlplane.kongmesh
```

We now have a `key.pem` and `cert.pem` and we'll save both of them as AWS secrets.

```
TLS_KEY=$(
  aws secretsmanager create-secret \
  --name ecs-demo/CPTLSKey \
  --description "Secret containing TLS private key for serving control plane traffic" \
  --secret-string file://key.pem \
  | jq -r .ARN)
TLS_CERT=$(
  aws secretsmanager create-secret \
  --name ecs-demo/CPTLSCert \
  --description "Secret containing TLS certificate for serving control plane traffic" \
  --secret-string file://cert.pem \
  | jq -r .ARN)
```

#### Deploy stack

```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-kong-mesh-cp \
    --parameter-overrides VPCStackName=ecs-demo-vpc \
      LicenseSecret=${LICENSE_SECRET} \
      ServerKeySecret=${TLS_KEY} \
      ServerCertSecret=${TLS_CERT} \
    --template-file deploy/controlplane.yaml
```

The ECS task fetches an admin API token and saves it in an AWS secret.

#### kumactl

Let's fetch the admin token from that secret:

```
TOKEN_SECRET_ARN=$(aws cloudformation describe-stacks --stack-name ecs-demo-kong-mesh-cp \
  | jq -r '.Stacks[0].Outputs[] | select(.OutputKey == "APITokenSecret") | .OutputValue')
```

Using those two pieces of information we can fetch the admin token and set up `kumactl`:

```
TOKEN=$(aws secretsmanager get-secret-value --secret-id ${TOKEN_SECRET_ARN} \
  | jq -r .SecretString)
kumactl config control-planes add \
  --name=ecs --address=https://${CP_ADDR}:5682 --overwrite --auth-type=tokens \
  --auth-conf token=${TOKEN} \
  --ca-cert-file cert.pem
```

#### GUI

We can also open the Kong Mesh GUI at `https://${CP_ADDR}:5682/gui` (you'll need to
force the browser to accept the self-signed certificate).

We now have our control plane running and can begin deploying applications!

### Demo app

We can now launch our app components and the workload identity feature will
handle authentication with the control plane.

```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-redis \
    --parameter-overrides VPCStackName=ecs-demo-vpc CPStackName=ecs-demo-kong-mesh-cp \
    --template-file deploy/counter-demo/redis.yaml
```

```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-demo-app \
    --parameter-overrides VPCStackName=ecs-demo-vpc CPStackName=ecs-demo-kong-mesh-cp \
    --template-file deploy/counter-demo/demo-app.yaml
```

See below under [Usage](#Usage) for more about how communcation between these
two services works and how to configure it.

The `demo-app` stack exposes the server on port `80` of the NLB so
our app is now running and accessible `http://${CP_ADDR}:80`.

### Cleanup

To cleanup the resources we created you can execute the following:

```
aws cloudformation delete-stack --stack-name ecs-demo-demo-app
aws cloudformation delete-stack --stack-name ecs-demo-demo-redis
aws cloudformation delete-stack --stack-name ecs-demo-kong-mesh-cp
aws secretsmanager delete-secret --secret-id ${TLS_CERT}
aws secretsmanager delete-secret --secret-id ${TLS_KEY}
aws secretsmanager delete-secret --secret-id ${LICENSE_SECRET}
aws cloudformation delete-stack --stack-name ecs-demo-vpc
```

### Further steps

#### Admin token

The control plane ECS task saves the generated admin token to an AWS secret.
After we have accessed the secret, we can remove the final two containers in our control plane task.

## Usage

When running Kong Mesh on ECS + Fargate, you'll need to list the services in the mesh
that your task communicates with. These are called
[_outbounds_](https://kuma.io/docs/1.4.x/documentation/dps-and-data-model/#dataplane-specification).

This entails editing the `Dataplane` template in the CloudFormation template
used to deploy your application.

We can see this in the [`demo-app`
template parameter `DPTemplate`](./deploy/counter-demo/demo-app.yaml):

```
        outbound:
        - port: 6379
          tags:
            kuma.io/service: redis
```

Here we're telling Kong Mesh that our `demo-app` will communicate with the `redis`
service. The sidecar is then configured to route requests to `redis:6379` to our
`redis` service.

## CI

This repository includes a [GitHub Workflow](.github/workflows/nightly.yaml)
that executes the above steps and tests that the demo works every night.
