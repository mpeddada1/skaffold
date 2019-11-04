---
title: "Skaffold API"
linkTitle: "Skaffold API"
weight: 60
---
When running [`skaffold dev`]({{< relref "/docs/workflows/dev" >}}) or [`skaffold debug`]({{< relref "/docs/workflows/debug" >}}), 
Skaffold starts a server that exposes an API over the lifetime of the Skaffold process.
Besides the CLI, this API is the primary way tools like IDEs integrate with Skaffold for **retrieving information about the
pipeline** and for **controlling the phases in the pipeline**.

To retrieve information about the Skaffold pipeline, the Skaffold API provides two main functionalities:
  
  * A [streaming event log]({{< relref "/docs/design/api#events-api">}}) created from the different phases in a pipeline run and
  
  * A snapshot of the [overall state]({{< relref "/docs/design/api#state-api" >}}) of the pipeline at any given time during the run.

To control the individual phases of the Skaffold, the Skaffold API provides [fine grained control over]({{< relref "/docs/design/api#controlling-build-sync-deploy" >}})
the individual phases of the pipeline (build, deploy and sync).


## Connecting to the Skaffold API
The Skaffold API is `gRPC` based, and it is also exposed via the gRPC gateway as a JSON over HTTP service.
The server is hosted locally on the same host where the skaffold process is running, and will serve by default on ports 50051 and 50052.
These ports can be configured through the `--rpc-port` and `--rpc-http-port` flags.

The server's gRPC service definitions and message protos can be found [here]({{< relref "/docs/references/api" >}}).


### HTTP server
The HTTP API is exposed on port `50052` by default. The default HTTP port can be overriden with the `--rpc-http-port` flag. 
If the HTTP API port is taken, Skaffold will find the next available port.
The final port can be found from Skaffold's startup logs.

```code
$ skaffold dev
WARN[0000] port 50052 for gRPC HTTP server already in use: using 50055 instead
```

### gRPC Server

The gRPC API is exposed on port `50051` by default and can be overriden with the `--rpc-port` flag.
As with the HTTP API, if this port is taken, Skaffold will find the next available port.
You can find this port from Skaffold's logs on startup.

```code
$ skaffold dev
WARN[0000] port 50051 for gRPC server already in use: using 50053 instead
```

#### Creating a gRPC Client
To connect to the `gRPC` server at default port `50051` use the following code snippet.

{{< alert title="Note" >}}
The skaffold gRPC server is not an HTTPS service, hence we need to specify `grpc.WithInSecure()`
{{</alert>}}

```golang
import (
  "log"
  pb "github.com/GoogleContainerTools/skaffold/proto"
  "google.golang.org/grpc"
)

func main(){
  conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
  if err != nil {
    log.Fatalf("fail to dial: %v", err)
  }
  defer conn.Close()
  client := pb.NewSkaffoldServiceClient(conn)
}
```


## API structure

Skaffold's API exposes the three main endpoints:

* Events API - stream of lifecycle events
* State API - retrieve the current state
* Control API - control build/deploy/sync

### Events API

Skaffold provides a continuous development mode [`skaffold dev`]({{< relref "/docs/workflows/dev" >}}) which rebuilds and redeploys
your application on changes. In a single development loop, one or more container images
may be built and deployed.

Skaffold exposes events for clients to get notified when phases within a development loop
start, succeed or fail.
Tools that integrate with Skaffold can use these events to kick-off parts of the development workflow depending on them.

Example scenarios:

* port forwarding events are used by Cloud Code to attach debuggers automatically to running containers.     
* when a port-forwarded frontend service is redeployed successfully, kick-off a suite of Selenium tests that test changes to the newly deployed service..

**Event API contract**

| protocol | endpoint | encoding |
| ---- | --- | --- |
| HTTP | `http://localhost:{HTTP_RPC_PORT}/v1/events` | newline separated JSON using chunk transfer encoding over HTTP|  
| gRPC | `client.Events(ctx)` method on the [`SkaffoldService`]({{< relref "/docs/references/api#skaffoldservice">}}) | protobuf 3 over HTTP |  


**Examples**

