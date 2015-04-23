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

#### Building 3rd Party Libraries with debug symbols
Third party libraries need to be built with debug symbols to enable probing by line number and fault injection, the following is instructions for building OpenSSL with debug symbols:  
- clone the repo
- checkout a tag corresponding to a release, because the branches don't always build
- `./config -d shared`
- `sudo make && sudo make install` (It should install to /usr/local/ssl)
- `ln -s /usr/local/ssl/include /usr/include/openssl`
- `ln -s /usr/local/ssl/lib /usr/lib64/openssl`

### Dynamic Failpoint
You can run the following scripts with `stap -g SCRIPT_NAME.stp`. The `-g` is for "guru mode" which lets you change variable values at runtime. The normal mode is read-only.

#### Replacing Some MONGO_FAIL_POINT
You can do most of what MONGO_FAIL_POINTs do with Systemtap. Here's an example of an existing `maxTimeNeverTimeOut` fail point.

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

#### Adding Failpoints in C code
In addition, you can place failpoints in parts of our code outside of the mongo:: namespace, e.g. in WT code, or shared libraries. This example sets the size of the `calloc` call to 0, effectively failing the memory allocation. (a better way would be to change the return value of calloc to NULL, but this illustrates the use of probing by line numbers)

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

### Profiling
The following code prints out the amount of time spent in `snappy_compress` every second

```
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

1. Print out detailed information on signals.  
```
dtrace -n 'proc:::signal-send /pid/ { printf("%s -%d %d",execname,args[2],args[1]->pr_pid); }'
```  
```
stap -e 'probe signal.send { if (pid()) printf("%s -%d %d\n",execname(), sig, sig_pid); }'
```  
2. Write size distribution by executable name.  
```
dtrace -n 'sysinfo:::writech { @dist[execname] = quantize(arg0); }'
```  
```
stap -e '
global bytes;
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
