# Server configuration with window size

```go
opts := []grpc.ServerOption {
    // This setting is for each individual stream i.e amount of data that can be sent without waiting for ack.
    grpc.InitialWindowSize(65536),    // 64KB
    // This setting is for connection level i.e governs the overall amount of data that can be sent over the connection, regardless of how many individual streams (RPCs) are active.
    grpc.InitialConnWindowSize(1 << 20), // 1MB,
    // This limits the number of concurrent streams per connection, which can help manage resource usage.
    grpc.MaxConcurrentStreams(1000),
}
```

* Pros: High throughput. Allows more data to be sent without waiting for ack.
* Cons: Too large, can cause excessive memory usage or overwhelm slower receivers.

# gRPC status codes
* https://grpc.github.io/grpc/core/md_doc_statuscodes.html  

# Flow control: Dynamic rate limiting (Simple)
* Adjust the sending rate based on the frequency of flow control errors.

```go
if err != nil {
    if err == io.EOF {
        return nil // Client has closed the stream
    }
    // The service is currently unavailable. This is most likely a transient condition, which can be corrected by retrying with a backoff. Note that it is not always safe to retry non-idempotent operations.
    if status.Code(err) == codes.Unavailable {
        // Flow control: back off and retry
        log.Println("Flow control: backing off...")
        // This gives the client time to process data and update its receive window.
        time.Sleep(100 * time.Millisecond)
        continue
    }
    return err
}
```

# Flow control: Dynamic rate limiting (Maintain client state)

<details>
<summary>Expand</summary>

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"sync"
	"time"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	pb "path/to/your/proto"
)

type logServer struct {
	pb.UnimplementedLogServiceServer
	mu           sync.Mutex
	clientStates map[string]*clientState
}

type clientState struct {
	lastSentTimestamp int64
	backoffDuration   time.Duration
}

