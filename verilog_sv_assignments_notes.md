# Verilog/SystemVerilog Notes - Assignment 2

These notes summarize the new Verilog/SystemVerilog ideas from Assignment 2.
The goal is to keep this as a practical running summary: syntax, what it does,
and small examples.

Related exercises:

- `01_decode_the_signal`
- `02_the_rendezvous`
- `03_generate_and_prove`

---

## 1. Arrays

### Syntax

```systemverilog
logic [7:0] bytes [4];
```

### What it does

This declares an unpacked array with 4 elements.
Each element is an 8-bit value.

```text
bytes[0] -> 8 bits
bytes[1] -> 8 bits
bytes[2] -> 8 bits
bytes[3] -> 8 bits
```

### Example

```systemverilog
bytes[0] = 8'h12;
bytes[1] = 8'h34;
bytes[2] = 8'h56;
bytes[3] = 8'h78;
```

Use arrays when you know the size in advance.

---

## 2. Queues

### Syntax

```systemverilog
int q[$];
```

The `[$]` means this is a queue.

### What it does

A queue is a dynamic list. You can add items and remove items during simulation.

### Useful methods

```systemverilog
q.push_back(10);     // add to the end
q.push_back(20);

int x;
x = q.pop_front();   // remove from the front, x = 10
```

After `pop_front`, the queue becomes:

```text
[20]
```

### Example: stimulus queue

This idea is useful in `03_generate_and_prove`.

```systemverilog
packet stim_q[$];
packet pkt;

initial begin
    repeat (10) begin
        pkt = new();

        if (!pkt.randomize())
            $fatal("Randomization failed");

        stim_q.push_back(pkt);
    end

    while (stim_q.size() > 0) begin
        pkt = stim_q.pop_front();
        pkt.print();
    end
end
```

---

## 3. Procedural `for` Loops

### Syntax

```systemverilog
for (int i = 0; i < 4; i++) begin
    // code
end
```

### What it does

Runs the same code multiple times during simulation.

### Example

```systemverilog
for (int i = 0; i < 4; i++) begin
    $display("i = %0d", i);
end
```

### Example: extracting bytes

Useful for `01_decode_the_signal`.

```systemverilog
logic [31:0] word;
logic [7:0]  bytes[$];

initial begin
    word = 32'h12345678;

    for (int i = 0; i < 4; i++) begin
        bytes.push_back(word[i*8 +: 8]);
    end
end
```

---

## 4. Bit Slicing

### Syntax

```systemverilog
signal[high:low]
```

### What it does

Selects part of a vector.

### Example

```systemverilog
logic [31:0] word;
logic [7:0]  byte0;

word  = 32'h12345678;
byte0 = word[7:0];     // 8'h78
```

Other slices:

```systemverilog
word[31:24] // 8'h12
word[23:16] // 8'h34
word[15:8]  // 8'h56
word[7:0]   // 8'h78
```

### Indexed part-select

```systemverilog
word[start +: width]
word[start -: width]
```

Examples:

```systemverilog
word[0 +: 8]   // word[7:0]
word[8 +: 8]   // word[15:8]
word[31 -: 8]  // word[31:24]
```

---

## 5. Concatenation

### Syntax

```systemverilog
{a, b, c}
```

### What it does

Combines several values into one larger vector.

### Example

```systemverilog
logic [7:0]  a;
logic [7:0]  b;
logic [15:0] result;

a = 8'h12;
b = 8'h34;

result = {a, b}; // 16'h1234
```

The left item goes to the higher bits.

### Example: decode fields

Useful for `01_decode_the_signal`.

```systemverilog
logic [31:0] signal;
logic [7:0]  opcode;
logic [7:0]  src;
logic [7:0]  dst;
logic [7:0]  data;

{opcode, src, dst, data} = signal;
```

---

## 6. Streaming Operators

### Syntax

