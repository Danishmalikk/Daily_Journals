# 2026-06-21

## What I Learned
- Java Concurrency. CAS and FAA

## Definations : 
Definition of CAS and FAA
CAS (Compare-and-Swap) and FAA (Fetch-and-Add) are atomic operations designed to ensure thread safety and synchronization in multithreaded applications. CAS allows comparing a variable’s value with an expected value and atomically updating it if the comparison succeeds. FAA provides atomic increments or decrements of variables, making it ideal for counters and aggregators.
Justification for the importance of atomic operations
Atomic operations play a significant role in multithreaded programming since they allow ensuring thread safety and synchronization without the use of locks. Locks can often lead to performance problems, such as throughput and scalability issues, as well as difficulties in debugging and analyzing code. Atomic operations provide efficient, non-blocking alternatives that can significantly improve the performance and stability of multithreaded applications.
A brief overview of the application of CAS and FAA in multithreaded applications
CAS and FAA are widely used to implement various multithreaded data structures and algorithms. In Java, for example, CAS is used in classes such as AtomicInteger, AtomicLong, and AtomicReference to provide thread-safe atomic operations on basic data types. FAA is used in classes like LongAdder and DoubleAdder for efficient variable incrementation with scalability in mind.

## Detailed Theory 
- In this article, we will examine the mechanisms of CAS and FAA in detail, their application in Java, and compare their performance, advantages, and disadvantages in different usage scenarios.

CAS (Compare-and-Swap)
1. Description of CAS mechanism
CAS (Compare-and-Swap) is an atomic operation that compares a variable’s value with an expected value and, if they match, updates the variable with a new value. The entire process of executing CAS is atomic, meaning it cannot be interrupted by other threads or operations.

The CAS operation includes three operands: the memory cell address (V), the expected old value (A), and the new value (B). The processor atomically updates the memory cell address (V) if the value in the memory cell matches the expected old value (A); otherwise, the modification is not recorded. In either case, the value that existed before the request will be returned. Some variations of the CAS method return information on the operation’s success rather than the current value. Essentially, CAS says:

Possibly, the value at address V equals A; if so, replace it with B, otherwise do not change it, but be sure to tell me the current value.

Examples of using CAS in Java
AtomicInteger, AtomicLong, and other atomic classes Java provides classes such as AtomicInteger, AtomicLong, and others that implement the CAS mechanism for working with primitive data types. Below is an example of using the AtomicInteger class to implement a thread-safe counter.

import java.util.concurrent.atomic.AtomicInteger;

public class Counter {
    private final AtomicInteger counter = new AtomicInteger(0);

    public void increment() {
        counter.incrementAndGet();
    }

    public int getCounter() {
        return counter.get();
    }
}
Flowchart of variable manipulation using CAS:

Press enter or click to view image in full size

Atomic operations on object fields using the AtomicReferenceFieldUpdater class.
The AtomicReferenceFieldUpdater class enables performing atomic operations on object fields using the CAS mechanism. Below is an example of using AtomicReferenceFieldUpdater to implement thread-safe modification of an object’s state.

import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

public class BankAccount {
    private static final AtomicReferenceFieldUpdater<BankAccount, Double> balanceUpdater =
            AtomicReferenceFieldUpdater.newUpdater(BankAccount.class, Double.TYPE, "balance");

    private volatile double balance;

    public BankAccount(double initialBalance) {
        this.balance = initialBalance;
    }

    public boolean withdraw(double amount) {
        double currentBalance;
        double newBalance;

        do {
            currentBalance = balance;
            newBalance = currentBalance - amount;

            if (newBalance < 0) {
                return false;
            }
        } while (!balanceUpdater.compareAndSet(this, currentBalance, newBalance));

        return true;
    }
}
Operation of CAS at the hardware level
CAS is implemented in hardware using special processor instructions that guarantee the atomicity of compare and update operations. Due to this, CAS provides high performance and low latency when updating shared data.

In the process of working with CAS at the hardware level, there are several key points:

