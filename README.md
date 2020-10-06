# How to switch easily between bool and sc_logic in SystemC

SystemC supports two types of value sets used to model signal levels: 
* the 4-valued ```sc_logic``` type consisting of values ```SC_LOGIC_0```, ```SC_LOGIC_1```, ```SC_LOGIC_X``` and ```SC_LOGIC_Z```
* the 2-valued ```bool``` type consisting of values ```true``` and ```false```

The 4-valued ```sc_logic``` type is inspired from HDLs like Verilog and VHDL and it's meant to be used mostly in design simulations to detect design bugs such as unconnected ports or ambiguous conditions that lead to X value propagation. While this can be beneficial for verification, it comes with the cost of slowing down simulations. Other times we might not be so concered about ambiguities and connectivity issues and we just want to run fast simulations with the 2-valued ```bool``` value set to test other types of scenarios.

It would be very conventient to have a simple way of switching between the two value sets automatically without having to change the types of variables over the entire source code by hand. 

## Use define switches
Let's introduce two #define switches ```SC_USE_BOOL_TYPES``` and ```SC_USE_LOGIC_TYPES``` corresponding to each of the value sets mentioned above. Defining one switch or the other at compilation time will force the chosen value set to be used automatically in the entire project provided that we respect some coding rules and guidelines that will be described later on. Let's also agree that by default we are going to use ```SC_USE_BOOL_TYPES``` if none of these switches are defined. The source code for such a default option would look like this:
```cpp
#if !defined(SC_USE_LOGIC_TYPES) && !defined(SC_USE_BOOL_TYPES)
#define SC_USE_BOOL_TYPES
#endif
```

## Differences between sc_logic and bool
Next we must understand what are the main differences between the ```sc_logic``` and ```bool``` types to figure out what needs to change when a value set switch is desired:
### Values 0 and 1 are represented differently:
* __bool__ uses ```false``` for 0 and ```true``` for 1 
* __sc_logic__ uses ```SC_LOGIC_0``` for 0 and ```SC_LOGIC_1``` for 1
* __sc_logic__ has two extra values ```SC_LOGIC_X``` and ```SC_LOGIC_Z```
* ```SC_LOGIC_0``` and ```SC_LOGIC_1``` are constant class instances not enum values nor integer constants
* __SOLUTION:__
  * Use two macros ```sc_0``` and ```sc_1``` that can translate to either ```true/false``` or ```SC_LOGIC_0/SC_LOGIC_1``` depending on which value set is chosen
### Data types for single bit and multi-bit variables are different:
* __single-bit bool__ variables are declared with the ```bool``` type
* __multi-bit bool__ variables are declared with the ```sc_bv``` type
* __single-bit sc_logic__ variables are declared with the ```sc_logic``` type
* __multi-bit sc_logic__ variables are declared with the ```sc_lv``` type
* __SOLUTION:__
  * When declaring single-bit variables use a macro ```sc_bit_type``` that translates to either ```bool``` or ```sc_logic``` depending on which value set is chosen
  * When declaring multi-bit variables use a macro ```sc_vec_type``` that translates to either ```sc_bv``` or ```sc_lv``` depending on which value set is chosen
### The logical negation operator for single-bit values is different
* __sc_logic__ uses uses the tilda ```~``` operator
* __bool__ uses the exclamation mark ```!``` operator
* Applying the ```~``` (bitwise negation) operator to ```bool``` variables will return unexpected results:
  * Applying ```~``` to ```false```(0x00) will return 0xFF which in the ```bool``` world is equal to ```true``` so there's no problem here
  * Applying ```~``` to ```true```(0x01) will return 0xFE which in the ```bool``` world is equal to ```true``` __which is wrong because we want the logical negation of ```true``` to be ```false```__
* __SOLUTION:__
  * Use a macro ```sc_not(x)``` that uses ```!``` or ```~``` depending on which value set is chosen
  * This macro should only be used for single-bit variables
  * For multi-bit variables you can still use ```!``` for logical negation and ```~``` for bitwise negation
