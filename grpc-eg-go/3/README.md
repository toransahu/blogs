<h3>Table of Content</h3>

[TOC]

<h1>Building a basic microservice with bidirectional-streaming gRPC using Golang</h1>

If you have been through part-2 of this blog series, you would have already got to know that the gRPC framework has got support for uni-directional streaming RPCs. But that is not the end. gRPC has support for bi-directional RPCs as well. Being said that, a gRPC client and a gRPC server can stream requests and responses simultaneously utilizing the same TCP connection.

## Objective

In this blog, you'll get to know what is bi-directional streaming RPCs. How to implement, test, and run them using a live, fully functional example.

> Previously in the [part-2]() of this [blog series](), we've learned the basics of uni-directional streaming gRPCs, how to implement those gRPC, how to write unit tests, how to launch the server & client. part-2 walks you through a step-by-step guide to implement a _Stack Machine_ server & client leveraging the uni-directional streaming RPC.

> If you've missed that, it is highly recommended to go through it to get familiar with the basics of the gRPC framework & streaming RPCs.

## Introduction

Let's understand how bi-directional streaming RPCs works at a very high-level.

Bidirectional streaming RPCs where:

- both sides send a sequence of messages using a read-write stream
- the two streams operate independently, so clients and servers can read and write in whatever order they like
- for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes

The best thing is, the order of messages in each stream is preserved.

Now let's improve the _"Stack Machine"_ server & client codes to support bidirectional streaming.

## Implementing Bidirectional Streaming RPC

We already have Server-streaming RPC `ServerStreamingExecute()` to handle `FIB` operation which streams the numbers from Fibonacci series, and Client-streaming RPC `Execute()` to handle the stream of instructions to perform basic `ADD/SUB/MUL/DIV` operations and return a single response.

In this blog we'll merge both the functionality to make the `Execute()` RPC a bidirectional streaming one.

### Update the protobuf

Let's update the `machine/machine.proto` to make Execute() a bi-directional (server & client streaming) RPC, so that the client can stream the instructions rather than sending a set of instructions to the server & the server can respond with a stream of results. Doing so, we're getting rid of `InstructionSet` and `ServerStreamingExecute()`.

The updated `machine/machine.proto` now looks like:

```proto
 service Machine {
-     rpc Execute(stream Instruction) returns (Result) {}
-     rpc ServerStreamingExecute(InstructionSet) returns (stream Result) {}
+     rpc Execute(stream Instruction) returns (stream Result) {}
 }
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/e0042dc83d2d34ad15af284a2d5c3318f2f18eeb">machine/machine.proto</a></center>


Notice the `stream` keyword at two places in the `Execute()` RPC declaration.

### Generating the updated client and server interface Go code

Now lets generate an updated golang code from the `machine/machine.proto` by running:

```
~/disk/E/workspace/grpc-eg-go
$ SRC_DIR=./
$ DST_DIR=$SRC_DIR
$ protoc \
  -I=$SRC_DIR \
  --go_out=plugins=grpc:$DST_DIR \
  $SRC_DIR/machine/machine.proto
```

You'll notice that declaration of `ServerStreamingExecute()` has been removed from `MachineServer` & `MachineClient` interfaces. However the signature of `Execute()` is intact.

```go
...

 type MachineServer interface {
 	Execute(Machine_ExecuteServer) error
- 	ServerStreamingExecute(*InstructionSet, Machine_ServerStreamingExecuteServer) error
 }

...

 type MachineClient interface {
	Execute(ctx context.Context, opts ...grpc.CallOption) (Machine_ExecuteClient, error)
-	ServerStreamingExecute(ctx context.Context, in *InstructionSet, opts ...grpc.CallOption) (Machine_ServerStreamingExecuteClient, error)
 }

...
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/f03265623e0b1b037e831a34d09e104f62955a89">machine/machine.pb.go</a></center>

### Update the Server

Now we need to update the server code to make `Execute()` a bi-directional (server & client streaming) RPC so that it should be able to accept stream the instructions from the client and at the same time it can respond with a stream of results.

