#程序的机器级表示 家庭作业
### 建议：在做作业的时候注意查阅机器指令。
### 4.45
A. 错误，压入的是%rsp-8。  
B. 交换两行即可。
### 4.46
A. 依旧是错误。最后结果是%rsp+8。  
B. 交换两行。
### 4.47
A. 稍微修改一下就可以了。
```C++
void bubble_b(long *data, long count)
{
    long i, *last;
    for (last = count - 1; last; last--)
    for (i = 0; i < last; i++)
    if (*(data + i + 1)< *(data + i))
    {
        long t = *(data + i + 1);
        *(data + i + 1) = *(data + i);
        *(data + i) = t;
    }
}
```
B. 自己写了一份代码，没有跑过，只保证理论正确。
```
rrmovq %rsi, %rax
irmovq $1, %r8
irmovq $8, %r9
LOOP1:
    subq %r8, %rax
    je DONE
    xorq %rbx, %rbx
    LOOP2:
        rrmovq %rbx, %r10
        subq %rax, %r10
        je LOOP1
        rrmovq %rbx, %r10
        addq %r10, %r10
        addq %r10, %r10
        addq %r10, %r10
        addq %rdi, %r10
        mrmovq (%r10), %rcx
        addq %r9, %r10
        mrmovq (%r10), %rdx
        rrmovq %rdx, %rbp
        subq %rcx, %rbp
        jge LOOP2
        mrmovq %rcx, (%r10)
        subq %r9, %r10
        mrmovq %rdx, (%r10)
        jmp LOOP2
DONE:ret
```
### 4.48, 4.49
直接给那个题目的答案吧。
```
rrmovq %rsi, %rax
irmovq $1, %r8
irmovq $8, %r9
LOOP1:
    subq %r8, %rax
    je DONE
    xorq %rbx, %rbx
    LOOP2:
        rrmovq %rbx, %r10
        subq %rax, %r10
        je LOOP1
        rrmovq %rbx, %r10
        addq %r10, %r10
        addq %r10, %r10
        addq %r10, %r10
        addq %rdi, %r10
        mrmovq (%r10), %rcx
        addq %r9, %r10
        mrmovq (%r10), %rdx
        rrmovq %rdx, %rbp
        subq %rcx, %rbp
//        jge LOOP2
        cmovge %rbp, %r12
        addq %rcx, %r12
        subq %rdx, %r12
        mrmovq %rcx, (%r10)
        subq %r9, %r10
        mrmovq %rdx, (%r10)
        jmp LOOP2
DONE:ret
```
### 4.50
```
switchv:
    irmovq 0xaaa, %r8
    irmovq 0xbbb, %r9
    irmovq 0xccc, %r10
    irmovq 0xddd, %r11
    irmovq $5, %r12
    irmovq table, %r13
    rrmovq %rdi, %rdx
    subq %r12, %rdx
    jg default
    addq %rdi, %rdi
    addq %rdi, %rdi
    addq %rdi, %rdi
    addq %rdi, %r13
    mrmovq (%r13), %r13
    pushq %r13
    ret

    rrmovq %r8, %rax
    jmp Done

    rrmovq %r9, %rax
    jmp Done

    rrmovq %r10, %rax
    jmp Done
default:
    rrmovq %r11, %rax
Done:
    ret
```
### 4.51
阶段|iaddq V, rB
:--:|:---------:
取指|icode:ifun<-M1[PC]<br>rA:rB<-M1[PC+1]<br>valC<-M2[PC]+2<br>valP<-PC+10
译码|valB<-R[rB]
执行|valE<-valB+valC
访存|
写回|R[rB]<-valE
更新PC|PC<-valP
### 4.52
实验题略
### 4.53