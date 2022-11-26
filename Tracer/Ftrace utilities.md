# Ftrace utilities 

## Function trace

**current_tracer**: The way to enable a tracer.

```Shell
# echo function > current tracer
# echo nop > current_tracer
```

**Function tracing**:

Trace almost any function in the kernel!

* See the possible functions in "available_filter_functions"

LImit what functions you trace

* set_ftrace_filter - Only trace these functions
* set_ftrace_notrace - Do not trace these functions ()

Just echo function names into the files:

```Shell
# echo foo > set_ftrace_notrace
```

Can add more than one a time (white space delimited)

```SHell
# echo foo bar > set_ftrace_filter
```

Append with the bash concatenation ">>"

```Shell
# echo zoot >> set_ftrace_filter
```

Clear with just writing nothing into it

```Shell
# echo > set_ftrace_notrace
```

Can handle minor wild cards "*" and "?"

```Shell
# echo '?lock*d' > set_ftrace_filter
```

Can use available_filter_functions for more complex filtering

```Shell
# cut -d' ' -f1 available_filter_functions | grep -E 'ipv(4|6)' > set_ftrace_filter
```

What happen if there are two function has the same name in different module?

**Function Graph tracing**:

Don't trust all the times (just take it as a guide).

Function graph tracer adds overhead

* You see the overhead of functions traced within other functions

Closest to actual time is a single entity

* The enter and exit of a function are together
* Denoted by single event with ';'
* Instead of two events with '{' and '}'

The set_ftrace_filter and set_ftrace_notrace affect function_graph.

## Trace Event

Function trace is great, it gives us the accurate timestamp which we can use it to evaluate the system performance. But it don't give the variable information, we don't know what happening within the kernel with ftrace.

So the trace events comes out. Which brings us to Trace Events:

* They are points in the kernel that write into the trace buffer
* Record specific data within the kernel
* Allows to see more detailed view of what is happening

### Event Triggers

Make a event do something special

* Turn off tracing
* Turn on tracing
* Take a "snapshot"
* Produce a stack dump
* Enable another event
* Disable another event

Filtered Events along with triggers

```Shell
# echo 'prev_comm == "bash"' > events/sched/sched_switch/filter
# echo 'stacktrace if prev_comm == "bash"' > events/sched/sched_switch/trigger
# echo 1 > events/sched/sched_switch/enable
# echo > trace
# cat trace
```

Triggers are a little more difficult to remove

```Shell
# echo '!stacktrace' > events/sched/sched_switch/trigger
```

## Trace-cmd

Using `splice` system call to do zero copy.

### trace-cmd record and start

"start" enables tracing but does no recording

* Use "show" to see the in-kernel ring buffer output "trace" file.

"record" records the trace data into a file (default: trace.dat)

* Use "report" to read the file
* The "record" will zero copy from the per_cpu files

"start" has most the same options as "record"

* To enable tracing
* "-e" for events
* "-p" for tracers (historically they were once called "plugins")

