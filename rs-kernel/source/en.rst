I do not write code for a living. Nor do I study computer science in university. And you know why this is great? Because it allows me to code completely stupid and useless things without anybody being on my back to tell me I shouldn’t. And what I am about to talk is precisely one of these codes…

.. important::

    The intended audience is composed as much of Linux kernel enthusiasts as of Rust enthusiasts, and I do not expect the formers to know much about Rust nor the latters about Linux kernel intrinsics. Furthermore, I will have to dive into some advanced aspects of Rust, and to reach the dreaded shores of x86_64 assembly, which I do not expect any of them both to know much about.

    Thus, I shall give a lot of explanations that you might deem unnecessary. If such is the case, feel free to skip these sections.

Rust is supposed to be an excellent systems programming language, having the ability to go extremely close to the bare metal, while offering a thick layer of security and expressiveness which C lacks of. And such an assumption *has* to be trialled!

Which Philipp Oppermann has already done by beginning to write a real OS in Rust and `writing about it`__. But this is not a *realistic* trial: an OS starting from scratch has almost zero chance to end up being finished, not to mention being actually used on real machines.

.. __: http://os.phil-opp.com/

So the idea would rather be to rewrite pieces of the Linux kernel in Rust, so the change can be incremental and one doesn’t need to rewrite the whole OS. And because you have to start somewhere, a good choice seems to be to parasitize the system call management system, which is the entry point to the kernel from userspace.

.. note::

    Testing a kernel is a real pain in the ass. So, to make things easier, I modified the sources of the Linux Mint distro I’ve got on my laptop. It is a 4.8.17 kernel, and you can find the whole source in a neat usable form on `Elixir`__.

.. __: http://elixir.free-electrons.com/linux/v4.8.17/source

But before we go further on, let’s discuss how system calls work in native Linux.

.. contents::

HowTo syscall
=============

For those who would not know, a syscall is the way Unix-like OSes allow softwares to do things that only the kernel should be allowed to do, like discussing with the hardware, creating new processes or allocating memory. Most coders don’t have to deal with syscalls: they are hidden inside the libc. But when you call ``printf()`` in C or ``println!()`` in Rust, for instance, your code is ultimately going to use the ``write()`` syscall.

To explain things very schematically, the OS is the only master on board in your computer. It decides who has the right to use memory, CPU time, and to access the hardware. And it creates a kind of guarded playground, where softwares are allowed to run, under the surveillance of the OS: this is called **userspace**.

Once in a while, the userspace softwares perform a syscall. How this is done in practice is architecture-dependent. For instance, on x86_64 processors, the ``syscall`` instruction triggers the syscall. But on x86, it’s ``int 0x80`` and on ARM and AArch64, it’s ``svc 0``.

And to explain to the OS what it awaits, the userspace has to put the arguments of the syscall (exactly like the arguments of a function) in very specific very small pieces of memory called registers: these are usually 32 ou 64 bits long, and have names like ``rax`` or ``rdi`` on x86_64, ``r0`` or ``ip`` on ARM, and ``r0`` or ``r4`` on PowerPC. For instance, this is the prototype of the ``write()`` syscall in C.

.. code:: c

    ssize_t write(int fd, const void *buf, size_t count)

On a x86_64 architecture, ``fd`` will go in the register called ``rdi``, ``buf`` in ``rsi`` and ``count`` in ``rdx``. Plus, the register called ``rax`` will contain ``1``, which is the code number for ``write``, and when the syscall is done, the result will be in ``rax`` again.

And to add to the difficulty, not only is the calling convention architecture-dependent, but also the numbering of the syscalls. For instance, ``write`` has number ``4`` on x86, on ARM, it can be ``4`` or ``0x900004`` depending on the application binary interface (ABI) used, ``4004`` on MIPS32, while having ``5001`` on MIPS64. And some syscalls only exist on some specific architectures.

But then, what happens when the syscall triggering instruction is called?

First of all, the OS takes back the hand and enters what is called **kernelspace**. On some architectures like x86_64, this induces a change in privileges at the CPU level, but this isn’t really important to us just now. Then, a general syscall-handling function is called, which performs a number of verifications, gets the arguments back from the registers, and saves the state of the memory to the point where it was left by the calling software.

