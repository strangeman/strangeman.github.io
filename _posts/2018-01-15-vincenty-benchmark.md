---
layout: post
categories: go rust python
title: "Benchmarking Python vs Go+Python vs Rust+Python for Vincenty formula "
description: "Benchmarking Python vs Go+Python vs Rust+Python for Vincenty formula "
keywords: "go, rust, python, benchmark"
---
# Table Of Contents
* TOC
{:toc}

# Benchmarking Python vs Go+Python vs Rust+Python

## Disclaimer

This is not too representative experiment, I just searching the fastest way to compute [Vincenty distance](https://en.wikipedia.org/wiki/Vincenty%27s_formulae) for my Telegram Python bot.

Algorithms for Go and Rust are equal (I just rewrote Go code into Rust), 'plain' Python uses `geopy` library.

Rust code may be not 'rust-way', but this is my first experience with this language.

All code located here: [https://github.com/strangeman/vincenty-benchmark](https://github.com/strangeman/vincenty-benchmark)

## Environment and tools

### OS

```
$ cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux buster/sid"

```

### Languages

```
$ python -VV
Python 3.6.4 (default, Jan  5 2018, 02:13:53)
[GCC 7.2.0]

$ go version
go version go1.9.2 linux/amd64

$ rustc -Vv
rustc 1.23.0 (766bd11c8 2018-01-01)
binary: rustc
commit-hash: 766bd11c8a3c019ca53febdcd77b2215379dd67d
commit-date: 2018-01-01
host: x86_64-unknown-linux-gnu
release: 1.23.0
LLVM version: 4.0
```

Rust code was compiled with and without `-O` (allow optimizations) option.

Unfortunately, that code wasn't compiling with `gccgo` compiler, so I didn't test it.

### Benchmarking

I used pytest-benchmark 3.1.1. Rust and Go code was compiled to `.so` library and called from Python code. All sources located in `/src` directory.

Benchmark parameters: `rounds=1000, warmup_rounds=5, iterations=100`.

## How to run it

* You need to have installed Python 3, Go and Rust;
* Run `make install` to install needed Python and Go dependencies;
* Run `make compile` to compile Go and Rust code;
* Run `make benchmark` to run benchmarks.


## Results

```
------------------------------------------------------------------------------------------------- benchmark: 4 tests ------------------------------------------------------------------------------------------------
Name (time in us)                               Min                 Max                Mean             StdDev              Median                IQR            Outliers  OPS (Kops/s)            Rounds  Iterations
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
test_benchmark_distance_rust_optimized      16.6895 (1.0)       35.6014 (1.0)       19.6811 (1.0)       2.8708 (1.0)       19.1832 (1.0)       1.3009 (1.06)      113;103       50.8103 (1.0)        1000         100
test_benchmark_distance_rust_default        22.5512 (1.35)      46.2139 (1.30)      26.5663 (1.35)      3.6022 (1.25)      25.8693 (1.35)      1.2280 (1.0)       113;198       37.6417 (0.74)       1000         100
test_benchmark_distance_go                  34.1294 (2.04)      68.4217 (1.92)      39.2860 (2.00)      3.5562 (1.24)      39.0943 (2.04)      2.7511 (2.24)       181;42       25.4543 (0.50)       1000         100
test_benchmark_distance_python             163.4739 (9.79)     365.9721 (10.28)    186.3141 (9.47)     21.2402 (7.40)     185.2215 (9.66)     15.8129 (12.88)       59;43        5.3673 (0.11)       1000         100
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

Several runs show similar results. Full report you can see [here](https://github.com/strangeman/vincenty-benchmark/benchmark_report.txt).

Looks like Rust code 2x faster than Go and almost 10x faster than Python.

### Size of binaries
```
$ ls -l1h build/

1,4K godistance.h
2,0M godistance.so
2,8M rustdistance_default.so
2,8M rustdistance_optimized.so

```

## Some additional thoughts

Rust looks interesting, but it a bit complex than Go (especially in OOP part). Also, Rust is younger and have less third-party modules and tools.