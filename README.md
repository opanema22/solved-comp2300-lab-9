Download Link: https://assignmentchef.com/product/solved-comp2300-lab-9
<br>
Before you attend this week’s lab, make sure:

<ol>

 <li>you understand <em>control flow</em>—what factors influence the order in which instructions get executed in your program (we have been talking about this since week 2!)</li>

 <li>you have attended (or watched) the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/">lectures</a> on the basics of interrupts</li>

 <li>you’re able to browse around and understand new assembly code (e.g. provided in a library) with the help of the <a class="acton-tabs-link-processed" href="https://sourceware.org/binutils/docs/as/">assembler documentation</a></li>

</ol>

In this week’s lab you will:

<ol>

 <li>configure a timer interrupt to periodically “hijack” the control flow of your program</li>

 <li>configure the GPIO pins connected to the joystick on your discoboard (the little blue diamond thingy) so that pressing down on the joystick triggers an interrupt</li>

 <li>write an interrupt handler function to <em>do something useful</em> when you press the joystick button</li>

 <li>use interrupt priorities to control what happens when different interrupts come in at the same time</li>

</ol>

<h2 id="introduction">Introduction</h2>

<p class="talk-box">Discuss with your neighbour—what does it mean for your program to have a “main loop”? On your discoboard, does your main loop have to <em>do</em> anything for the program to be useful?

So far, following the control flow through your program has been easy. In most cases, the execution (which you can track through the <code>pc</code> register) just flows from one assembly instruction (i.e. a line of assembly code) to the next. Sometimes you jump around with branch instructions (e.g. <code>b</code> and <code>bl</code>), and in certain cases you even make <em>conditional</em> branches using the condition flags in the status register (e.g. <code>beq</code>, <code>bgt</code> or <code>bmi</code>).

In today’s lab, this all changes. You’re going to configure a <strong>timer interrupt</strong> which will periodically “interrupt” the flow of your program, execute a special interrupt handler function, and then return back to where your “main” program was executing. Then you’ll go further by showing how the discoboard can handle <em>multiple</em> interrupts, each with their own handler function, and how each interrupt has a <strong>priority</strong> so that <em>interrupts can interrupt one another</em>. It <em>sounds</em> confusing… but it’s not, really. You’ll get the hang of it &#x1f642;

Plug in your discoboard, fork &amp; clone the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-lab-9">lab 9 template</a> and let’s get started.

<h2 id="exercise-1">Exercise 1: enabling the SysTick timer</h2>

A timer is a hardware component which holds a value (like a register) which counts down (or up) over time. Timers come in various shapes and sizes; some are simple and don’t have much potential for configuration, while others are <em>extremely</em> configurable, e.g. counting down to zero vs counting up from zero, counting at different rates, etc. Any given microcontroller can include many different timers, all with different names and configuration options, and multiple timers can be used simultaneously.

Your discoboard has a timer called the <strong>SysTick</strong> timer, described in the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARM reference manual</a> in <em>Section B3.3</em>. As with all things on your discoboard, you configure the SysTick timer by reading and writing to special hardware registers. To configure and use the SysTick timer your program needs to:

<ol>

 <li>enable the timer using the <em>SysTick Control and Status Register</em> (<code>SYST_CSR</code>), (also set the <code>CLKSOURCE</code> bit to use the processor clock);</li>

 <li>set the <em>SysTick Reload Value Register</em> (<code>SYST_RVR</code>)—this is the value which gets loaded into the register when it is “reloaded”, i.e. after it runs down to zero;</li>

 <li>read the current value of the timer register using the <em>SysTick Current Value Register</em> (<code>SYST_CVR</code>).</li>

</ol>

For example, if the SysTick timer is enabled (in <code>SYST_CSR</code>) and the value of <code>SYST_RVR</code> is <code>0x4000</code> then the timer will take 16384 cycles to count down to zero. How long this takes in wall-clock time depends on the CPU frequency (cycles per second) of the board.

