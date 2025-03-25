# MetalBear Playground

This repository contains different microservices and Kubernetes manifests to deploy them.
Each microservice has it's own `app.yaml` that should contain all of it's dependencies (besides other microservices).

## Setting up

### SQS

To enable SQS:

1. Install mirrord Operator in cluster (with SQS splitting enabled)
2. `aws iam create-user --user-name SQSPlayground`
3. `aws iam create-access-key --user-name SQSPlayground` - save data to file
4. `aws sqs create-queue --queue-name IpCount` - take QueueUrl to be used in deployment.yaml
5. You need to edit `ip-visit-sqs-consumer/policy.json` and set REGION and ACCOUNT_ID
6. `aws iam create-policy --policy-name SQSPlaygroundPolicy --policy-document file://ip-visit-sqs-consumer/policy.json`
7. `aws iam attach-user-policy --policy-arn arn:aws:iam::526936346962:policy/SQSPlaygroundPolicy --user-name SQSPlayground`
8. Set Region in app.yaml in `ip-visit-counter` and `ip-visit-sqs-consumer`


### Proto

To build proto:

```bash
cd proto
protoc --go_out=../protogen --go_opt=paths=source_relative \
        --go-grpc_out=../protogen --go-grpc_opt=paths=source_relative ./ipinfo.proto
```

### minikube/ local cluster

To set up the playground on minikube/ a local cluster:

```bash
kubectl apply -k ./base/local
```

## Playing on the playground

If you're not sure where to start after you have everything set up, try running the `ip-visit-counter` microservice locally
with mirrord, using this config:

```json
{
    "feature": {
        "network": {
            "incoming": {
                "mode": "steal",
                "http_filter": {
                    "header_filter": "X-PG-Tenant: YourNameHere"
                }
            },
            "outgoing": true
        },
        "fs": "read",
        "env": true
    },
    "target": "deployment/ip-visit-counter"
}
```

If you're using an IDE plugin, you can set some breakpoints in the `main` function in [`ip-visit-counter`](/ip-visit-counter/main.go)
to see various mirrord features at play while it completes setup, like fetching remote env vars, reading from the remote file system,
resolving DNS remotely, etc.

Once it's running, you can curl the `/count` endpoint and optionally, if you put a breakpoint somewhere in the `getCount`
function, trip the breakpoint. For example on a local cluster, the curl request would look something like this: 

```bash
curl --header "X-PG-Tenant: YourNameHere" localhost:PORT_NUMBER/count
```

You can find the required port number by running `kubectl get svc` and looking for the `ip-visit-counter` NodePort.

Note that the `X-PG-Tenant` header in the curl request and `feature.network.incoming.http_filter` configuration are measures
to prevent you from stealing someone else's traffic on the same cluster, so if it's just you and a local cluster you won't
need them (although it _does_ prevent mirrord from stealing health checks to the target, which can be useful).