```systemverilog
{<<{expr}}
{>>{expr}}
{<<8{expr}}
{>>8{expr}}
```

### What it does

Streaming operators repack bits from an expression.
They are often used to reverse or reorder chunks of bits.

### Example: byte reorder

```systemverilog
logic [31:0] word;
logic [31:0] reversed;

word     = 32'h12345678;
reversed = {<<8{word}};
```

This kind of operation is commonly used when byte order matters.

### Difference from concatenation

Concatenation is explicit:

```systemverilog
reversed = {word[7:0], word[15:8], word[23:16], word[31:24]};
```

Streaming is more compact:

```systemverilog
reversed = {<<8{word}};
```

When learning, use a small hex example and check the simulator result.

---

## 7. `task`

### Syntax

```systemverilog
task send_byte(input logic [7:0] data);
    // code
endtask
```

### What it does

A `task` is a reusable block of procedural code.
Unlike a `function`, a task may consume simulation time.

### Example

```systemverilog
task send_req(input logic [7:0] data);
    req <= 1'b1;
    bus <= data;
    @(posedge clk);
    req <= 1'b0;
endtask
```

This task waits for a clock edge, so it must be a `task`, not a `function`.

---

## 8. `function`

### Syntax

```systemverilog
function int add(input int a, input int b);
    return a + b;
endfunction
```

### What it does

A `function` computes and returns a value.
It should not consume simulation time.

### Example

```systemverilog
int result;
result = add(3, 4); // 7
```

### Task vs function

| Feature | `task` | `function` |
|---|---|---|
| Can consume time | Yes | No |
| Can use `@(posedge clk)` | Yes | No |
| Returns a value | Not directly | Yes |
| Common use | Drive protocol | Calculate value |

---

## 9. `automatic task`

### Syntax

```systemverilog
task automatic do_work(input int id);
    int local_counter;
    // code
endtask
```

### What it does

`automatic` gives each call its own local variables.
This matters when the same task runs more than once at the same time.

### Example

```systemverilog
task automatic wait_and_print(input int id, input int delay_time);
    #delay_time;
    $display("id=%0d done at %0t", id, $time);
endtask

initial begin
    fork
        wait_and_print(1, 10);
        wait_and_print(2, 20);
    join
end
```

Good habit in testbenches:

```systemverilog
task automatic ...
function automatic ...
```

---

## 10. `fork`, `join`, `join_any`, `join_none`

### `fork ... join`

Runs branches in parallel and waits for all of them.

```systemverilog
fork
    begin
        #10;
        $display("A");
    end

    begin
        #20;
        $display("B");
    end
join

$display("Both finished");
```

### `fork ... join_any`

Waits only for the first branch to finish.

```systemverilog
fork
    begin
        #10;
        $display("A");
    end

    begin
        #20;
        $display("B");
    end
join_any

$display("At least one finished");
disable fork;
```

`disable fork` stops the remaining branches.

### `fork ... join_none`

Starts the branches and continues immediately.

```systemverilog
fork
    begin
        #10;
        $display("A");
    end
join_none

$display("Continues immediately");
```

Useful for background processes.

---

## 11. `event`

### Syntax

```systemverilog
event done_event;

-> done_event;       // trigger event
@(done_event);       // wait for event
```

### What it does

An `event` is a simulation synchronization mechanism.
It means "something happened now".

### Example

Useful for `02_the_rendezvous`.

```systemverilog
event producer_done;
event consumer_ready;

initial begin
    fork
        begin
            #10;
            -> producer_done;
        end

        begin
            #20;
            -> consumer_ready;
        end

        begin
            @(producer_done);
            @(consumer_ready);
            $display("Rendezvous complete");
        end
    join
end
```

Important: an `event` is not a signal. It does not stay high or low.
If nobody is waiting when the event is triggered, it may be missed.

---

## 12. Classes, Objects, Handles, and `new`

### Syntax

