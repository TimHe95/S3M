# S3M (performance Mismatch in I/O Size, ParalleliSm and Sequentiality)

`confSysTaint` is the core of `S3M`, `confSysTaint` is based on LLVM IR, it analyzes the control and data dependency between configuration variable and specific syscalls. An example shows how it works:

<img width="412" alt="截屏2022-09-03 12 02 02" src="https://user-images.githubusercontent.com/18543932/188259997-0e1b7269-9e90-41a6-b153-edeeeb4c47ca.png">

## Target syscalls

In section **`4.1 Test Input Generation`**, we mention that `S3M` focuses on four series of syscalls, theay are obtained by our manual investigation on every Linux syscall (335 found in kernel version 5.4.0) by reading the official manual, followed by cross-checking. The filtered out 21 syscalls that may affect I/O size, parallelism, and sequentiality:

| **read series**                                                                                | **write series**                                                                                    | **sync series**                                        | **thread series**              |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------|--------------------------------|
| `read` `pread64` `readv` `preadv` `preadv2` `io_submit` `io_getevents` `madvise` `open` `mmap` | `write` `pwrite64` `writev` `pwritev` `pwritev2` `io_submit` `io_getevents` `madvise` `open` `mmap` | `fsync` `fdatasync` `syncfs` `sync_file_range` `fcntl` | `clone(pthread_create)` `fork` |

## Scenarios that `confSysTaint` supports

### Data flow

 - **basic LLVM "Use" support**  
   <img width="250" alt="截屏2022-09-03 15 27 06" src="https://user-images.githubusercontent.com/18543932/188260809-360ee1bd-6966-4fa7-80fe-7c5a290174be.png">
 - **Field sensitive analysis**  
   <img width="582" alt="截屏2022-09-03 15 44 43" src="https://user-images.githubusercontent.com/18543932/188261335-a40776f5-6a85-4224-90a1-157dd872b1fb.png">
 - **Inter-procedure (with pointer)**  
   <img width="417" alt="截屏2022-09-03 15 53 39" src="https://user-images.githubusercontent.com/18543932/188261592-d35d625b-e808-4f43-9a01-01a431ed94bb.png">  
   <img width="417" alt="截屏2022-09-03 16 02 01" src="https://user-images.githubusercontent.com/18543932/188261868-e32bc710-4e42-4dfb-be9a-a6dfc957211b.png">
 - **Our extended data-flow (`phi-node`)**  
   <img width="600" alt="截屏2022-09-03 16 06 28" src="https://user-images.githubusercontent.com/18543932/188262007-992034c9-a3d2-4fce-96e3-2cc85589dae3.png">  
   - How to formaly determine if a `phi-node` will be tainted  
     Given a `phiNode` like:
     ```
        phi i32 [ %5, %bb1.i ], [ 0, %bb1 ]
                   pre_node      pre_node2
     ```
     we check if:
      <img width="400" alt="截屏2022-09-03 16 07 53" src="https://user-images.githubusercontent.com/18543932/188262045-11a5b2c0-48a3-4cf5-9039-077f8ffffb7c.png">

### Control flow

Formaly define how the control flow:
 - **Control Dependency**: A block **Y** is control dependent on block **X** if and only if: **Y** post-dominates at least one but not all successors of **X**. 
   - **Transitivity**：if **A** control dependent on **B**, **B** control dependent on **C**, then **A** control dependent on **C**.

An example, where the yellow square indicats the complicated code structures that motivate the use of the formal definition.  
<img width="1000" alt="截屏2022-09-03 16 39 03" src="https://user-images.githubusercontent.com/18543932/188263144-892a1294-d302-4dea-82bb-eedd21f18e78.png">

   
## Usage

### Dependency

 - llvm-10.0.0
 - [gllvm](https://github.com/SRI-CSL/gllvm) 

### Build
```
cd tainter
cmake -DCMAKE_CXX_COMPILER=/usr/bin/clang++-10 -DCMAKE_C_COMPILER=/usr/bin/clang-10 -DLLVM_DIR=/usr/lib/llvm-10/cmake . 
make
```

### Run
```
cd test/demo
../../tainter test.bc test-var.txt
```
For real DBMS, use `gllvm` to obtain the `.bc` file (e.g., mysqld.bc).

### Check results
```
cat test-records.dat
```

### Example result
```
Tainted Functions (group by Caller-Functions): 

		Clone_Handle::open_file <------------ func-1 of "srv_unix_file_flush_method"
				Clone_Task_Manager::set_error ----- Tainted Function.

		Clone_Snapshot::update_block_size <-- func-2 of "srv_unix_file_flush_method"
				os_event_set -------------------\ 
				pfs_unlock_mutex_v1              |_ Tainted Function.
				sync_array_object_signalled      |
				ut_dbg_assertion_failed --------/

		Double_write::sync_page_flush <------ func-3 of "srv_unix_file_flush_method"
				__clang_call_terminate ---------\ 
				buf_page_io_complete             |-- Tainted Function.
				fil_flush ----------------------/
        
    ...
```