When this is done, this multiplexer function calls the actual handler corresponding to the syscall number. Here, it would be the ``sys_write()`` function from ``fs/read_write.c``. When the handler is done, the result is put back in the result register (here, ``rax``), the memory state is restored, a few more verifications are done, and finally, the flow is sent back to userspace. On x86_64, this is done by the ``sysret`` instruction.

The fact that a great deal of the useful information switches from C variables to ASM registers and back, and that the syscall numbering has to be architecture-dependent, and that impossible values can be passed as argument due to the non-specific nature of ``int``\ s and so on, is of course very error-prone.

On the other hand, Rust gives us the ability to make that a lot safer, and here is how it works.

syscall.rs
==========

Let me give you the full code of the Rust part of the syscall-management code, and I shall explain further on what it does.

.. code:: rust

    #![crate_type = "rlib"]
    #![crate_name = "syscall"]

    #![feature(asm)]

    #![no_std]

    pub enum Syscall    {
        Useless(u32, Option<u32>)
    }

    impl Syscall    {
        #[cfg(feature = "userspace")]
        pub fn call(&mut self)  {
            let pointer = self as *mut Syscall;

            if (pointer as usize) < 1024    {
                // This shouldn’t happen, but we should
                // have something ready in case of.
            }

            unsafe { asm_syscall(pointer); }
        }

        #[cfg(feature = "kernel")]
        pub fn handle(&mut self)    {
            match *self {
                Syscall::Useless(data, ref mut ret) => *ret = Some(data + 15),
            };
        }
    }

    #[cfg(all(feature = "kernel", target_arch = "x86_64"))]
    #[no_mangle]
    pub unsafe extern fn rust_syscall_handle() {
        let pointer : *mut Syscall;

        asm!(""
           : "={rax}"(pointer)
         : : "memory"
           : "intel", "volatile"
        );

        (*pointer).handle();
    }

    #[cfg(all(feature = "userspace", target_arch = "x86_64"))]
    #[inline(always)]
    pub unsafe fn asm_syscall(ptr : *mut Syscall)   {
        asm!("syscall"
         : : "{rax}"(ptr)
           : "rcx", "r11", "memory"
           : "intel", "volatile"
        );
    }

First of all, a bit of configuration.

.. code:: rust

    #![feature(asm)]

Our code is obviously going to need to do a little bit of assembly. And while this is rather easy, due to an inline assembly system very close to the one used in C, it is also terribly unsafe, thus you have to tell the compiler in advance you are going to use it, so it can get emotionally prepared.

.. code:: rust

    #![no_std]

Still obviously, our code should really *not* be linked to the standard Rust library, which in turn uses the libc. Still, we can use the ``core`` library, which contains a whole bunch of useful things that don’t rely on the system except for memory allocation, like containers, or string functions.

.. code:: rust

    pub enum Syscall    {
        Useless(u32, Option<u32>)
    }

Now we are getting to actual business, and defining a type to describe our syscalls. But don’t get fooled by the keyword: an ``enum`` in Rust has little to do with an ``enum`` in C, it is much closer to an algebraic data type from Haskell and such languages.

In other words, an ``enum`` is a type that can have multiple forms, each form being able to contain almost anything, as long as this anything remains the same. Here, our ``enum`` only has one variant, but we could imagine to port both the ``write()`` and ``exit()`` syscalls (the only ones needed for a basic Hello world), and here is what it would look like.

.. code:: rust

    pub enum Syscall    {
        Exit(i32),
        Write(i32, *const u8, usize)
    }

But this would actually by a very poor adaptation, and for instance, the second variant would rather be something like ``Write(&File, String)``, and the ``File`` type would have to be defined elsewhere. But back to the actual code.

.. code:: rust

    pub enum Syscall    {
        Useless(u32, Option<u32>)
    }

As you might have guessed, ``u32`` is an unsigned 32 bits integer, that is an ``unsigned int`` from C. And you saw earlier that you also have ``u8`` (unsigned 8 bits integer), ``i32`` (signed 32 bits integer), and the whole family up to ``u64``/``i64``. And to be complete, ``usize`` and ``isize`` are integers whose width is the one of a pointer on a given architecture, that is 64 bits on x86_64.

On the other hand, ``Option<u32>`` might not be familiar to you at all if you are only used to C: it is the exact equivalent of Haskell’s ``Maybe`` type, that is a type to represent a value that might or might not be defined. Let’s try to be clearer.