```systemverilog
class packet;
    bit [7:0] data;
endclass

packet pkt;
pkt = new();
```

### What it does

`class packet` defines a type.
`packet pkt;` creates a handle.
`pkt = new();` creates the actual object.

### Example

```systemverilog
class packet;
    bit [7:0] data;

    function void print();
        $display("data=%0h", data);
    endfunction
endclass

initial begin
    packet pkt;

    pkt = new();
    pkt.data = 8'hAA;
    pkt.print();
end
```

### Common mistake

Wrong:

```systemverilog
packet pkt;
pkt.data = 8'hAA;
```

Correct:

```systemverilog
packet pkt;
pkt = new();
pkt.data = 8'hAA;
```

Without `new`, the handle is `null`.

---

## 13. `rand` and `randomize`

### Syntax

```systemverilog
class packet;
    rand int length;
    rand bit urgent;
endclass
```

### What it does

Fields marked as `rand` can receive random values when calling `randomize()`.

### Example

```systemverilog
packet pkt;

initial begin
    pkt = new();

    if (!pkt.randomize())
        $fatal("Randomization failed");
end
```

`randomize()` returns:

- `1` if randomization succeeded
- `0` if it failed

So it is better to check the return value.

---

## 14. Constraints

### Syntax

```systemverilog
constraint c_length {
    length inside {[2:12]};
}
```

### What it does

A constraint limits the legal random values.

### Example

```systemverilog
class packet;
    rand int length;

    constraint c_length {
        length >= 2;
        length <= 12;
    }
endclass
```

### Common syntax mistake

Wrong:

```systemverilog
constraint c_length {
    length inside {[2:12]};
};
```

Correct:

```systemverilog
constraint c_length {
    length inside {[2:12]};
}
```

There is no semicolon after the closing `}` of a constraint block.

---

## 15. `inside`

### Syntax

```systemverilog
x inside {[2:12]}
x inside {1, 3, 5}
```

### What it does

Checks whether a value is inside a set or range.

### Example

```systemverilog
constraint c_opcode {
    opcode inside {8'h10, 8'h20, 8'h30};
}
```

Another example:

```systemverilog
constraint c_length {
    length inside {[2:12]};
}
```

---

## 16. `dist`

### Syntax

```systemverilog
urgent dist {1 := 20, 0 := 80};
```

### What it does

Defines weighted random distribution.

### Example

```systemverilog
class packet;
    rand bit urgent;

    constraint c_urgent {
        urgent dist {1 := 20, 0 := 80};
    }
endclass
```

This means approximately:

```text
urgent = 1 -> 20%
urgent = 0 -> 80%
```

Useful in `03_generate_and_prove` for generating more realistic stimulus.

---

## 17. `always`

### Syntax

```systemverilog
always begin
    // code
end
```

or:

```systemverilog
always @(posedge clk) begin
    // code
end
```

### What it does

`always` is a general procedural block that repeats forever.

### Example: clock generator

```systemverilog
initial clk = 0;

always #5 clk = ~clk;
```

This toggles `clk` every 5 time units.

---

## 18. `always_ff`

### Syntax

```systemverilog
always_ff @(posedge clk) begin
    q <= d;
end
```

### What it does

`always_ff` describes sequential logic, such as flip-flops and registers.
It updates only on a clock edge.

### Example

```systemverilog
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        counter <= 0;
    else
        counter <= counter + 1;
end
```

Use nonblocking assignment `<=` inside `always_ff`.

---

## 19. `always_comb`

### Syntax

```systemverilog
always_comb begin
    y = a & b;
end
```

### What it does

`always_comb` describes combinational logic.
The output should depend only on current inputs, not on clock history.

### Example: decoder

Useful for `01_decode_the_signal`.

```systemverilog
always_comb begin
    case (opcode)
        2'b00: action = IDLE;
        2'b01: action = READ;
        2'b10: action = WRITE;
        default: action = ERROR;
    endcase
end
```