To configure the SysTick timer you’ll need to use the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/#load-twiddle-store">load-twiddle-store</a> pattern from lab 5 all over again. This time, the relevant information (addresses, offsets, bits) starts at <em>Section B3.3.2</em> on page 677 of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARMv7 reference manual</a> and includes the next couple of sections as well.

We’re in week 9 now, so you now have the tools to read the manual and figure it out for yourself (although don’t be afraid to ask your tutor for help). Here are a few things to be mindful of:

<ul>

 <li>remember that these are memory-mapped registers, so e.g. to read the current value into a general-purpose CPU register (e.g. <code>r0</code>) you need to use an <code>ldr</code> instruction with the appropriate memory address</li>

 <li>you can find the memory-mapped addresses for both of these registers in the table in <em>Section B3.3.2</em></li>

 <li>to enable the timer, you’ll need to set the <em>enable</em> bit in <code>SYST_CSR</code> and also set the clock source to use the processor clock</li>

 <li>even though the timer will count down automatically (once tick per clock cycle) your program still needs to be running, so make sure you’ve got an infinite “run” loop in your program</li>

 <li>the initial clock speed of your discoboard when you first turn it on is 4MHz so keep that in mind when you’re setting the <code>SYST_RVR</code> reload value</li>

</ul>

For exercise 1, all you need to do is enable the SysTick timer, start it running, and watch the values from the <code>SYST_CVR</code>.

<p class="push-box">Write an assembly program which configures the SysTick timer to count down from <code>4000000</code>, and goes into a <code>finished</code> infinite loop when the timer reaches zero. Commit and push your program to GitLab.

<h2 id="exercise-2">Exercise 2: configuing the interrupt</h2>

You may have noticed that there’s another bit in the <code>SYST_CSR</code> configuration register which you didn’t set in the last exercise, but which looks interesting: the <strong>TICKINT</strong> bit. The <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARMv7 refernce manual</a> says that this particular bit:

<blockquote>

 indicates whether counting to 0 causes the status of the SysTick exception to change to pending

</blockquote>

