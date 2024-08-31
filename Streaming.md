# Server configuration

```
opts := []grpc.ServerOption{
    grpc.InitialWindowSize(65536),    // 64KB
    grpc.InitialConnWindowSize(1 << 20), // 1MB
}
```