Use blocking assignment `=` inside `always_comb`.

---

## 20. Multiple Drivers

### What it means

A signal has multiple drivers when more than one procedural block writes to it.

### Example problem

```systemverilog
logic req;

initial begin
    req = 0;
end

always_ff @(posedge clk) begin
    req <= next_req;
end
```

Both the `initial` block and the `always_ff` block write to `req`.
With `always_ff`, this is illegal.

### Better version

```systemverilog
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        req <= 1'b0;
    else
        req <= next_req;
end
```

Rule of thumb:

```text
A signal assigned in always_ff should be assigned only in that always_ff block.
```

---

## 21. Blocking vs Nonblocking Assignment

### Blocking assignment: `=`

```systemverilog
a = b;
c = a;
```

Executes immediately, in order.

### Nonblocking assignment: `<=`

```systemverilog
a <= b;
c <= a;
```

Schedules updates for the end of the current time step.
This models flip-flops better.

### Rule of thumb

```systemverilog
always_ff   -> use <=
always_comb -> use =
```

### Example

```systemverilog
always_ff @(posedge clk) begin
    a <= b;
    b <= a;
end
```

This swaps `a` and `b` on the clock edge.

---

## 22. `@(posedge clk)`

### Syntax

```systemverilog
@(posedge clk);
```

### What it does

Waits until the next rising edge of `clk`.

A rising edge is:

```text
0 -> 1
```

### Example

```systemverilog
task automatic wait_cycles(input int n);
    repeat (n) begin
        @(posedge clk);
    end
endtask
```

---

## 23. Immediate Assertions

### Syntax

```systemverilog
assert(condition)
else $error("message");
```

### What it does

Checks a condition immediately when the simulator reaches that line.

### Example

```systemverilog
assert(length inside {[2:12]})
else $fatal("Invalid length");
```

Immediate assertions are similar to runtime checks.

---

## 24. Concurrent Assertions

### Syntax

```systemverilog
assert property (@(posedge clk) req |-> ##[1:3] done);
```

### What it does

Checks behavior over time.
Usually sampled on a clock edge.

### Example

Useful for `03_generate_and_prove`.

```systemverilog
assert property (@(posedge clk) req |-> ##[1:3] done)
else $error("done did not arrive after req");
```

Meaning:

```text
If req is high now, done must be high 1 to 3 cycles later.
```

---

## 25. `property`

### Syntax

```systemverilog
property p_req_done;
    @(posedge clk) req |-> ##[1:3] done;
endproperty

assert property (p_req_done);
```

### What it does

A `property` gives a name to an assertion rule.
This makes assertions easier to read and reuse.

---

## 26. `sequence`

### Syntax

```systemverilog
sequence s_req_then_done;
    req ##[1:3] done;
endsequence
```

### What it does

A `sequence` describes a pattern over time.
It can be used inside a `property`.

### Example

```systemverilog
sequence s_req_then_done;
    req ##[1:3] done;
endsequence

property p_req_done;
    @(posedge clk) s_req_then_done;
endproperty

assert property (p_req_done);
```

---

## 27. Implication: `|->` and `|=>`

### `|->`

```systemverilog
a |-> b
```

If `a` is true, `b` must be true starting in the same cycle.

Example:

```systemverilog
assert property (@(posedge clk) req |-> busy);
```

### `|=>`

```systemverilog
a |=> b
```

If `a` is true, `b` must be true starting in the next cycle.

Example:

```systemverilog
assert property (@(posedge clk) req |=> busy);
```

### Quick comparison

| Operator | Meaning |
|---|---|
| `|->` | same-cycle implication |
| `|=>` | next-cycle implication |

---

## 28. SVA Delay: `##`

### Exact delay

```systemverilog
a ##3 b
```

Means:

```text
b must happen exactly 3 clock cycles after a.
```

### Range delay

```systemverilog
a ##[1:5] b
```

