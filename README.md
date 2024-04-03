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

- [Single sysbench run with Alpine 3.18.5 docker image](https://metrics.green-coding.io/stats.html?id=b0cb7f90-8c4c-4d66-a5ac-ab44186039b5)
- [Single sysbench run with Alpine 3.19.0 docker image](https://metrics.green-coding.io/stats.html?id=f8c56841-b70f-4f66-877f-f9b9cdc74d78)
- [Cumulative comparison of 10 runs each](https://metrics.green-coding.io/compare.html?ids=b0cb7f90-8c4c-4d66-a5ac-ab44186039b5,f8c56841-b70f-4f66-877f-f9b9cdc74d78,da146255-19f0-41d7-9140-69a4c364532d,d8b5e53d-7726-4ecf-9439-f9fb2beeb04e,d46f4148-7bfc-4435-b398-1ce903565815,9aeb8db8-2ef8-4905-967d-2ad444b23c25,330d49a1-e908-45da-982a-986edbf3b43f,1684ba97-96f9-42dd-8e8a-9b2e0aee3bad,739147d8-0f14-4b6f-86ea-03d281f1f19e,0914e700-3b12-4acf-8751-7a366456b782,e7bb50a7-a335-47c5-95fa-6dc9b95ff571,bfc3e0db-4ded-42f6-b2f0-9411a270c174,1b943114-a5a7-4ad4-9f36-a6956ccc7296,bd29bcf9-aa94-40b8-9397-a6f23fbd0449,42a96472-6649-4c63-8d47-ebc4bb32c319,3b133cad-c87f-4775-b94c-d3cdd1281ff0,73072192-4eb0-4578-93f0-5e8b9af8a4a4,fe326915-4f7c-43d6-8ea0-78c4c68da36e,441b6c0c-e354-4864-98bc-fb601bf1bf99,332d93b5-5b93-4c81-889e-44e368de33b9,801ab9b4-5226-405a-be1d-a79bab58e4f0,a0759dec-0b4f-491d-b696-4f73718e84b0)


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
