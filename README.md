# Gleam
[![Build Status](https://travis-ci.org/chrislusf/gleam.svg?branch=master)](https://travis-ci.org/chrislusf/gleam)
[![GoDoc](https://godoc.org/github.com/chrislusf/gleam/flow?status.svg)](https://godoc.org/github.com/chrislusf/gleam/flow)
[![Wiki](https://img.shields.io/badge/docs-wiki-blue.svg)](https://github.com/chrislusf/gleam/wiki)
[![Go Report Card](https://goreportcard.com/badge/github.com/chrislusf/gleam)](https://goreportcard.com/report/github.com/chrislusf/gleam)
[![codecov](https://codecov.io/gh/chrislusf/gleam/branch/master/graph/badge.svg)](https://codecov.io/gh/chrislusf/gleam)

Gleam is a high performance and efficient distributed execution system, and also 
simple, generic, flexible and easy to customize.

Gleam is built in Go, and the user defined computation can be written in Go, Lua, 
Unix pipe tools, or any streaming programs.

It is convenient to write logic in Lua, but Lua is optional. Go is also supported with a little bit extra effort.

### High Performance

* Go itself has high performance and concurrency. Optional LuaJIT also has high performance comparable to C, Java, Go.
* LuaJIT stream processes data, no context switches between Go and Lua.
* Data flows through memory, optionally to disk.
* Multiple map reduce steps are merged together for better performance.


### Memory Efficient

* Gleam does not have the common GC problem that plagued other languages. Each executor is run in a separated OS process. The memory is managed by the OS. One machine can host many more executors.
* Gleam master and agent servers are memory efficient, consuming about 10 MB memory.

The shuffle step in map-shuffle-reduce is costly because it usually needs to go through disk, since usually there are not enough executors to process the data partitions at the same time. But when executors are memory efficient, they can process more all data without touching disk.

Gleam also tries to automatically adjust the required memory size based on data size hints, avoiding the try-and-error manual memory tuning effort.

### Flexible
* The Gleam flow can run standalone or distributed.
* Adjustable in memory mode or OnDisk mode.

### Easy to Customize
* The Go code is much simpler to read than Scala, Java, C++.
* LuaJIT FFI library can easily invoke any C functions, for even more performance or use any existing C libraries.
* (future) Write SQL with UDF written in Lua.

# One Flow, Multiple ways to execute
Gleam code defines the flow, specifying each dataset(vertex) and computation step(edge), and build up a directed
acyclic graph(DAG). There are multiple ways to execute the DAG.

The default way is to run locally. This works in most cases. 

Here we mostly talk about the distributed mode.

## Distributed Mode
The distributed mode has several names to explain: Master, Agent, Executor, Driver.

### Gleam Driver

* Driver is the program users write, it defines the flow, and talks to Master, Agents, and Executors.

### Gleam Master

* The Master is one single server that collects resource information from Agents.
* It stores transient resource information and can be restarted.
* When the Driver program starts, it asks the Master for available Exeutors on Agents.

### Gleam Agent

* Agents runs on any machine that can run computations.
* Agents periodically send resource usage updates to Master.
* When the Driver program has executors assigned, it talks to the Agents to start Executors.
* Agents also manage datasets generated by each Executors.

### Gleam Executor
* Executors are started by Agents. It will read inputs from external or previous datasets, process them, and output to a new dataset.

### Dataset

* The datasets are managed by Agents. By default, the data run only through memory and network, not touching slow disk.
* Optionally the data can be persist to disk.

By leaving it in memory, the flow can have back pressure, and can support stream computation naturally.

# Documentation
* [Gleam Wiki] (https://github.com/chrislusf/gleam/wiki)
* [Installation](https://github.com/chrislusf/gleam/wiki/Installation)
* [Gleam Flow API GoDoc](https://godoc.org/github.com/chrislusf/gleam/flow)
* [gleam-dev on Slack](https://gleam-dev.slack.com)

# Standalone Example

## Word Count
The full source code, not snippet, for word count:
```go
package main

import (
	"os"

	"github.com/chrislusf/gleam/flow"
)

func main() {

	flow.New().TextFile("/etc/passwd").FlatMap(`
		function(line)
			return line:gmatch("%w+")
		end
	`).Map(`
		function(word)
			return word, 1
		end
	`).ReduceBy(`
		function(x, y)
			return x + y
		end
	`).Fprintf(os.Stdout, "%s,%d\n").Run()
}

```

Another way to do the similar:
```go
package main

import (
	"os"

	"github.com/chrislusf/gleam/flow"
)

func main() {

	flow.New().TextFile("/etc/passwd").FlatMap(`
		function(line)
			return line:gmatch("%w+")
		end
	`).Pipe("sort").Pipe("uniq -c").Fprintf(os.Stdout, "%s\n").Run()
}

```

## Join two CSV files. 

Assume there are file "a.csv" has fields "a1, a2, a3, a4, a5" and file "b.csv" has fields "b1, b2, b3". We want to join the rows where a1 = b2. And the output format should be "a1, a4, b3".

```go
package main

import (
	"os"

	. "github.com/chrislusf/gleam/flow"
	"github.com/chrislusf/gleam/plugins/csv"
)

func main() {

	f := New()
	a := f.ReadFile(csv.New("a.csv")).Select(Field(1,4)) // a1, a4
	b := f.ReadFile(csv.New("b.csv")).Select(Field(2,3)) // b2, b3
	
	a.Join(b).Fprintf(os.Stdout, "%s,%s,%s\n").Run()  // a1, a4, b3

}

```

## Parallel Execution
Unix Pipes are easy for sequential pipes, but limited to fan out, and even more limited to fan in.

With Gleam, fan-in and fan-out parallel pipes become very easy. 

This example get a list of file names, partitioned into 3 groups, and then process them in parallel.

```go
// word_count.go
package main

import (
	"log"
	"os"
	"path/filepath"

	"github.com/chrislusf/gleam/flow"
)

func main() {

	fileNames, err := filepath.Glob("/Users/chris/Downloads/txt/en/ep-08-*.txt")
	if err != nil {
		log.Fatal(err)
	}

	flow.New().Strings(fileNames).Partition(3).PipeAsArgs("cat $1").FlatMap(`
      function(line)
        return line:gmatch("%w+")
      end
    `).Map(`
      function(word)
        return word, 1
      end
    `).ReduceBy(`
      function(x, y)
        return x + y
      end
    `).Fprintf(os.Stdout, "%s\t%d").Run()

}

```

# Distributed Computing
## Setup Gleam Cluster
Start a gleam master and serveral gleam agents
```go
// start "gleam master" on a server
> go get github.com/chrislusf/gleam/distributed/gleam
> gleam master --address=":45326"

// start up "gleam agent" on some different servers or ports
// if a different server, remember to install Luajit and copy the MessagePack.lua file also.
> gleam agent --dir=2 --port 45327 --host=127.0.0.1
> gleam agent --dir=3 --port 45328 --host=127.0.0.1
```

## Change Execution Mode.

After the flow is defined, the Run() function can be executed in different ways: local mode, distributed mode, or planner mode.

```go
  f := flow.New()
  ...
  // local mode
  f.Run()

  // distributed mode
  import "github.com/chrislusf/gleam/distributed"
  f.Run(distributed.Option())
  f.Run(distributed.Option().SetMaster("master_ip:45326"))

  // distributed planner mode to print out logic plan
  import "github.com/chrislusf/gleam/distributed"
  f.Run(distributed.Planner())

```
# Write Mapper Reducer in Go

LuaJIT is easy, but sometimes we really need to write in Go. It is a bit more complicated, but not much. Gleam allows us to write a simple Go code with mapper or reducer logic, and automatically send it over to Gleam agents to execute. See https://github.com/chrislusf/gleam/wiki/Write-Mapper-Reducer-in-Go

# Important Features

* Fault tolerant [OnDisk()](https://godoc.org/github.com/chrislusf/gleam/flow#Dataset.OnDisk).
* Support Cassandra Fast Data Extraction.
* Read data from Local, HDFS, or S3.

# Status
Gleam is just beginning. Here are a few todo items:
* Add metadata management.
* Add better SQL database support.
* Add streaming functions.
* Caching of often re-calculated data.

Especially Need Help Now:
* Go implementation to read Parquet files.
* Design a good plugin system for accessing external data.

Please start to use it and give feedback. Help is needed. Anything is welcome. Small things count: fix documentation, adding a logo, adding docker image, blog about it, share it, etc.

[![](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=EEECLJ8QGTTPC) 

## License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