So what does this mean, exactly? Well, as discussed in lectures, an interrupt/exception is “a signal to the processor emitted by hardware or software indicating an event that needs immediate attention” (from <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Interrupt">Wikipedia</a>). If the <strong>TICKINT</strong> bit is set in <code>SYST_CSR</code>, then the SysTick timer triggers an interrupt every time it counts down to zero. Your CPU handles this interrupt by branching to an <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Interrupt_handler">interrupt handler</a> which will (hopefully) branch back when it’s finished. In words, when an interrupt comes in then the CPU stops what it’s doing and branches somewhere else.

The ARM CPU in your discoboard recognises many different types of interrupts. Some are triggered by timers, some are triggered by external peripherals (like the joystick), some are triggered by other chips or wires connected to the discoboard.

All interrupts on your discoboard have:

<ol>

 <li>an index (which is just a number for identifying the source of the interrupt)</li>

 <li>a priority</li>

 <li>an entry in the <strong>vector table</strong>, which is a region of the discoboard’s memory where the addresses (i.e. the place to branch to) of the <em>handler</em> routine for each interrupt</li>

</ol>

You might be wondering—<em>where</em> does my code branch to when the interrupt comes in? Well, that’s what the <strong>vector table</strong> is for. It’s a special part of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/#chapter-2-hardwaresoftware-interface">memory address space</a> (starting at <code>0x0</code>) where the addresses of the different interrupt handler functions are stored. Think of it like a bunch of “jump-off points”—the <em>code</em> for handling the interrupt will be stored somewhere else, the vector table just has the address of the starting point for that code.

You can see your program’s vector table in the <code>lib/startup.S</code> file starting at around line 60

<pre><code class="language-ARM hljs"><span class="hljs-symbol">.section</span> .rodata.vtable    <span class="hljs-meta">.word</span> _stack_end    <span class="hljs-meta">.word</span> Reset_Handler    <span class="hljs-meta">.word</span> NMI_Handler    <span class="hljs-meta">.word</span> HardFault_Handler    <span class="hljs-meta">.word</span> MemManage_Handler    <span class="hljs-meta">.word</span> <span class="hljs-keyword">BusFault_Handler</span>    <span class="hljs-meta">.word</span> UsageFault_Handler    <span class="hljs-meta">.word</span> <span class="hljs-number">0</span>    <span class="hljs-meta">.word</span> <span class="hljs-number">0</span>    <span class="hljs-meta">.word</span> <span class="hljs-number">0</span>    <span class="hljs-meta">.word</span> <span class="hljs-number">0</span>    <span class="hljs-meta">.word</span> <span class="hljs-keyword">SVC_Handler</span>    <span class="hljs-meta">.word</span> DebugMon_Handler    <span class="hljs-meta">.word</span> <span class="hljs-number">0</span>    <span class="hljs-meta">.word</span> PendSV_Handler    <span class="hljs-meta">.word</span> SysTick_Handler    <span class="hljs-comment">@</span>    <span class="hljs-comment">@ more entries follow...</span>    <span class="hljs-comment">@</span></code></pre>

<p class="think-box">What does it mean if there’s a <code>0</code> in a particular “slot” in the vector table?

Try and find the vector table for yourself in the startup file. Look for the <code>.section .rodata.vtable</code> directive—can you see how it mirrors the table from <em>Section B1.5.2</em>? You can see that there’s already a <code>SysTick_Handler</code> label in there in the 16th slot in the vector table, but my “hot tip” to you is that the <code>SysTick_Handler</code> function isn’t very interesting at the moment, it’s just defined to be equal to the <code>Default_Handler</code> (which is just an infinite loop) down at the bottom of the file.

Your job in Exercise 2 is to build on the counter program you wrote in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/#exercise-1">Exercise 1</a> and add a couple of things:

<ol>

 <li>when you configure the timer, set the <strong>TICKINT</strong> bit as well</li>

 <li>somewhere in your program, write a <strong>function</strong> (i.e. something which you can <code>bl</code> to and which does a <code>bx lr</code> at the end) called <code>SysTick_Handler</code></li>

</ol>

If you set it up correctly, your <code>Systick_Handler</code> function will get called every time the counter gets to zero.

Again, here are a couple of things to be careful of:

<ul>

 <li>you’ll need to declare <code>SysTick_Handler</code> as a label with <code>.global</code> visibility so that the address of <em>your</em> <code>SysTick_Handler</code> function will get used in the vector table in <code>src/startup.S</code>, not the boring default one down the bottom of that file)</li>

 <li>similarly, make sure <code>SysTick_Handler</code> is declared as a function with the usual <code>.type SysTick_Handler, %function</code><sup id="fnref:interworking" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/#fn:interworking">1</a></sup></li>

 <li>remember that the interrupt handler (in this case <code>SysTick_Handler</code>) needs to be a function, and also to play nice and obey the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/courses/comp2300/lectures/#chapter-3-functions">AAPCS</a> (otherwise it might mess with other parts of your program)</li>

</ul>

<p class="push-box">Using the <code>led.S</code> library provided, write a program which uses the <code>SysTick_Handler</code> interrupt to toggle the red LED on and off with a frequency of 1Hz (two toggles per second). Commit and push your program to GitLab.

<h2 id="exercise-3">Exercise 3: GPIO interrupts</h2>

<p class="think-box">Ok, so the <code>SysTick_Handler</code> looks after the SysTick timer interrupt, but what about the other peripherals on your discoboard? Is there a <code>Joystick_Handler</code> for handling presses on the joystick? If not, where <em>can</em> you put your code to be executed when the joystick is pressed?