{{% tabs %}}
{{% tab "HTTP API" %}}
Using `curl` and `HTTP_RPC_PORT=50052`, an example output of a `skaffold dev` execution on our [getting-started example](https://github.com/GoogleContainerTools/skaffold/tree/master/examples/getting-started)
```bash
 curl localhost:50052/v1/events
{"result":{"timestamp":"2019-10-16T18:26:11.385251549Z","event":{"metaEvent":{"entry":"Starting Skaffold: {Version:v0.39.0-16-g5bb7c9e0 ConfigVersion:skaffold/v1beta15 GitVersion: GitCommit:5bb7c9e078e4d522a5ffc42a2f1274fd17d75902 GitTreeState:dirty BuildDate:2019-10-03T15:01:29Z GoVersion:go1.13rc1 Compiler:gc Platform:linux/amd64}"}}}}
{"result":{"timestamp":"2019-10-16T18:26:11.436231589Z","event":{"buildEvent":{"artifact":"gcr.io/k8s-skaffold/skaffold-example","status":"In Progress"}},"entry":"Build started for artifact gcr.io/k8s-skaffold/skaffold-example"}}
{"result":{"timestamp":"2019-10-16T18:26:12.010124246Z","event":{"buildEvent":{"artifact":"gcr.io/k8s-skaffold/skaffold-example","status":"Complete"}},"entry":"Build completed for artifact gcr.io/k8s-skaffold/skaffold-example"}}
{"result":{"timestamp":"2019-10-16T18:26:12.391721823Z","event":{"deployEvent":{"status":"In Progress"}},"entry":"Deploy started"}}
{"result":{"timestamp":"2019-10-16T18:26:12.847239740Z","event":{"deployEvent":{"status":"Complete"}},"entry":"Deploy complete"}}
..
```
{{% /tab %}}
{{% tab "gRPC API" %}}
To get events from the `gRPC` server, first create [`gRPC` client]({{< relref "/docs/design/api#creating-a-grpc-client" >}})

```golang
func main() {
  ctx, ctxCancel := context.WithCancel(context.Background())
  defer ctxCancel()
  // `client` is the gRPC client with connection to localhost:50051.
  // See code above to create it
  logStream, err := client.Events(ctx)
  if err != nil {
  	log.Fatalf("could not get events: %v", err)
  }
  for {
  	entry, err := logStream.Recv()
  	if err == io.EOF {
  		break
  	}
  	if err != nil {
  		log.Fatal(err)
  	}
  	log.Println(entry)
  }
}
```
{{% /tab %}}
{{% /tabs %}}

Each [Entry log]({{<relref "/docs/references/api#proto.LogEntry" >}}) contains an [Event]({{< relref "/docs/references/api#proto.Event" >}}) in the `LogEntry.Event` field and
a string description of the event in `LogEntry.entry` field.


### State API

The State API provides a snapshot of the current state of the following components:

- build state per artifacts 
- deploy state
- file sync state 
- status check state per resource 
- port-forwarded resources

**Event API contract**  

| protocol | endpoint | encoding |
| ---- | --- | --- |
| HTTP | `http://localhost:{HTTP_RPC_PORT}/v1/state` | newline separated JSON using chunk transfer encoding over HTTP|  
| gRPC | `client.GetState(ctx)` method on the [`SkaffoldService`]({{< relref "/docs/references/api#skaffoldservice">}}) | protobuf 3 over HTTP |  


**Examples** 
{{% tabs %}}
{{% tab "HTTP API" %}}
Using `curl` and `HTTP_RPC_PORT=50052`, an example output of a `skaffold dev` execution on our [microservices example](https://github.com/GoogleContainerTools/skaffold/tree/master/examples/microservices)
```bash
 curl localhost:50052/v1/state | jq
 {
   "buildState": {
     "artifacts": {
       "gcr.io/k8s-skaffold/leeroy-app": "Complete",
       "gcr.io/k8s-skaffold/leeroy-web": "Complete"
     }
   },
   "deployState": {
     "status": "Complete"
   },
   "forwardedPorts": {
     "9000": {
       "localPort": 9000,
       "remotePort": 8080,
       "namespace": "default",
       "resourceType": "deployment",
       "resourceName": "leeroy-web"
     },
     "50055": {
       "localPort": 50055,
       "remotePort": 50051,
       "namespace": "default",
       "resourceType": "service",
       "resourceName": "leeroy-app"
     }
   },
   "statusCheckState": {
     "status": "Succeeded"
   },
   "fileSyncState": {
     "status": "Not Started"
   }
 }
```
{{% /tab %}}
{{% tab "gRPC API" %}}
To get events over `gRPC` server, first create [`gRPC` client]({{< relref "/docs/design/api#creating-a-grpc-client" >}})
```code
func main() {
  // Create a gRPC client connection to localhost:50051.
  // See code above
  ctx, ctxCancel := context.WithCancel(context.Background())
  defer ctxCancel()
  grpcState, err = client.GetState(ctx, &empty.Empty{})
  ...
}
```
{{% /tab %}}
{{% /tabs %}}

### Controlling Build/Sync/Deploy

TODO: https://github.com/GoogleContainerTools/skaffold/issues/3143