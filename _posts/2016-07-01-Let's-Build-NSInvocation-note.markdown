---
layout:     post
title:      "读「Let's Build NSInvocation」博文"
subtitle:   "MAInvocation 源码浅析"
date:       2016-07-01 14:00:00
author:     "Shin Cheung"
header-img: "img/post-bg-ios9-web.jpg"
header-mask: 0.3
catalog:    true
tags:
    - iOS
    - 博客阅读记
---

>原作地址<br/>
[Let's Build NSInvocation,Part I](https://www.mikeash.com/pyblog/friday-qa-2013-03-08-lets-build-nsinvocation-part-i.html)
<br/>
[Let's Build NSInvocation,Part II](https://www.mikeash.com/pyblog/friday-qa-2013-03-22-lets-build-nsinvocation-part-ii.html)

>源码地址:<br/>
[MAInvocation](https://github.com/mikeash/MAInvocation)
<br/>

##Overview
NSInvocation对象代表一个方法调用。一个方法调用必须有target、selector、一堆参数及返回值。

一个NSInvocation对象可以根据一个特定的object交互。我们平时编码时一般写作：
<pre class="prettyprint"><code>[target message: argument];
</code></pre>

实则等于
<pre class="prettyprint"><code>[obj forwardInvocation: (id)inv];
</code></pre>

NSInvocation的相比于常见的方法调用，target、messgae及参数都是完全在runtime时决定的。这样就实现了可以随意的修改target、message及参数了，甚至修改返回值。

NSInvocation总的来说有2个相互补充的业务逻辑：<br/>

<li>*在某个方法调用是传递参数且获取返回值。*
<li>*接收方法调用，获取参数，取得一个随意的返回值给调用者。*

##Calling Conventions
原文基于x86_64架构的调用协定，有兴趣可以查阅[Wikipedia](https://en.wikipedia.org/wiki/Calling_convention).
总结x86_64一般使用16个通用寄存器，分别是rax, rbx, rcx, rdx, rbp, rsp, rsi, rdi, r8, r9, r10, r11, r12, r13, r14, and r15. 
 
 当调用一个方法时，前6个参数依次存储于rdi, rsi, rdx, rcx, r8, and r9。剩余的额外参数，分配在64bit的栈上，所以随后的参数可以在内存中RSP，RSP + 8，RSP + 16，等。
 
 如果一个方法有一个返回值，则存储于rax寄存器。如果有2个返回值，像NSRange这样的struct数据，rdx寄存器则存储第二个返回值。如果是一个比较大的struct数据，调用者则需要分配足够的内存去存储，然后该内存的指针作为隐藏的第一个参数存储在rdi寄存器中，这样，显式的参数则依次向后面的寄存器移动。
 
值得注意的是Objective-c语言方法调用时，前2个参数self，_cmd分别存储于rdi和rsi中（如果返回参数是larger struct时，向后移动就是rsi和rdx）。显式参数则在这两个隐式参数之后。

##Start

为了能完成一个function的调用，MAInvocation需要持有该function的参数，且将前6个参数存放在合适的寄存器之中，额外的参数存放在栈上，需要持有该function的实际内存地址。在处理返回值上，需要记录返回值在2个返回值寄存器中。

为了能接收function的调用，MAInvocation需要记录前6个参数的寄存器值，sp寄存器的位置，且根据这些提取参数值。在处理返回值时，需要将返回值存放于2个返回值寄存器中。赋值到寄存器和栈上的逻辑代码可以使用Objective-c编写，但是操作寄存器和栈的代码只能使用汇编代码编写了。

###数据结构
为了能够使汇编代码和OC代码通信，MAInvocation定义了一个struct来持有所有的相关代码。当调用发生时，MAInvocation就能完美的创建这个struct对象，并和胶水代码交互。当接收到一个调用时，汇编语言的胶水代码会在当前的状态下构造出该struct对象，然后传递给OC代码。

<pre class="prettyprint"><code>
struct RawArguments
{
    //the address of the function to call
    void *fptr;
    /*
     There are sixteen general-purpose registers: rax, rbx, rcx, rdx, rbp, rsp, rsi, rdi, r8, r9, r10, r11, r12, r13, r14, and r15. The first half are all inherited from Intel's 32-bit architecture, while the second half are new additions for x86-64. Each register holds 64 bits.
     */
    
    /*
     When calling a function, the first six parameters are passed by filling these registers in order: rdi, rsi, rdx,
     rcx,r8, and r9. Additional arguments, if any, are passed on the stack as 64-bit quantities, so subsequent parameters
     can be found in memory at rsp, rsp + 8, rsp + 16, etc.
     */
    
    /*
     one return value:rax.
     two return values(eg NSRange):rax->first value,rdx->second return value.
     larger struct:caller allocate enough memory to hold it,and then a pointer to that memory is passed as an implicit first argument to the function in rdi,with all of the explicit parameters moved down by one.
     
     Note that, for Objective-C methods, the first two parameters are self and _cmd, which are therefore passed in rdi and rsi (or, if the method returns a larger struct, in rsi and rdx). The explicit parameters, if any, come after those two.
     */
    
    /*
     AL/AH/AX/EAX/RAX: Accumulator
     BL/BH/BX/EBX/RBX: Base index (for use with arrays)
     CL/CH/CX/ECX/RCX: Counter (for use with loops and strings)
     DL/DH/DX/EDX/RDX: Extend the precision of the accumulator (e.g. combine 32-bit EAX and EDX for 64-bit integer operations in 32-bit code)
     SI/ESI/RSI: Source index for string operations.
     DI/EDI/RDI: Destination index for string operations.
     SP/ESP/RSP: Stack pointer for top address of the stack.
     BP/EBP/RBP: Stack base pointer for holding the address of the current stack frame.
     IP/EIP/RIP: Instruction pointer. Holds the program counter, the current instruction address.
     */
    
    /*
     CS: Code segment
     DS: Data segment
     SS: Stack segment
     ES: Extra data
     FS: Extra data #2
     GS: Extra data #3
     */
    
    //it stores the values of the six 64-bit parameter-passing registers
    uint64_t rdi;
    uint64_t rsi;
    uint64_t rdx;
    uint64_t rcx;
    uint64_t r8;
    uint64_t r9;
    
    //It then stores the address of the arguments passed on the stack,
    //as well as how many stack arguments there are:
    uint64_t stackArgsCount;
    uint64_t *stackArgs;
    
    // it stores the two return-value registers
    /*
     rdx already exists in the parameter-passing section, but it's easier to make a separate entry for return values than to reuse that field.
     */
    uint64_t rax_ret;
    uint64_t rdx_ret;
    
    /*
     it keeps a flag that records whether or not the call uses struct return conventions, i.e. whether the rdi is used to store a pointer to space allocated for the return value. In Objective-C runtime terminology, such calls are called stret, short for "struct return".
     */
    uint64_t isStretCall;
};
</code></pre>

###Function Call Glue
<pre class="prettyprint"><code>
void MAInvocationCall(struct RawArguments *);
</code></pre>
<pre class="prettyprint"><code>
void MAInvocationForward(void);
</code></pre><pre class="prettyprint"><code>
void MAInvocationForwardStret(void);
</code></pre>

这些胶水代码都会使用汇编实现，且定义在 MAInvocation-asm.h 文件中。

接下来看 MAInvocation-asm.s 文件中的具体汇编实现：
定义一个全局的标签
<pre class="prettyprint"><code>.globl _MAInvocationCall
_MAInvocationCall:
// Save and set up frame pointer
pushq %rbp
movq %rsp, %rbp

// Save r12-r15 so we can use them
pushq %r12
pushq %r13
pushq %r14
pushq %r15

// Move the struct RawArguments into r12 so we can mess with rdi
mov %rdi, %r12

// Save the current stack pointer to r15 before messing with it
mov %rsp, %r15

// Copy stack arguments to the stack

// Put the number of arguments into r10
movq 56(%r12), %r10 #get the number of arguments to r10.

// Put the amount of stack space needed into r11 (# args << 3)
movq %r10, %r11
shlq $3, %r11

// Put the stack argument pointer into r13
movq 64(%r12), %r13

// Move the stack down
subq %r11, %rsp

// Align the stack
andq $-0x10, %rsp

// Track the current argument number, start at 0
movq $0, %r14

// Copy loop
stackargs_loop:

// Stop the loop when r14 == r10 (current offset equals stack space needed
cmpq %r14, %r10
je done

/*
for(int i = 0; i != r10; i++)
        rsp[i] = r13[i];
*/
// Copy the current argument (r13[r14]) to the current stack slot
movq 0(%r13, %r14, 8), %rdi
movq %rdi, 0(%rsp, %r14, 8)

// Increment the current argument number
inc %r14

// Back to the top of the loop
jmp stackargs_loop

done:

// Copy registers over
movq 8(%r12), %rdi
movq 16(%r12), %rsi
movq 24(%r12), %rdx
movq 32(%r12), %rcx
movq 40(%r12), %r8
movq 48(%r12), %r9

// Call the function pointer
callq *(%r12)

// Copy the result registers into the args struct
movq %rax, 72(%r12)
movq %rdx, 80(%r12)

// Restore the stack pointer
mov %r15, %rsp

// Restore r12-15 for the caller
popq %r15
popq %r14
popq %r13
popq %r12

// Restore the frame pointer and return
leave
ret
</code></pre>

MAInvocationForwardStret 和 _MAInvocationForward 也差不多的方式：

<pre class="prettyprint"><code>
.globl _MAInvocationForwardStret
_MAInvocationForwardStret:

// Set %r10 to indicate that this is a stret call
movq $1, %r10

// Jump to the common handler
jmp _MAInvocationForwardCommon


.globl _MAInvocationForward
_MAInvocationForward:

// Set %r10 to indicate that this is a normal call
movq $0, %r10

// Jump to the common handler
jmp _MAInvocationForwardCommon


.globl _MAInvocationForwardCommon
_MAInvocationForwardCommon:

// Calculate the location of the stack arguments, which are at rsp + 8
movq %rsp, %r11
addq $8, %r11

// Save and set up frame pointer
pushq %rbp
movq %rsp, %rbp

//-----start construct the RawArguments struct-----

// Push RawArguments components one by one

// Push isStretCall, which is the value of r10
pushq %r10

// Push a dummy rax and rdx
pushq $0
pushq $0

// Push stackArgs pointer, which is in r11
pushq %r11

// Push a dummy stackArgsCount
pushq $0

// Push argument registers
pushq %r9
pushq %r8
pushq %rcx
pushq %rdx
pushq %rsi
pushq %rdi

// Push a dummy fptr
pushq $0

//-----end construct the RawArguments struct-----

// Save the pointer to the newly constructed struct to pass it to the C function
movq %rsp, %rdi

// Also save it in r12 so we can get to it afterwards
movq %rdi, %r12

// Align the stack
andq $-0x10, %rsp

// Call into C
callq _MAInvocationForwardC

// Copy the return value out of the RawArguments
movq 72(%r12), %rax
movq 80(%r12), %rdx

// Restore the frame pointer and return
leave
ret
</code></pre>

**汇编代码就不过多分析了，详细请看注释。**

接下来看MAInvocation的定义：
<pre class="prettyprint"><code>@interface MAInvocation : NSObject

+ (MAInvocation *)invocationWithMethodSignature:(NSMethodSignature *)sig;

- (NSMethodSignature *)methodSignature;

- (void)retainArguments;
- (BOOL)argumentsRetained;

- (id)target;
- (void)setTarget:(id)target;

- (SEL)selector;
- (void)setSelector:(SEL)selector;

- (void)getReturnValue:(void *)retLoc;
- (void)setReturnValue:(void *)retLoc;

- (void)getArgument:(void *)argumentLocation atIndex:(NSInteger)idx;
- (void)setArgument:(void *)argumentLocation atIndex:(NSInteger)idx;

- (void)invoke;
- (void)invokeWithTarget:(id)target;

@end
</code></pre>

与NSInvocation无异。

###MAInvocation的实现

####入口
<br/>
**首先在initialize方法中注册handler**
<pre class="prettyprint"><code>+ (void)initialize
{
    objc_setForwardHandler(MAInvocationForward, MAInvocationForwardStret);
}
</code></pre>

####private工具方法
target和selector的getter及setter方法此处就略过了，主要看private的方法。
<pre class="prettyprint"><code>
// 获取特定index的参数
- (uint64_t *)argumentPointerAtIndex: (NSInteger)idx;
//获取特定index参数的大小
- (NSUInteger)sizeAtIndex: (NSInteger)idx;
//返回值大小
- (NSUInteger)returnValueSize;
//特定类型的大小
- (NSUInteger)sizeOfType: (const char *)type;
//遍历处理retain的参数
- (void)iterateRetainableArguments: (void (^)(NSUInteger idx, id obj, id block, char *cstr))block;
//是否是larger struct返回值
- (BOOL)isStretReturn;
//返回值的地址
- (void *)returnValuePtr;
//特定的struct的类型
- (enum TypeClassification)classifyStructType: (const char *)type;
//是否是Integer类型
- (BOOL)isIntegerClass: (enum TypeClassification)classification;
//枚举并处理struct元素
- (void)enumerateStructElementTypes: (const char *)type block: (void (^)(const char *type))block;

</code></pre>

这些工具方法都是为了处理返回值和参数时使用你给的。

####核心方法
<pre class="prettyprint"><code>void MAInvocationForwardC(struct RawArguments *r)
{
    id obj;
    SEL sel;
    
    if(r->isStretCall)
    {
        obj = (id)r->rsi;
        sel = (SEL)r->rdx;
    }
    else
    {
        obj = (id)r->rdi;
        sel = (SEL)r->rsi;
    }
    
    NSMethodSignature *sig = [obj methodSignatureForSelector: sel];
    
    MAInvocation *inv = [[MAInvocation alloc] initWithMethodSignature: sig];
    inv->_raw.rdi = r->rdi;
    inv->_raw.rsi = r->rsi;
    inv->_raw.rdx = r->rdx;
    inv->_raw.rcx = r->rcx;
    inv->_raw.r8 = r->r8;
    inv->_raw.r9 = r->r9;
    
    memcpy(inv->_raw.stackArgs, r->stackArgs, inv->_raw.stackArgsCount * sizeof(uint64_t));
    
    [obj forwardInvocation: (id)inv];
    
    r->rax_ret = inv->_raw.rax_ret;
    r->rdx_ret = inv->_raw.rdx_ret;

    if(r->isStretCall && inv->_stretBuffer)
    {
        memcpy((void *)r->rdi, inv->_stretBuffer, [inv returnValueSize]);
    }
    
    [inv release];
}
</code></pre>

这里是根据自定义的RawArguments* 数据来构造MAInvocation对象，且调用。

##总结
<br/>
<li>汇编胶水代码实现方法调用时的参数、返回值及函数指针能正确的存储在寄存器和栈中。
<li>MAInvocation在Initialize方法中指定objc_setForwardHandler的两个函数指针。
<li>MAInvocationForwardC方法中根据struct RawArguments *来构造MAInvocation对象且调用。
<li>MAInvocation中的各种逻辑处理（target、selector、返回值及参数的处理）。
