# Walking on water with this profiler for RISC Zero

<img src="title.png" align="right" alt="A young boy walking on water heading to a place with Bonsai." width="300"/>

This repo presents a plugin for RISC Zero programs that counts the number of cycles contributing by different parts of the program, 
detects execution steps that lead to significant number of cycles, and explains the underlying reasons.

Developers can add `start_timer!`, `stop_start_timer!`, and `stop_timer!` in the program to trace where the cycles come from. An example
is as follows.

```rust
start_timer!("Load data");
......

    start_timer!("Read from the host");
    ......

    stop_start_timer!("Check the length");
    ......

    stop_start_timer!("Hash");
    ......

    stop_timer!();

stop_timer!();
```

The profiler will output colorized information about the breakdown of the cycles. Specifically, if the profiler sees a single execution step 
that, however, leads to a large number of cycles, it would call it out and find out the underlying reasons. 

![An example output of the profiler.](profiler-example.png)

## How to use?

There are necessary changes that need to be made on the RISC Zero program's host and guest.

#### Host

The host should have [cycle_trace.rs](host/src/cycle_trace.rs) in place and use `ExecutorEnv` to run the program.

```rust
let cycle_tracer = Rc::new(RefCell::new(CycleTracer::default()));

let env = ExecutorEnv::builder()
        .write_slice(&task.a)
        .write_slice(&task.b)
        .write_slice(&task.long_form_c)
        .write_slice(&task.k)
        .write_slice(&task.long_form_kn)
        .trace_callback(|e| {
            cycle_tracer.borrow_mut().handle_event(e);
            Ok(())
        })
        .build()
        .unwrap();

let mut exec = ExecutorImpl::from_elf(env, METHOD_ELF).unwrap();
let _ = exec.run().unwrap();

cycle_tracer.borrow().print();
```

In the example above, we first create the cycle tracer.

```rust
let cycle_tracer = Rc::new(RefCell::new(CycleTracer::default()));
```

Then, we use `trace_callback` to ask `ExecutorEnv` to send back execution trace to the profiler.

```rust
.trace_callback(|e| {
    cycle_tracer.borrow_mut().handle_event(e);
    Ok(())
})
```

After the execution is done, ask the cycle tracer to output the profiling results.
```rust
cycle_tracer.borrow().print();
```

#### Guest

Guest also has its own [cycle_trace.rs](methods/guest/src/cycle_trace.rs). Put it in place.

When the program starts, as shown below.

```rust
fn main() {
    cycle_trace::init_trace_logger();
    start_timer!("Total");
    ......
    stop_timer!();
}
```

We first initialize the trace logger.
```rust
cycle_trace::init_trace_logger();
```

Then, the guest can use the macros to break down the program into smaller pieces for examination.

## How does it work?

The way that the profiler works is similar to a hardware watchpoint. 

When the guest program starts, the guest-side cycle tracer---uses a special instruction---to notify the 
host-side cycle tracer about the buffers to be watched. The code is [here](https://github.com/l2research/profiler0/blob/main/methods/guest/src/cycle_trace.rs#L14C5-L28C6).
```rust
#[inline(always)]
pub fn init_trace_logger() {
    unsafe {
        core::arch::asm!(
            r#"
            nop
            li x0, 0xCDCDCDCD
            la x0, TRACE_MSG_CHANNEL
            la x0, TRACE_MSG_LEN_CHANNEL
            la x0, TRACE_SIGNAL_CHANNEL
            nop
        "#
        );
    }
}
```

The host-side cycle tracer will then watch over these three channels. If the program writes data to these memory locations, the host-side cycle tracer can 
catch these changes and get the information in the channels.

For example,

- `start_timer!(msg)` copies the message `msg` into `TRACE_MSG_CHANNEL` and writes the message length into `TRACE_MSG_LEN_CHANNEL`, which triggers the host-side
  cycle tracer to mark that a new timer has started.
- `end_timer!()` writes a zero into `TRACE_SIGNAL_CHANNEL`, which triggers the host-side cycle tracer to mark that the previous timer has stopped.

Both timers are designed to be minimalistic, in that we want them not to incur too many cycles. This is a significant improvement from previous approach that uses 
`eprintln!("{}", env::get_cycle_count());` in the guest, which would by itself create a lot of cycles and affect the calculation.

## License

Please refer to [LICENSE](./LICENSE).
