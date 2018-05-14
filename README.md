# Getting Started with Microservices using Go, gRPC and Kubernetes

## Prerequisites
- Install [Minikube](https://github.com/kubernetes/minikube)
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- Install [Docker](https://docs.docker.com/engine/installation/)
- Install [Protocol Buffers](https://github.com/google/protobuf)

## Run Minikube
```s
$ minikube start [--vm-driver=<driver>]
$ kubectl cluster-info
```

## To speed up the development, set docker to reuse Docker daemon running inside the Minikube virtual machine.
```s
$ docker-machine create -d virtualbox minikube
$ docker-machine env minikube
$ eval $(minikube docker-env)
```

## delete machine
```s
$ docker-machine rm minikube
```

## Defining communication protocol
Create `gcd.proto` file inside `proto` directory

### Navigate to the pb directory and run the following command.
```s
$ protoc -I . --go_out=plugins=grpc:. ./*.proto
```
Compilation should produce `gcd.proto.go` file.

## Greatest common divisor service
Create `main.go` file inside `server` directory.
```go
package main
import (
  "log"
  "net"
  // Change this for your own project
  "gcd-labs/proto"
  context "golang.org/x/net/context"
  "google.golang.org/grpc"
  "google.golang.org/grpc/reflection"
)

type server struct{}

func main() {
  lis, err := net.Listen("tcp", ":3000")
  if err != nil {
    log.Fatalf("Failed to listen: %v", err)
  }
  s := grpc.NewServer()
  proto.RegisterGCDServiceServer(s, &server{})
  reflection.Register(s)
  if err := s.Serve(lis); err != nil {
    log.Fatalf("Failed to serve: %v", err)
  }
}

// Implement service
func (s *server) Computer(ctx context.Context, r *proto.GCDRequest) (*proto.GCDResponse, error) {
  a, b := r.A, r.B
  for b != 0 {
    a, b = b, a%b
  }
  return &proto.GCDResponse{Result: a}, nil
}
```

## Frontend API service
Frontend service uses [gin](https://github.com/gin-gonic/gin) web framework to serve a REST API and calls the GCD service for the actual calculation.

Create `main.go` file inside `api` directory.
```go
package main

import (
	"fmt"
	"gcd-labs/proto"
	"log"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial("gcd-service:3000", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Fial failed: %v", err)
	}
	gcdClient := proto.NewGCDServiceClient(conn)

	r := gin.Default()
	r.GET("/gcd/:a/:b", func(c *gin.Context) {
		// Parse parameters
		a, err := strconv.ParseUint(c.Param("a"), 10, 64)
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid parameter A"})
			return
		}
		b, err := strconv.ParseUint(c.Param("b"), 10, 64)
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid parameter B"})
			return
		}
		// Call GCD service
		req := &proto.GCDRequest{A: a, B: b}
		if res, err := gcdClient.Computer(c, req); err == nil {
			c.JSON(http.StatusOK, gin.H{
				"result": fmt.Sprint(res.Result),
			})
		} else {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		}
	})

	// Run the server
	if err := r.Run(":3000"); err != nil {
		log.Fatalf("Failed to run server: %v", err)
	}
}
```

## Building Docker images
Create `Dockerfile.api`
```s
FROM golang:1.9.4

WORKDIR /go/src/gcd-labs/api
COPY api .
COPY proto ../proto

RUN go get -v ./..
RUN go install -v ./..

EXPOSE 3000

CMD ["api"]
```
And the `Dockerfile.gcd`
```s
FROM golang:1.9.4

WORKDIR /go/src/gcd-labs/server
COPY server .
COPY proto ../proto

RUN go get -v ./..
RUN go install -v ./..

EXPOSE 3000

CMD ["server"]
```

### Build both images. 
If you switched to Minikube's Docker daemon, they will become available inside the VM.
```s
$ docker build -t local/server -f Dockerfile.gcd .
$ docker build -t local/api -f Dockerfile.api .
```

### Run a Docker Image as a Container

To list the docker images
```s
$ docker images
```

If your application wants to run in 3000 port
```s
$ docker run -d --restart=always -p port:port image_name:version
```

Run `server` the following commands.
```s
$ docker run -d --restart=always -p 3000:3000 local/server:latest
```

Run `client` the following commands.
```s
$ docker run -d --restart=always -p 4000:3000 local/api:latest
```

## Deploying to Kubernetes cluster
Create `gcd.yaml` file.
```yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: gcd-deployment
  labels:
    app: gcd
spec:
  selector:
    matchLabels:
      app: gcd
  replicas: 3
  template:
    metadata:
      labels:
        app: gcd
    spec:
      containers:
      - name: gcd
        image: local/gcd
        imagePullPolicy: Never
        ports:
        - name: gcd-service
          containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: gcd-service
spec:
  selector:
    app: gcd
  ports:
  - port: 3000
    targetPort: gcd-service
```
Create `api.yaml` file. 
```yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  selector:
    matchLabels:
      app: api
  replicas: 1
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: local/api
        imagePullPolicy: Never
        ports:
        - name: api-service
          containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort
  selector:
    app: api
  ports:
  - port: 3000
    targetPort: api-service
```
To create these resources inside the cluster, run the following commands.
```s
$ kubectl create -f api.yaml
$ kubectl create -f gcd.yaml
```

Check if all pods are running. By specifying -w flag, you can watch for changes.
```s
$ kubectl get pods -w
NAME                             READY     STATUS    RESTARTS   AGE
api-deployment-778049682-3vd0z   1/1       Running   0          3s
```

Get the URL of the API service.
```s
$ minikube service api-service --url
```

Finally, try it out.
```s
$ curl http://192.168.99.100:32602/gcd/294/462
```

Build images and deploy to Kubernetes cluster:
```s
$ ./build.sh
```

Try it out.
```s
$ curl $(minikube service api-service --url)/gcd/294/462
```

## Reference
[https://outcrawl.com/getting-started-microservices-go-grpc-kubernetes/](https://outcrawl.com/getting-started-microservices-go-grpc-kubernetes/)