### When assigning literals to variables there are both similarities and differences
* both __sc_bv__ and __sc_lv__ variables can be assigned the following types of literals:
  * integer literals: e.g. 85 (decimal), 0213 (octal), 0x4b (hexadecimal), 30 (int), 30u (unsigned int), 30l (long), 30ul (unsigned long), etc.
  * string literals representing integer values: e.g.  "0b10100",  "0d478", "0xFF", etc. (See the SystemC LRM for more details on the syntax of these literals)
* assigning any invalid string literals (not representing numeric values) to ```sc_bv``` or ```sc_lv``` will result in an error
* assigning string literals containing X/Z values to ```sc_bv``` will result in an error
* both ```bool``` and ```sc_logic``` variables can be assigned the following literals:
  * integer literals
    * ```bool``` will see zero values as false and non-zero values as true
    * ```sc_logic``` will only accept integer literals equal to 0 or 1, otherwise an error message will the printed and the conversion will return a value of SC_LOGIC_X
  * character literals e.g. 'a' 'b' 'c'
    * ```bool``` will see character '\0' as false and everythin else as true
    * ```sc_logic``` will only accept chars with ASCII codes that are less than 127 otherwise an error message will the printed and the conversion will return a value of SC_LOGIC_X
      * for characters with ASCII codes 0, 1, 2 and 3 the conversion will return values SC_LOGIC_0, SC_LOGIC_1, SC_LOGIC_Z, SC_LOGIC_X
      * for characters '0', '1', 'z'/'Z' and 'x'/'X' the conversion will return value SC_LOGIC_0, SC_LOGIC_1, SC_LOGIC_Z, SC_LOGIC_X
      * for any other ASCII code the conversion will return value SC_LOGIC_X
  * ```true/false``` literals 
* ```bool``` will aslo accept floating point literals e.g. 0.0 (false), 1.27 (true), 3.14159 (true) etc.
* __SOLUTION:__
  * Do not assign floating point literals to ```sc_bit_type``` or ```sc_vec_type``` variables, it won't compile when SC_USE_LOGIC_TYPES is defined.
  * Do not assign character literals to ```sc_bit_type``` because conversion is different for ```bool``` and ```sc_logic``` 
  * For better performance it's recommended to assign integer literals to ```sc_vec_type``` variables
  * For better readability it's recommended to assign string or character literals to ```sc_vec_type``` variables
  * When it's imperative to use literals that contain X/Z values (e.g. "01X", 'x'. 'Z', SC_LOGIC_X) the code where they are used must be enclosed with ```#ifdef SC_USE_LOGIC_TYPES ... #endif```__ to avoid compilation or runtime errors. See example below:
```cpp
sc_bit_type d = data_signal.read();
sc_bit_type v = data_valid.read();

void check_for_data_xz() {
#ifdef SC_USE_LOGIC_TYPES
  if( (d == SC_LOGIC_X) && v == sc_1 ) // this could cause a compilation error without the ifdef
    SC_REPORT_ERROR("DATA_X_ERR", "Data can't be X while the valid signal is high.")
#endif
}

void drive_data_x() {
#ifdef SC_USE_LOGIC_TYPES
  data_signal.write(SC_LOGIC_X); // this could cause a compilation error without the ifdef
  data_bus.write("100X"); // this could cause a runtime error without the ifdef
#endif
}
```
### Integer arithmetic operations can't be performed on sc_logic, sv_bv and sc_lv and the workarounds can yield different effects
* Types __sc_logic__, __sc_bv__ and __sc_lv__ don't support integer arithmetic operators +, - , * and /
* They only support bitwise logic operators &, | and ^
* In order to do integer arithmetic computations with variables of these types, their contents need to be copied to ```sc_int``` or ```sc_uint``` or other integer variables or data structures, do the computations and then convert the results back to types ```sc_logic```, ```sc_bv``` or ```sc_lv```
* __bool__ variables are already represented internally as integers so they can be used directly in arithmetic expressions without coversions or casts.
* If ```sc_logic``` or ```sc_lv``` variables containing X/Z values are assigned to ```sc_int``` or ```sc_uint``` variables then the X/Z bits become '1', e.g. "XXZZ01" becomes "111101". This means that X/Z values get lost when doing arithmetic operations with ```sc_logic``` and ```sc_lv``` variables.
* __SOLUTION:__
  * Introduce a function ```bool has_xz(sc_vec_type)``` that checks if an ```sc_vec_type``` variable contains X/Z values
  * Don't use ```sc_bit_type``` variables directly in integer arithmetic expressions as this can cause compilation errors when SC_USE_LOGIC_TYPES is defined. Always copy them to variables which support the desired arithmetic operators and then convert the result back to ```sc_bit_type```.
  * Use the has_xz function to test ```sc_vec_type``` variables before using their contents in arithmetic operations and decide how to treat the cases when X/Z values are present. See example below:
```cpp
sc_vec_type<3> vec_var;
sc_vec_type<3> result_var;
// ...
if(!has_xz(vec_var)) { // no X/Z => do integer arithmetic
  sc_uint<3> int_var = vec_var;
  int_var = int_var/2 + 5;
  result_var = int_var;
} 
#ifdef SC_USE_LOGIC_TYPES
else
  result_var = "XXX"; // Pessimistic X propagation
#endif
```
    
## Putting it all together
Most of the solutions mentioned above can be put together in a header file ```sc_type_switch.h``` that would look like this:
```cpp
// File: sc_type_switch.h
#ifndef __SC_TYPE_SWITCH__
#define __SC_TYPE_SWITCH__

#if !defined(SC_USE_LOGIC_TYPES) && !defined(SC_USE_BOOL_TYPES)
#define SC_USE_BOOL_TYPES
#endif

// Define sc_bit_type, sc_vec_type, sc_0, sc_1 and sc_not
#if defined(SC_USE_LOGIC_TYPES)
  #define sc_bit_type sc_logic
  #define sc_vec_type sc_lv
  #define sc_0 SC_LOGIC_0
  #define sc_1 SC_LOGIC_1
  #define sc_not(x) (~(x))
#elif defined(SC_USE_BOOL_TYPES)
  #define sc_bit_type bool
  #define sc_vec_type sc_bv
  #define sc_0 false
  #define sc_1 true
  #define sc_not(x) (!(x))
#endif

// Test if sc_bit_type is X or Z
bool has_xz(sc_bit_type val) {
#ifdef SC_USE_BOOL_TYPES
  return false;
#else
  return !val.is_01();
#endif
}

// Check if sc_vec_type has X or Z values
template<W>
bool has_xz(sc_vec_type<W> val) {
  return !val.is_01();
}

#endif // __SC_TYPE_SWITCH__
```

Include the ```sc_type_switch.h``` file in your project and switch between value sets by defining either SC_USE_BOOL_TYPES (default) or SC_USE_LOGIC_TYPES.

Besides including the header you must also respect the following rules and guidelines to avoid compilation or runtime errors:
* Use only ```sc_bit_type``` for single bit variables/signals and ```sc_vec_type<WIDTH>``` for multi-bit variables/signals
* Use only the sc_0 and sc_1 to represent the single-bit values 0 and 1
* Use only the ```sc_not``` macro to negate single-bit variables
* Do not assign character literals to ```sc_bit_type``` because conversion is different for ```bool``` and ```sc_logic``` 
* Do not assign floating point literals to ```sc_bit_type``` or ```sc_vec_type``` variables, it won't compile when SC_USE_LOGIC_TYPES is defined.
* When it's imperative to use literals that contain X/Z values (e.g. "01X", 'x'. 'Z', SC_LOGIC_X) the code where they are used must be enclosed with ```#ifdef SC_USE_LOGIC_TYPES ... #endif```__ to avoid compilation or runtime errors.
* Always use the has_xz function to test ```sc_vec_type``` variables before using their contents in arithmetic operations and decide how to treat the cases when X/Z values are present.
* Do not use ```sc_bit_type``` variables directly in integer arithmetic expressions as this can cause compilation errors when SC_USE_LOGIC_TYPES is defined. Always copy them to variables which support the desired arithmetic operators and then convert the result back to ```sc_bit_type```.
* Input ports that are meant to be connected to ```sc_clock``` generators must always be declared as ```sc_in<bool>``` because ```sc_clock``` only generates bool values.
* According to the "SystemC Synthesis Subset v1.4.7" the inpot port type ```sc_in<sc_logic>``` is not synthesizable so make sure that you define SC_USE_BOOL_TYPES to make code synthesizable. Another solution is to declare all input ports as sc_in<bool> and propagate their values to internal sc_logic variables.
* For better performance it's recommended to assign integer literals to ```sc_vec_type``` variables
* For better readability it's recommended to assign string or character literals to ```sc_vec_type``` variables

