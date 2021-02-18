<h3>Table of Content</h3>

[TOC]

<h1>Building a basic microservice with unidirectional-streaming gRPC using Golang</h1>

Have you ever wondered while developing a REST API that if the server could have got the capability to stream responses using the same TCP connection? Or, reversely if the REST client could have got the capability to stream the requests to the server, this could have saved the cost of bringing up another service (like WebSocket) just for the sake of fulfilling such requirement.

Then REST isn’t the only API architecture available, and for such use-cases, the gRPC model has begun to play a crucial role. gRPC's unidirectional-streaming RPC feature has got your back on those requirements.

## Objective

In this blog, you'll get to know what is client streaming & server streaming uni-directional RPCs. How to implement, test, and run them using a live, fully functional example.

> Previously in the [part-1]() of this [blog series](), we've learned the basics of gRPC, how to implement a Simple/Unary gRPC, how to write unit tests, how to launch the server & client. part-1 walks you through a step-by-step guide to implement a _Stack Machine_ server & client leveraging Simple/Unary RPC.

> If you've missed that, it is highly recommended to go through it to get familiar with the basics of the gRPC framework.

## Introduction

Let's understand how Client streaming & Server streaming RPCs works at a very high-level.

Client streaming RPCs where:

- the client writes a sequence of messages and sends them to the server using a provided stream
- once the client has finished writing the messages, it waits for the server to read them and return its response

Server streaming RPCs where:

- the client sends a request to the server and gets a stream to read a sequence of messages back
- the client reads from the returned stream until there are no more messages

The best thing is gRPC guarantees message ordering within an individual RPC call.

Now let's improve the _"Stack Machine"_ server & client codes to support unidirectional streaming.

## Implementing Server Streaming RPC

We'll see an example of Server Streaming first by implementing the `FIB` operation.

Where the `FIB` RPC will:

- perform a Fibonacci operation
- accept an integer input i.e. generate first N numbers of the Fibonacci series
- will respond with a stream of integers i.e. first N numbers of the Fibonacci series

And later we'll see how Client Streaming can be implemented so that a client can input a stream of Instructions to the Stack Machine in real-time rather than sending a single request comprised of a set of Instructions.

### Update the protobuf

We already have defined the gRPC service `Machine` and a Simple (Unary) RPC method `Execute` inside our service definition in part-1 of the blog series. Now, let's update the service definition to add one server streaming RPC called `ServerStreamingExecute`.

- A server streaming RPC where the client sends a request to the server using the stub and waits for a response to come back as a stream of result
- To specify a server-side streaming method, need to place the `stream` keyword before the response type

```proto
// ServerStreamingExecute accepts a set of Instructions from client and returns a stream of Result.
rpc ServerStreamingExecute(InstructionSet) returns (stream Result) {}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/689f3ec5cbd33fa1ee6ca68baa48586603b3c381">machine/machine.proto</a></center>

### Generating the updated client and server interface Go code

We need to generate the gRPC client and server interfaces from our `machine/machine.proto` service definition.

```
~/disk/E/workspace/grpc-eg-go
$ SRC_DIR=./
$ DST_DIR=$SRC_DIR
$ protoc \
  -I=$SRC_DIR \
  --go_out=plugins=grpc:$DST_DIR \
  $SRC_DIR/machine/machine.proto
```

You can observe that the declaration of `ServerStreamingExecute()` in the `MachineClient` and `MachineServer` interface has been auto-generated:

```go
...

 type MachineClient interface {
 	Execute(ctx context.Context, in *InstructionSet, opts ...grpc.CallOption) (*Result, error)
+ 	ServerStreamingExecute(ctx context.Context, in *InstructionSet, opts ...grpc.CallOption) (Machine_ServerStreamingExecuteClient, error)
 }

...

 type MachineServer interface {
 	Execute(context.Context, *InstructionSet) (*Result, error)
+ 	ServerStreamingExecute(*InstructionSet, Machine_ServerStreamingExecuteServer) error
 }
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/707207380c5e9b32dc30b89d5bb744d43e8335bf">machine/machine.pb.go</a></center>