Atomicity: A CAS instruction must be atomic, that is, it must be fully executed or not executed at all. This ensures that other threads cannot change the value of the variable at the same time.
Addressing: A CAS instruction must be addressable, that is, the position in memory where the variable is located must be specified.
Comparison: The CAS instruction must compare the value of the variable with the expected value. If the values match, then the instruction writes the new value to memory.
Atomic Write: The CAS instruction must write the new value of the variable to memory atomically.
Each processor may implement the CAS mechanism differently. For example, some processors may use special registers that store the current value of a variable and the expected value. In this case, the values are compared in registers, and if they match, the new value is written to memory.

Other processors may use a more complex mechanism that uses cache memory and multithreading. In this case, the values are compared in the cache, and if the values match, the new value is written to the memory.

Thus, the operation of CAS at the hardware level depends on the specific implementation in the microarchitecture of the processor. However, regardless of the specific implementation, the CAS instruction ensures the atomicity of the operation of reading, checking, and changing the value of a variable.

The specific implementation of the CAS mechanism at the hardware level depends on the manufacturer and model of the processor. Here are some examples:

Intel x86: Intel x86 processors support the CAS instruction, which compares the value of a variable with the expected value and writes a new value if they match. This instruction is available in real and protected modes.
ARM: ARM processors support an LDREX (Load exclusive) instruction to read the value of a variable, and a STREX (Store exclusive) instruction to write a new value to a variable if it hasn’t changed since the read. Both instructions must be executed in a single transaction.
PowerPC: PowerPC processors support the LWARX (Load word and reserve) instruction to read the value of a variable and reserve the corresponding memory, as well as the STWCX (Store word conditional) instruction to write a new value to a variable if it has not been modified by other processors.
SPARC: SPARC processors support the CASA (Compare and Swap Atomic) instruction to compare the value of a variable with the expected value and write a new value if they match. This instruction can be used for atomic operations on 32-bit and 64-bit values.
IBM z/Architecture: IBM z/Architecture processors support the CS (Compare and Swap) instruction to compare the value of a variable with the expected value and write a new value if they match. This instruction can be used for atomic operations on 32-bit and 64-bit values.
As you can see from these examples, each processor architecture implements the CAS mechanism differently, but in general, they work on the same principle — comparing the value of a variable with the expected value and writing a new value if they match.

Advantages and Disadvantages of CAS
Advantages of CAS:
Fast and low-cost operation compared to traditional locking mechanisms.
A non-blocking approach that reduces the likelihood of deadlocks and improves application performance.
Enables optimistic synchronization strategies that can work effectively under low contention (competition between threads).
Disadvantages of CAS:
CAS can lead to the problem of “starvation,” where one thread constantly tries to perform a CAS operation but cannot do so due to competition with other threads.
CAS operations can lead to the “ABA problem,” where the value of a variable changes from A to B and then back to A, which can cause errors in some algorithms.
Requires knowledge and experience for correct algorithm implementation, especially when solving the ABA problem.
Guarantees of Progress for CAS
CAS provides a progress guarantee called “obstruction-free,” which means that if a thread executes a CAS operation without competition from other threads (i.e., other threads do not interfere), it will always be able to successfully complete the operation. However, in the case of active competition between threads, progress is not guaranteed, and this can lead to problems such as the starvation problem and the ABA problem.

To provide strict progress guarantees, such as “lock-free” or “wait-free,” additional mechanisms and algorithms may be required, such as using double-word CAS, versioning, or additional data structures such as descriptors.

FAA (Fetch-and-Add)
Description of the FAA mechanism
Fetch-and-Add (FAA) is an atomic operation that reads the current value of a variable, adds a specified number to it, and then returns the old value of the variable. FAA is one of the key primitives for implementing atomic operations in multi-threaded applications.

FAA involves two operands: the address of a memory location (V) and the value (S) to add to the old value stored at the memory location (V). Thus, FAA can be represented as follows: extract the value at the specified address (V) and temporarily store it. Then, write the previously stored value, incremented by the value of the second operand (S), to the specified address (V). It is important to note that all the aforementioned operations are performed atomically and implemented at the hardware level.

