# go-optimisation

Code is the example Bill Kennedy uses in this video
https://www.youtube.com/watch?v=2557w0qsDV0

Note that Google found an expanded example with 2 more algos at:
https://go.dev/play/p/9h53-8jZUW

## Benchmarks

To run benchmarks
`go test -run none -bench . -benchtime 3s -benchmem`

Starting code outputs:
```
BenchmarkAlgOne-16       2452267              1472 ns/op              53 B/op          2 allocs/op
BenchmarkAlgTwo-16       8997981               395.8 ns/op             0 B/op          0 allocs/op
```