``Option<T>`` is (1) a parametrized type and (2) an ``enum`` with two variants ``None`` and ``Some(T)``. (1) means that ``Option`` is a generic type, and has to be specialized by specifying the type it will contain, through the use of the ``<>`` syntax (the same you can find in C++).

In conclusion, ``Option<u32>`` means “An ``u32`` or nothing, it depends.”. Here, we will use it for the return value of the syscall: as long as the syscall has not been called, it will be ``None``, and when the syscall is over, it will be changed to ``Some(result_of_the_syscall)``.

This type is extremely powerful, and makes it possible to get rid of null pointers, dummy values for arguments so they don’t get taken into account, and optional structure members, for instance.

Finally, the ``pub`` keyword makes the type public, which more or less corresponds to exporting the symbol. The behavior is more complicated than that, but for our example, it is all you need to understand.

What follows might be a bit more arduous to swallow. So let’s take it piece by piece.

.. code:: rust

    impl Syscall    {
        #[cfg(feature = "userspace")]
        pub fn call(&mut self)  {
            let pointer = self as *mut Syscall;

            if (pointer as usize) < 1024    {
                // This shouldn’t happen, but we should
                // have something ready in case of.
            }

            unsafe { asm_syscall(pointer); }
        }

        #[cfg(feature = "kernel")]
        pub fn handle(&mut self)    {
            match *self {
                Syscall::Useless(data, ref mut ret) => *ret = Some(data + 15),
            };
        }
    }

The ``impl Syscall { }`` is used to implement methods for our ``Syscall`` type. Yet again, don’t get fooled by the name, it is not a full blown C++-like OOP system with inheritance and stuff. In Rust, methods are just namespaced functions that can ben called trough the method syntax ``var.method(args)`` instead of the canonical syntax ``Type::method(var, args)``. They also give the possibility to use ``self`` as the first argument, which must be of the type you are being implementing.

So ``pub fn call(&mut self)`` is a public function that takes a mutable reference (more on this in a moment) to a ``Syscall`` variable, and might be called through ``my_syscall.call()`` which is more readable than ``Syscall::call(&mut my_syscall)``.

What about the mutable reference, then? Well, first of all, a reference is more or less a safe pointer: it cannot be created to a nonexistent variable, it cannot become dangling, and it comes in two flavors, and that’s the mutable part.

In Rust, variables are non mutable by default, you cannot change their value after having assigned them one. If you want them to be mutable, you have to explicitly declare them such, with the keyword ``mut``. The references go the same way: they can be unmutable, that is read-only (``&variable``), or mutable, that is read-write (``&mut variable``), as long as the variable it points to is also mutable.

This might seem overkill, but there’s a subtlety: you cannot have more than one mutable reference to a given variable at the same time, and when you have a mutable reference to a given variable, you cannot have immutable references to it until the mutable reference goes dead. This makes data races impossible.

But let’s take a look at these nice lines of code.

.. code:: rust

    #[cfg(feature = "userspace")]

    #[cfg(feature = "kernel")]

This is Rust’s system for conditional compilation. Rust allows a lot of tweaking of the code depending on a lot of parameters, and I will definitely not go into much details on the matter: just go check the `official learning book`__ and the `language reference`__.

.. __: https://doc.rust-lang.org/book/conditional-compilation.html
.. __: https://doc.rust-lang.org/reference/attributes.html#conditional-compilation

This here is the equivalent of the bunch of ``#ifdef CONFIG_WHATEVER`` you find in the code of the kernel. You just have to call Rust’s compiler with the parameter ``--cfg feature=\"kernel\"`` to make all blocks marked with ``#[cfg(feature = "kernel")]`` be included in the code, whereas the blocks that have not received their ``--cfg feature=stuff`` will just disappear from the code.

How useful might this be in the present context? Well, it means we have a single source code for both the kernel and the standard library that will use the kernel, only differentiated by a simple compiler option. That way, you are guarantied that both will remain consistent.

Now, we shall follow the flow of our syscall from the call in userspace down to the handling in kernelspace.

.. code:: rust

        pub fn call(&mut self)  {
            let pointer = self as *mut Syscall;

            if (pointer as usize) < 1024    {
                // This shouldn’t happen, but we should
                // have something ready in case of.
            }

            unsafe { asm_syscall(pointer); }
        }

