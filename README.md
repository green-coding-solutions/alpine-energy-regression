This repository aims to evaluate an energy regression (increase in energy usage) happening after the update from
3.18.5 to 3.19.0 of the alpine base image.

## Benchmark

In order to pin down the regression we are using sysbench.

We are building the base docker image of alpine (in two different versions 3.18.5 and 3.19.0) and then only installing
sysbench (both times in the exact same version) as dependency.

The testrun is done with the following commands:

`$ sysbench cpu run --cpu-max-prime=25000 --threads=8 --time=15`

which executes a CPU bound calculation for prime numbers on 8 threads, no event limit, no rate limit, but 15 seconds time
limit.

## Results for measurement cluster

We are using the Green Metrics Tool Measurement cluster to get the results

Find the detailed results from a measurement cluster here:

## Results with perf

A more easy way to reproduce can be done with perf if your CPU supports Intel RAPL.

`$ sudo perf stat -a -r 10 -e power/energy-pkg/ runuser -u gc -- sh -c "docker run --rm --init alpine_3.18.5 sysbench cpu run --cpu-max-prime=25000 --threads=8 --time=15"

This will encapsultae the sysbench run in a `docker run` statement and additionally also encapsulate it in perf stat
which will read the `power/energy-pkg/` PMU. The benchmark here is automatically repeated 10 times (`-r 10`)

Results are similar to:

### With alpine 3.19.0:
```
$ sudo perf stat -a -r 10 -e power/energy-pkg/ runuser -u gc -- sh -c "docker run --rm --init alpine_3.19.0 sysbench cpu run --cpu-max-prime=25000 --threads=8 --time=15 --events=0 --rate=0 --debug=off | grep total"
# 33783 Events
# 1018.81 Joules power/energy-pkg/ ( +-  0.19% )
```

### With alpine 3.18.5:
```
$ sudo perf stat -a -r 10 -e power/energy-pkg/ runuser -u gc -- sh -c "docker run --rm --init alpine_3.18.5 sysbench cpu run --cpu-max-prime=25000 --threads=8 --time=15 --events=0 --rate=0 --debug=off | grep total"
# 34079 Events
# 973.22 Joules power/energy-pkg/ ( +-  0.06% )
```

### Summary
- Event amount dropped by around 1%
- Energy increased by around 5%

So we no only have a performance regression, but also an energy increase.
