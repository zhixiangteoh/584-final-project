Usage
-----
Setup:
1. Clone forked version of duckdb repo at https://github.com/zhixiangteoh/duckdb
2. Clone working set size (WSS) measuring tool repo at https://github.com/brendangregg/wss
3. Measure WSS for each benchmark, when local DRAM capacity "unbounded" (i.e., far exceeds WSS):
  - for TPC-H benchmark:
    - in one terminal, cd into `duckdb` folder, then run `build/release/benchmark/benchmark_runner "benchmark/tpch/sf50/.*"`
    - in a separate terminal, cd into `wss` folder, then run `./wss.pl -s 1 $(pgrep -f benchmark_runner) 0.01`
      - this prints to screen 0.01s-interval WSS measurements every second: monitor and read the highest value in the RSS (MB) column as the WSS of this benchmark workload
  - repeat for TPC-DS benchmark

See also https://github.com/duckdb/duckdb/blob/main/benchmark/README.md for duckdb benchmark running instructions.

Run baseline benchmark experiments, varying bounds for local DRAM capacity:
1. Run each benchmark with local DRAM capacity set to 1%, 10%, 50%, 80%, 100% of the benchmark workload's measured WSS (from setup), and unbounded local DRAM
  - for TPC-H benchmark:
    - cd into `duckdb` folder, then run `build/release/benchmark/benchmark_runner "benchmark/tpch/sf50_bounded/.*"`
      - for different local DRAM capacities, modify the `SET memory_limit = '<local DRAM capacity in KB, MB or GB>';` line in the `benchmark/tpch/sf50_bounded/load.sql` file

Run CXL-emulated benchmark experiments, varying bounds for local DRAM capacity:
1. Run each benchmark with local DRAM capacity set to 1%, 10%, 50%, 80%, 100% of the benchmark workload's measured WSS (from setup), and unbounded remote NUMA node memory (emulating CXL memory)
  - for TPC-H benchmark:
    - cd into `duckdb` folder, then run `build/release/benchmark/benchmark_runner "benchmark/tpch/sf50_bounded/.*"`
      - for different local DRAM capacities, modify the `SET memory_limit = '<local DRAM capacity in KB, MB or GB>';` line in the `benchmark/tpch/sf50_bounded/load.sql` file
      - set the memory spill location to remote filesystem that functions as remote memory, via `PRAGMA temp_directory='<remote/dir/path>'`;

OS commands to set up local/remote two-tier memory system:
Three ways: 
1. Set local and remote sockets, and alter local and remote DRAM capacity via GRUB file, then adjust remote memory access latency and bandwidth as needed
  - Setup a multi-tiered memory system to simulate local DRAM / CXL memory device
    - Isolate all CPU cores on one socket (which will then be the remote socket), so the remaining DRAM on this socket becomes remote DRAM for CPU cores on the other (local) socket
    - At the same time, limit memory capacity of local DRAM so that local DRAM has much smaller capacity than remote DRAM; and when application uses up all local DRAM, it accesses slower remote DRAM
    - Code (example GRUB file line):
    ```
    GRUB_CMDLINE_LINUX="isolcpus=0-63,128-191 memmap=680G!90G memmap=770G!780G"
    ```
  - Increase latency of remote memory access
    - lower uncore frequency of remote node
    ```
    sudo modprobe msr
    sudo wrmsr 0x620 0x0101
    ```

2. Use `taskset` and/or `numactl` utilities to choose CPU and memory nodes to run benchmark program on, then adjust remote memory access latency and bandwidth as needed
Example: 
```
numactl --cpubind=0 --membind=0,1 ...
```
This above command will run the benchmark program on node 0 CPUs, but use both NUMA nodes' memory

3. As above, set spill location to be "fake" disk (that is actually memory) on remote node via duckdb


Implementation(s)
-----------------
This work does not write or build any new software solution, nor does it modify existing software product(s). Instead, it describes and evaluates a viable system-level configuration and memory subsystem design that we emulate using operating system tools.

Reference docs: https://docs.google.com/document/d/1w0JCWKd07tD7i7R2kP6NnuJhxiGuJnMxLYh3XrtcLLU/edit?usp=sharing
Reference sheets: https://docs.google.com/spreadsheets/d/1sI27AjldQyZnX6Hby-SHtog6jbVNhSm5PgWu-zPTxlE/edit?usp=sharing