```go
func (s *MachineServer) Execute(stream machine.Machine_ExecuteServer) error {
	var stack stack.Stack
	for {
		instruction, err := stream.Recv()
		if err == io.EOF {
			log.Println("EOF")
			return nil
		}
		if err != nil {
			return err
		}

		operand := instruction.GetOperand()
		operator := instruction.GetOperator()
		op_type := OperatorType(operator)

		fmt.Printf("Operand: %v, Operator: %v\n", operand, operator)

		switch op_type {
		case PUSH:
			stack.Push(float32(operand))
		case POP:
			stack.Pop()
		case ADD, SUB, MUL, DIV:
			item2, popped := stack.Pop()
			item1, popped := stack.Pop()

			if !popped {
				return status.Error(codes.Aborted, "Invalid sets of instructions. Execution aborted")
			}

			var res float32
			if op_type == ADD {
				res = item1 + item2
			} else if op_type == SUB {
				res = item1 - item2
			} else if op_type == MUL {
				res = item1 * item2
			} else if op_type == DIV {
				res = item1 / item2
			}

			stack.Push(res)
			if err := stream.Send(&machine.Result{Output: float32(res)}); err != nil {
				return err
			}
		case FIB:
			n, popped := stack.Pop()

			if !popped {
				return status.Error(codes.Aborted, "Invalid sets of instructions. Execution aborted")
			}

			if op_type == FIB {
				for f := range utils.FibonacciRange(int(n)) {
					if err := stream.Send(&machine.Result{Output: float32(f)}); err != nil {
						return err
					}
				}
			}
		default:
			return status.Errorf(codes.Unimplemented, "Operation '%s' not implemented yet", operator)
		}
	}
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/a4e39d671a604d135822868ed7169402c19d0c1d">server/machine.go</a></center>

### Update the Client

Let's update the client code to make `client.Execute()` a bi-directional streaming RPC so that the client can stream the instructions to the server and can receive a stream of results at the same time.

```go
func runExecute(client machine.MachineClient, instructions []*machine.Instruction) {
	log.Printf("Streaming %v", instructions)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	stream, err := client.Execute(ctx)
	if err != nil {
		log.Fatalf("%v.Execute(ctx) = %v, %v: ", client, stream, err)
	}
	waitc := make(chan struct{})
	go func() {
		for {
			result, err := stream.Recv()
			if err == io.EOF {
				log.Println("EOF")
				close(waitc)
				return
			}
			if err != nil {
				log.Printf("Err: %v", err)
			}
			log.Printf("output: %v", result.GetOutput())
		}
	}()

	for _, instruction := range instructions {
		if err := stream.Send(instruction); err != nil {
			log.Fatalf("%v.Send(%v) = %v: ", stream, instruction, err)
		}
		time.Sleep(500 * time.Millisecond)
	}
	if err := stream.CloseSend(); err != nil {
		log.Fatalf("%v.CloseSend() got error %v, want %v", stream, err, nil)
	}
	<-waitc
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/fe079ba56b348c5ce2e465501be5ee6093bdf5c3">client/machine.go</a></center>

### Test

Before we start updating the unit test, let's generate mocks for `MachineClient`, `Machine_ExecuteClient`, and `Machine_ExecuteServer` interfaces to mock the `stream` type while testing the bidirectional streaming RPC `Execute()`.

```
~/disk/E/workspace/grpc-eg-go
$ mockgen github.com/toransahu/grpc-eg-go/machine MachineClient,Machine_ExecuteServer,Machine_ExecuteClient > mock_machine/machine_mock.go
```

The updated `mock_machine/machine_mock.go` should look like [this](https://github.com/toransahu/grpc-eg-go/commit/85a45cec309e3cabb58ca9e46f64fe669b899cf1).

Now, we're good to write a unit test for bidirectional streaming RPC `Execute()`.

#### Server

Let's update the unit test to test the server-side logic of `Execute()` RPC for bidirectional streaming using mock:

```go
func TestExecute(t *testing.T) {
	s := MachineServer{}

	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockServerStream := mock_machine.NewMockMachine_ExecuteServer(ctrl)

	mockResults := []*machine.Result{}
	callRecv1 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operand: 1, Operator: "PUSH"}, nil)
	callRecv2 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operand: 2, Operator: "PUSH"}, nil).After(callRecv1)
	callRecv3 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operator: "MUL"}, nil).After(callRecv2)
	callRecv4 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operand: 3, Operator: "PUSH"}, nil).After(callRecv3)
	callRecv5 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operator: "ADD"}, nil).After(callRecv4)
	callRecv6 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operator: "FIB"}, nil).After(callRecv5)
	mockServerStream.EXPECT().Recv().Return(nil, io.EOF).After(callRecv6)
	mockServerStream.EXPECT().Send(gomock.Any()).DoAndReturn(
		func(result *machine.Result) error {
			mockResults = append(mockResults, result)
			return nil
		}).AnyTimes()
	wants := []float32{2, 5, 0, 1, 1, 2, 3, 5}

	err := s.Execute(mockServerStream)
	if err != nil {
		t.Errorf("Execute(%v) got unexpected error: %v", mockServerStream, err)
	}
	for i, result := range mockResults {
		got := result.GetOutput()
		want := wants[i]
		if got != want {
			t.Errorf("got %v, want %v", got, want)
		}
	}
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/01bd6dfcebd100f1c027ed13e8e7d52871bc8aff">server/machine_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test server/machine.go server/machine_test.go
ok      command-line-arguments  0.004s
```

#### Client

Now, add unit test to test client-side logic of `Execute()` RPC for bidirectional streaming using mock:

```go
func testExecute(t *testing.T, client machine.MachineClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	instructions := []*machine.Instruction{}
	instructions = append(instructions, &machine.Instruction{Operand: 5, Operator: "PUSH"})
	instructions = append(instructions, &machine.Instruction{Operand: 6, Operator: "PUSH"})
	instructions = append(instructions, &machine.Instruction{Operator: "MUL"})

	stream, err := client.Execute(ctx)
	if err != nil {
		log.Fatalf("%v.Execute(%v) = _, %v: ", client, ctx, err)
	}
	for _, instruction := range instructions {
		if err := stream.Send(instruction); err != nil {
			log.Fatalf("%v.Send(%v) = %v: ", stream, instruction, err)
		}
	}
	result, err := stream.Recv()
	if err != nil {
		log.Fatalf("%v.Recv() got error %v, want %v", stream, err, nil)
	}

	got := result.GetOutput()
	want := float32(30)
	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
}

func TestExecute(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockMachineClient := mock_machine.NewMockMachineClient(ctrl)

	mockClientStream := mock_machine.NewMockMachine_ExecuteClient(ctrl)
	mockClientStream.EXPECT().Send(gomock.Any()).Return(nil).AnyTimes()
	mockClientStream.EXPECT().Recv().Return(&machine.Result{Output: 30}, nil)

	mockMachineClient.EXPECT().Execute(
		gomock.Any(), // context
	).Return(mockClientStream, nil)

	testExecute(t, mockMachineClient)
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/7a5160c2b9aa6ad038d47225cd14321ef05fd019">mock_machine/machine_mock_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test mock_machine/machine_mock.go mock_machine/machine_mock_test.go
ok      command-line-arguments  0.003s
```

### Run

Now we are assured through unit tests that the business logic of the server & client codes is working as expected, let’s try running the server and communicating to it via our client code.

#### Server

To spin up the server we need to run the previously created `cmd/run_machine_server.go` file.

```
~/disk/E/workspace/grpc-eg-go
$ go run cmd/run_machine_server.go
```

#### Client

Now, let’s run the client code `client/machine.go`.

```
~/disk/E/workspace/grpc-eg-go
$ go run client/machine.go
Streaming [operator:"PUSH" operand:1  operator:"PUSH" operand:2  operator:"ADD"  operator:"PUSH" operand:3  operator:"DIV"  operator:"PUSH" operand:4  operator:"MUL"  operator:"FIB"  operator:"PUSH" operand:5  operator:"PUSH" operand:6  operator:"SUB" ]
output: 3
output: 1
output: 4
output: 0
output: 1
output: 1
output: 2
output: 3
output: -1
EOF
```

## Bonus

There are situations when one has to choose between _mocking a dependency_ versus _incorporating the dependencies_ into the test environment & running them live\_.

The decision (whether to mock or not) could be made based on:

1. how many dependencies are there
1. which are the essential & most used dependencies
1. is it feasible to install dependencies on the test (and even the developer's) environment
1. etc.

To one extreme we can mock everything. But the mocking effort should pay us off.

For the gRPC framework, we can run the gRPC server live & write client codes to test against the business logic.
But spinning up a server from the test file can lead to unintended consequences that may require you to allocate a TCP port (parallel runs, multiple runs under the same CI server).

To solve this gRPC community has introduced a package called [bufconn](google.golang.org/grpc/test/bufconn) under gRPC's testing package. `bufconn` is a package that provides a `Listener` object that implements `net.Conn`. We can substitute this listener in a gRPC server - allowing us to spin up a server that acts as a full-fledged server that can be used for testing that talks over an in-memory buffer instead of a real port.

As `bufconn` already comes with the [grpc](google.golang.org/grpc) go module - which we already have installed, we don't need to install it explicitly.

So, let's create a new test file `server/machine_live_test.go` write the following test code to launch the gRPC server live using `bufconn`, and write a client to test the bidirectional RPC `Execute()`.

```go
const bufSize = 1024 * 1024

var lis *bufconn.Listener

func init() {
	lis = bufconn.Listen(bufSize)
	s := grpc.NewServer()
	machine.RegisterMachineServer(s, &MachineServer{})
	go func() {
		if err := s.Serve(lis); err != nil {
			log.Fatalf("Server exited with error: %v", err)
		}
	}()
}

func bufDialer(context.Context, string) (net.Conn, error) {
	return lis.Dial()
}

func testExecute_Live(t *testing.T, client machine.MachineClient, instructions []*machine.Instruction, wants []float32) {
	log.Printf("Streaming %v", instructions)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	stream, err := client.Execute(ctx)
	if err != nil {
		log.Fatalf("%v.Execute(ctx) = %v, %v: ", client, stream, err)
	}
	waitc := make(chan struct{})
	go func() {
		i := 0
		for {
			result, err := stream.Recv()
			if err == io.EOF {
				log.Println("EOF")
				close(waitc)
				return
			}
			if err != nil {
				log.Printf("Err: %v", err)
			}
			log.Printf("output: %v", result.GetOutput())
			got := result.GetOutput()
			want := wants[i]
			if got != want {
				t.Errorf("got %v, want %v", got, want)
			}
			i++
		}
	}()

	for _, instruction := range instructions {
		if err := stream.Send(instruction); err != nil {
			log.Fatalf("%v.Send(%v) = %v: ", stream, instruction, err)
		}
	}
	if err := stream.CloseSend(); err != nil {
		log.Fatalf("%v.CloseSend() got error %v, want %v", stream, err, nil)
	}
	<-waitc
}

func TestExecute_Live(t *testing.T) {
	ctx := context.Background()
	conn, err := grpc.DialContext(ctx, "bufnet", grpc.WithContextDialer(bufDialer), grpc.WithInsecure())
	if err != nil {
		t.Fatalf("Failed to dial bufnet: %v", err)
	}
	defer conn.Close()
	client := machine.NewMachineClient(conn)

	// try Execute()
	instructions := []*machine.Instruction{
		{Operand: 1, Operator: "PUSH"},
		{Operand: 2, Operator: "PUSH"},
		{Operator: "ADD"},
		{Operand: 3, Operator: "PUSH"},
		{Operator: "DIV"},
		{Operand: 4, Operator: "PUSH"},
		{Operator: "MUL"},
		{Operator: "FIB"},
		{Operand: 5, Operator: "PUSH"},
		{Operand: 6, Operator: "PUSH"},
		{Operator: "SUB"},
	}
	wants := []float32{3, 1, 4, 0, 1, 1, 2, 3, -1}
	testExecute_Live(t, client, instructions, wants)
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/95afc5b5d158252b6ff04e6b2193c30effa270ec">server/machine_live_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test server/machine.go server/machine_live_test.go
ok      command-line-arguments  0.005s
```

Fantastic!!! Everything worked as expected.

---

At the end of this blog, we’ve learned:

- How to define an interface for bi-directional streaming RPC using protobuf
- How to write gRPC server & client logic for bi-directional streaming RPC
- How to write and run the unit test for bi-directional streaming RPC
- How to write and run the unit test for bi-directional streaming RPC by running the server live leveraging the `bufconn` package
- How to run the gRPC server and a client can communicate to it

The source code of this example is available at [toransahu/grpc-eg-go](https://github.com/toransahu/grpc-eg-go).

See you on the next blog.
