---
layout: post
title: Testing arithmetic latency
---
This post is about testing the floating point speed of a standard CPU.
It is said that a modern x86_64 CPU can execute something like 3 to 4 floating
point operations per cycle. If the arithmetic [instruction latency][1] is 3-5 cycles,
how is it possible to reach anywhere close the theoretical maximum speed?
Assuming instruction latency of 4 cycles then in order to execute 4 operations per
cycle 16 calculation, in avarage, needs to be in flight all the time.

Instruction latency is the time needed to wait for result to be available for
a dependent operation. The key here is the dependent operation. A dependent
operation is one the depends on the result of some previous operation. 

{% highlight c %}
  float x, y, a, b;
  x = x*a + b;
  y = x*a + b;  // depends on x
{% endhighlight %}


If the calculations can be broken to series of independent operations it may be 
possible to fill the execution pipeline and keep the cpu busy and hopefully approach
the maximum speed. Pipelined instructions executed in parallel is called instruction
level parallelism (ILP). 

In order to test how many independent operations are needed to fill the pipeline
I wrote a simple test program inspired by Vasily Volkov's [GPU benchmarking papers][2].
The core of the test is tight loop that mimicks matrix-matrix multiplication - a
multiplication and a dependent addition operation.

{% highlight c %}
  double x[N];
  for (j = 0; j < ROWS; j++) {
    for (i = 0; i < ROWS; i += N) {
      x[0]   = x[0]*a + b;
      // ....
      x[N-1] = x[N-1]*a + b;
    }
  }
{% endhighlight %}

Running the test on a Lenovo X121e laptop with Core i3 processor clocked at 1.4Ghz
produces following results. (cpuid: *Core(TM) i3-2367M CPU@1.40GHz*)

![double precission ops/cycle][one]


Changing the data type to single precision float gives approximately same results.
How do I interpret the results? I don't know the CPU architecture well enough to say
whether results are as expected for single instruction, single data (SISD) computation.
However, as the Core architecture has two FPUs then two operations per cycle seems
plausible. The CPU also supports vectorized, single instruction multiple data,
computation with 128bit SSE and 256bit AVX instruction sets. The 128bit wide SSE
instructions handle two double precision and four single precision floats with one
instruction. And the 256bit wide AVX set handles four double precision and eight
single precision floats respectively. 

![all ops/cycle][all]

That looks like expected, the vectorized performance being multiplies of the
unvectorized version by the vector width. And according to [this document][3]
the Core-i3 CPU can sustain 16 single precision and 8 double precision flops/cycle.

What do I conclude from the results? Clearly more than eight indepent operations does 
not give any significant additional performance and is enough to hide arithmetic latency
almost completely. As the test code does not access any
memory and is running within CPU registers it gives some sort of upper
bound for floating point performance. An actual usefull calculation would have
more memory access and that would bring the memory latencies into the picture.


[1]: http://www.agner.org/optimize/instruction_tables.pdf
[2]: http://www.cs.berkeley.edu/~volkov/
[3]: http://www.realworldtech.com/sandy-bridge/6/
[one]: /assets/x121e_ops_per_tick500x360.png "Double precision performance, ops/cycle"
[all]: /assets/x121e_all_ops_per_tick500x360.png "Performance, ops/cycle"