## sc_bit is deprecated
Instead of ```bool``` you could also use ```sc_bit``` but be aware that ```bool``` is faster since it's a native C++ type and that ```sc_bit``` has been marked as deprecated in the latest version of SystemC.

## SR Latch example
Let's apply this solution to a simple example: an SR latch. SR latches are level sensitive storage elements with two inputs S(Set) and R(Reset) and two outputs Q and NQ. It stores a value of 1 (Q=1, NQ=0) when the S input toggles from low to high while R is low. It stores a value of 0 (Q=0, NQ=1) when the R input toggles from low to high while S is low. When both S and R are low the latch holds the currently stored value. When S and R are both high the stored value can be anything so the output is X. See below the code of an SR latch model and a testbench module that applies some stimuli to its inputs:
```cpp
// File: sr_latch_example.cpp
#include <systemc>

#include "sc_type_switch.h"

using namespace sc_core;
using namespace sc_dt;

struct sr_latch_example: public sc_module {
    SC_HAS_PROCESS(sr_latch_example);
    // Ports
    sc_in<sc_bit_type> s;
    sc_in<sc_bit_type> r;
    sc_out<sc_bit_type> q;
    sc_out<sc_bit_type> nq;

    // Variables
    sc_bit_type stored_value;

    // SC_METHOD - the latch logic
    void propagate() {
        // Propagate inputs
        if(s.read() == sc_0 && r.read() == sc_0) { 
            // Do nothing, hold value
        } else
        if(s.read() == sc_1 && r.read() == sc_0) {
            // Set value
            stored_value = sc_1;
        }
        else 
        if(s.read() == sc_0 && r.read() == sc_1) {
            // Reset value
            stored_value = sc_0;
        } else
        if(s.read() == sc_1 && r.read() == sc_1) {
            // Illegal input, output X if sc_logic is used
            #ifdef SC_USE_LOGIC_TYPES
            stored_value = SC_LOGIC_X;
            #endif
        } else { // X or Z on inputs
            // Do nothing, hold value
        }

        // Propagate outputs
        q.write(stored_value);
        nq.write(sc_not(stored_value));
    }

    sr_latch_example(sc_module_name name):sc_module(name),
    s("s"), r("r"), q("q"), nq("nq")
    {
        SC_METHOD(propagate);
        sensitive << s << r;
    }

};

struct sr_latch_example_tb: public sc_module {
    SC_HAS_PROCESS(sr_latch_example_tb);

    // Signals
    sc_signal<sc_bit_type> s;
    sc_signal<sc_bit_type> r;
    sc_signal<sc_bit_type> q;
    sc_signal<sc_bit_type> nq;

    // Latch instance
    sr_latch_example latch;

    // Trace file
    sc_trace_file* tf;

    void test() {
        // Wait some time to see initial values
        wait(10, SC_NS);
        
        // Apply hold inputs - nothing will happen, stored value will be the same
        s.write(sc_0);
        r.write(sc_0);
        wait(10, SC_NS);

        // Apply a set pulse
        s.write(sc_1);
        wait(2, SC_NS);
        s.write(sc_0);

        // Wait some time to see that the value holds
        wait(10, SC_NS);

        // Apply a reset pulse
        r.write(sc_1);
        wait(2, SC_NS);
        r.write(sc_0);

        // Wait some time to see that the value holds
        wait(10, SC_NS);

        // Apply illegal combination s=1 r=1, should output X if sc_logic is used
        s.write(sc_1);
        r.write(sc_1);
        wait(5, SC_NS);
        s.write(sc_0);
        r.write(sc_0);

        // Wait some time to see that the value holds
        wait(10, SC_NS);

        // Apply Z to one of the inputs to see what happens
        #ifdef SC_USE_LOGIC_TYPES
        s.write(SC_LOGIC_Z);
        #endif
        wait(5, SC_NS);
        s.write(sc_0);
        r.write(sc_0);

        // Wait some time to see that the value holds
        wait(10, SC_NS);
    }

    sr_latch_example_tb(sc_module_name name):
    s("s"), r("r"), q("q"), nq("nq"), latch("latch"),
    sc_module(name) {
        // Connect signals
        latch.s(s);
        latch.r(r);
        latch.q(q);
        latch.nq(nq);

        // Trace signals
        #ifdef SC_USE_LOGIC_TYPES
        tf = sc_create_vcd_trace_file("latch_logic");
        #else
        tf = sc_create_vcd_trace_file("latch_bool");
        #endif
        sc_trace(tf, s, "s");
        sc_trace(tf, r, "r");
        sc_trace(tf, q, "q");
        sc_trace(tf, nq, "nq");

        // Start test thread
        SC_THREAD(test);
    }

    ~sr_latch_example_tb() {
        sc_close_vcd_trace_file(tf);
    }
};

int sc_main(int argc, char** argv) {
    sr_latch_example_tb latch_example_rb("latch_example_rb");
    sc_start();
    return 0;
}
```