The discoboard includes a Nested Vectored Interrupt Controller (NVIC), a special bit of hardware which is responsible for watching the various bits of hardware (and software) which can trigger interrupts in your discoboard.

A brief recap: remember that interrupts are a method of triggering an <em>interruption</em> to the sequence of assembly instructions being executed by the discoboard. Configuring interrupts requires (at a minimum) enabling the interrupt and creating an <strong>interrupt handler</strong>—the function which gets called when the interrupt is triggered.

<img decoding="async" alt="Single-interrupt timeline" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-9/single-interrupt-timeline.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-9/single-interrupt-timeline.png?w=980&amp;ssl=1" alt="Single-interrupt timeline" data-recalc-dims="1">

 </noscript>

In this exercise we’re going to configure an interrupt based on the GPIO pins. In this lab, we’re going to be using our GPIO pins as <strong>input</strong> devices to register a “click” on the discoboard’s blue diamond-shaped joystick (pictured below). Finally, you can give (physical) input to your discoboard!

<img decoding="async" alt="Discoboard joystick" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-9/disco-button.jpg?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-9/disco-button.jpg?w=980&amp;ssl=1" alt="Discoboard joystick" data-recalc-dims="1">

 </noscript>

Compared to the SysTick interrupt, there’s a slightly different process in configuring GPIO pins as sources of interrupts. This is because SysTick interrupt is one of the 16 “built-in” ARM Cortex interrupts—it’s not just something which ST decided to put in when they designed your discoboard, it’s part of the ARM standard. The GPIO pins, on the other hand, aren’t part of a standard—each microcontroller manufacturer is free to include (or not) any number of GPIO pins on their board, and the way that they are wired into the CPU is up to them (although there are some conventions, so most of them do things in pretty much the same way).

On your discoboard, the GPIO pins are managed through the Extended Interrupts and Events Controller (EXTI), which is described in detail in <strong>Section 12</strong> of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/stm32-L476G-discovery-reference-manual.pdf">discoboard reference manual</a>. From that section:

<blockquote>

 The extended interrupts and events controller (EXTI) manages the external and internal asynchronous events/interrupts and generates the event request to the CPU/Interrupt Controller (the NVIC) and a wake-up request to the Power Controller.

</blockquote>

This means that raising a GPIO-triggered interrupt is really a two-stage process (at least from the hardware’s perspective):

<ul>

 <li>the EXTI notices the hardware event (e.g. an edge trigger on a GPIO line, or a timer event from one of the discoboard’s many timers) and raises an interrupt line into the NVIC</li>

 <li>the NVIC deals with that interrupt, <em>potentially</em> saving the current register context to the stack and switching to the handler function (depending on whether the interrupt is currently enabled, whether any higher priority interrupts are already running, etc.)</li>

</ul>

So, to configure your discoboard so that when you press the central joystick button an interrupt is triggered (which you can then write a handler for) you need to enable &amp; configure the interrupt in both the EXTI and the NVIC. As for most things on your discoboard, this is done by reading &amp; writing the right bits in the right places to the various EXTI &amp; NVIC configuration registers.

<p class="think-box">If this is still a bit confusing, you’ll get another chance to work through it (in more detail) in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#input-with-interrupts">next week’s lab</a>.

<p class="think-box">There are often more things to configure (i.e. GPIO pins) than there are bits in a 32-bit register—can you guess how the designers of the discoboard get around this limitation?

Since you’re hopefully well acquainted with the load-twiddle-store process for writing the configuration registers like this, to start this lab we’ve provided some files with the initialisation code to save you some time. It’s worth having a look through, though—if you don’t understand what they’re doing (and you can’t figure it out from the <a class="acton-tabs-link-processed" href="https://sourceware.org/binutils/docs/as/">assembler manual</a>) then you need to ask for help—it’s not too late!