So the ``call()`` function takes a mutable reference to a ``Syscall`` as its only argument. And first, it converts it into a mutable C-like pointer (``*mut Syscall``, ``let`` being the keyword to define a variable). Such pointers of course lose all the benefits of references, and as such are deemed unsafe in most their usages, but have the advantage of containing no more information than the bare address it points to.

The next block is a conditional block, whose syntax you should easily understand. As of what it serves for, we shall come back to it later.

Finally, the function ``asm_syscall()`` is called on the raw pointer, and the actions being done there are totally unsafe. So we have to put the call inside a ``unsafe { }`` block, to tell the compiler that, yes, it is unsafe, but we take responsibility over what might happen, because we know what we are doing.

Now jumping to this ``asm_syscall()`` function.

.. code:: rust

    #[cfg(all(feature = "userspace", target_arch = "x86_64"))]
    #[inline(always)]
    pub unsafe fn asm_syscall(ptr : *mut Syscall)   {
        asm!("syscall"
         : : "{rax}"(ptr)
           : "rcx", "r11", "memory"
           : "intel", "volatile"
        );
    }

First, notice the configuration block? The ``all()`` part is a logical AND, both conditions have to be met: the feature ``userspace`` must have been asked for, and the target architecture must be x86_64. Which obviously come from the fact that we are doing x86_64 assembly inside the function.

Then, ``#[inline(always)]`` is surely very clear to you. This function is just a single assembly instruction, it would be nonsensical to deal with the overhead of jumping to it and then back. But this single assembly instruction depends on the architecture we are compiling for, so it is easier to put it in an individual function and conditional-compiling it, than to condition-compile the whole ``call()`` method.

Thirdly, notice the keyword ``unsafe`` in the function definition: it tells that the whole body of the function is unsafe. You could also write it that way. But it would be stupid.

.. code:: rust

    #[cfg(all(feature = "userspace", target_arch = "x86_64"))]
    #[inline(always)]
    pub fn asm_syscall(ptr : *mut Syscall)  {
        unsafe  {
            asm!("syscall"
             : : "{rax}"(ptr)
               : "rcx", "r11", "memory"
               : "intel", "volatile"
            );
        }
    }

Finally, the ``asm!()`` block has more or less the same syntax as C inline assembly, so I will not detail much. Basically, it says that the assembly instruction ``syscall`` must be issued; that prior to it, ``ptr`` must be put in the register ``rax``; that the registers ``rcx`` and ``r11`` and the general memory will be affected by the assembly code, so Rust’s compiler must not make any assumption on the value of these memory bits after the block is executed; that I use Intel syntax instead of the AT&T syntax the gas assembler uses (which objectively is crap); that the code must remain at this exact specific place and not be displaced for optimization reasons.

And finally, the system call is done, the kernel is going to take the hand! But what did actually happen? Instead of putting a syscall number in ``rax``, and then its arguments in other registers, and finally getting the answer in ``rax`` again, we put the address of the ``Syscall`` in ``rax`` and triggered the syscall with just that information.

This address might be comprised between ``0`` and ``0xffffffffffffffff``, while syscall numbers only amount up to a bit more than 512. Which means that unless you are being very unlucky and the ``Syscall`` has an address lower than 1024 (this should actually never happen, that part of the memory is supposed to be used by the kernel), then it will never be confused with an old style C syscall.

The whole point of this is that a bit of assembly magic (which we shall see later) will dispatch between old style syscalls with ``rax`` under 1024, that will be sent to the already existing syscall handler, and our own syscalls with ``rax`` above 1024, that will be sent to our own handler written in Rust. And this is our next step.

.. code:: rust

    #[cfg(all(feature = "kernel", target_arch = "x86_64"))]
    #[no_mangle]
    pub unsafe extern fn rust_syscall_handle() {
        let pointer : *mut Syscall;

        asm!(""
           : "={rax}"(pointer)
         : : "memory"
           : "intel", "volatile"
        );

        (*pointer).handle();
    }

