# VHDL Design Practices

General Good practices:
- Label your loops, processes etc. Makes it easier to debug

## Combinatorial circuts:

This is for parts of the circuit that do not involve registers, so everything gets processed simultaneously. I will take the example of a basic OR gate. The program is written as:
```
library ieee;
use ieee.numeric_std.all;
use ieee.std_logic_1164.all;

entity test_or is
	port (
	a : in std_logic;
	b : in std_logic;
	c : out std_logic);
end test_or;

architecture or_beh of test_or is

begin

process(a, b)
begin
	c <= a or b;
end process;

end or_beh;
```

Some things to note here: the sensitivity list contains both a and b, and generally if the circuit part is purely combinatorial then it should always contain all of the inputs of the part in the sensitivity list. So you can look at the '<=' and everything that is at the right hand side should always be in the sensitivity list. The above example synthesizes correctly and gives the netlist as 

![OR](https://github.com/taitaisama/VHDL_design_practices/blob/main/OR.png?raw=true)

Now if I change the process part to

```
process(a)
begin
   c <= a or b;
end process;
```

and then compile this then first of all I get the warning: 

`Warning (10492): VHDL Process Statement warning at test_or.vhd(18): signal "b" is read inside the Process Statement but isn't in the Process Statement's sensitivity list`

Now does this synthesize? Yes it does, however the synthesis tool will still generate the same netlist, which will be a simple OR gate with a and b as inputs and c as output. But the thing to note is that the program will behave differently from the gate that is synthesized. You can tell because an OR gate output will also change when the input b changes, however the vhdl code will not behave like this and c will only update when a changes. 

So Rule 1: Always have the right hand side of all combinatorial assignments in the sensitivity list of the process, otherwise you'll get a design mismatch. 

Now we modify this example and say that we want only an OR gate if there is a boolean signal "sel" set as true. The naive implementation will be

```
library ieee;
use ieee.numeric_std.all;
use ieee.std_logic_1164.all;

entity test_or is
	port (
	a : in std_logic;
	b : in std_logic;
	c : out std_logic;
	sel: in boolean);
end test_or;

architecture or_beh of test_or is

begin

process(a, b, sel)
begin
	if (sel) then
		c <= a or b;
	end if;		
end process;

end or_beh;
```

However there is an issue with this implementation. Lets try and synthesize this and see what happens. 

I get the warning :

Warning (10631): VHDL Process Statement warning at test_or.vhd(17): inferring latch(es) for signal or variable "c", which holds its previous value in one or more paths through the process

And now my netlist looks like:

![latch](https://github.com/taitaisama/VHDL_design_practices/blob/main/ORLATCH.png?raw=true)

The thing here is that the value of c is set only in one branch of the if statement branches, so if for example sel is set to false then the signal 'c' needs to be aware of what its previous value was and accordingly keep that same value. In combinatorial circuits this behaviour is undesirable and you usually dont want to have a latch, so to fix this you have to set a default value to c when sel is false. We change the process to

```
process(a, b, sel)
begin
	if (sel) then
		c <= a or b;
	else
		c <= '0';
	end if;		
end process;
```

And now my netlist looks like 

![ORMULTIPLEX](https://github.com/taitaisama/VHDL_design_practices/blob/main/ORMULTIPLEX.png?raw=true)

So Rule 2 is: In combinatorial circuits if you are assigning a value to a signal then do it in all branches of an if statement or a switch case statement. Note that this is relevant for setting carryout bit in ALU, you might think that it isn't important to set it in an instruction like AND, however you have to always give it a default value if it isn't being set, unless you specifically want a latch (usually you wont tho).

## Registers

Now we come to Modeling registers, or edge triggered flip flops. I'll take the example of a simple register that takes the value of the input 'a' when the clock is on its rising edge. The general way to do this is to keep the sensitivity list of the process as just the clock and then have a `if(rising_edge(clock))` statement inside that. The code is

```

library ieee;
use ieee.numeric_std.all;
use ieee.std_logic_1164.all;

entity test_reg is
	port (a : in std_logic;
			clock : in std_logic;
			reg : out std_logic);
end test_reg;

architecture reg_beh of test_reg is

begin

process(clock)
begin
	if (rising_edge(clock)) then
		reg <= a;
	end if;
  
end process;

end reg_beh;
```

And as expected, we get our netlist as:

![REGISTER](https://github.com/taitaisama/VHDL_design_practices/blob/main/REGISTER.png?raw=true)

So Rule 3 is: To initialise variables that you want to be in registers you put them in a process which just has one sensitivity, `clock`, and then initialise them in an if condition as `rising_edge(clock)` or `falling_edge(clock)`. The only variables that have to be set here are registers, so don't put anything that you don't want as a register inside this if statement. Also registers need to be only set inside such an if statement, don't change their values outside.

## Some more VHDL Concepts

### Generics

Suppose your design needs registers of varying sizes: 8, 16 and 32 bits. The naive way of doing this is to make three register components `reg8`, `reg16` and `reg24`, but we can use generics to pass the size attribute to the register. Here's a simple example:

```
entity reg is
    generic (size : integer := 32 );
    port (
        data_in  : in std_logic_vector((size-1) downto 0);
        data_out : out std_logic_vector((size-1) downto 0);
        clock    : in std_logic;
        w_en     : in std_logic
    );
end reg;
```

note that size is a `generic`, whose default value is 32. We can instantiate this register as follows:
```
    reg1: entity work.reg generic map (16) port map (data_in, data_out, clock, w_en)
```

This creates a 16-bit register. Omitting the `generic map ()` statement will create a 32-bit register, which is the default register size.

### `for ... generate`

Suppose you have a 32-bit sequential adder that you need to create with 32 one-bit adders. The naive way is to manually write out all 32 entries (or write a script to generate them), but the `for ... generate` statement allows us to generate and chain all these adders using a for loop

```
entity full_adder is port (x,y,c_in: in std_logic; s,c_out: out std_logic);
end full_adder;
.
.
entity int_adder is 
    port (
        a,b: in std_logic_vector(31 downto 0); 
        ans: out std_logic_vector(31 downto 0);
        c_in: in std_logic;
        c_out: out std_logic;
    );
end int_adder;

architecture int_adder_df of int_adder is
    signal carries: std_logic_vector(32 downto 0);
begin
    gen_adders: for i in 0 to 31 generate
        ADDX : entity work.int_adder port map
            (a(i), b(i), carries(i), ans(i), carries(i+1));
    end generate gen_adders;

    .
    .
end int_adder_df;
```

## References

1. _Perry, Douglas L_. VHDL: Programming by Example. 4th ed, McGraw-Hill, 2002.