### Update the Server

Just in case if you're wondering, _What if my service doesn't implement some of the RPCs declared in the `machine.pb.go` file_, then you'll encounter the following error while launching your gRPC server.

```
~/disk/E/workspace/grpc-eg-go
$ go run cmd/run_machine_server.go
# command-line-arguments
cmd/run_machine_server.go:32:44: cannot use &server.MachineServer literal (type *server.MachineServer) as type machine.MachineServer in argument to machine.RegisterMachineServer:
        *server.MachineServer does not implement machine.MachineServer (missing ServerStreamingExecute method)

```

So, it's always the best practice to keep your service in sync with the service definition i.e. `machine/machine.proto` & `machine/machine.pb.go`. If you do not want to support a particular RPC, or its implementation is not yet ready, just respond with `Unimplemented` error status. Example:

```go
// ServerStreamingExecute runs the set of instructions given and streams a sequence of Results.
func (s *MachineServer) ServerStreamingExecute(instructions *machine.InstructionSet, stream machine.Machine_ServerStreamingExecuteServer) error {
	return status.Error(codes.Unimplemented, "ServerStreamingExecute() not implemented yet")
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/2467249a8824525439186d28374b74067d93b0e9">server/machine.go</a></center>

Before we implement the `ServerStreamingExecute()` RPC, let's write a Fibonacci series _generator_ called `FibonacciRange()`.

```go
package utils

