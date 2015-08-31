
### What is Systemtap
SystemTap provides a command line interface and scripting language for instrumenting for a live running kernel and user-space applications.
SystemTap is great at:
- live analysis
- programmable on-line response (aka fault injection)
- whole-system symbolic access
- simple tracing jobs

### How to Get Systemtap Up and Running
Systemtap works best on CentOS or RHEL (since it's conceived by Red Hat), the instructions below are for CentOS 7

#### Installation
1. `sudo yum install systemtap`
2. `sudo stap-prep` which installs `kernel-debuginfo` and `kernel-devel` for the current kernel version. This is required for probing kernel functions and system calls
3. `sudo usermod -a -G stapdev,stapusr $USER` ...which adds your current user to the stapdev and stapusr group, which lets you run systemtap scripts. You can use `id $USER` to verify you have been added to those groups


#### Building MongoDB With Debug Symbols
There are two ways to add probe points in userspace, one is by file name and line number and the other is by function name, the former requires you to compile mongod with `--dbg=on --opt=off` to enable debug symbols.

### Dynamic Failpoint
You can run the following scripts with `stap -g SCRIPT_NAME.stp`. The `-g` is for "guru mode" which lets you change variable values at runtime. The normal mode is read-only.

#### Example: Probing by Line Number
This example demonstrates how to probe by line number. Note that you can place systemtap probes in parts of our codebase outside of the `mongo::` namespace, e.g. in WT code, or shared libraries. This example sets the size of the `calloc` call to 0, effectively failing the memory allocation. (a better way would be to change the return value of calloc to NULL, but this illustrates the use of probing by line numbers, which I found more useful).

Code:

```
// this line is 39
if ((p = calloc(number, size)) == NULL)
    WT_RET_MSG(session, __wt_errno(), "memory allocation");
```

Fault Injection:

```
probe process("/home/guo/mongo/mongod").statement("*@os_alloc.c:39") {
  $size = 0
}
```

#### Example: Probing by Function Name
This example shows how to probe by giving the function name. It also shows how to simulate some of the functionality of MONGO_FAIL_POINTs. Here's the current version of the code with a `maxTimeNeverTimeOut` fail point.

```
bool CurOp::maxTimeHasExpired() {
        if (MONGO_FAIL_POINT(maxTimeNeverTimeOut)) {
            return false;
        }
        if (_maxTimeMicros > 0 && MONGO_FAIL_POINT(maxTimeAlwaysTimeOut)) {
            return true;
        }
        return _maxTimeTracker.checkTimeLimit();
    }
```

You can replace this with:

```
probe begin {
    println("systemtap is starting")
}
// function abcd (ARG_1:TYPE, ARG_2:TYPE) %{ SOME_C_CODE; %}
// this is the syntax for embedding C code
function set_return_val () %{
  STAP_RETVALUE = true; // set the return value to true or false
%}
probe process("mongod").function("_ZN5mongo5CurOp17maxTimeHasExpiredEv").return {
  println("injecting return value")
  $return = set_return_val() // $return sets the return value
}
// alternatively, you can set it to 1 or 0 for this particular use case
// but you'll need to embed C code if you want booleans since systemtap doesn't have this data type
```

Note that you have to use the mangled function name, which you can find by going through the output of `objdump --sym mongod` (Although the next release of systemtap should allow you to probe with unmangled names)

#### Example: Probing by Markers
If neither approach works great, there is another option to modify the code probe by adding a DTrace marker. Here's the example from Systemtap's website

In the source file:

```
#include <sys/sdt.h>
```

and insert the following at each location instrumentation is desired. `provider` is an arbitrary symbol identifying your application or subsystem, and `name` is an arbitrary symbol identifying your probe point. Markers can include a fixed number of arguments that are either integer or pointer values. Below is an example of a marker with four arguments:

```
DTRACE_PROBE4(provider, name, arg1, arg2, arg3, arg4)
```

Systemtap can attach to these markers using this syntax. You may use wildcards or fully spell out the provider and marker names in the DTRACE_PROBE. Within the probe handlers, arguments may be accessed with $arg1 for values, user_string($arg2) for dereferencing pointers, or pretty-printed with $$parms. The values $$name and $$provider are also available to match up the current probe pointer. Process names may also be abbreviated.

```
probe process("a.out").provider("p").mark("n") { println($arg1) }
```


#### Example: Basic Profiling
The following is a simple example that prints out the amount of time spent in `snappy_compress` every second. It also shows how to use systemtap variables.

A note on performance: the probes should have low overhead. When I ran the following script on an insert only workload, there's about a 5%-15% drop in performance.

```
global start_epoch
global time_counter

probe process("/home/guo/mongo/mongod").function("wt_snappy_compress") {
  start_epoch = gettimeofday_us()
}
probe process("/home/guo/mongo/mongod").function("wt_snappy_compress").return {
  time_counter = gettimeofday_us() - start_epoch
}
probe timer.ms(1000) {
  printf("time spent in compress in basis points: %ld\n", time_counter/100)
  time_counter = 0
}
```

### Dtrace and Systemtap

#### Translating DTrace to Systemtap
Examples of Some Scripts in DTrace and Systemtap

- Print out detailed information on signals.  
```
dtrace -n 'proc:::signal-send /pid/ { printf("%s -%d %d",execname,args[2],args[1]->pr_pid); }'
```  
```
stap -e 'probe signal.send { if (pid()) printf("%s -%d %d\n",execname(), sig, sig_pid); }'
```  
- Write size distribution by executable name.  
```
dtrace -n 'sysinfo:::writech { @dist[execname] = quantize(arg0); }'
```  
```
stap -e '
global bytes
probe syscall.write.return, syscall.writev.return, syscall.pwrite.return {
    if ($return>=0) bytes[execname()] <<< $return
}
probe end {
    foreach (e in bytes) {printf("%s\n", e); print(@hist_log(bytes[e]))}
}
'
```  

Official wiki on how to port over a script:  
https://sourceware.org/systemtap/wiki/PortingDTracetoSystemTap
#### Systemtap vs. DTrace Feature Comparison
https://sourceware.org/systemtap/wiki/SystemtapDtraceComparison
### Appendix
1. List of functions available in Systemtap:  
https://sourceware.org/systemtap/tapsets/