The template provides some starter code for using the joystick in <code>joystick.S</code>. Make sure you read and understand what the functions in that library are doing—if you don’t, you’ll have trouble later on. Here are a few things worth noticing:

<ol>

 <li>the LED library (which builds on the LED code from the last couple of labs) now supports the green LED as well—have a look at the functions marked <code>.global</code> at the top of <code>led.s</code> to get an idea of the things you can do with this library</li>

 <li>the joystick init code sets the GPIO pins to <strong>input mode</strong> using the <code>GPIOA_MODER</code> register (this is different to the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/">blinky lab</a>, where we only used the pins in output mode)</li>

 <li>the central joystick button is connected to <strong>PA0</strong> which (at least as configured in the setup code) will trigger the <code>EXTI0_IRQHandler</code> you’ll need to write—don’t forget to declare the handler as <code>.type EXTI0_IRQHandler, %function</code></li>

 <li>unlike the SysTick interrupt, the EXTI interrupts are not enabled by default—they must be enabled by setting the relevant bit in the interrupt mask register <code>EXTI_IMR1</code> (described in <em>Section 12.5.1</em> of the <strong>disco-board</strong> reference manual, mapped to address <code>0x40010400</code>)</li>

 <li>the EXTI controller can use either a <strong>rising edge</strong> (when the signal goes from <code>0</code> to <code>1</code>) or <strong>falling edge</strong> (when it goes from <code>1</code> to <code>0</code>) or <strong>both</strong> as a trigger for the interrupt—it’s currently set to rising edge in the <code>joystick_init</code> function (think: what will the difference be if you set a falling edge trigger instead/as well?)</li>

 <li>once the interrupt handler function has done whatever it needs to do, it needs to tell the EXTI controller that it’s finished handling interrupt <em>n</em> by writing the <em>n</em>th bit in the <code>EXTI_PR1</code> “pending” register (<code>0x40010414</code>)</li>

</ol>

<p class="push-box">Write a program where pressing the central joystick button toggles the green LED on and off. Hint: the interrupt configuration &amp; enabling is already done in <code>joystick_init</code>—for this Exercise you only need to write the interrupt handler function and add it’s symbol to the vector table. Commit &amp; push your program to GitLab.

<p class="extension-box">In this exercise you’ve turned on the “centre” button on the joystick (PA0), but the “directions” (up/down/left/right) don’t work—they’re connected to pins 1 to 5 of the same GPIO port (GPIOA). If you’re keen, you can turn on the other ones (same process as before except some of the register addresses &amp; bit indexes will be different).

<h2 id="exercise-4">Exercise 4: interrupt priorities</h2>

What happens when you are busy handling interrupt and another interrupt happens? In this exercise you will construct such a scenario and see how interrupt priorities work.

Use the following code as the <code>SysTick_Handler</code> function:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">.type</span> SysTick_Handler, %<span class="hljs-meta">function</span><span class="hljs-symbol">SysTick_Handler</span>:  <span class="hljs-keyword">bl </span>red_led_on<span class="hljs-symbol">SysTick_Handler_infloop</span>:  <span class="hljs-keyword">nop</span>  <span class="hljs-keyword">b </span>SysTick_Handler_infloop<span class="hljs-symbol">.size</span> SysTick_Handler, .-SysTick_Handler</code></pre>

This handler just turns on the red LED and then goes into an infinite loop. The effect is that when the first SysTick interrupt happens, control flow will get stuck in this handler code.

<p class="info-box">This is actually a <strong>bad</strong> idea for writing an interrupt handler. Usually you want the interrupt handling to be quick (it’s an interrupt, not the right place to do computationally intensive work). But it’s useful to do it this way to see how interrupt priorities work.

Now if you press the joystick button when the red LED is on, what happens?

Do you remember that the <strong>N</strong> in <strong>N</strong>VIC stands for <em>nested</em>? This means that the interrupts can happen inside of one another. Here’s a diagram to show what it might look like:

<img decoding="async" alt="Multi-interrupt timeline" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-9/multi-interrupt-timeline.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-9/multi-interrupt-timeline.png?w=980&amp;ssl=1" alt="Multi-interrupt timeline" data-recalc-dims="1">

 </noscript>

This isn’t the full story, though—the discoboard doesn’t always “kick out” the currently running interrupt for the new one, it depends on the priority. On the discoboard (as in life) some things are more important than others, and each interrupt has a priority associated with it. On your discoboard, this priority is represented by a 4-bit number, with 0 being the highest priority and 15 being the lowest. When an interrupt handler is running and a new interrupt is triggered, it will only preempt (i.e. interrupt) the currently running interrupt handler if the priority is lower. If it’s the same or higher, that interrupt handler will be run once the currently running one finishes (i.e. returns with <code>bx lr</code>).

If your green LED doesn’t turn on when you press the joystick and the red LED is on, this means that either the SysTick interrupt has the same or higher priority (i.e. a smaller number as the priority value) than the EXTI0 interrupt. To change the interrupt priority so that you can click the green LED on even when the red one is blinking (i.e. when the SysTick interrupt handler is running) you’ll need to lower the priority (give a higher number) to the SysTick interrupt.

Because the two interrupts (the SysTick timer interrupt and the EXTI0 joystick interrupt) have some differences as mentioned earlier (one is part of the core ARM Cortex standard, one is a discoboard-specific thing) you need to set their interrupt priorities in slightly different places:

<ul>

 <li>for the SysTick interrupt, you can set the interrupt priority by writing bits 28-31 of the System Handler Priority Register 3 (<code>SHPR3</code>, base address <code>0xE000ED20</code>) described in B3.2.12 of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARM architecture reference manual</a></li>

 <li>for the PA0 interrupt, you can set the interrupt priority by writing bits 20-23 of the NVIC interrupt priority register (<code>NVIC_IPR1</code>, base address <code>=0xE000E404</code>)</li>

</ul>

<p class="push-box">Modify the priority of your SysTick interrupt handler so that it <em>does</em> get preempted by the EXTI0 handler and the green light comes on when the red LED is on. Commit and push your program to GitLab. Experiment with different priority values—what happens if they’re the same?

<p class="extension-box">If you’re wondering how to figure out exactly which bits to set to control the priorities, then <a class="acton-tabs-link-processed" href="http://www.ocfreaks.com/interrupt-priority-grouping-arm-cortex-m-nvic/">here’s an article which might help you out</a>. It’s for a different ARM Cortex-M board (i.e. not your discoboard) the main principle is the same.

<h2 id="quickclick">Exercise 5: QuickClick</h2>

In the final exercise, your job is to take your new knowledge of interrupts and make a game called <strong>QuickClick</strong>. It’s a simple game:

<ol>

 <li>you blink the red LED on your discoboard for a short time every 5 seconds</li>

 <li>the player’s goal is then to press the joystick button when the red LED is on</li>

 <li>if you get the timing right (i.e. the red LED <em>is</em> on when the button is pressed) the green LED comes on</li>

 <li>each time you get it right, the red blink duration gets shorter (so that it’s harder to get the timing right for the next round).</li>

</ol>

<p class="info-box">When you’re clicking your joystick, try and be a <em>bit</em> gentle on your discoboard. Make sure you’re on a flat surface (and that none of the pins are likely to get bent). Don’t get too carried away and smash your fist down on the joystick—it’s not built to handle that &#x1f642;

For this exercise, you can take advantage of your ability to enable &amp; disable different interrupts in software to make it easy to implement the “is the red light on? if so, then clicking the button will turn on the green” logic:

<ol>

 <li>perform all the configurations steps necessary (including defining the handler function) to use the joystick as an input device</li>

 <li>in your <code>SysTick_Handler</code>:

  <ul>

   <li>enable the joystick interrupt by setting the bit in the interrupt <strong>set</strong> enable register <code>NVIC_ISER0</code> (address: <code>0xE000E100</code>) The bit you are looking to set can be found in the position column of the interrupt vector table in section 11.3 of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/stm32-L476G-discovery-reference-manual.pdf">discoboard reference manual</a></li>

   <li>You may also want to clear <strong>both</strong> EXTI and NVIC interrupt pending bits <strong>before</strong> enabling the EXTI interrupt. Otherwise the pending interrupt will trigger when you enable it, causing the green LED to turn on as soon as the red is on.</li>

   <li>blink the red LED in a <strong>blocking</strong> fashion (i.e. use a delay, so that the red LED goes on and then off again before the <code>SysTick_Handler</code> exits)</li>

   <li>before <code>SysTick_Handler</code> exits, disable the joystick interrupt by setting the bit in the interrupt <strong>clear</strong> enable register <code>NVIC_ICER0</code> (address: <code>0xE000E180</code>)</li>

  </ul></li>

</ol>

Can you see how you can use this technique to temporarily enable the joystick interrupt in the SysTick interrupt handler so that the joystick will only work when the red LED is on?

<p class="push-box">Implement the QuickClick game following the steps above (you can use as much of the startup code provided earlier as you like). Commit and push your program to GitLab.

<h2 id="extension-ideas">Extension ideas</h2>

5/5 - (1 vote)

There’s one more “gotcha” to be aware of when dealing with the <em><strong>clear</strong> enable</em> and <em><strong>clear</strong> pending</em> NVIC control registers (e.g. <code>NVIC_ICER0</code> or <code>NVIC_ICPR0</code>). As described above, to disable an interrupt (ICER) or clear a pending interrupt (ICPR) you write a <code>1</code> to the corresponding bit (e.g. to disable the interrupt in position 6 of the NVIC you write a <code>1</code> to the 7th bit from the right in <code>NVIC_ICER0</code>).

However, you might have noticed something if you were reading Sections B3.4.5 (p684) and B3.4.7 (p685) the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARM reference manual</a> really closely. In the description for those registers it says:

<blockquote>

 <strong>1</strong>: On reads, interrupt enabled

</blockquote>

which means that if an interrupt is enabled, then a <em>read</em> from that (memory-mapped) register will show the corresponding bit as <code>1</code>.

This is a problem for the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/#load-twiddle-store">load-twiddle-store</a> pattern, because the point of the load twiddle store is to leave all the bits unchanged except for the one you’re interested in. However, this means that all of the currently enabled interrupts (whose bits will read as <code>1</code> in the <em>load</em> phase) will be disabled when you write the bits back in the <em>store</em> phase—which (almost certainly) isn’t what you want!

Again, here’s an example: say there are 3 interrupts currently enabled, then a load from the corresponding register would have <code>1</code>s in those three positions, and <code>0</code>s elsewhere. If you load/twiddle/store the value, then all three of those interrupts would be <em>cleared</em> by the store operation.

This means that for the <em><strong>clear</strong> enable/pending</em> registers you should just write a <code>1</code> for the particular interrupt you’re interested in, and a <code>0</code> in all the other bits.

Next week’s lab goes into more detail about how the interrupts are routed in your discoboard, so if you’re still confused/curious then <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/">jump ahead</a> and see how it works in more detail.

There are <em>heaps</em> of things you can do to stretch yourself further:

<ul>

 <li>can you modify QuickClick to make it more fun? (e.g. using the direction buttons on the joystick as well?)</li>

 <li>can you re-implement the QuickClick game without interrupts?</li>

 <li>can you turn on the discoboard’s random number generator (RNG) and use it so that the red LED blinks on randomly, rather than at regular intervals? Hint: <em>Section 24</em> of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/">discoboard reference manual</a> is the place to find the configuration steps required to get the RNG working—it’s not too difficult.</li>