func (s *logServer) StreamLogs(req *pb.LogRequest, stream pb.LogService_StreamLogsServer) error {
	clientID := fmt.Sprintf("%v", stream.Context().Value("client-id"))
	log.Printf("Received log streaming request from client %s for type: %v", clientID, req.LogType)

	s.mu.Lock()
	if s.clientStates == nil {
		s.clientStates = make(map[string]*clientState)
	}
	if _, exists := s.clientStates[clientID]; !exists {
		s.clientStates[clientID] = &clientState{backoffDuration: 10 * time.Millisecond}
	}
	s.mu.Unlock()

	for {
		select {
		case <-stream.Context().Done():
			return status.Error(codes.Canceled, "Client canceled the stream")
		default:
			// Simulate log generation
			logEntry := &pb.LogEntry{
				Timestamp: time.Now().Unix(),
				Message:   fmt.Sprintf("Log message for type %v", req.LogType),
			}

			// Check if enough time has passed since the last successful send
			s.mu.Lock()
			clientState := s.clientStates[clientID]
			if time.Now().Unix()-clientState.lastSentTimestamp < int64(clientState.backoffDuration.Seconds()) {
				s.mu.Unlock()
				time.Sleep(10 * time.Millisecond)
				continue
			}
			s.mu.Unlock()

			// Attempt to send the log entry
			err := stream.Send(logEntry)
			if err != nil {
				if err == io.EOF {
					return nil // Client has closed the stream
				}
				if status.Code(err) == codes.Unavailable {
					// Flow control: increase backoff
					s.mu.Lock()
					clientState.backoffDuration *= 2
					if clientState.backoffDuration > 5*time.Second {
						clientState.backoffDuration = 5 * time.Second
					}
					s.mu.Unlock()
					log.Printf("Flow control: backing off for client %s, new duration: %v", clientID, clientState.backoffDuration)
					time.Sleep(clientState.backoffDuration)
					continue
				}
				return err
			}

			// Successful send: update last sent timestamp and reduce backoff
			s.mu.Lock()
			clientState.lastSentTimestamp = time.Now().Unix()
			clientState.backoffDuration /= 2
			if clientState.backoffDuration < 10*time.Millisecond {
				clientState.backoffDuration = 10 * time.Millisecond
			}
			s.mu.Unlock()

			// Simulate some processing time
			time.Sleep(50 * time.Millisecond)
		}
	}
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	opts := []grpc.ServerOption{
		grpc.InitialWindowSize(65536),       // 64KB
		grpc.InitialConnWindowSize(1 << 20), // 1MB
	}

	s := grpc.NewServer(opts...)
	pb.RegisterLogServiceServer(s, &logServer{})

	log.Println("Starting gRPC server on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```
</details>


# Flow control: Dynamic rate limiting: Connection limiting
```go
type logServer struct {
    // ...
    activeClients   int32
    maxClients      int32
}

if atomic.LoadInt32(&s.activeClients) >= s.maxClients {
    return status.Error(codes.ResourceExhausted, "Server at maximum capacity")
}
```

<details> 
<summary>Expand code</summary>

```go

package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"sync"
	"sync/atomic"
	"time"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	pb "path/to/your/proto"
)

type logServer struct {
	pb.UnimplementedLogServiceServer
	mu              sync.Mutex
	clientStates    map[string]*clientState
	activeClients   int32
	maxClients      int32
}

type clientState struct {
	lastSentTimestamp int64
	backoffDuration   time.Duration
}

func (s *logServer) StreamLogs(req *pb.LogRequest, stream pb.LogService_StreamLogsServer) error {
	// Check if we've reached the maximum number of clients
	if atomic.LoadInt32(&s.activeClients) >= s.maxClients {
		return status.Error(codes.ResourceExhausted, "Server at maximum capacity")
	}

	// Increment the active client count
	atomic.AddInt32(&s.activeClients, 1)
	defer atomic.AddInt32(&s.activeClients, -1)

	clientID := fmt.Sprintf("%v", stream.Context().Value("client-id"))
	log.Printf("Received log streaming request from client %s for type: %v", clientID, req.LogType)

	s.mu.Lock()
	if s.clientStates == nil {
		s.clientStates = make(map[string]*clientState)
	}
	if _, exists := s.clientStates[clientID]; !exists {
		s.clientStates[clientID] = &clientState{backoffDuration: 10 * time.Millisecond}
	}
	s.mu.Unlock()

	for {
		select {
		case <-stream.Context().Done():
			return status.Error(codes.Canceled, "Client canceled the stream")
		default:
			// ... (rest of the StreamLogs function remains the same)
		}
	}
}

func (s *logServer) monitorServerLoad() {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		activeClients := atomic.LoadInt32(&s.activeClients)
		log.Printf("Current active clients: %d/%d", activeClients, s.maxClients)
		
		// Here you could add logic to adjust maxClients based on system resources
		// For example:
		// if systemLoad() > 0.8 {
		//     atomic.AddInt32(&s.maxClients, -10)
		// } else if systemLoad() < 0.6 {
		//     atomic.AddInt32(&s.maxClients, 10)
		// }
	}
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	opts := []grpc.ServerOption{
		grpc.InitialWindowSize(65536),       // 64KB
		grpc.InitialConnWindowSize(1 << 20), // 1MB
		grpc.MaxConcurrentStreams(1000),     // Adjust based on your requirements
	}

	server := &logServer{
		maxClients: 10000, // Set this based on your system's capacity
	}

	s := grpc.NewServer(opts...)
	pb.RegisterLogServiceServer(s, server)

	go server.monitorServerLoad()

	log.Println("Starting gRPC server on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```
</details>

# Flow control: Buffering
1. Set the max buffer size.
2. Determine how often you want to flush the buffer. If the buffer reaches its maximum size, it triggers a flush.


<details> 
<summary>Expand code</summary>

```go

package main

import (
	"context"
	"fmt"
	"log"
	"sync"

	pb "path/to/your/proto"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type server struct {
	pb.UnimplementedJobWorkerServer
	logStore     LogStore
	clientStreams map[string]map[string]pb.JobWorker_StreamLogsServer
	streamsMu    sync.RWMutex
}

func (s *server) StreamLogs(req *pb.StreamLogsRequest, stream pb.JobWorker_StreamLogsServer) error {
	jobID := req.GetJobId()
	clientID := fmt.Sprintf("%p", stream) // Use pointer address as a unique client identifier

	// Fetch existing logs
	logs, err := s.logStore.GetLogs(jobID)
	if err != nil {
		return status.Errorf(codes.Internal, "failed to get logs: %v", err)
	}

	// Send existing logs
	for _, logEntry := range logs {
		if err := stream.Send(&pb.StreamLogsResponse{LogEntry: logEntry}); err != nil {
			return status.Errorf(codes.Internal, "failed to send log entry: %v", err)
		}
	}

	// Get or create log channel
	logCh, err := s.logStore.GetOrCreateLogChannel(jobID)
	if err != nil {
		return status.Errorf(codes.Internal, "failed to get log channel: %v", err)
	}

	// Register this client's stream
	s.addStream(jobID, clientID, stream)
	defer s.removeStream(jobID, clientID)

	// Context for handling client disconnection
	ctx := stream.Context()

	// Stream new logs
	for {
		select {
		case logEntry, ok := <-logCh:
			if !ok {
				// Channel closed, job finished
				return status.Error(codes.OK, "job finished")
			}
			if err := stream.Send(&pb.StreamLogsResponse{LogEntry: logEntry}); err != nil {
				log.Printf("failed to send log entry to client %s: %v", clientID, err)
				return status.Errorf(codes.Internal, "failed to send log entry: %v", err)
			}
		case <-ctx.Done():
			// Client disconnected
			return status.Error(codes.Canceled, "client disconnected")
		}
	}
}

func (s *server) addStream(jobID, clientID string, stream pb.JobWorker_StreamLogsServer) {
	s.streamsMu.Lock()
	defer s.streamsMu.Unlock()

	if s.clientStreams == nil {
		s.clientStreams = make(map[string]map[string]pb.JobWorker_StreamLogsServer)
	}
	if s.clientStreams[jobID] == nil {
		s.clientStreams[jobID] = make(map[string]pb.JobWorker_StreamLogsServer)
	}
	s.clientStreams[jobID][clientID] = stream
}

func (s *server) removeStream(jobID, clientID string) {
	s.streamsMu.Lock()
	defer s.streamsMu.Unlock()

	if streams, ok := s.clientStreams[jobID]; ok {
		delete(streams, clientID)
		if len(streams) == 0 {
			delete(s.clientStreams, jobID)
		}
	}
}

type LogStore interface {
	GetLogs(jobID string) ([]*pb.LogEntry, error)
	GetOrCreateLogChannel(jobID string) (<-chan *pb.LogEntry, error)
}
```
</details>

# Flow control: Ring buffer and a channel


# Flow control: Batching
* Group multiple log entries into a single message to reduce overhead.

# Flow control: Client feedback 
* Implement a bidirectional stream where clients can send feedback about their processing capacity.

# Client sides considerations
1. Ensure receive buffers are large enough to handle the incoming data rate. How? 

