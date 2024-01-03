
XLua includes two built-in tools for analyzing performance-related issues: one is an analysis tool for measuring the duration of Lua function calls to C# functions (which is not necessarily the same as CPU time consumption, as time spent during coroutine yields is also included in the call duration); the other is a tool for locating memory leaks.

## Function Call Duration Analysis Tool

### Typical Use Cases:

Explanation:

The API is very straightforward, consisting of just three functions: start, stop, and report. Both start and stop take no parameters and are self-explanatory, marking the beginning and end of the measurement period, respectively. You can call report multiple times between start and stop (it's also possible to omit calling stop). Each report generates a function duration statistical report from the start to the point of invoking report (returned as a string). The report function has a single optional parameter that allows sorting by total time (with the string "TOTAL" as the parameter, which is the default), average time per call ("AVERAGE"), or number of calls ("CALLED"). A typical duration statistical report is as follows:

The first column lists the function names.

The second column shows the source code. If it's a Lua file, it will include the file and line number. If it's exported C# code, it will be marked with [C#]. If it's a C function, it will be denoted as [C].

The subsequent columns display the total time, average time per call, percentage of total recorded time, and the number of calls.

## Memory Leak Locating Tool

### Typical Use Cases:

Explanation:

There are only two functions in the API. Total retrieves the memory usage of the Lua virtual machine in Kbytes and returns it as a Lua number. Snapshot returns the current memory snapshot information. A typical memory snapshot report is as follows:

The first column is the name of the table variable.

The second column indicates the size of the table, including any subtables.

The third column specifies the variable type. UPVALUE refers to a variable within a closure (an inner function's reference to an outer local variable), GLOBAL denotes a global variable, and REGISTRY refers to some private data on the C side (including that within the virtual machine). The primary focus is usually on the first two types.

The fourth column is the variable's ID, essentially the memory pointer. If the ID remains the same across reports without reallocation, it can be used to identify whether it's the same table.

The final column provides additional information, such as which functions reference the variable, which can be useful when multiple closure variables share the same name, to assist in pinpointing the location.

How to locate memory leaks: Detect continuous memory growth using memory.total, then pinpoint the source of the leak with memory.snapshot (memory leaks in Lua typically manifest as data being added to a table without being deleted).