I do not insist on the configuration line, you already understood it. Now, ``#[no_mangle]``. Rust is like C++, it tampers with the name of its exported symbols, so that our ``handle()`` method from above will actually be exported as something like ``_ZN7syscall7Syscall6handle17hd86ed5a2914e5b27E``, and this name changes every time the code is compiled. So how are we supposed to call such a function from C code? By telling Rust’s compiler not to mess with the name and let it the way we intended. With ``#[no_mangle]``.

The function definition line got another keyword: ``extern``. Rust uses it’s own calling convention for functions, that is not the one used by C. So if you want a given function to follow a non Rust-standard calling convention, you have to tell it using ``extern "calling_convention_name"``. But to make things easier, and because it is the most common case, ``extern`` alone means ``extern "C"``.

What happens next is that the content of register ``rax`` is put back into the variable ``pointer``. Then the raw pointer is dereferenced (``*pointer``), which is a totally unsafe action, be it known, and finally the method ``handle()`` is called on the resulting ``Syscall``.

.. code:: rust

    #[cfg(feature = "kernel")]
    pub fn handle(&mut self)    {
        match *self {
            Syscall::Useless(data, ref mut ret) => *ret = Some(data + 15),
        };
    }

Again, the function takes a mutable reference to a ``Syscall``. If you follow well, this is the exact same ``Syscall`` that was used by the ``call()`` method: same place in memory, same content, and all. Which means that we sent all the arguments for the syscall to the kernel without ever putting them in registers.

Then, we are going to pattern match the ``Syscall`` we just received. This again comes from functional programming and is the other side of the “algebraic data types” coin, which makes it so damn powerful. It looks like a ``switch`` in C-like languages, but on steroids’ steroids.

To try to explain it simply, the variable will be tried against every variant of the ``enum``, and then execute the block or expression that follows the corresponding ``=>``. Just so you know, Rust forces you to define a behavior for every single variant of an ``enum``, to make sure there is no undefined behavior. And that, when you add a new variant, you implemented a behavior for it in every place the ``enum`` is used.

But then, it is more than just that: you can in a single movement associate the content of the variant to local variables, which you will in turn use in the right arm of ``=>``. Here, our ``u32`` is put in ``data``, and we take a mutable reference to our ``Option<u32>``, called ``ret``.

Because this ``Option<u32>`` will be used as a return value for the syscall, the way I explained long above. Finally, ``*ret = Some(data + 15)`` just does so that the ``None`` contained in the original syscall is replaced by a ``Some()`` containing the syscall’s argument + 15. Yeah, it *is* called ``Useless`` after all…

So when the code flow will be sent back to userspace by some more assembly magic, the ``Syscall``\ ’s last content will not say “Hey, nothing to see here.” anymore, but “Look! I’ve got your answer!”.

Makefile
========

The only thing missing for you to fully understand how the new style syscalls work is the assembly part. But first, we shall see how to integrate the Rust code inside a fully C kernel. First thing to know, the code that makes the interface between userspace and the kernel on x86 and x86_64 machines is to be found in the ``/arch/x86/entry`` folder (full source for the kernel I used is on `Elixir`__).

.. __: http://elixir.free-electrons.com/linux/v4.8.17/source/arch/x86/entry

So we shall have to modify the Makefile of this specific directory, whose complete content I offer you here.

.. code:: make

    #
    # Makefile for the x86 low level entry code
    #

    OBJECT_FILES_NON_STANDARD_entry_$(BITS).o   := y
    OBJECT_FILES_NON_STANDARD_entry_64_compat.o := y

    CFLAGS_syscall_64.o		+= $(call cc-option,-Wno-override-init,)
    CFLAGS_syscall_32.o		+= $(call cc-option,-Wno-override-init,)
    obj-y				:= entry_$(BITS).o thunk_$(BITS).o syscall_$(BITS).o
    obj-y				+= common.o

    obj-y				+= vdso/
    obj-y				+= vsyscall/

    obj-$(CONFIG_IA32_EMULATION)	+= entry_64_compat.o syscall_32.o

This is not very intuitive, so before you read on, check out this `great explanation`__. Yes, it dates back to 2003 and the 2.5 kernel, but turns out it still holds true.

.. __: https://lwn.net/Articles/21835/

And here is the modified Makefile I have been using.

