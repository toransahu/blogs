# Building a basic microservice with gRPC using Golang - Part 1

## What is gRPC?

[gRPC](https://grpc.io/) - Remote Procedure Call

- gRPC is a high performance, open source universal RPC Framework. 
- It enables the server and client applications to communicate transparently and build connected systems. 
- gRPC is developed and open sourced by Google (but no, the [g](https://github.com/grpc/grpc/blob/master/doc/g_stands_for.md) doesn’t stand for Google).

## Why use gRPC?

- Better Design
    - With gRPC we can define our service once in a .proto file and implement clients and servers in any of gRPC’s supported languages
    - Ability to auto-generate and publish SDKs as opposed to publishing the APIs for services
- High Performance
    - Advantages of working with protocol buffers, including efficient serialization, a simple IDL, and easy interface updating
    - Advantages of improved features of HTTP/2
        - *Multiplexing*: This forces service-client to utilise a single TCP connection to simultaneously handle multiple requests.
        - *Binary Framing and Compression*
- Multi-way communication
    - Simple/Uni-ary RPC
    - Server-side streaming RPC
    - Client-side streaming RPC
    - Bidirectional streaming RPC

## Where to use gRPC?

The “where” is pretty easy: we can leverage gRPC almost anywhere we have two computers communicating over a network:

- Microservices
- Client-Server Applications
- Integrations and APIs
- Browser-based Web Applications

## How to use gRPC?
Our example is a simple *“Stack Machine”* as a service that lets clients perform operations like, `PUSH`, `ADD`, `SUB`, `MUL`, `DIV`, `FIBB`, `AP`, `GP`. 

In Part-1 we’ll focus on Simple RPC implementation, in Part-2 we’ll focus on Server-side & Client-side streaming RPC, and in Part-3 we’ll implement Bidirectional streaming RPC.

### Prerequisites

#### Go 

- Version 1.6 or higher.
- For installation instructions, see Go’s Getting Started guide.

#### gRPC

Use the following command to install gRPC.

```
~/disk/E/workspace/grpc-eg-go
$ go get -u google.golang.org/grpc
```

#### Protocol Buffers v3

- Install the protoc compiler that is used to generate gRPC service code. (https://developers.google.com/protocol-buffers/)

```
~/disk/E/workspace/grpc-eg-go
$ go get -u github.com/golang/protobuf/proto
```

- Update the environment variable PATH to include the path to the protoc binary file.
- Install the protoc plugin for Go

```
~/disk/E/workspace/grpc-eg-go
$ go get -u github.com/golang/protobuf/protoc-gen-go
```

#### Setting Project Structure

```
~/disk/E/workspace/grpc-eg-go
$ go mod init github.com/toransahu/grpc-eg-go
$ mkdir machine
$ mkdir server
$ mkdir client

$ tree
.
├── client/
├── go.mod
├── machine/
└── server/
```

### Defining the service
Our first step is to define the gRPC service and the method request and response types using [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview).

To define a service, we specify a named service in our `.proto` file:

```
service Machine {
   ...
}
```


Then we define a Simple RPC method inside our service definition, specifying their request and response types. 

- A simple RPC where the client sends a request to the server using the stub and waits for a response to come back

```
// Execute accepts a set of Instructions from client and returns a Result.
rpc Execute(InstructionSet) returns (Result) {}
```


- `.proto` file also contains protocol buffer message type definitions for all the request and response types used in our service methods. 

```
// Result represents the output of execution of the instruction(s).
message Result {
  float output = 1;
}
```

Our `.proto` file should look like [this](https://github.com/toransahu/grpc-eg-go/commit/58c3b4a7962030a9998836e879d230979a906b74) considering Part-1 of this blog series.


### Generating client and server code

We need to generate the gRPC client and server interfaces from our `.proto` service definition. 

```
~/disk/E/workspace/grpc-eg-go
$ SRC_DIR=./
$ DST_DIR=$SRC_DIR
$ protoc \
  -I=$SRC_DIR \
  --go_out=plugins=grpc:$DST_DIR \
  $SRC_DIR/machine/machine.proto
```

Running this command generates the following file in the machine directory under the repository:

```
~/disk/E/workspace/grpc-eg-go
$ tree machine/
.
├── machine/
│   ├── machine.pb.go
│   └── machine.proto
```

### Server
Let's create the server.

There are two parts to making our Machine service do its job:
- Create `server/machine.go`:
    Implementing the service interface generated from our service definition; writing the business logic of our service.
- Running the Machine gRPC server:
    Run the server to listen for requests from clients and dispatch them to the right service implementation.

Have a look how our `MachineServer` interface should look like: [grpc-eg-go/server/machine.go](https://github.com/toransahu/grpc-eg-go/commit/63604cf8cd1711d1559e5e34540b23a0e4db3cb0)

```
type MachineServer struct{}

// Execute runs the set of instructions given.
func (s *MachineServer) Execute(ctx context.Context, instructions *machine.InstructionSet) (*machine.Result, error) {
    return nil, status.Error(codes.Unimplemented, "Execute() not implemented yet")
}
```


#### Implementing Simple RPC
`MachineServer` implements only `Execute()` service method as of now - as per Part-1 of this blog series.
`Execute()`, just gets a `InstructionSet` from the client and returns the value in a `Result` by executing every `Instruction` in the `InstructionSet` into our *Stack Machine*.

Before implementing `Execute()`, let's implement a basic `Stack`. It should look like [this](https://github.com/toransahu/grpc-eg-go/commit/6951692c4fafa4e818f6680d180fd136c619ef3b).

```
type Stack []float32

func (s *Stack) IsEmpty() bool {
    return len(*s) == 0
}

func (s *Stack) Push(input float32) {
    *s = append(*s, input)
}

func (s *Stack) Pop() (float32, bool) {
    if s.IsEmpty() {
        return -1.0, false
    }
    item := (*s)[len(*s)-1]
    *s = (*s)[:len(*s)-1]
    return item, true
}
```

Now, let’s implement the `Execute()`. It should look like [this](https://github.com/toransahu/grpc-eg-go/commit/77af4f0ac3a8840d6e5dcae2b8e42a575b456fe4).

```
type OperatorType string

const (
    PUSH OperatorType   = "PUSH"
    POP               = "POP"
    ADD               = "ADD"
    SUB               = "SUB"
    MUL               = "MUL"
    DIV               = "DIV"
)

type MachineServer struct{}

// Execute runs the set of instructions given.
func (s *MachineServer) Execute(ctx context.Context, instructions *machine.InstructionSet) (*machine.Result, error) {
    if len(instructions.GetInstructions()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "No valid instructions received")
    }

    var stack stack.Stack

    for _, instruction := range instructions.GetInstructions() {
        operand := instruction.GetOperand()
        operator := instruction.GetOperator()
        op_type := OperatorType(operator)

        fmt.Printf("Operand: %v, Operator: %v", operand, operator)

        switch op_type {
        case PUSH:
            stack.Push(float32(operand))
        case POP:
            stack.Pop()
        case ADD, MUL, DIV:
            item2, popped := stack.Pop()
            item1, popped := stack.Pop()

            if !popped {
                return &machine.Result{}, status.Error(codes.Aborted, "Invalide sets of instructions. Execution aborted")
            }

            if op_type == ADD {
                stack.Push(item1 + item2)
            } else if op_type == MUL {
                stack.Push(item1 * item2)
            } else if op_type == DIV {
                stack.Push(item1 / item2)
            }

        default:
            return nil, status.Errorf(codes.Unimplemented, "Operation '%s' not implemented yet", operator)
        }

    }

    item, popped := stack.Pop()
    if !popped {
        return &machine.Result{}, status.Error(codes.Aborted, "Invalide sets of instructions. Execution aborted")
    }
    return &machine.Result{Output: item}, nil
}
```

We have implemented the `Execute()` to handle basic instructions like `PUSH`, `POP`, `ADD`, `SUB`, `MUL`, and `DIV` with proper error handling. On completion of the execution of instructions set, it pops the result from `Stack` and returns as a `Result` object to the client.



#### Code to run the gRPC server

To run the gRPC server we need to:
- Create a new instance of gRPC struct and make it listen to one of the TCP ports at our localhost address. As a convention default port selected for gRPC is 9111.
- In order to serve our `StackMachine` service over gRPC server, we need to register the service with the newly created gRPC server.

For the development purpose, the basic insecure code to run the gRPC server should look like [this](https://github.com/toransahu/grpc-eg-go/commit/c39d5675ca47e7778cd410a0eb24ccd1f9dd4542). 

```
var (
    port = flag.Int("port", 9111, "Port on which gRPC server should listen TCP conn.")
)

func main() {
    flag.Parse()
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    grpcServer := grpc.NewServer()
    machine.RegisterMachineServer(grpcServer, &server.MachineServer{})
    grpcServer.Serve(lis)
    log.Printf("Initializing gRPC server on port %d", *port)
}
```

We must consider strong TLS based security for our production environment. I’ll try planning to include an example of TLS implementation to this blog series.

### Client
As we already know that the same `.proto` file, which is our [IDL](https://en.wikipedia.org/wiki/Interface_description_language) (Interface Definition Language) is capable of generating interfaces for clients as well. One has to just implement those interfaces to communicate with the gRPC server.

With having the `.proto` either service provider can implement an SDK, or consumer/customer of the service itself can implement a client in desired programming language.

Lets implement our version of a basic client code, which will call the `Execute()` method of the service. The client should look like [this](https://github.com/toransahu/grpc-eg-go/commit/5aa509cc8a1cffa7ffcc126fd2a30e85d350fa14).

```
var (
    serverAddr = flag.String("server_addr", "localhost:9111", "The server address in the format of host:port")
)

func runExecute(client machine.MachineClient, instructions *machine.InstructionSet) {
    log.Printf("Executing %v", instructions)
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    result, err := client.Execute(ctx, instructions)
    if err != nil {
        log.Fatalf("%v.Execute(_) = _, %v: ", client, err)
    }
    log.Println(result)
}

func main() {
    flag.Parse()
    var opts []grpc.DialOption
    opts = append(opts, grpc.WithInsecure())
    opts = append(opts, grpc.WithBlock())
    conn, err := grpc.Dial(*serverAddr, opts...)
    if err != nil {
        log.Fatalf("fail to dial: %v", err)
    }
    defer conn.Close()
    client := machine.NewMachineClient(conn)

    // try Execute()
    instructions := []*machine.Instruction{}
    instructions = append(instructions, &machine.Instruction{Operand: 5, Operator: "PUSH"})
    instructions = append(instructions, &machine.Instruction{Operand: 6, Operator: "PUSH"})
    instructions = append(instructions, &machine.Instruction{Operator: "MUL"})
    runExecute(client, &machine.InstructionSet{Instructions: instructions})
}
```

### Test

#### Server
Let's write a unit test to validate our business logic of `Execute()` method. 
- Create a test file `server/machine_test.go`
- Write the unit test, it should look like [this](https://github.com/toransahu/grpc-eg-go/commit/eab052d4a66bdcf0a1ae7fd9111687c8ff4d5113).

Run the test file.

```
~/disk/E/workspace/grpc-eg-go
$ go test server/machine.go server/machine_test.go -v
=== RUN   TestExecute
--- PASS: TestExecute (0.00s)
PASS
ok      command-line-arguments    0.004s
```

#### Client

To test client-side code without the overhead of connecting to a real server, we'll use `Mock`. Mocking enables users to write light-weight unit tests to check functionalities on client-side without invoking RPC calls to a server.

To write a unit test to validate client side business logic of calling the `Execute()` method:
- Install [golang/mock](https://github.com/golang/mock) package
- Generate mock for `MachineClient`
- Create a test file `mock/machine_mock_test.go`
- Write the unit test


As we are leveraging the [golang/mock](https://github.com/golang/mock) package, to install the package we need to run the following command:

```
~/disk/E/workspace/grpc-eg-go
$ go get github.com/golang/mock/mockgen@latest
```

To generate a mock of the `MachineClient` run the following command, the file should look like [this](https://github.com/toransahu/grpc-eg-go/commit/6e7dfc2cfdcd41bad1cd5a6a6e525cf94db33738).

```
~/disk/E/workspace/grpc-eg-go
$ mkdir mock_machine && cd mock_machine
$ mockgen github.com/toransahu/grpc-eg-go/machine MachineClient > machine_mock.go
```

Write the unit test, it should look like [this](https://github.com/toransahu/grpc-eg-go/commit/b69256d7bc6cdb41418fdb67dcf3a251072db63f).

Run the test file.

```
~/disk/E/workspace/grpc-eg-go
$ go test mock_machine/machine_mock.go  mock_machine/machine_mock_test.go -v
=== RUN   TestExecute
2020/03/28 14:50:11 output:30
--- PASS: TestExecute (0.00s)
PASS
ok      command-line-arguments    0.004s
```

### Run
As now we are assured through unit tests that business logic of the server & client codes are working as expected, let’s try running the server and communicating to it via our client code.

#### Server
To turn on the server we need to run the previously created `cmd/run_machine_server.go` file.

```
~/disk/E/workspace/grpc-eg-go
$ go run cmd/run_machine_server.go 
```

#### Client
Now, let’s run the client code `client/machine.go`.

```
~/disk/E/workspace/grpc-eg-go
$ go run client/machine.go
2020/03/28 12:29:49 Executing instructions:<operator:"PUSH" operand:5 > instructions:<operator:"PUSH" operand:6 > instructions:<operator:"MUL" >
2020/03/28 12:29:49 output:30
```

Hurray!!! It worked.

At the end of this blog, we’ve learned:
- Importance of gRPC - What, Why, Where
- How to install all the prerequisites
- How to define an interface using protobuf
- How to write gRPC server & client logic for Simple RPC
- How to write and run unit test for server & client logic
- How to run the gRPC server and a client can communicate to it

Source code of this example is available at [toransahu/grpc-eg-go](https://github.com/toransahu/grpc-eg-go). See you on the next part of this blog series.