Notice how throughout the entire code only ```sc_bit_type``` is used for single-bit variables, how only ```sc_1``` and ```sc_0``` are used to represent high and low signal values and how the code portions where X values must be used are isolated with #ifdef statements. This coding style will allow us to easily choose one value set or the other and force its use throughout the entire simulation simply by passing a -D argument to the compiler.

Compiling and running with this command ```g++ -o sim -DSC_USE_LOGIC_TYPES -lsystemc sr_latch_example.cpp; ./sim``` will generate this waveform:
![picture alt](https://github.com/verificationcontractor/sc_type_switch/blob/master/images/logic_vcd.PNG "latch_logic.vcd")

Compiling and running with this command ```g++ -o sim -DSC_USE_BOOL_TYPES -lsystemc sr_latch_example.cpp; ./sim``` will generate this waveform:
![picture alt](https://github.com/verificationcontractor/sc_type_switch/blob/master/images/bool_vcd.PNG "latch_bool.vcd")

And this is how you run the same scenario with two different value sets.

## Conclusion
Having the option to switch between slow but realistic simulations and fast but less realistic simulations can help solve many problems during the development and testing of complex SystemC projects. Running fast 2-valued simulations can expose functional bugs and also ease the debugging of simulations that would otherwise take much longer to finish. Running slower 4-valued simulations can expose connectivity bugs and ambiguous conditions that would be impossible to spot in a 2-valued simulation. It's up to the engineers themselves to decide which value set to use in order to meet the desired goal.

## Further reading
The X value is the only value that does not exist in real life circuits. SystemC is very flexible in how it models the behavior of X values leaving a lot of open options for the developer. In other languages, like SystemVerilog for example, the behavior of X is more strictly defined. In some cases propagated X values affect only a part of the result (optimistic X) while in other cases a propagated X value will turn all the bits of the result to X (pessimistic X). There are advantages and disadvantages in each case with respect to realism and accuracy. This problem is explored in more depth by Stuart Sutherland in his article [I’m Still In Love With My X!](https://sutherland-hdl.com/papers/2013-DVCon_In-love-with-my-X_paper.pdf)

The book [SystemC From the Gound Up](https://www.springer.com/gp/book/9780387699578) also explores in depth the problem of value sets showing you among other things how to use the ```sc_signal_resolved``` channel type that supports multiple drivers and how to model some system-level bus concepts such as pull-ups, pull-downs, open-source or open-drain variations (see chapter 9.3).

If you want to write RTL code in SystemC you must make sure that your code is synthesizable. While different synthesis tools might support different synthesizable subsets of SystemC it's worth having a look over Accellera's [SystemC Synthesizable Subset](https://www.accellera.org/images/downloads/standards/systemc/SystemC_Synthesis_Subset_1_4_7.pdf) and an older guide [Describing
Synthesizable RTL
in SystemC](https://www.cl.cam.ac.uk/research/srg/han/ACS-P35/documents/synopsys-rtl-synth.pdf) from Synopsys.

---

I hope you found this article useful. For any questions or comments on the topic please use the "Issues" section of this Github repo.
