# Beehive
A distributed messaging platform focused on simplicity. Our goal
is to create a programming model that is almost identical to a
centralized application yet can automatically be distributed and
optimized. Beehive comes with built-in support for transactions,
replication, fault-tolerance, runtime instrumentation, and optimized
placement.

Beehive is written in Go and uses [ectd](https://github.com/coreos/etcd)'s
[implementation](https://github.com/coreos/etcd/tree/master/raft)
of the [Raft](http://raftconsensus.github.io/) consensus algorithm.
Beehive has no external dependencies.

## Overview
Beehive denotes each logical computing node (say, a physical
or a virtual machine) as a hive. Hives can form, join to, and
leave a cluster. All hives in the same cluster must have the
same set of applications running. An application is defined as
a set of message handlers. Message handlers simply process
async messages and store their state in application dictionaries.
Application dictionaries are in-memory key-value stores (that support
persistence and replication, we'll get to that later).

```
                 +-----------------+---+---+---+-------------------+
                 |                 |   |   |   |                   |
                 |                 |   |   |   |                   |
     +-----------v-----------+     v   v   v   v       +-----------v-----------+
     | hive 1                |                         |  hive N               |
     |                       |                         |                       |
     |                       |                         |                       |
     | +-------------------+ |                         | +-------------------+ |
     | |app1               | |                         | |app1               | |
     | |-------------------| |                         | |-------------------| |
     | |handler1 (map, rcv)| |                         | |handler1 (map, rcv)| |
     | |handler2 (map, rcv)| |     ..............      | |handler2 (map, rcv)| |
     | +-------------------+ |                         | +-------------------+ |
     |                       |                         |                       |
     | +-------------------+ |                         | +-------------------+ |
     | |app2               | |                         | |app2               | |
     | |-------------------| |                         | |-------------------| |
     | |handler1 (map, rcv)| |                         | |handler1 (map, rcv)| |
     | +-------------------+ |                         | +-------------------+ |
     +-----------------------+                         +-----------------------+
```

We want to distribute the entries in the dictionaries (or
as we call them, _cells_) over multiple hives to scale. As such,
async messages are distributed among hives in a way that the
application's behavior remains identical to when we use only
a single, centralized hive. To that end, we need to ensure that
each cell in an application dictionary is accessed (i.e., read and
written) on the same hive. Otherwise, we can't guarantee that
the application state remains consistent when distributed over
multiple hives.

In addition to the `rcv` function that basically processes
the message, a message handler has a `map` function that maps
the incoming message to keys in application dictionaries,
or as we call it, to the _mapped cells_ of the incoming message.

When a message enters a hive, we first pass that message
to the `map` function of the registered message handlers.
Then, we relay the message to the hive that has all the keys
in the mapped cell and that hive in return calls the `rcv`
function of that message handler.
If there is no such hive, we elect one and relay the message.

Internally, each hive has a set of go-routines called _bees_.
Each bee exclusively owns a set of cells. These cells are
the cells that must be collocated to preserve state
consistency. Cells are locked by bees using the internal
distributed consensus mechanism. Bees can be persistent and
fault-tolerant. When a hive crashes, we reload all the bees.
And when a bee fails, we hand its cells and workload to other
bees in the cluster.
Moreover, for replicated applications the main bee will form
a colony of bees (itself and some other bees on other hives)
and will consistently replicate its cells.

Beehive is:

- Transactional: Message handlers are either ran successfully, or
  otherwise won't have any side effects.
- Replicated: State of each application is replicated using a
  distributed consensus mechanism.
- Fault Tolerant: When a machine fails, its workload will be
  handed off to other machines in the cluster.
- Instrumented: You can see how your applications interact and
  how they exchange messages.
- Optimized: Beehive is able to adjust the placement of
  applications in the cluster to optimize their performance.

## Installation

_Prerequisite_ you need to install go (preferably version 1.2+) and set up your GOPATH.

To install Beehive, run:

```
# go get github.com/kandoo/beehive
```

To test your setup, enter Beehive's root directory and run:
```
# go build
```

## Hello World!
Let's write an application that prints hello to a given name
and counts the number of hellos it has said to that name.
To implement that, we first need a message handler, a rcv
function and a map function:

```go
func mapf(msg bh.Msg, ctx bh.MapContext) bh.MappedCells {
	return bh.MappedCells{{helloDict, msg.Data().(string)}}
}
```

Here, we map an incoming message based on the string in its data
(i.e., the name in our example). This ensures that all messages
with that name will be processed by the same bee.

```go
func rcvf(msg bh.Msg, ctx bh.RcvContext) error {
	name := msg.Data().(string)
	v, err := ctx.Dict(helloDict).Get(name)
	if err != nil {
		v = []byte{0}
	}

	cnt := v[0] + 1
	fmt.Printf("%v> hello %s (%d)!\n", ctx.ID(), name, cnt)
	ctx.Dict(helloDict).Put(name, []byte{cnt})
	return nil
}
```

In the receive function, we simply lookup the name in the
hello dictionary and find out how many times we have said
hello for this name. Then we say hello accordingly!
Note that `ctx.ID()` prints the bee ID for us. Later, we will
see that two different names will be handled by two different bees.

To use these functions, one needs to create an application
and register these functions as a handler for type `string`.

```go
func main() {
	app := bh.NewApp("HelloWorld", bh.AppPersistent(1))
	app.HandleFunc(string(""), mapf, rcvf)
	...
}
```

After that, she needs to start the hive and emit messages:

```go
func main() {
	app := bh.NewApp("HelloWorld", bh.AppPersistent(1))
	app.HandleFunc(string(""), mapf, rcvf)
	name1 := "1st name"
	name2 := "2nd name"
	for i := 0; i < 3; i++ {
		go bh.Emit(name1)
		go bh.Emit(name2)
	}
	bh.Start()
}
```

The output of this application will be (or with a slightly different
order and with an occasional raft logs ;) ):

```
# go run helloworld.go

2> hello 2nd name (1)!
2> hello 2nd name (2)!
1> hello 1st name (1)!
2> hello 2nd name (3)!
1> hello 1st name (2)!
1> hello 1st name (3)!
```

Note that "1st name" and "2nd name" are handled by different bees.

In our example, we made the application persisent and transactional:
```go
bh.NewApp(..., bh.AppPersistent(1))
```

This means that any update to the state will be written to disk,
if the message handler successfully processes the message.
If you re-run the program (you can exit with `ctrl+c`), 
you see that the counts are preserved!

```
# go run helloworld.go

2> hello 2nd name (4)!
2> hello 2nd name (5)!
2> hello 2nd name (6)!
1> hello 1st name (4)!
1> hello 1st name (5)!
1> hello 1st name (6)!
```

You can find the complete code on
[the Hello World example](https://github.com/kandoo/beehive/tree/master/examples/helloworld/helloworld.go)

## More Examples
- [Calculator](https://github.com/kandoo/beehive/tree/master/examples/calc)
- [Ping Pong](https://github.com/kandoo/beehive/tree/master/examples/pingpong)

## Projects using Beehive:
- [Beehive Distributed SDN Controller](https://github.com/kandoo/beehive-netctrl)


## Discussions
Google group: [https://groups.google.com/forum/#!forum/beehive-dev](https://groups.google.com/forum/#!forum/beehive-dev)

Please report bugs in github, not in the group.

## Publications
Soheil Hassas Yeganeh, Yashar Ganjali,
[Beehive: Towards a Simple Abstraction for Scalable Software-Defined Networking](http://conferences.sigcomm.org/hotnets/2014/papers/hotnets-XIII-final17.pdf),
HotNets XIII, 2014.
