# gRPC
Experimenting/learning with gRPC

# Install
```
// https://grpc.io/docs/languages/go/quickstart/

brew install protobuf


go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
export PATH="$PATH:$(go env GOPATH)/bin"
```

# How to use protoc
```
protoc --go_out=. --go-grpc_out=. path/to/your/file.proto
```

# 

1. First write your gRPC service
    * Write your contract (what your service does?)
    * Request entity
    * Response entity
2. Generate RPC code for your RPC service using protoc. This should generate 2 files
    * _grpc.pb.go
    * .pb.go
    * This will generate : 
        1. NewGreeterClient implementing the service Greeter
        2. Server: Mapping of your methods to the handlers that will do the work
3. Write your server code     
    1. Take dependency of the gRPC service
    2. Start your gRPC service
        1. Listen on a port
        2. Start a new gRPC server
        3. Register your service with the gRPC server 
        4. Start listening on the port on step #1 with the server on step #2
```go
  net.Listen("tcp", ...)   
  s := grpc.NewServer()
  pb.RegisterGreeterServer(s, &server{})
  s.Serve(lis)
```  
    3. Implement the Service and can invoke the server method.
4. Write your client code 
    1. Start a new connection using grpc.NewClient(....) with the server running
    2. Take dependency of the gRPC service. Call that to initiate a new client.