Means:

```text
b must happen 1 to 5 clock cycles after a.
```

### Example

```systemverilog
assert property (@(posedge clk) $rose(req) |-> ##[2:12] done);
```

Meaning:

```text
When req rises, done must arrive 2 to 12 cycles later.
```

---

## 29. Vacuous Pass

### What it means

An assertion like this:

```systemverilog
assert property (@(posedge clk) req |-> ##[1:3] done);
```

passes if `req` never happens.

That is called a vacuous pass.

The assertion says:

```text
If req happens, done must follow.
```

If `req` never happens, the rule was never really tested.

### How to check that the scenario happened

Use `cover property`.

```systemverilog
cover property (@(posedge clk) req);
cover property (@(posedge clk) req ##[1:3] done);
```

---

## 30. `$rose`, `$fell`, `$stable`

### `$rose`

```systemverilog
$rose(req)
```

True when `req` changed from 0 to 1.

Example:

```systemverilog
assert property (@(posedge clk) $rose(req) |-> ##[1:3] done);
```

### `$fell`

```systemverilog
$fell(req)
```

True when `req` changed from 1 to 0.

Example:

```systemverilog
assert property (@(posedge clk) $fell(req) |-> !busy);
```

### `$stable`

```systemverilog
$stable(data)
```

True when `data` did not change since the previous sample.

Example:

```systemverilog
assert property (@(posedge clk) busy |-> $stable(data));
```

Meaning:

```text
While busy is high, data must not change.
```

---

## 31. Assignment 2 Examples

### `01_decode_the_signal`

Main ideas:

- bit slicing
- concatenation
- streaming operators
- loops

Example:

```systemverilog
logic [31:0] signal;
logic [7:0]  opcode;
logic [7:0]  src;
logic [7:0]  dst;
logic [7:0]  payload;

initial begin
    signal = 32'hA1_B2_C3_D4;
    {opcode, src, dst, payload} = signal;
end
```

Same idea using slicing:

```systemverilog
opcode  = signal[31:24];
src     = signal[23:16];
dst     = signal[15:8];
payload = signal[7:0];
```

### `02_the_rendezvous`

Main ideas:

- `fork`
- `event`
- `-> event`
- `@(event)`
- `join`

Example:

```systemverilog
event a_done;
event b_done;

initial begin
    fork
        begin
            #10;
            -> a_done;
        end

        begin
            #20;
            -> b_done;
        end

        begin
            @(a_done);
            @(b_done);
            $display("Both completed");
        end
    join
end
```

### `03_generate_and_prove`

Main ideas:

- classes
- `rand`
- `randomize`
- constraints
- queues as stimulus
- `always_ff`
- assertions

Example:

```systemverilog
class packet;
    rand int length;
    rand bit urgent;

    constraint c_length {
        length inside {[2:12]};
    }

    constraint c_urgent {
        urgent dist {1 := 20, 0 := 80};
    }
endclass

packet q[$];

initial begin
    packet pkt;

    repeat (20) begin
        pkt = new();

        if (!pkt.randomize())
            $fatal("Randomization failed");

        q.push_back(pkt);
    end
end
```

Example assertion:

```systemverilog
property p_done_after_req;
    @(posedge clk) $rose(req) |-> ##[2:12] done;
endproperty

assert property (p_done_after_req)
else $error("done did not arrive in time");
```

---

## 32. Short Checklist

Before running the simulation, check:

- Did every class handle get `new()` before use?
- Did every `randomize()` call check the return value?
- Did every constraint avoid the extra semicolon after `}`?
- Is every `always_ff` signal assigned from only one procedural block?
- Are you using `<=` in `always_ff`?
- Are you using `=` in `always_comb`?
- Did you verify bit order in slicing, concatenation, or streaming?
- If using `join_any`, do you need `disable fork`?
- If an assertion can pass vacuously, did you add `cover property`?
