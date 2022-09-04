---
title: "Inefficient ioutil.ReadAll"
date: 2022-08-22T13:19:43+02:00
description: In this article we consider why is using ioutil.ReadAll could be inefficient and how to workaround that issue.
tags: [perfomance, profiling, memory]
categories: [ golang, dev ]
---

Let's consider ReadAll function from ioutil package.
We use it a lot in our code base.
In our infrastructure, we have a backend box and we use it for the background tasks.
Mostly it's about converting external data files to our internal format.


One sunny day, we got a notification from OOM Killer. One of out task was killed.
That task is about converting JSON file to some smart binary format
for the fast data access.
The size of the input JSON file is around 4.5G.


After quick investigation we found that the largest memory allocation comes from the following code:

```bash
go tool pprof -list problematic_package -alloc_space master.mem.pprof

         .          .    100:	}
         .          .    101:
         .       16GB    102:	data, err := ioutil.ReadAll(reader)
         .          .    103:	if err != nil {
         .          .    104:		return nil, fmt.Errorf("can't read file: %w", err)
         .          .    105:	}
```

There were some research why it's like that.
In short, when you work with the reader there's no an easy way to get information about
the file size. Thus, it's not possible to allocate the buffer with the necessary size.
Therefore, ioutil starts with the predefined minimal buffer size:

```go
// MinRead is the minimum slice size passed to a Read call by
// Buffer.ReadFrom. As long as the Buffer has at least MinRead bytes beyond
// what is required to hold the contents of r, ReadFrom will not grow the
// underlying buffer.
const MinRead = 512
```

This MinRead used as a value to grow the data buffer while reading if there is more data to read:
```go
	if int64(int(capacity)) == capacity {
		buf.Grow(int(capacity))
	}
```

and under the hood it do the allocations
```go
		buf := makeSlice(2*c + n)
		copy(buf, b.buf[b.off:])
		b.buf = buf
```
In other words, when you read huge files there happens a lot of new allocations.

Taking in account all that, we prepared a hotfix like the following:

```go
func readFile(r io.Reader, size int64) ([]byte, error) {
	data := make([]byte, size)
	offset := 0
	for {
		n, err := r.Read(data[offset:])
		if n == 0 {
			break
		}
		offset += n
		if err != nil {
			if err != io.EOF {
				return nil, fmt.Errorf("can't read file: %w", err)
			}
			break
		}
	}
	return data, nil
}

func parseJSON(path string) {
    ...
	r, err := os.Open(inPath)
	if err != nil {
		return fmt.Errorf("can't open file %q: %w", path, err)
	}
	defer r.Close()

	fi, err := os.Stat(path)
	if err != nil {
		return fmt.Errorf("can't get file info %q: %w", path, err)
	}

	data, err := readFile(r, fi.Size())
    ...
}
```

After this optimization memory profile changed to the predictable state:
```bash
go tool pprof -list problematic_package -alloc_space branch.mem.pprof

         .          .     99:	}
         .          .    100:
    4.32GB     4.32GB    101:	data, err := readFile(r, fi.Size())
         .          .    102:	if err != nil {
```

And profiles diff shows the following optimization:
```bash
go tool pprof -top -alloc_space -base master.mem.pprof branch.mem.pprof
File: main
Type: alloc_space
Time: Sep 22, 2020 at 12:30am (CEST)
Showing nodes accounting for -11.92GB, 28.25% of 42.22GB total
Dropped 36 nodes (cum <= 0.21GB)
      flat  flat%   sum%        cum   cum%
     -16GB 37.90% 37.90%      -16GB 37.90%  bytes.makeSlice
    4.32GB 10.24% 27.66%   -11.68GB 27.67%  component.glob..func3
   -0.25GB  0.59% 28.25%    -0.25GB   0.6%  component.CreateCache
         0     0% 28.25%      -16GB 37.90%  bytes.(*Buffer).ReadFrom
         0     0% 28.25%      -16GB 37.90%  bytes.(*Buffer).grow
         0     0% 28.25%   -11.93GB 28.27%  component.ConvertJSON2Binary
         .....
```

The main disadvantage with such hotfix is that we need to extend the function signature and
pass the file size together with the io.Reader interface.
But the advantage is shown above.

If there's some mistakes in understanding or there's a better solution, I'll be happy to get known.