func FibonacciRange(n int) <-chan int {
	ch := make(chan int)
	fn := make([]int, n+1, n+2)
	fn[0] = 0
	fn[1] = 1
	go func() {
		defer close(ch)
		for i := 0; i <= n; i++ {
			var f int
			if i < 2 {
				f = fn[i]
			} else {
				f = fn[i-1] + fn[i-2]
			}
			fn[i] = f
			ch <- f
		}
	}()
	return ch
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/5f2a7611c8a2e8d94cb3b2a71ddee34b6f0500bf">utils/fibonacci.go</a></center>

> The blog series assumes that you're familiar with Golang basics & its concurrency paradigms & concepts like `Channels`. You can read more about the `Channels` from the [official document](https://tour.golang.org/concurrency/2).

This function yields the numbers of Fibonacci series till the Nth position.

Let's also add a small unit test to validate the `FibonacciRange()` generator.

```go
package utils

import (
	"testing"
)

func TestFibonacciRange(t *testing.T) {
	fibOf5 := []int{0, 1, 1, 2, 3, 5}
	i := 0
	for f := range FibonacciRange(5) {
		if f != fibOf5[i] {
			t.Errorf("got %d, want %d", f, fibOf5[i])
		}
		i++
	}
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/aa80539e6c79060336a6bfc3ba1b64928e9f6cf1">utils/fibonacci_test.go</a></center>

Let's implement `ServerStreamingExecute()` to handle the basic instructions `PUSH`/`POP`, and `FIB` with proper error handling. On completion of the execution of instructions set, it should `POP` the result from the Stack and should respond with a `Result` object to the client.

```go
func (s *MachineServer) ServerStreamingExecute(instructions *machine.InstructionSet, stream machine.Machine_ServerStreamingExecuteServer) error {
	if len(instructions.GetInstructions()) == 0 {
		return status.Error(codes.InvalidArgument, "No valid instructions received")
	}

	var stack stack.Stack

	for _, instruction := range instructions.GetInstructions() {
		operand := instruction.GetOperand()
		operator := instruction.GetOperator()
		op_type := OperatorType(operator)

		log.Printf("Operand: %v, Operator: %v\n", operand, operator)

		switch op_type {
		case PUSH:
			stack.Push(float32(operand))
		case POP:
			stack.Pop()
		case FIB:
			n, popped := stack.Pop()

			if !popped {
				return status.Error(codes.Aborted, "Invalid sets of instructions. Execution aborted")
			}

			if op_type == FIB {
				for f := range utils.FibonacciRange(int(n)) {
					log.Println(float32(f))
					stream.Send(&machine.Result{Output: float32(f)})
				}
			}
		default:
			return status.Errorf(codes.Unimplemented, "Operation '%s' not implemented yet", operator)
		}
	}
	return nil
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/9d3e72a01e455599a88a57b9464d0a6fa8276721">server/machine.go</a></center>

### Update the Client

Now, update the client code to call `ServerStreamingExecute()` where the client will be receiving numbers of the Fibonacci series through the `stream` and print the same.


```go
func runServerStreamingExecute(client machine.MachineClient, instructions *machine.InstructionSet) {
	log.Printf("Executing %v", instructions)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	stream, err := client.ServerStreamingExecute(ctx, instructions)
	if err != nil {
		log.Fatalf("%v.Execute(_) = _, %v: ", client, err)
	}
	for {
		result, err := stream.Recv()
		if err == io.EOF {
			log.Println("EOF")
			break
		}
		if err != nil {
			log.Printf("Err: %v", err)
			break
		}
		log.Printf("output: %v", result.GetOutput())
	}
	log.Println("DONE!")
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/f115c8dffd4e977110d63b534b62c30be5e1b4d6">client/machine.go</a></center>

### Test

To write the unit test we'll need to generate the mock of multiple `interface` as required.
[`mockgen`](https://github.com/golang/mock) is the ready-to-go framework for mocking in Golang, so we'll be leveraging it in our unit tests.

#### Server

As we've upgdated our interface i.e. `machine/machine.pb.go`, let's update the mock for `MachineClient` interface. And as we've introduced a new RPC `ServerStreamingExecute()`, let's generate the mock for `ServerStream` interface `Machine_ServerStreamingExecuteServer` as well.

```
~/disk/E/workspace/grpc-eg-go
$ mockgen github.com/toransahu/grpc-eg-go/machine MachineClient,Machine_ServerStreamingExecuteServer > mock_machine/machine_mock.go
```

The updated `mock_machine/machine_mock.go` should look like [this](https://github.com/toransahu/grpc-eg-go/commit/ab642a2e48ffe2263dbfd9cf054755fb7123c994">mock_machine/machine_mock.go).

Now, we're good to write unit test for server-side streaming RPC `ServerStreamingExecute()`:

```go
func TestServerStreamingExecute(t *testing.T) {
	s := MachineServer{}

	// set up test table
	tests := []struct {
		instructions []*machine.Instruction
		want         []float32
	}{
		{
			instructions: []*machine.Instruction{
				{Operand: 5, Operator: "PUSH"},
				{Operator: "FIB"},
			},
			want: []float32{0, 1, 1, 2, 3, 5},
		},
		{
			instructions: []*machine.Instruction{
				{Operand: 6, Operator: "PUSH"},
				{Operator: "FIB"},
			},
			want: []float32{0, 1, 1, 2, 3, 5, 8},
		},
	}

	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockServerStream := mock_machine.NewMockMachine_ServerStreamingExecuteServer(ctrl)
	for _, tt := range tests {
		mockResults := []*machine.Result{}
		mockServerStream.EXPECT().Send(gomock.Any()).DoAndReturn(
			func(result *machine.Result) error {
				mockResults = append(mockResults, result)
				return nil
			}).AnyTimes()

		req := &machine.InstructionSet{Instructions: tt.instructions}

		err := s.ServerStreamingExecute(req, mockServerStream)
		if err != nil {
			t.Errorf("ServerStreamingExecute(%v) got unexpected error: %v", req, err)
		}
		for i, result := range mockResults {
			got := result.GetOutput()
			want := tt.want[i]
			if got != want {
				t.Errorf("got %v, want %v", got, want)
			}
		}
	}
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/bcb695bc3a6335fbc4965dceecfe74618765d5a8">server/machine_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test server/machine.go server/machine_test.go
ok      command-line-arguments  0.003s
```

#### Client

For our new RPC `ServerStreamingExecute()`, let's add the mock for `ClientStream` interface `Machine_ServerStreamingExecuteClient` as well.

```
~/disk/E/workspace/grpc-eg-go
$ mockgen github.com/toransahu/grpc-eg-go/machine MachineClient,Machine_ServerStreamingExecuteServer,Machine_ServerStreamingExecuteClient > mock_machine/machine_mock.go
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/24db23a4110b241f17a4d45091f21ca389878574">mock_machine/machine_mock.go</a></center>

Let's add unit test to test client-side logic for server-side streaming RPC `ServerStreamingExecute()` using mock `MockMachine_ServerStreamingExecuteClient` :

```go
func TestServerStreamingExecute(t *testing.T) {
	instructions := []*machine.Instruction{}
	instructions = append(instructions, &machine.Instruction{Operand: 1, Operator: "PUSH"})
	instructions = append(instructions, &machine.Instruction{Operator: "FIB"})
	instructionSet := &machine.InstructionSet{Instructions: instructions}

	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockMachineClient := mock_machine.NewMockMachineClient(ctrl)
	clientStream := mock_machine.NewMockMachine_ServerStreamingExecuteClient(ctrl)

	clientStream.EXPECT().Recv().Return(&machine.Result{Output: 0}, nil)

	mockMachineClient.EXPECT().ServerStreamingExecute(
		gomock.Any(),   // context
		instructionSet, // rpc uniary message
	).Return(clientStream, nil)

	if err := testServerStreamingExecute(t, mockMachineClient, instructionSet); err != nil {
		t.Fatalf("Test failed: %v", err)
	}
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/b06988aa4f120a0f85ac514739603585358ead7a">mock_machine/machine_mock_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test mock_machine/machine_mock.go mock_machine/machine_mock_test.go
ok      command-line-arguments  0.003s
```

### Run

As we are assured through unit tests that the business logic of the server & client codes is working as expected, let’s try running the server and communicating to it via our client code.

#### Server

To start the server we need to run the previously created `cmd/run_machine_server.go` file.

```
~/disk/E/workspace/grpc-eg-go
$ go run cmd/run_machine_server.go
```

#### Client

Now, let’s run the client code `client/machine.go`.

```
~/disk/E/workspace/grpc-eg-go
$ go run client/machine.go
Executing instructions:<operator:"PUSH" operand:5 > instructions:<operator:"PUSH" operand:6 > instructions:<operator:"MUL" >
output:30
Executing instructions:<operator:"PUSH" operand:6 > instructions:<operator:"FIB" >
output: 0
output: 1
output: 1
output: 2
output: 3
output: 5
output: 8
EOF
DONE!
```

Awesome! A Server Streaming RPC has been successfully implemented.

---

## Implementing Client Streaming RPC

We have learned how to implement a Server Streaming RPC, now it's time to explore the Client Streaming RPC.
To do so, we'll not introduce another RPC, rather we'll update the existing `Execute()` RPC to accept a stream of Instructions from the client in real-time rather than sending a single request comprised of a set of Instructions.

### Update the protobuf

So, let's update the interface:

```proto
 service Machine {
-     rpc Execute(InstructionSet) returns (Result) {}
+     rpc Execute(stream Instruction) returns (Result) {}
      rpc ServerStreamingExecute(InstructionSet) returns (stream Result) {}
 }
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/18e609afa8d7f8eb52270aa1cff8b42fd1d6728c">machine/machine.proto</a></center>

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

You'll notice that declaration of `Execute()` has been updated from `MachineServer` & `MachineClient` interfaces.

```go
 type MachineServer interface {
- 	Execute(context.Context, *InstructionSet) (*Result, error)
+ 	Execute(Machine_ExecuteServer) error
 	ServerStreamingExecute(*InstructionSet, Machine_ServerStreamingExecuteServer) error
 }

 type MachineClient interface {
-	 Execute(ctx context.Context, in *InstructionSet, opts ...grpc.CallOption) (*Result, error)
+	 Execute(ctx context.Context, opts ...grpc.CallOption) (Machine_ExecuteClient, error)
	 ServerStreamingExecute(ctx context.Context, in *InstructionSet, opts ...grpc.CallOption) (Machine_ServerStreamingExecuteClient, error)
 }
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/825bdd8a5a97af41d97a4a60dc87191e9b1e0e59">machine/machine.pb.go</a></center>

### Update the Server

Let's update the server code to make `Execute()` a client streaming uni-directional RPC so that it should be able to accept stream the instructions from the client and respond with a `Result` struct.


```go
func (s *MachineServer) Execute(stream machine.Machine_ExecuteServer) error {
	var stack stack.Stack
	for {
		instruction, err := stream.Recv()
		if err == io.EOF {
			log.Println("EOF")
			output, popped := stack.Pop()
			if !popped {
				return status.Error(codes.Aborted, "Invalid sets of instructions. Execution aborted")
			}

			if err := stream.SendAndClose(&machine.Result{
				Output: output,
			}); err != nil {
				return err
			}

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

			if op_type == ADD {
				stack.Push(item1 + item2)
			} else if op_type == SUB {
				stack.Push(item1 - item2)
			} else if op_type == MUL {
				stack.Push(item1 * item2)
			} else if op_type == DIV {
				stack.Push(item1 / item2)
			}

		default:
			return status.Errorf(codes.Unimplemented, "Operation '%s' not implemented yet", operator)
		}
	}
}
```

<center>Source: <a href="https://github.com/toransahu/grpc-eg-go/commit/0136804fe1567922ae3f64796dc0e78638975109">server/machine.go</a></center>

### Update the Client

Now update the client code to make `client.Execute()` a uni-directional streaming RPC, so that the client can stream the instructions to the server and can receive a `Result` struct once the streaming completes.

```go
func runExecute(client machine.MachineClient, instructions *machine.InstructionSet) {
	log.Printf("Streaming %v", instructions)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	stream, err := client.Execute(ctx)
	if err != nil {
		log.Fatalf("%v.Execute(ctx) = %v, %v: ", client, stream, err)
	}
	for _, instruction := range instructions.GetInstructions() {
		if err := stream.Send(instruction); err != nil {
			log.Fatalf("%v.Send(%v) = %v: ", stream, instruction, err)
		}
	}
	result, err := stream.CloseAndRecv()
	if err != nil {
		log.Fatalf("%v.CloseAndRecv() got error %v, want %v", stream, err, nil)
	}
	log.Println(result)
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/91a35a9c36a69ad8005becf48bd171ca311864f4">client/machine.go</a></center>

### Test

Generate mock for `Machine_ExecuteClient` and `Machine_ExecuteServer` interface to test client-streaming RPC `Execute()`:

```
~/disk/E/workspace/grpc-eg-go
$ mockgen github.com/toransahu/grpc-eg-go/machine MachineClient,Machine_ServerStreamingExecuteClient,Machine_ServerStreamingExecuteServer,Machine_ExecuteServer,Machine_ExecuteClient > mock_machine/machine_mock.go
```

The updated `mock_machine/machine_mock.go` should look like [this](https://github.com/toransahu/grpc-eg-go/commit/6a6031df42d8a1e11346e29c6d0ff8ead898932c).

#### Server

Let's update the unit test to test the server-side logic of client streaming `Execute()` RPC using mock:

```go
func TestExecute(t *testing.T) {
	s := MachineServer{}

	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockServerStream := mock_machine.NewMockMachine_ExecuteServer(ctrl)

	mockResult := &machine.Result{}
	callRecv1 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operand: 5, Operator: "PUSH"}, nil)
	callRecv2 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operand: 6, Operator: "PUSH"}, nil).After(callRecv1)
	callRecv3 := mockServerStream.EXPECT().Recv().Return(&machine.Instruction{Operator: "MUL"}, nil).After(callRecv2)
	mockServerStream.EXPECT().Recv().Return(nil, io.EOF).After(callRecv3)
	mockServerStream.EXPECT().SendAndClose(gomock.Any()).DoAndReturn(
		func(result *machine.Result) error {
			mockResult = result
			return nil
		})

	err := s.Execute(mockServerStream)
	if err != nil {
		t.Errorf("Execute(%v) got unexpected error: %v", mockServerStream, err)
	}
	got := mockResult.GetOutput()
	want := float32(30)
	if got != want {
		t.Errorf("got %v, wanted %v", got, want)
	}
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/77468e91e82a6769f7edd17d0502455a6d1cd040">server/machine_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test server/machine.go server/machine_test.go
ok      command-line-arguments  0.003s
```

#### Client

Now, add unit test to test client-side logic of client streaming `Execute()` RPC using mock:

```go
func TestExecute(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockMachineClient := mock_machine.NewMockMachineClient(ctrl)

	mockClientStream := mock_machine.NewMockMachine_ExecuteClient(ctrl)
	mockClientStream.EXPECT().Send(gomock.Any()).Return(nil).AnyTimes()
	mockClientStream.EXPECT().CloseAndRecv().Return(&machine.Result{Output: 30}, nil)

	mockMachineClient.EXPECT().Execute(
		gomock.Any(), // context
	).Return(mockClientStream, nil)

	testExecute(t, mockMachineClient)
}
```

<center>source: <a href="https://github.com/toransahu/grpc-eg-go/commit/40f434a4eb79fea421d58ba8eb2269c4bb3807a4">mock_machine/machine_mock_test.go</a></center>

Let's run the unit test:

```
~/disk/E/workspace/grpc-eg-go
$ go test mock_machine/machine_mock.go mock_machine/machine_mock_test.go
ok      command-line-arguments  0.003s
```

Run all the unit tests at once:

```
~/disk/E/workspace/grpc-eg-go
$ go test ./...
?       github.com/toransahu/grpc-eg-go/client  [no test files]
?       github.com/toransahu/grpc-eg-go/cmd     [no test files]
?       github.com/toransahu/grpc-eg-go/machine [no test files]
ok      github.com/toransahu/grpc-eg-go/mock_machine    (cached)
ok      github.com/toransahu/grpc-eg-go/server  (cached)
ok      github.com/toransahu/grpc-eg-go/utils   (cached)
?       github.com/toransahu/grpc-eg-go/utils/stack     [no test files]
```

### Run

Now we are assured through unit tests that the business logic of the server & client codes is working as expected, let’s try running the server and communicating to it via our client code.

#### Server

To launch the server we need to run the previously created `cmd/run_machine_server.go` file.

```
~/disk/E/workspace/grpc-eg-go
$ go run cmd/run_machine_server.go
```

#### Client

Now, let’s run the client code `client/machine.go`.

```
~/disk/E/workspace/grpc-eg-go
$ go run client/machine.go
Streaming instructions:<operator:"PUSH" operand:5 > instructions:<operator:"PUSH" operand:6 > instructions:<operator:"MUL" >
output:30
Executing instructions:<operator:"PUSH" operand:6 > instructions:<operator:"FIB" >
output: 0
output: 1
output: 1
output: 2
output: 3
output: 5
output: 8
EOF
DONE!
```

Awesome!! We have successfully transformed a Unary RPC into Server Streaming RPC.

---

At the end of this blog, we’ve learned:

- How to define an interface for uni-directional streaming RPCs using protobuf
- How to write gRPC server & client logic for uni-directional streaming RPCs
- How to write and run the unit test for server-streaming & client-streaming RPCs
- How to run the gRPC server and a client can communicate to it

The source code of this example is available at [toransahu/grpc-eg-go](https://github.com/toransahu/grpc-eg-go).
You can also `git checkout` to [this](https://github.com/toransahu/grpc-eg-go/tree/b06988aa4f120a0f85ac514739603585358ead7a) commit SHA for [Part-2(a)](#implementing-server-streaming-rpc) and to [this](https://github.com/toransahu/grpc-eg-go/tree/40f434a4eb79fea421d58ba8eb2269c4bb3807a4) commit SHA for [Part-2(b)](#implementing-client-streaming-rpc).

See you in the next part of this blog series.
