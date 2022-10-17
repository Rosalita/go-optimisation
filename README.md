# go-optimisation

Code is the example Bill Kennedy uses in this video
https://www.youtube.com/watch?v=2557w0qsDV0

Note that Google found an expanded example with 2 more algos at:
https://go.dev/play/p/9h53-8jZUW

## Benchmarks and Memory Allocation Analysis

To run benchmarks
`go test -run none -bench . -benchtime 3s -benchmem`

explanation of go test flags at 
`https://pkg.go.dev/cmd/go/internal/test`


Starting code outputs:
```
BenchmarkAlgOne-16       2452267              1472 ns/op              53 B/op          2 allocs/op
BenchmarkAlgTwo-16       8997981               395.8 ns/op             0 B/op          0 allocs/op
```
ns/op os nanoseconds per op
B/op is bytes per op

The benchmarks can generate a memory profile
`go test -run none -bench . -benchtime 3s -benchmem -memprofile mem.out`

Note that this generates mem.out, it also generates symbols in `foldername.test.exe`

run pprof
`go tool pprof -alloc_space go-optimisation.test.exe mem.out`

This opens pprof in interactive mode. Type `exit` to leave.

list command
`list algOne`

this shows
```
$ go tool pprof -alloc_space go-optimisation.test.exe mem.out
File: go-optimisation.test.exe
Type: alloc_space
Time: Oct 17, 2022 at 4:08pm (BST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) list algOne
Total: 188.51MB
ROUTINE ======================== github.com/Rosalita/go-optimisation.algOne in C:\dev\go\src\github.com\Rosalita\go-optimisation\main.go
   19.50MB   184.51MB (flat, cum) 97.88% of Total
         .          .     80:// reads a minimum number of bytes required and then starts processing
         .          .     81:// new bytes as they are provided in the stream.
         .          .     82:func algOne(data []byte, find []byte, repl []byte, output *bytes.Buffer) {
         .          .     83:
         .          .     84:   // Use a bytes buffer to provide a stream to process.
         .   165.01MB     85:   input := bytes.NewBuffer(data)
         .          .     86:
         .          .     87:   // The number of bytes we are looking for.
         .          .     88:   size := len(find)
         .          .     89:
         .          .     90:   // Declare the buffers we need to process the stream.
   19.50MB    19.50MB     91:   buf := make([]byte, size)
         .          .     92:   //      tmp := make([]byte, 1)
         .          .     93:   end := size - 1
         .          .     94:
         .          .     95:   // Read in an initial number of bytes we need to get started.
         .          .     96:   if n, err := io.ReadFull(input, buf[:end]); err != nil {
(pprof)
```

pprof can't look at the allocations of algTwo because it has no allocations.

To see the call graphs as a web visualisation type `web` in interactive mode. (Note: this requires both `dot` and `graphviz` to be installed. On windows `choco install graphviz`)

To interpret the call graph
https://github.com/google/pprof/blob/main/doc/README.md#interpreting-the-callgraph

## Escape Analysis
To see an escape analysis run 
`go build -gcflags=-m=2`

Reading the escape analysis for line 85

```
.\main.go:85:26: inlining call to bytes.NewBuffer
```
firstly it gets inlined.

```
.\main.go:85:26: &bytes.Buffer{...} escapes to heap:
.\main.go:85:26:   flow: ~R0 = &{storage for &bytes.Buffer{...}}:
.\main.go:85:26:     from &bytes.Buffer{...} (spill) at .\main.go:85:26
.\main.go:85:26:     from ~R0 = &bytes.Buffer{...} (assign-pair) at .\main.go:85:26
.\main.go:85:26:   flow: input = ~R0:
.\main.go:85:26:     from input := ~R0 (assign) at .\main.go:85:8
.\main.go:85:26:   flow: io.r = input:
.\main.go:85:26:     from input (interface-converted) at .\main.go:115:28
.\main.go:85:26:     from io.r, io.buf := input, buf[:end] (assign-pair) at .\main.go:115:28
.\main.go:85:26:   flow: {heap} = io.r:
.\main.go:85:26:     from io.ReadAtLeast(io.r, io.buf, len(io.buf)) (call parameter) at .\main.go:115:28
```
Escape analysis shows this value is escaping to the heatp at line 115.
The reason for this is that it is being converted to an interface.
`io.ReadFull` accepts an io reader so input is converted to an ioreader.

Replaced `io.ReadFull` with `input.Read()`