.. code:: make

    #
    # Makefile for the x86 low level entry code
    #

    OBJECT_FILES_NON_STANDARD_entry_$(BITS).o   := y
    OBJECT_FILES_NON_STANDARD_entry_64_compat.o := y

    $(obj)/rs-syscall.o: $(src)/syscall.rs
	    rustc -O --cfg feature=\"kernel\" -C prefer-dynamic $(src)/syscall.rs
	    ar x libsyscall.rlib syscall.0.o
	    mv syscall.0.o $(src)/rs-syscall.o
	    rm libsyscall.rlib
	    rustc -O --cfg feature=\"userspace\" -C prefer-dynamic $(src)/syscall.rs

    CFLAGS_syscall_64.o		+= $(call cc-option,-Wno-override-init,)
    CFLAGS_syscall_32.o		+= $(call cc-option,-Wno-override-init,)
    obj-y				:= entry_$(BITS).o thunk_$(BITS).o syscall_$(BITS).o
    obj-y				+= common.o
    obj-y				+= rs-syscall.o

    obj-y				+= vdso/
    obj-y				+= vsyscall/

    obj-$(CONFIG_IA32_EMULATION)	+= entry_64_compat.o syscall_32.o

First thing first, the object ``rs-sycall.o`` has been added to the list of the object files needed to compile the kernel. Then, we say that this object file depends on the ``syscall.rs`` source file, and we describe the procedure to obtain the object file from it.

Rust’s compiler can issue a number of outputs: a fully executable binary, a dynamic library (``libxxxx.so``), a classic C static library (``libxxxx.a``), or a Rust lib a. k. a. rlib (``libxxxx.rlib``). The first two lines of the source code, which we have not addressed yet, are meant to tell the compiler we wish it to output a rlib, and to call it ``libsyscall.rlib``.

.. code:: rust

    #![crate_type = "rlib"]
    #![crate_name = "syscall"]

A rlib is very interesting output, because it contains the compiled code, but also all the exported types, functions, and so on. That is, it serves as a static library and a header file all at once.

Furthermore, it is nothing complicated: it is really an ar archive containing object files, just like classic C static libraries. And it happens that the object file containing our whole code is called ``syscall.0.o``.

So ``rustc -O --cfg feature=\"kernel\" -C prefer-dynamic $(src)/syscall.rs`` compiles the source code, only the kernel parts, with optimizations (``-O``), and linking it dynamically to the Rust stdlib, or rather here the ``core`` lib (``-C prefer-dynamic``), so we just have the object files corresponding to our source code in ``libsyscall.rlib``.

Then, we extract ``syscall.0.o`` from the rlib, move it into ``/arch/x86/entry`` while renaming it ``rs-syscall.o``, and finally delete the rlib.

And finally, we compile the rlib anew, but this time only the userspace parts, which makes a nice “stdlib” for us to use later. It will be found in the root directory of the source code.

entry_64.S
==========

Now take a deep breath, because we are putting our hands into assembly code. And assembly code written in AT&T syntax which, as previously noted, is a hell of a crap. The original file is almost 1500 lines long, so I shall paste here only the part that will be of interest to us.

