<!--
Copyright (c) 2010 Yahoo! Inc., 2012 - 2016 YCSB contributors.
All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you
may not use this file except in compliance with the License. You
may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied. See the License for the specific language governing
permissions and limitations under the License. See accompanying
LICENSE file.
-->

# YCSB-DIO
To fully test the adaptive ability of DIO, we developed a customized variant of YCSB, named YCSB-DIO, that supports running multiple workloads sequentially without restart on one database instance, abrupt online switching between workloads, and phase-aware performance observability.

## Getting Started
1. Download this repository and the [DIO project](https://github.com/DynamicIndexOrganization/rocksdb_DIO)
2. Compile DIO using normal execution mode and generate its Java target library using the following command:
```
make -sj12 clean jclean
make -sj12 rocksdbjavastatic
```
3. Find the Java<code>(.jar)</code> file, normally located at <code><dio_project_path>/java/target</code> directory.
4. Update the <code><systemPath></code> in DIO dependency information in file <code>rocksdb/pom.xml</code>:
```
    # Replace your-path-to-jar-file
    # Currently, it is set to the path within the DIO experiment environment.
    <dependency>
      <groupId>org.rocksdb_DIO</groupId>
      <artifactId>rocksdbjni</artifactId>
      <version>LOCAL</version>
      <scope>system</scope>
      <systemPath> your-path-to-jar-file </systemPath>
    </dependency>
```
We also leave a placeholder dependency for the baseline build, which is disabled by default:
```
<!-- <dependency>
  <groupId>org.rocksdbBase</groupId>
  <artifactId>rocksdbjni</artifactId>
  <version>LOCAL</version>
  <scope>system</scope>
  <systemPath>/home/user/research/rocksdb_baseline/java/target/rocksdbjni-10.1.0-linux64.jar</systemPath>
</dependency> -->
```
You can fork RocksDB, compile it, and modify the information here accordingly to run the baseline easily.

5. After the pom file is updated, use the following command to ensure the linkage is error-free
```
# This command will clean old binding, bind DIO to YCSB, and run some basic tests.
mvn -pl site.ycsb:rocksdb-binding -am clean package -e
```
6. Compile a multi-workload using the guide below, or leverage any multi-workload saved in the directory <code>exp_workload</code>, which are the workload files used by the DIO evaluation.
7. Run the following command to start the test:
```
# running ycsb using 16 Client threads with the workload 5050SWto2080GW
./bin/ycsb run rocksdb -s -threads 16 -P exp_workload/5050SWto2080GW
```

## Multi-Workload File
YCSB-DIO supports multiple workloads using one instance without restarting between workloads. In this framework, multiple workloads are executed consecutively within the same benchmark run, where each workload phase runs to completion before transitioning to the next. We defined a new workload file format based on the original workload syntax.

The original workload file is defined in the following format:
```
workload=site.ycsb.workloads.CoreWorkload
recordcount=1000000
operationcount=3000000
...
readproportion=0.2
updateproportion=0
scanproportion=0
insertproportion=0.8
...
requestdistribution=latest
```
The query proportion listed above defines the workload: it contains 80\% INSERT queries and 20\% READ or POINT\_GET queries. Attribute _recordcount_ defines the size of the key space, _operationcount_ defines how many operations will be run in this workload, and _requestdistribution_ defines what key distribution the current workload uses.

In YCSB-DIO, we support defining multiple workloads explicitly within one workload file while maintaining other basic attributes. Besides the basic attributes mentioned above, a multi-workload file used in YCSB-DIO has the following format:
```
...
workloadcount=2
...
readproportion_1=0
updateproportion_1=0    
scanproportion_1=0.2
insertproportion_1=0.8
stopcondition_1=duration
workloadduration_1=60

readproportion_2=0
updateproportion_2=0
scanproportion_2=0
insertproportion_2=1
stopcondition_2=opcount
workloadopcount_2=2000000
```
In our multi-workload file, we use _workloadcount_ to specify how many workloads are defined. Each workload **i** is defined by a set of proportion attributes with suffix **_i**. We support specifying the stopping condition of each workload either by duration or operation count. As the example shown above, we have 2 workloads: the first workload to run is a mix of 80\% INSERT operations and 20\% RANGE\_SCAN operations, and the second workload contains 100\% INSERT operations, which is an INSERT-ONLY workload. The workload 1 will run for 60 seconds, and the workload 2 will run until 2 million operations are done. When the stopping condition of workload 1 is reached, all threads will immediately start processing queries from workload 2. At the end of the execution of the multi-workload, it will print out the throughput of each workload separately, along with the overall throughput. 


### Note
1. To run DIO properly, we offered a sample option file in <code>rocksdb/dio_options.ini</code>, which is also used by the DIO evaluation. We added the following new options for DIO
```
  # To enable DIO
  enable_dynamic_index_organization=true
  # To avoid memtable type oscillation when two memtable types have close cost
  dynamic_index_organization_cost_adjust_factor=0.9
  # To initialize the memtable factory at system initialization. Keep them untouched
  skip_list_memtable_factory=SkipListRepFactory
  vector_memtable_factory=VectorRepFactory
  hash_skip_list_memtable_factory=HashSkipListRepFactory
```
You can also use this option file directly. We offered another file for baseline run saved at <code>rocksdb/baseline_options.ini</code>.

2. Multi-Workload files in <code>exp_workload</code> have the following default setting:
```
  # Set the database directory
  rocksdb.dir=/home/user/research/data
  # Set the default option file to use during system initialization
  rocksdb.optionsfile=/home/user/research/YCSB_DIO/rocksdb/options.ini
```
Please update them accordingly before your run.

For any question, please check the [original YCSB README](README_YCSB.md)
