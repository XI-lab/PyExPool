# PyExPool

A Lightweight Multi-Process Execution Pool to schedule Jobs execution with *per-job timeout*, optionally grouping them into Tasks and specifying optional execution parameters considering NUMA architecture peculiarities:

- automatic CPU affinity management and maximization of the dedicated CPU cache for a worker process
- automatic rescheduling and *load balancing* (reduction) of the worker processes and on low memory condition for the *in-RAM computations* (requires [psutil](https://pypi.python.org/pypi/psutil), can be disabled)
- *chained termination* of related worker processes and jobs rescheduling to satisfy *timeout* and *memory limit* constraints
- timeout per each Job (it was the main initial motivation to implement this module, because this feature is not provided by any Python implementation out of the box)
- onstart/ondone *callbacks*, ondone is called only on successful completion (not termination) for both Jobs and Tasks (group of jobs)
- stdout/err output, which can be redirected to any custom file or PIPE
- custom parameters for each Job and respective owner Task besides the name/id

> Automatic rescheduling of the workers on low memory condition for the in-RAM computations is an optional and the only feature that requires external package, namely [psutil](https://pypi.python.org/pypi/psutil).

Implemented as a *single-file module* to be *easily included into your project and customized as a part of your distribution* (like in [PyCaBeM](//github.com/eXascaleInfolab/PyCABeM)), not as a separate library.  
The main purpose of this single-file module is the **asynchronous execution of modules and external executables with cache / parallelization tuning and automatic balancing of the worker processes for the in-RAM computations**.  
In case asynchronous execution of the *Python functions* is required and usage of external dependences is not a problem, or automatic jobs scheduling for in-RAM computations is not required, then more handy and straightforward approach is to use [Pebble](https://pypi.python.org/pypi/Pebble) library.

The **load balancing** is enabled when global variables `_LIMIT_WORKERS_RAM` and `_CHAINED_CONSTRAINTS` are set, jobs `.category` and relative `.size` (if known) specified. The balancing is performed to use as much RAM and CPU resources as possible performing in-RAM computations and meeting the specified timeout, memory limit and CPU cache (processes affinity) constraints.  
Large executing jobs are rescheduled for the later execution with less number of worker processes after completion of the smaller jobs. The number of workers is reduced automatically (balanced) on the jobs queue processing. It is recommended to add jobs in the order of the increasing memory/time complexity if possible to reduce the number of worker processes terminations on jobs execution postponing (rescheduling).

\author: (c) Artem Lutov <artem@exascale.info>  
\license:  Apache License, Version 2.0  
\organizations: [eXascale Infolab](http://exascale.info/), [Lumais](http://www.lumais.com/), [ScienceWise](http://sciencewise.info/)  
\date: 2017-06  

## Content
- [Dependencies](#dependencies)
- [API](#api)
	- [Job](#job)
	- [Task](#task)
	- [ExecPool](#execpool)
	- [Accessory Routines](#accessory-routines)
- [Usage](#usage)
	- [Usage Example](#usage-example)
	- [Failsafe Termination](#failsafe-termination)
- [Related Projects](#related-projects)

## Dependencies

[psutil](https://pypi.python.org/pypi/psutil) is required only in case of dynamic jobs balancing for the in-RAM computations (`_LIMIT_WORKERS_RAM = True`), otherwise there are no any dependencies.  
To install `psutil`:
```
$ sudo pip install psutil
```
[mock](https://pypi.python.org/pypi/mock) is required exclusively for the unit testing under Python2, `mock` is included in the standard lib of Python3.  
To install `mock`:
```
$ sudo pip install mock
```

## API

Flexible API provides *automatic CPU affinity management, maximization of the dedicated CPU cache, limitation of the minimal dedicated RAM per worker process, balancing of the worker processes and rescheduling of chains of the related jobs on low memory condition for the in-RAM computations*, optional automatic restart of jobs on timeout, access to job's process, parent task, start and stop execution time and more...  
`ExecPool` represents a pool of worker processes to execute `Job`s that can be grouped into `Tasks`s for more flexible management.

### Job

```python
# Global Parameters
# Limit the amount of virtual memory (<= RAM) used by worker processes
# NOTE: requires import of psutils
_LIMIT_WORKERS_RAM = True

# Use chained constraints (timeout and memory limitation) in jobs to terminate
# also related worker processes and/or reschedule jobs, which have the same
# category and heavier than the origin violating the constraints
 CHAINED_CONSTRAINTS = True


Job(name, workdir=None, args=(), timeout=0, ontimeout=False, task=None
	, startdelay=0, onstart=None, ondone=None, params=None, category=None, size=0
	, slowdown=1., omitafn=False, stdout=sys.stdout, stderr=sys.stderr):
	"""Initialize job to be executed

	Job is executed in a separate process via Popen or Process object and is
	managed by the Process Pool Executor

	# Main parameters
	name  - job name
	workdir  - working directory for the corresponding process, None means the dir of the benchmarking
	args  - execution arguments including the executable itself for the process
		NOTE: can be None to make make a stub process and execute the callbacks
	timeout  - execution timeout in seconds. Default: 0, means infinity
	ontimeout  - action on timeout:
		False  - terminate the job. Default
		True  - restart the job
	task  - origin task if this job is a part of the task
	startdelay  - delay after the job process starting to execute it for some time,
		executed in the CONTEXT OF THE CALLER (main process).
		ATTENTION: should be small (0.1 .. 1 sec)
	onstart  - callback which is executed on the job starting (before the execution
		started) in the CONTEXT OF THE CALLER (main process) with the single argument,
		the job. Default: None
		ATTENTION: must be lightweight
		NOTE: can be executed a few times if the job is restarted on timeout
	ondone  - callback which is executed on successful completion of the job in the
		CONTEXT OF THE CALLER (main process) with the single argument, the job. Default: None
		ATTENTION: must be lightweight
	params  - additional parameters to be used in callbacks
	stdout  - None or file name or PIPE for the buffered output to be APPENDED.
		The path is interpreted in the CONTEXT of the CALLER
	stderr  - None or file name or PIPE or STDOUT for the unbuffered error output to be APPENDED
		ATTENTION: PIPE is a buffer in RAM, so do not use it if the output data is huge or unlimited.
		The path is interpreted in the CONTEXT of the CALLER

	# Scheduling parameters
	omitafn  - omit affinity policy of the scheduler, which is actual when the affinity is enabled
		and the process has multiple treads
	category  - classification category, typically context or part of the name; requires _CHAINED_CONSTRAINTS
	size  - size of the processing data, >= 0; requires _LIMIT_WORKERS_RAM or _CHAINED_CONSTRAINTS
		0 means undefined size and prevents jobs chaining on constraints violation
	slowdown  - execution slowdown ratio (inversely to the [estimated] execution speed), E (0, inf)

	# Execution parameters, initialized automatically on execution
	tstart  - start time is filled automatically on the execution start (before onstart). Default: None
	tstop  - termination / completion time after ondone
	proc  - process of the job, can be used in the ondone() to read it's PIPE
	vmem  - consuming virtual memory (smooth max, not just the current value) or the least expected value
		inherited from the jobs of the same category having non-smaller size; requires _LIMIT_WORKERS_RAM
	"""
```
### Task
```python
Task(name, timeout=0, onstart=None, ondone=None, params=None, stdout=sys.stdout, stderr=sys.stderr):
	"""Initialize task, which is a group of jobs to be executed

	Task is a managing container for Jobs

	name  - task name
	timeout  - execution timeout. Default: 0, means infinity
	onstart  - callback which is executed on the task starting (before the execution
		started) in the CONTEXT OF THE CALLER (main process) with the single argument,
		the task. Default: None
		ATTENTION: must be lightweight
	ondone  - callback which is executed on successful completion of the task in the
		CONTEXT OF THE CALLER (main process) with the single argument, the task. Default: None
		ATTENTION: must be lightweight
	params  - additional parameters to be used in callbacks
	stdout  - None or file name or PIPE for the buffered output to be APPENDED
	stderr  - None or file name or PIPE or STDOUT for the unbuffered error output to be APPENDED
		ATTENTION: PIPE is a buffer in RAM, so do not use it if the output data is huge or unlimited

	tstart  - start time is filled automatically on the execution start (before onstart). Default: None
	tstop  - termination / completion time after ondone
	"""
```
### ExecPool
```python
ExecPool(wksnum=cpu_count(), afnstep=None, vmlimit=0., latency=0.)
	"""Multi-process execution pool of jobs

	wksnum  - number of resident worker processes, >=1. The reasonable value is
			<= NUMA nodes * node CPUs, which is typically returned by cpu_count(),
			where node CPUs = CPU cores * HW treads per core.
			To guarantee minimal average RAM per a process, for example 2.5 Gb:
				wksnum = min(cpu_count(), max(ramfracs(2.5), 1))
		afnstep  - affinity step, integer if applied. Used to bind worker to the
			processing units to have warm cache for single thread workers.
			Typical values:
				None  - do not use affinity at all (recommended for multi-threaded workers),
				1  - maximize parallelization (the number of worker processes = CPU units),
				cpucorethreads()  - maximize the dedicated CPU cache (the number of
					worker processes = CPU cores = CPU units / hardware treads per CPU core).
			NOTE: specification of the afnstep might cause reduction of the workers number.
		vmlimit  - limit total amount of VM (automatically reduced to the amount of physical
			RAM if the larger value is specified) in gigabytes that can be used by worker
			processes to provide in-RAM computations.
			Dynamically reduce the number of workers to consume total virtual memory
			not more than specified. The workers are rescheduled starting from the
			most memory-heavy processes. >= 0
			NOTE:
				- applicable only if _LIMIT_WORKERS_RAM
				- 0 means unlimited (some jobs might be [partially] swapped)
				- value > 0 is automatically limited with total physical RAM to process
					jobs in RAM almost without the swapping
		latency  - approximate minimal latency of the workers monitoring in sec, float >= 0.
			0 means automatically defined value (recommended, typically 2-3 sec).
	"""

	execute(job, async=True):
		"""Schedule the job for the execution

		job  - the job to be executed, instance of Job
		async  - async execution or wait until execution completed
		  NOTE: sync tasks are started at once
		return  - 0 on successful execution, process return code otherwise
		"""

	join(timeout=0):
		"""Execution cycle

		timeout  - execution timeout in seconds before the workers termination, >= 0.
			0 means absence of the timeout. The time is measured SINCE the first job
			was scheduled UNTIL the completion of all scheduled jobs.
		return  - True on graceful completion, False on termination by the specified timeout
		"""

	__del__():
		"""Force termination of the pool"""

	__finalize__():
		"""Force termination of the pool"""
```
### Accessory Routines
```python
def ramfracs(fracsize):
	"""Evaluate the minimal number of RAM fractions of the specified size in GB

	Used to estimate the reasonable number of processes with the specified minimal
	dedicated RAM.

	fracsize  - minimal size of each fraction in GB, can be a fractional number
	return the minimal number of RAM fractions having the specified size in GB
	"""

def cpucorethreads():
	"""The number of hardware treads per a CPU core

	Used to specify CPU afinity step dedicating the maximal amount of CPU cache.
	"""

def cpunodes():
	"""The number of NUMA nodes, where CPUs are located

	Used to evaluate CPU index from the affinity table index considerin the NUMA architectore.
	"""

def afnicpu(iafn, corethreads=1, nodes=1, crossnodes=True):
	"""Affinity table index mapping to the CPU index

	Affinity table is a reduced CPU table by the non-primary HW treads in each core.
	Typically CPUs are evumerated across the nodes:
	NUMA node0 CPU(s):     0,2,4,6,8,10,12,14,16,18,20,22,24,26,28,30
	NUMA node1 CPU(s):     1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31
	So, in case the number of threads per core is 2 then the following CPUs should be bound:
	0, 1, 4, 5, 8, ...
	2 -> 4, 4 -> 8
	#i ->  i  +  i // cpunodes() * cpunodes() * (cpucorethreads() - 1)

	iafn  - index in the affinity table to be mapped into the respective CPU index
	corethreads  - HW threads per CPU core or just some affinity step,
		1  - maximal parallelization with the minimal CPU cache size
	nodes  - NUMA nodes containing CPUs
	crossnodes  - cross-nodes enumeration of the CPUs in the NUMA nodes

	return CPU index respective to the specified index in the affinity table
	"""

```

## Usage

Target version of the Python is 2.7+ including 3.x, also works fine on PyPy.

The workflow consists of the following steps:

1. Create Execution Pool.
1. Create and schedule Jobs with required parameters, callbacks and optionally packing them into Tasks.
1. Wait on Execution pool until all the jobs are completed or terminated, or until the global timeout is elapsed.

See [unit tests (TestExecPool)](mpepool.py) for the advanced examples.

### Usage Example
```python
from multiprocessing import cpu_count
from sys import executable as PYEXEC  # Full path to the current Python interpreter

# 1. Create Multi-process execution pool with the optimal affinity step to maximize the dedicated CPU cache size
execpool = ExecPool(max(cpu_count() - 1, 1), cpucorethreads())
global_timeout = 30 * 60  # 30 min, timeout to execute all scheduled jobs or terminate them


# 2. Schedule jobs execution in the pool

# 2.a Job scheduling using external executable: "ls -la"
execpool.execute(Job(name='list_dir', args=('ls', '-la')))


# 2.b Job scheduling using python function / code fragment,
# which is not a goal of the design, but is possible.

# 2.b.1 Create the job with specified parameters
jobname = 'NetShuffling'
jobtimeout = 3 * 60  # 3 min

# The network shuffling routine to be scheduled as a job,
# which can also be a call of any external executable (see 2.alt below)
args = (PYEXEC, '-c',
"""import os
import subprocess

basenet = '{jobname}' + '{_EXTNETFILE}'
#print('basenet: ' + basenet, file=sys.stderr)
for i in range(1, {shufnum} + 1):
	netfile = ''.join(('{jobname}', '.', str(i), '{_EXTNETFILE}'))
	if {overwrite} or not os.path.exists(netfile):
		# sort -R pgp_udir.net -o pgp_udir_rand3.net
		subprocess.call(('sort', '-R', basenet, '-o', netfile))
""".format(jobname=jobname, _EXTNETFILE='.net', shufnum=5, overwrite=False))

# 2.b.2 Schedule the job execution, which might be postponed
# if there are no any free executor processes available
execpool.execute(Job(name=jobname, workdir='this_sub_dir', args=args, timeout=jobtimeout
	# Note: onstart/ondone callbacks, custom parameters and others can be also specified here!
))

# Add another jobs
# ...


# 3. Wait for the jobs execution for the specified timeout at most
execpool.join(global_timeout)  # 30 min
```

In case the execution pool is required locally then it can be used in the following way:
```python
...
# Limit of the virtual memory for the all worker processes with max(32 GB, RAM)
# and provide latency of 1.5 sec for the jobs rescheduling
with ExecPool(max(cpu_count()-1, 1), vmlimit=32, latency=1.5) as xpool:
	job =  Job('jvmem_proc', args=(PYEXEC, '-c', TestProcMemTree.allocAndSpawnProg(
		allocDelayProg(inBytes(amem), duration), allocDelayProg(inBytes(camem), duration)))
		, timeout=timeout, ondone=mock.MagicMock())
	jobtr =  Job('jvmem_tree', args=(PYEXEC, '-c', TestProcMemTree.allocAndSpawnProg(
		allocDelayProg(inBytes(amem), duration), allocDelayProg(inBytes(camem), duration)))
		, timeout=timeout, ondone=mock.MagicMock())
	...
	xpool.execute(job)
	xpool.execute(jobtr)
	...
	xpool.join(10)  # Timeout for the execution of all jobs is 10 sec [+latency]
```

### Failsafe Termination
To perform *graceful termination* of the Jobs in case of external termination of your program, signal handlers can be set:
```python
import signal  # Intercept kill signals

# Use execpool as a global variable, which is set to None when all jobs are done,
# and recreated on jobs scheduling
execpool = None

def terminationHandler(signal=None, frame=None, terminate=True):
	"""Signal termination handler

	signal  - raised signal
	frame  - origin stack frame
	terminate  - whether to terminate the application
	"""
	global execpool

	if execpool:
		del execpool  # Destructors are caled later
		# Define _execpool to avoid unnessary trash in the error log, which might
		# be caused by the attempt of subsequent deletion on destruction
		execpool = None  # Note: otherwise _execpool becomes undefined
	if terminate:
		sys.exit()  # exit(0), 0 is the default exit code.

# Set handlers of external signals, which can be the first lines inside
# if __name__ == '__main__':
signal.signal(signal.SIGTERM, terminationHandler)
signal.signal(signal.SIGHUP, terminationHandler)
signal.signal(signal.SIGINT, terminationHandler)
signal.signal(signal.SIGQUIT, terminationHandler)
signal.signal(signal.SIGABRT, terminationHandler)

# Define execpool to schedule some jobs
execpool = ExecPool(max(cpu_count() - 1, 1))

# Failsafe usage of execpool ...
```
Also it is recommended to register the termination handler for the normal interpreter termination using [**atexit**](https://docs.python.org/2/library/atexit.html):
```python
import atexit
...
# Set termination handler for the internal termination
atexit.register(terminationHandler, terminate=False)
```

**Note:** Please, [star this project](//github.com/eXascaleInfolab/PyExPool) if you use it.

## Related Projects
- [ExecTime](https://bitbucket.org/lumais/exectime)  -  *failover* lightweight resource consumption profiler (*timings and memory*), applicable to multiple processes with optional *per-process results labeling* and synchronized *output to the specified file* or `stderr`: https://bitbucket.org/lumais/exectime
- [PyCABeM](https://github.com/eXascaleInfolab/PyCABeM) - Python Benchmarking Framework for the Clustering Algorithms Evaluation. Uses extrinsic (NMIs) and intrinsic (Q) measures for the clusters quality evaluation considering overlaps (nodes membership by multiple clusters).