.. code:: gas

    /*
     * 64-bit SYSCALL instruction entry. Up to 6 arguments in registers.
     *
     * […]
     */

    ENTRY(entry_SYSCALL_64)
	    /*
	     * Interrupts are off on entry.
	     * We do not frame this tiny irq-off block with TRACE_IRQS_OFF/ON,
	     * it is too small to ever cause noticeable irq latency.
	     */
	    SWAPGS_UNSAFE_STACK
	    /*
	     * A hypervisor implementation might want to use a label
	     * after the swapgs, so that it can do the swapgs
	     * for the guest and jump here on syscall.
	     */
    GLOBAL(entry_SYSCALL_64_after_swapgs)

	    movq	%rsp, PER_CPU_VAR(rsp_scratch)
	    movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp

	    TRACE_IRQS_OFF

	    /* Construct struct pt_regs on stack */
	    pushq	$__USER_DS			/* pt_regs->ss */
	    pushq	PER_CPU_VAR(rsp_scratch)	/* pt_regs->sp */
	    pushq	%r11				/* pt_regs->flags */
	    pushq	$__USER_CS			/* pt_regs->cs */
	    pushq	%rcx				/* pt_regs->ip */
	    pushq	%rax				/* pt_regs->orig_ax */
	    pushq	%rdi				/* pt_regs->di */
	    pushq	%rsi				/* pt_regs->si */
	    pushq	%rdx				/* pt_regs->dx */
	    pushq	%rcx				/* pt_regs->cx */
	    pushq	$-ENOSYS			/* pt_regs->ax */
	    pushq	%r8				/* pt_regs->r8 */
	    pushq	%r9				/* pt_regs->r9 */
	    pushq	%r10				/* pt_regs->r10 */
	    pushq	%r11				/* pt_regs->r11 */
	    sub	$(6*8), %rsp			/* pt_regs->bp, bx, r12-15 not saved */

	    /*
	     * If we need to do entry work or if we guess we'll need to do
	     * exit work, go straight to the slow path.
	     */
	    testl	$_TIF_WORK_SYSCALL_ENTRY|_TIF_ALLWORK_MASK, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
	    jnz	entry_SYSCALL64_slow_path

    entry_SYSCALL_64_fastpath:
	    /*
	     * Easy case: enable interrupts and issue the syscall.  If the syscall
	     * needs pt_regs, we'll call a stub that disables interrupts again
	     * and jumps to the slow path.
	     */
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)
    #if __SYSCALL_MASK == ~0
	    cmpq	$__NR_syscall_max, %rax
    #else
	    andl	$__SYSCALL_MASK, %eax
	    cmpl	$__NR_syscall_max, %eax
    #endif
	    ja	1f				/* return -ENOSYS (already in pt_regs->ax) */
	    movq	%r10, %rcx

	    /*
	     * This call instruction is handled specially in stub_ptregs_64.
	     * It might end up jumping to the slow path.  If it jumps, RAX
	     * and all argument registers are clobbered.
	     */
	    call	*sys_call_table(, %rax, 8)
    .Lentry_SYSCALL_64_after_fastpath_call:

	    movq	%rax, RAX(%rsp)
    1:

	    /*
	     * If we get here, then we know that pt_regs is clean for SYSRET64.
	     * If we see that no exit work is required (which we are required
	     * to check with IRQs off), then we can go straight to SYSRET64.
	     */
	    DISABLE_INTERRUPTS(CLBR_NONE)
	    TRACE_IRQS_OFF
	    testl	$_TIF_ALLWORK_MASK, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
	    jnz	1f

	    LOCKDEP_SYS_EXIT
	    TRACE_IRQS_ON		/* user mode is traced as IRQs on */
	    movq	RIP(%rsp), %rcx
	    movq	EFLAGS(%rsp), %r11
	    RESTORE_C_REGS_EXCEPT_RCX_R11
	    movq	RSP(%rsp), %rsp
	    USERGS_SYSRET64

    1:
	    /*
	     * The fast path looked good when we started, but something changed
	     * along the way and we need to switch to the slow path.  Calling
	     * raise(3) will trigger this, for example.  IRQs are off.
	     */
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)
	    SAVE_EXTRA_REGS
	    movq	%rsp, %rdi
	    call	syscall_return_slowpath	/* returns with IRQs disabled */
	    jmp	return_from_SYSCALL_64

Yes, this is ugly, and I honestly don’t fully understand what is happening. This might partially be due to the AT&T syntax which is… well, I think you got the point. Just after we called ``syscall`` in userspace and the CPU did its black magic to give the hand back to the kernel, the code flow resumes at this point: ``ENTRY(entry_SYSCALL_64)``.

Down until ``sub $(6*8), %rsp``, the code more or less saves the state of the machine to be able to reestablish it at the end of the syscall. Then it performs a test to maybe switch to a different entry procedure, and this is where I introduced the first part of my code.

.. code:: gas

    /* Rust syscall handling system */
    cmpq	$1024, %rax
    jae	rust_entry_syscall

It’s actually very simple: we compare the value of ``rax`` with 1024, and if it is above or equal to 1024, we jump to the ``rust_entry_syscall`` label. In other words, as described earlier, if the syscall was called from Rust and ``rax`` contains a pointer to a ``Syscall``, we branch towards our Rust syscall handler, otherwise, we let the code continue as previously intended.

The rest of the code never stops until ``jmp return_from_SYSCALL_64``: this in an unconditional jump, which means that we can put some code just after it without any risk that the flow might get there without us explicitly jumping to it. And so we introduce the second part of the Rust syscall handler.

