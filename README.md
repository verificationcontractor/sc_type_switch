# How to switch easily between bool and sc_logic in SystemC

When we run pin level simulations in SystemC we sometimes want to use the 4-valued type ```sc_logic``` in order to detect design bugs such as unconnected ports or ambiguous conditions that lead to X value propagation. While this can be beneficial for verification, it comes with the cost of slowing down the simulation. Other times, for example in the late stages of pre-silicon verification when our trust in the design is higher, we just want to run fast simulations by using the 2-valued ```bool``` type. 

It is possible to automate the switch between the two types over an entire project by using the C macros shown below:
```cpp
// sc_type_switch.h
#ifndef __SC_TYPE_SWITCH__
#define __SC_TYPE_SWITCH__

// This is where you decide what value set to use
#define SC_USE_LOGIC_TYPES
// or #define SC_USE_LOGIC_TYPES

#if defined(SC_USE_LOGIC_TYPES)
  #define sc_bit_type sc_logic
  #define sc_vec_type sc_lv
  #define sc_0 SC_LOGIC_0
  #define sc_1 SC_LOGIC_1
#elif defined(SC_USE_BOOL_TYPES)
  #define sc_bit_type bool
  #define sc_vec_type sc_bv
  #define sc_0 false
  #define sc_1 true
#endif

#endif // __SC_TYPE_SWITCH__
```
First you have to include the ```sc_type_switch.h``` file (listed above) in your project. Then, if you use only ```sc_bit_type``` for single bit variables/signals and ```sc_vec_type``` for multi-bit variables/signals, switching between ```bool``` and ```sc_logic``` all over your project can be as simple as activating one of the ```SC_USE_BOOL_TYPES/SC_USE_LOGIC_TYPES``` compilation switches either in the code or with a -D argument to the compiler. 

Taking a closer look at the code you can see that all it does is deal with the main differences between the ```bool``` and ```sc_logic``` types:
* How values 0 and 1 are represented - ```true/false``` vs. ```SC_LOGIC_0/SC_LOGIC_1```
* What data types are used for single bit variables and multi-bit variables - ```bool/sc_bv``` vs ```sc_logic/sc_lv```

## Assigning/writing constants
Assigning constant values to variables and writing to signals should be done only using __string values and the ```sc_0/sc_1``` macros__ as shown below:
```cpp
sc_bit_type bit_val;
sc_vec_type<3> vec_val;
sc_signal<sc_bit_type> bit_signal;
sc_signal<sc_vec_val<3> > vec_signal;
// ...
bit_val = sc_1; // use only the sc_0/sc_1 macros for single bit variables
vec_val = "101"; // use only strings for multi-bit variables
bit_signal.write(sc_0);
vec_signal.write("010");
```
## Using X and Z values
If you absolutely need to use values ```SC_LOGIC_X``` and/or ```SC_LOGIC_Z``` in expressions (for example in checkers) or to write "X"/"Z" values to signals to generate error scenarios you need to __enclose the entire statement with ```#ifdef SC_USE_LOGIC_TYPES ... #endif```__ to avoid compilation or runtime errors. For example:
```cpp
sc_bit_type d = data_signal.read();
sc_bit_type v = data_valid.read();

void check_for_data_xz() {
#ifdef SC_USE_LOGIC_TYPES
  if( (d == SC_LOGIC_X || d == SC_LOGIC_Z) && v == sc_1 ) // this could cause a compilation error without the ifdef
    SC_REPORT_ERROR("DATA_X_ERR", "Data can't be X or Z while the valid signal is high.")
#endif
}

void drive_data_x() {
#ifdef SC_USE_LOGIC_TYPES
  data_signal.write(SC_LOGIC_X); // this could cause a compilation error without the ifdef
  data_bus.write("100X"); // this could cause a runtime error without the ifdef
#endif
}
```
## sc_bit is deprecated
Instead of ```bool``` you could also use ```sc_bit``` but be aware that ```bool``` is faster since it's a native C++ type and that ```sc_bit``` has been marked as deprecated in the latest version of SystemC.

## Conclusion
Having the option to switch between slow but realistic simulations and fast but less realistic simulations can help solve many problems during the development and testing of complex SystemC projects. Running fast 2-valued simulations can expose functional bugs and also ease the debugging of simulations that would otherwise take much longer to finish. Running slower 4-valued simulations can expose connectivity bugs and ambiguous conditions that would be impossible to spot in a 2-valued simulation. It's up to the engineers themselves to decide which value set to use in order to meet the desired goal.

## Further reading
In practice, 4-valued simulations are a double edged sword because some discovered bugs are not actually something that you would encounter in real life (false negatives) but rather some glitch of the simulation software while other bugs could go completely undiscovered in simulation and show up later in real life (false positives) even if the simulation was 4-vlued. This problem is explored in more depth by Stuart Sutherland in his article [I’m Still In Love With My X!](https://sutherland-hdl.com/papers/2013-DVCon_In-love-with-my-X_paper.pdf)

The book [SystemC From the Gound Up](https://www.springer.com/gp/book/9780387699578) also explores in depth the problem of value sets showing you among other things how to use the ```sc_signal_resolved``` channel type that supports multiple drivers and how to model some system-level bus concepts such as pull-ups, pull-downs, open-source of open-drain variations (see chapter 9.3).

If you want to write RTL code in SystemC and make use of 4-value logic you must make sure that your code is synthesizable. While different synthesis tools support different synthesizable subsets of SystemC it's worth having a look over Accellera's [SystemC Synthesizable Subset](https://www.accellera.org/images/downloads/standards/systemc/SystemC_Synthesis_Subset_1_4_7.pdf) and the [Describing
Synthesizable RTL
in SystemC](https://www.cl.cam.ac.uk/research/srg/han/ACS-P35/documents/synopsys-rtl-synth.pdf) book from Synopsys.

---

I hope you found this article useful. For any questions or comments on the topic please use the "Issues" section of this Github repo.