Running the benchmark again this shows
```
BenchmarkAlgOne-16       3663192               970.3 ns/op             5 B/op          1 allocs/op
BenchmarkAlgTwo-16      10462761               345.7 ns/op             0 B/op          0 allocs/op
```

Now line 91 escapes to heap, this is a make call where size is a variable.
Compiler needs to know size at compile time to keep values on the stack.
Hard coding this as 5 reduces the allocations to 0

```
BenchmarkAlgOne-16       3899920               926.3 ns/op             0 B/op          0 allocs/op
BenchmarkAlgTwo-16      10743843               338.9 ns/op             0 B/op          0 allocs/op
```

## Benchmark and CPU Profiling

The benchmarks can generate a cpu profile
`go test -run none -bench . -benchtime 3s -benchmem -cpuprofile cpu.out`

to open this in pprof
`go tool pprof go-optimisation.test.exe cpu.out`

Then type `web` to see a call graph. Watchout for graphs that go too deep

Can list functions again. Listing algOne shows
```
Total: 5.57s
ROUTINE ======================== github.com/Rosalita/go-optimisation.algOne in C:\dev\go\src\github.com\Rosalita\go-optimisation\main.go
     370ms      2.86s (flat, cum) 51.35% of Total
         .          .     89:
         .          .     90:   // Declare the buffers we need to process the stream.
         .          .     91:   // original:
         .          .     92:   // buf := make([]byte, size)
         .          .     93:   // optimised:
      70ms       70ms     94:   buf := make([]byte, 5)
         .          .     95:   //      tmp := make([]byte, 1)
         .          .     96:   end := size - 1
         .          .     97:
         .          .     98:   // Read in an initial number of bytes we need to get started.
         .          .     99:   // original line:
         .          .    100:   // if n, err := io.ReadFull(input, buf[:end]); err != nil {
         .          .    101:   // optimised line:
         .       20ms    102:   if n, err := input.Read(buf[:end]); err != nil {
         .          .    103:           output.Write(buf[:n])
         .          .    104:           return
         .          .    105:   }
         .          .    106:
         .          .    107:   for {
         .          .    108:
         .          .    109:           // Read in one byte from the input stream.
         .          .    110:
         .          .    111:           // original line:
         .          .    112:           // if _, err := io.ReadFull(input, buf[end:]); err != nil {
         .          .    113:           // optimised line:
     120ms      520ms    114:           if _, err := input.Read(buf[end:]); err != nil {
         .          .    115:                   // Flush the reset of the bytes we have.
      10ms       10ms    116:                   output.Write(buf[:end])
         .          .    117:                   return
         .          .    118:           }
         .          .    119:
         .          .    120:           // if we have a match, replace the bytes.
         .          .    121:           // original:
         .          .    122:           // if bytes.Compare(buf, find) == 0 {
         .          .    123:           // optimised:
      30ms      1.81s    124:           if bytes.Compare(buf, find) == 0 {
     100ms      190ms    125:                   output.Write(repl)
         .          .    126:
         .          .    127:                   //Read a new initial number of bytes.
         .          .    128:                   // original line
         .          .    129:                   // if n, err := io.ReadFull(input, buf[:end]); err != nil {
         .          .    130:                   // optimised line
         .       60ms    131:                   if n, err := input.Read(buf[:end]); err != nil {
      30ms       40ms    132:                           output.Write(buf[:n])
         .          .    133:                           return
         .          .    134:                   }
         .          .    135:
         .          .    136:                   continue
         .          .    137:           }
         .          .    138:
         .          .    139:           //Write the front byte since it has been compared.
      10ms      140ms    140:           output.WriteByte(buf[0])
         .          .    141:
         .          .    142:           // Slice that front byte out.
         .          .    143:           copy(buf, buf[1:])
         .          .    144:   }
         .          .    145:}
(pprof)
```

The two lines which look slow are:
114, which reads a byte using input.Read()
124, which does bytes.Compare.

Replacing bytes.Compare for bytes.Equal results in the following benchmark, it's gone down about 80ns

```
BenchmarkAlgOne-16       4269470               845.0 ns/op             0 B/op          0 allocs/op
BenchmarkAlgTwo-16       8088073               445.9 ns/op             0 B/op          0 allocs/op
```

Replacing input.Read() for input.ReadByte() results in the following benchmark:

```
BenchmarkAlgOne-16       4726576               761.7 ns/op             0 B/op          0 allocs/op
BenchmarkAlgTwo-16       8589955               419.4 ns/op             0 B/op          0 allocs/op
```