Examples of using similar FAA functionality in Java
AtomicInteger and AtomicLong
In Java, the AtomicInteger and AtomicLong classes provide the getAndAdd() and addAndGet() methods, which implement similar FAA functionality. Here is an example of using AtomicInteger to implement a thread-safe counter:

import java.util.concurrent.atomic.AtomicInteger;

public class Counter {
    private final AtomicInteger counter = new AtomicInteger(0);

    public void increment() {
        counter.getAndAdd(1);
    }

    public int getCounter() {
        return counter.get();
    }
}
LongAdder and DoubleAdder
Java also provides the LongAdder and DoubleAdder classes, which are designed for high-performance and scalable counter implementations. Here is an example of using LongAdder to implement a thread-safe counter.

FAA operation at the hardware
level Like CAS, FAA is implemented at the hardware level using special processor instructions that guarantee the atomicity of read, add, and update operations. This provides high performance and low latency when updating shared data.

When a thread wants to perform an FAA operation, the processor first loads the value of the variable into a register and returns it. Then, the processor performs an addition operation between this value and the passed operand, stores the result in the register, and writes the new value back to memory. Thus, the FAA operation includes three steps: reading the value of the variable, performing the addition operation, and writing the new value.

For example, if we want to perform an FAA operation on a variable counter containing the value 10 and pass an operand of 2, the processor will perform the following steps:

Load the value of the counter variable (10) into a register.
Perform an addition operation between this value (10) and the passed operand (2), resulting in (12).
Store the result (12) in the register.
Write the new value of the variable (12) back to memory.
As a result of the FAA operation, the value of the counter variable will be increased by 2.

Like CAS, the processor must ensure the atomic execution of the FAA operation to ensure the correct operation of multi-threaded applications. The exact implementation of the FAA mechanism at the hardware level depends on the specific processor architecture, but in general, it works on the principle described above.

The implementation of the FAA mechanism at the hardware level depends on the manufacturer and model of the processor. Some modern processors, such as Intel x86 and ARM, support atomic FAA operations at the hardware level.

Become a Medium member
For example, Intel processors starting with the Intel Pentium architecture from the late 1990s support the “xadd” instruction, which performs FAA operations on 32-bit and 64-bit integer values. This instruction reads the value from the operands, performs the addition operation between them, stores the result in the register, and writes the new value to memory. Thus, the FAA operation can be performed atomically at the hardware level using this instruction.

Similarly, ARM processors starting with the ARMv6K architecture also support the “LDREX” instruction for reading values from memory and the “STREX” instruction for writing a new value back to memory. These instructions can be used to perform atomic FAA operations at the hardware level.

The implementation of the FAA mechanism on other processor architectures may be different, but in general, the principle of operation will be similar.

Advantages and Disadvantages of FAA
Advantages of FAA:
Fast and low-cost operation compared to traditional locking mechanisms.
The non-blocking approach reduces the likelihood of deadlocks and improves application performance.
Simplifies implementation of atomic counters and aggregators, such as LongAdder and DoubleAdder.
Disadvantages of FAA:
May not be suitable for all use cases as it assumes atomic increment by a fixed amount, which may not be a universal solution for all tasks.
Like CAS, FAA requires a certain level of expertise and knowledge for correct algorithm implementation, especially under high thread contention.
Guarantees of Progress for FAA
FAA also provides a progress guarantee called “obstruction-free”, which means that if a thread executes an FAA operation without contention from other threads (i.e. no interference), it will always be able to successfully complete the operation. However, in the case of active thread contention, progress is not guaranteed, and more complex algorithms and mechanisms may be required to ensure strict progress guarantees, such as “lock-free” or “wait-free”.

In some cases, combined approaches may be used to ensure strict progress guarantees, such as combining CAS and FAA mechanisms or using additional data structures, such as descriptors, to reduce contention between threads and improve performance.

Comparison of CAS and FAA
Similarities and differences between CAS and FAA
Similarities between CAS and FAA:

Both are atomic operations that ensure thread safety in multithreaded applications.
Both mechanisms are implemented at the hardware level using special processor instructions.
Both provide “obstruction-free” progress guarantee.
Differences between CAS and FAA:

CAS compares the current value of a variable with an expected value and replaces it with a new value only if it matches the expected value. FAA reads the current value of a variable, increases it by a specified number, and returns the old value of the variable.
CAS can be used to implement more complex algorithms and data structures such as non-blocking stacks and queues. FAA is primarily used for atomic counters and aggregators.
CAS may encounter the ABA problem, while FAA usually does not have such problems.
Bandwidth and scalability
CAS and FAA provide high bandwidth and scalability compared to traditional locking mechanisms. However, due to differences in usage scenarios and operation characteristics, bandwidth and scalability may vary.

In the case of CAS, bandwidth may decrease in high competition between threads due to frequent failed update attempts. FAA typically has better bandwidth and scalability, especially when using classes such as LongAdder and DoubleAdder, which were developed to reduce contention between threads.

Use cases for CAS and FAA
CAS:

Implementation of non-blocking data structures such as stacks, queues, and lists.
Implement optimistic synchronization strategies where contention between threads is low or medium.
Solving problems related to locks and deadlocks in multithreaded applications.
FAA:

Implementing thread-safe counters and aggregators where high performance and scalability are critical.
Solving problems where atomic incrementing or decrementing of variables is required.
Implementation of accumulative counters in monitoring and statistics systems.
Code examples:

CAS in AtomicInteger:

AtomicInteger atomicInt = new AtomicInteger(0);

boolean success = atomicInt.compareAndSet(0, 1);
System.out.println("CAS success: " + success + ", new value: " + atomicInt.get());
FAA in AtomicInteger:

AtomicInteger atomicInt = new AtomicInteger(0);

int oldValue = atomicInt.getAndAdd(1);
System.out.println("Old value: " + oldValue + ", new value: " + atomicInt.get());
In conclusion, CAS and FAA provide powerful and flexible mechanisms for implementing thread-safe operations in multi-threaded applications. They offer high performance and scalability compared to traditional locking mechanisms but require careful understanding and analysis to be used correctly in different scenarios and tasks.

Conclusion
Summary of the main ideas of the article
In this article, we have examined two important mechanisms for ensuring thread safety in multithreaded applications — CAS (Compare-and-Swap) and FAA (Fetch-and-Add). We discussed the basic principles of how these mechanisms work, examples of their use in Java, and compared their characteristics, advantages, and disadvantages.

CAS allows for atomic comparison of the value of a variable with the expected value and updates it if the comparison is successful. This is especially useful for implementing non-blocking data structures and optimistic synchronization strategies.

FAA provides atomic increments or decrements of variables, making it ideal for counters and aggregators. In Java, the LongAdder and DoubleAdder classes were specifically developed to provide high performance and scalability when incrementing variables.

The importance of choosing the right mechanism for multithreaded applications.
Selecting an appropriate synchronization mechanism is critically important for ensuring the performance and reliability of multithreaded applications. Incorrect choices can lead to performance issues, blocking, and even race conditions. Therefore, developers need to carefully study the available mechanisms and choose the most suitable one depending on the specific task and usage scenario.

Directions for further study and development of the topic
Multithreaded programming is a complex and constantly evolving field, and there are numerous directions for further study and development. Some of these include:

Studying more advanced synchronization mechanisms, such as lock-free and wait-free algorithms, provide strict progress guarantees and can ensure even higher performance and scalability.
Developing and analyzing new data structures and algorithms that can provide even greater thread safety and performance for multithreaded applications.
Studying different memory models and their impact on the behavior and performance of multithreaded applications.
Considering alternative synchronization approaches, such as the actor model and message-based parallel programming, can offer other principles and strategies for developing multithreaded systems.
Applying formal methods and static analysis to detect and prevent synchronization issues and race conditions during development.
In conclusion, CAS and FAA provide powerful tools for implementing thread-safe operations in multithreaded applications, and developers need to carefully study and apply them to achieve the best results in the performance, reliability, and scalability of their systems.

## Solution
- Register filter correctly.

## Resources
- https://medium.com/@pravvich/cas-and-faa-through-the-eyes-of-a-java-developer-8a028f213624
