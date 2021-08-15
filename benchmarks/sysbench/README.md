# Sysbench

## Testing conditions

- Sysbench 0.4.12
- Mysql by presslabs mysql operators, containers on kubernetes, Specs requests (1vCPU 2Gb memory) limits (2 vCPU 4GB memory) 50 GB GP2 SSD EBS
- Mysql RDS, Specs db.t3.micro 2vCPU 1GB memory 50 GB GP2 SSD EBS

## Testing

1. OLTP

```
$ sysbench --db-driver=mysql --mysql-host=mysql-cluster-mysql --mysql-port=3306 --mysql-user=sbtest --mysql-password="MyPassword"  --mysql-db=sbtest --num-threads=10 --test=oltp --oltp-table-size=10000000 prepare
$ sysbench --db-driver=mysql --mysql-host=mysql-cluster-mysql --mysql-port=3306 --mysql-user=sbtest --mysql-password="MyPassword"  --mysql-db=sbtest --num-threads=10 --test=oltp --oltp-table-size=10000000 --max-time=60 --max-requests=0 --oltp-read-only=on  run
```

Results:

```
sysbench 0.4.12:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 10

Doing OLTP test.
Running mixed OLTP test
Doing read-only test
Using Special distribution (12 iterations,  1 pct of values are returned in 75 pct cases)
Using "BEGIN" for starting transactions
Using auto_inc on the id column
Threads started!
Time limit exceeded, exiting...
(last message repeated 9 times)
Done.

OLTP test statistics:
    queries performed:
        read:                            272720
        write:                           0
        other:                           38960
        total:                           311680
    transactions:                        19480  (324.55 per sec.)
    deadlocks:                           0      (0.00 per sec.)
    read/write requests:                 272720 (4543.68 per sec.)
    other operations:                    38960  (649.10 per sec.)

Test execution summary:
    total time:                          60.0219s
    total number of events:              19480
    total time taken by event execution: 599.5743
    per-request statistics:
         min:                                  4.30ms
         avg:                                 30.78ms
         max:                                158.83ms
         approx.  95 percentile:              57.14ms

Threads fairness:
    events (avg/stddev):           1948.0000/45.29
    execution time (avg/stddev):   59.9574/0.03
```

```
$ sysbench --db-driver=mysql --mysql-host=<<RDS_HOST>> --mysql-port=3306 --mysql-user=sbtest --mysql-password="MyPassword" --mysql-db=sbtest --num-threads=10 --test=oltp --oltp-table-size=10000000 prepare
$ sysbench --db-driver=mysql --mysql-host=<<RDS_HOST>> --mysql-port=3306 --mysql-user=sbtest --mysql-password="MyPassword" --mysql-db=sbtest --num-threads=10 --test=oltp --oltp-table-size=10000000 --max-time=60 --max-requests=0 --oltp-read-only=on  run
```

Results:

```
sysbench 0.4.12:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 10

Doing OLTP test.
Running mixed OLTP test
Doing read-only test
Using Special distribution (12 iterations,  1 pct of values are returned in 75 pct cases)
Using "BEGIN" for starting transactions
Using auto_inc on the id column
Threads started!
Time limit exceeded, exiting...
(last message repeated 9 times)
Done.

OLTP test statistics:
    queries performed:
        read:                            229656
        write:                           0
        other:                           32808
        total:                           262464
    transactions:                        16404  (273.23 per sec.)
    deadlocks:                           0      (0.00 per sec.)
    read/write requests:                 229656 (3825.17 per sec.)
    other operations:                    32808  (546.45 per sec.)

Test execution summary:
    total time:                          60.0381s
    total number of events:              16404
    total time taken by event execution: 600.1008
    per-request statistics:
         min:                                 27.50ms
         avg:                                 36.58ms
         max:                                267.10ms
         approx.  95 percentile:              52.53ms

Threads fairness:
    events (avg/stddev):           1640.4000/4.15
    execution time (avg/stddev):   60.0101/0.01
```