.. code:: gas

    rust_entry_syscall:
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)

	    call	rust_syscall_handle

	    jmp	.Lentry_SYSCALL_64_after_fastpath_call

To understand well, this is a reduced version of what happens when an old style C syscall is being called.

.. code:: gas

    /* */
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)
	    call	*sys_call_table(, %rax, 8)
    .Lentry_SYSCALL_64_after_fastpath_call:

I don’t exactly understand what the first two lines do, but they are obviously needed, so I reproduce them in my own code. Then, the handler function associated with the syscall number is called. In my version, on the other hand, I call the ``pub unsafe extern fn rust_syscall_handle()`` we discussed earlier.

And finally, when it is done, I jump back to the label that immediately follows the call of the syscall handler in the original code, and I let said original code handle the return back to userspace. I tried to write my own more concise code for it, but failed miserably, so let’s do it the easy way for now on…

And now you have the whole chain that goes from Rust calling a syscall, to Rust handling that same syscall inside the kernel, then back again to userspace. Sooo… time to see it in action!

It works!
=========

.. code:: rust

    extern crate syscall;

    use syscall::Syscall;

    fn main()   {
        let mut lets_be_useless = Syscall::Useless(15, None);
        lets_be_useless.call();

        let Syscall::Useless(_, ref result) = lets_be_useless;
        println!("The syscall has sent back : {}", result.unwrap());
    }

You should now be able to understand most of what this code does. ``extern crate syscall;`` is more or less ``#include "syscall.h"``, except the functions and types and so on are namespaced, so ``use syscall::Syscall;`` makes sure we get rid of the annoying ``syscall::`` we should have put in front of every use of ``Syscall``.

If you remember well, the return value of the syscall is a ``Option<u32>``, to which we take an immutable reference called ``result``. Then, ``result.unwrap()`` is a standard function that returns the content of ``Some()`` and panics if the value is ``None``. But here, we are sure it is a ``Some()``.

And here is the result.

.. code:: console

    carnufex@KAMARAD-PC $ ./test
    The syscall has sent back : 30

So, obviously, the code is far from perfect. For instance, instead of an ``Option``, the syscall should return a ``Result``, which gives the possibility to return either (did I tell you this type is called ``Either`` in Haskel? :3) an actual result or an error code, and to know which is which. Instead of the crappy “A negative but low value is the opposite of the error code, while a positive or high negative value is an actual result” system used in C.

And the assembly part is rather messy, I might not have thought of every possible case. And to be completely honest, I have no idea if this way of doing introduces security breaches, even if I think not, given that C syscalls use pointers to userspace all the time. And I don’t know either whether my code is thread-safe or not.

There are also a few drawbacks to this method. The most important, in my opinion, is that Rust gives absolutely no guaranty on the internal representation of its ``enum``\ s. That means that when you add new variants to the ``enum``, the actual numerical value associated to each variant might change.

Thus, userspace code compiled with one version of the source might not be compatible with a kernel compiled with another version of the code. The fact that the stdlib and the kernel are compiled at the same time and from the exact same source attenuates the problem, as long as the softwares are dynamically linked to the stdlib.

On the other hand, software statically linked might need to be recompiled to continue working. One way around might be to never change the order of the variants, and only adding new ones at the end of the ``enum``, so the number of the preexisting ones should remain the same across versions.

Secondly, kernel code will have to be very careful if it wishes to return a container type like ``String`` or ``Vec``. Indeed, a ``String`` is really just an array of ``char``\ s allocated on the heap on one hand, a pointer to this array and some more info like length or capacity allocated on the stack on the other hand.

And if a ``String`` gets passed back as a return value to the syscall, only the stack part will actually be written in userspace memory, the array of ``char``\ s will remain where it was allocated by the kernel, that is in kernelspace memory. And when userspace code will try to access it, it will segfault. It is possible to avoid this problem, but it will request extra care from the developers.

Conclusion
==========

But all these are just details. The important part, is that it actually works for real on a real-world kernel, and it opens the way to progressively rewriting chunks of the kernel in Rust, offering more safety and more expressive code. Enjoy!

Now, my job here is done. Feel free to comment, insult me or my mother, or send boob pics, as you see most fit.

-----

Legal notice: I have absolutely no idea where the cute Tux of the logo comes from. If it is yours and you do not wish to see it here, feel free to tell me.
