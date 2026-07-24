---
title: A custom x86 mini assembly emulator (and why i made it)
date: 2026-07-24 00:00:00 +0000
categories: [C, Emulation]
tags:
  - low level
  - emulation
  - c
---

It's currently 11 am as im writing this and my classes start in an hour lmao. Let's get down to what i wanted to achieve in this project.


# Goals

- Try to emulate the close if not replicable behaviour of the .data .code and .stack segments of a program in assembly
- Build different addressing modes for our operands
- Different logical operations and their flags in flags register
- Proper JMP functions and utility


# How tf do you start?

>When you start with a big task like building a *full turing complete assembly emulator* from the start you need to work backward from what your goal is

My honest first thought was building the mov instruction was my safest bet if i had to build all the micro-ops myself.

But the fact of the matter is that MOV instruction in assembly is highly flexible and incredibly lethal and used everywhere

> MOV [operand1+offset],operand2

Before building the MOV instruction i realised "Wow, i dont have a system in place for identifying variables or creating them", and since they're kinda defined from the beginning of the program it's really required.

By keeping a separate malloc for variables i wanted to ensure that identifying variables for the compiler was as sound as having their addresses in hand - 

```c
int value = 10;
add_var("myvar", &memory, 4, (unsigned char *)&value, &var_ptr, vartoaddr);
```
```c
*nametoaddr = (Variable *)malloc(VARIABLE_LIMIT * sizeof(Variable));
```
```c
typedef struct {
  char name[10];
  int *address;
  int size;
} Variable;
```

Get this - 
>I was going to be only storing the address of a Variable in my malloc, because in assembly, generally all the variable memory and their addresses for memory are stored in the .data section

So when a variable is updated, the VALUE of the variable goes to the .data section, and on variable declaration it simply goes to malloc and checks where it is.

# Building the MOV instruction

Now this one was tricky because i knew that things were going to go sideways, because of these reasons -


- I haven't implemented a global parser
- I haven't implemented addressing
- There's no difference between a host machine pointer and a virtual pointer
- Different cases like Reg to Reg, Memory to Reg, and Reg to Memory


I made a quick get variable func and we got to work.

We survived it, I tried out a very basic implementation without any addressing modes or such -

```c
int mov(char *dest_addr, char *src_addr, int size) {
  // identify register to register or register to mem or mem to register
  bool destreg = false;
  Register *dest_reg;
  if (strcmp(dest_addr, "eax") == 0) {
    dest_reg = &eax;
    destreg = true;
  }
  if (strcmp(dest_addr, "ebx") == 0) {
    dest_reg = &ebx;
    destreg = true;
  }
  if (strcmp(dest_addr, "ecx") == 0) {
    dest_reg = &ecx;
    destreg = true;
  }
  if (strcmp(dest_addr, "edx") == 0) {
    dest_reg = &edx;
    destreg = true;
  }

  bool srcreg = false;
  Register *src_reg;
  if (strcmp(dest_addr, "eax") == 0) {
    src_reg = &eax;
    srcreg = true;
  }
  if (strcmp(dest_addr, "ebx") == 0) {
    src_reg = &ebx;
    srcreg = true;
  }
  if (strcmp(dest_addr, "ecx") == 0) {
    src_reg = &ecx;
    srcreg = true;
  }
  if (strcmp(dest_addr, "edx") == 0) {
    src_reg = &edx;
    srcreg = true;
  }

  if (srcreg && destreg) {
    memcpy(dest_reg->value, src_reg->value, 4);
  } else if (destreg && !srcreg) {
    Variable *ptr = get_var(src_addr, vartoaddr, &memory, var_ptr);
    memcpy(dest_reg, ptr->address, 4);
  } else if (!destreg && srcreg) {
    Variable *ptr = get_var(dest_addr, vartoaddr, &memory, var_ptr);
    memcpy(ptr->address, src_reg, 4);
  }
}

```


Now i bet you're wondering - "Yuck, why is this why so stupid, why is his code so repetitive and unorganized"

>I don't care.

But it's fine, it started working properly (ish) for the first few times.

# Parser

So this was the parser that was going to take care of all my addressing - 

```c
void parser(char *string, int *offset, bool *deref, char *var_name) {
  int len = strlen(string);
  int start_idx = 0;

  if (string[0] == '[' && string[len - 1] == ']') {
    *deref = true;
    start_idx = 1;
  } else {
    *deref = false;
    start_idx = 0;
  }

  char name[10] = {0};
  int i;
  int name_ptr = 0;

  for (i = start_idx;
       i < len && string[i] != '+' && string[i] != '-' && string[i] != ']';
       i++) {
    if (name_ptr < 9) {
      name[name_ptr++] = string[i];
    }
  }
  name[name_ptr] = '\0';
  strcpy(var_name, name);

  if (i < len && (string[i] == '+' || string[i] == '-')) {
    int multiplier = (string[i] == '-') ? -1 : 1;
    i++;

    char offset_str[10] = {0};
    int offset_ptr = 0;

    for (; i < len && string[i] != ']'; i++) {
      if (offset_ptr < 9) {
        offset_str[offset_ptr++] = string[i];
      }
    }
    offset_str[offset_ptr] = '\0';

    *offset = atoi(offset_str) * multiplier;
  } else {
    *offset = 0;
  }
}
```

Addressing modes are kinda witchcraft, to me personally.

They only kind of made sense because of my background in C pointers and the fact that i was building them.

This parser could also add and subtract offsets from addresses so that was nice.

The main thing I kept getting stuck at was pointer arithmetic with making a 32 bit system on a 64 bit system. Every goddamn time i tried to switch my host pointer over to virtual, the compiler threw so many warnings i had to fix.



# The refactor

So after adding all the ALU and LU operators, i had a massive epiphany

>Yo this code is unreadable, what am i doing

And i realised, the code was professionally so complicated for a microop to parse that it made me actually want to throw my laptop. It was so bad.

Look at this -

```c
void offset_and_write_reg(unsigned char *offset_addr1,
                          unsigned char *offset_addr2, int offset1, int offset2,
                          bool deref1, bool deref2, Variable *ptr,
                          Register *dest_reg) {

void mov_(char *dest_addr, char *src_addr) {

  int offset1;
  bool deref1;
  char name1[10];
  parser(src_addr, &offset1, &deref1, name1);

  int offset2;
  bool deref2;
  char name2[10];
  parser(dest_addr, &offset2, &deref2, name2);

  bool destreg = false;
  Register *dest_reg;
  if (get_register(name2) != NULL) {
    dest_reg = get_register(name2);
    destreg = true;
  }

  bool srcreg = false;
  Register *src_reg;
  if (get_register(name1) != NULL) {
    src_reg = get_register(name1);
    srcreg = true;
  }

	if (srcreg && destreg) {
    int offset1;
    bool deref1;
    char name1[10];
    parser(dest_addr, &offset1, &deref1, name1);
		if (deref1) {
      memcpy(memory.data + *(int *)dest_reg->value, src_reg->value, 4);
    } else {
      memcpy(dest_reg->value, src_reg->value, 4);
    }
	
  } else if (destreg && !srcreg) {

    int offset1;
    bool deref1;
    char name1[10];
    parser(src_addr, &offset1, &deref1, name1);

    int offset2;
    bool deref2;
    char name2[10];
    parser(dest_addr, &offset2, &deref2, name2);

    Variable *ptr = get_var(name1, vartoaddr, &memory, var_ptr);

    if (!ptr) {
      fprintf(stderr, "Asm Error: Label/Value '%s' not found!\n", name1);
      exit(1);
    }
    unsigned char *offset_addr1 = ptr->address;
    unsigned char *offset_addr2 = dest_reg->value;
    unsigned char *final_dest_ptr = NULL;

    // printf("%p %s %d \n", ptr->address, ptr->name, ptr->size);
    // printf("%d \n", ptr->is_immediate);
    printf("%d %d \n", deref1, deref2);
    if (ptr->is_immediate == true) {
      // printf("immediate_val %d", immediate_val);
      memcpy(dest_reg->value, &immediate_val, 4);
		uintptr_t addr_as_int;

		if (deref1) {
			offset_addr1 = offset_addr1 + offset1;

		} else {
			addr_as_int = (uintptr_t)(ptr->address - memory.data);
			addr_as_int = (int)addr_as_int;
			// printf("addr_as_int %d \n", addr_as_int);
		offset_addr1 = (unsigned char *)&addr_as_int;
      }
      if (deref2 && deref1) {
        fprintf(stderr, "Asm Error: mem to mem not allowed! %s %s \n", name1,
                name2);
        exit(1);
      }
		} else if (!destreg && srcreg) {
    int offset1;
    bool deref1;
    char name1[10];
    parser(src_addr, &offset1, &deref1, name1);

    int offset2;
    bool deref2;
    char name2[10];
    parser(dest_addr, &offset2, &deref2, name2);

    if (!deref2) {
      fprintf(
          stderr,
          "Asm Error: Destination must be dereferenced! Did you mean [%s]?\n",
          dest_addr);
		  exit(1);
    }

    Variable *ptr = get_var(name2, vartoaddr, &memory, var_ptr);
    if (!ptr) {
      fprintf(stderr, "Asm Error: Label/Value '%s' not found!\n", name2);
      exit(1);
    }
    unsigned char *offset_addr1 = src_reg->value;
    unsigned char *offset_addr2 = ptr->address;

		if (deref2) {
      offset_addr2 = offset_addr2 + offset2;
    } else {
      offset_addr2 = ptr->address + offset2;
    }

		printf("%p %s %d \n", ptr->address, ptr->name, ptr->size);
    memcpy(offset_addr2, offset_addr1, ptr->size);

		} else {
    fprintf(stderr,
            "Asm Error: Invalid memory-to-memory operation or bad syntax! "
            "(e.g., mov [var1], [var2] is not supported by x86 hardware)\n");
    exit(1);
		}
```

I mean, what the f**k is that? You could write it off as a "really interesting groundbreaking code".

So post refactor it turned to these 3 functions each specifying the case in which it would work - 

```c
#include <compiler.h>
#include <globals.h>
#include <microops.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void offset_and_write_reg(unsigned char *offset_addr1,
                                 unsigned char *offset_addr2, int offset1,
                                 int offset2, bool deref1, bool deref2,
                                 Variable *ptr, Register *dest_reg) {

  if (ptr->is_immediate == true) {
    memcpy(dest_reg->value, &immediate_val, 4);
  } else {

    int addr_as_int = (int)(ptr->address - memory.data);

    if (deref1) {
      offset_addr1 = offset_addr1 + offset1;
    } else {
      offset_addr1 = (unsigned char *)&addr_as_int;

      memcpy(offset_addr2, offset_addr1, ptr->size);
      return;
    }

    if (deref2 && deref1) {
      fprintf(stderr, "Asm Error: mem to mem not allowed!");
      exit(1);
    }

    // final write
    memcpy(offset_addr2, offset_addr1, ptr->size);
  }
}

static void offset_and_write_addr(unsigned char *offset_addr1,
                                  unsigned char *offset_addr2, int offset1,
                                  int offset2, bool deref1, bool deref2,
                                  Variable *ptr) {
  if (deref2) {
    offset_addr2 = offset_addr2 + offset2;
  } else {
    offset_addr2 = ptr->address + offset2;
  }
  memcpy(offset_addr2, offset_addr1, ptr->size);
}

void offset_and_write_reg_from_reg(Register *dest_reg, Register *src_reg,
                                   int offset2, bool deref2) {

  if (deref2) {
    memcpy(memory.data + *(int *)dest_reg->value + offset2, src_reg->value, 4);
  } else {
    memcpy(dest_reg->value, src_reg->value, 4);
  }
}

void mov_(char *dest_addr, char *src_addr) {
  operand_parse(dest_addr, src_addr, offset_and_write_reg,
                offset_and_write_addr, offset_and_write_reg_from_reg);
}
```

where my operand_parse looked a lil bit like this

```c

void operand_parse(char *dest_addr, char *src_addr,
                   write_reg offset_and_write_reg,
                   write_addr offset_and_write_addr,
                   write_reg_from_reg offset_and_write_reg_from_reg) {

  int offset1 = 0;
  bool deref1 = false;
  char name1[10];
  parser(src_addr, &offset1, &deref1, name1);

  int offset2 = 0;
  bool deref2 = false;
  char name2[10];
  parser(dest_addr, &offset2, &deref2, name2);

  bool destreg = false;

  Register *dest_reg;
  if (get_register(name2) != NULL) {
    dest_reg = get_register(name2);
    destreg = true;
  }

  bool srcreg = false;
  Register *src_reg;
  if (get_register(name1) != NULL) {
    src_reg = get_register(name1);
    srcreg = true;
  }

  if (srcreg && destreg) {
    offset_and_write_reg_from_reg(dest_reg, src_reg, offset2, deref2);

  } else if (destreg && !srcreg) {
    // need this
    Variable *ptr = get_var(name1, vartoaddr, &memory, var_ptr);

    if (!ptr) {
      fprintf(stderr, "Asm Error: Label/Value '%s' not found!\n", name1);
      exit(1);
    }

    unsigned char *offset_addr1 = ptr->address;
    unsigned char *offset_addr2 = dest_reg->value;

    offset_and_write_reg(offset_addr1, offset_addr2, offset1, offset2, deref1,
                         deref2, ptr, dest_reg);

  } else if (!destreg && srcreg) {

    if (!deref2) {
      fprintf(
          stderr,
          "Asm Error: destination must be dereferenced! did you mean [%s]?\n",
          dest_addr);
      exit(1);
    }

    Variable *ptr = get_var(name2, vartoaddr, &memory, var_ptr);

    if (!ptr) {
      fprintf(stderr, "Asm Error: label/value '%s' not found!\n", name2);
      exit(1);
    }
    unsigned char *offset_addr1 = src_reg->value;
    unsigned char *offset_addr2 = ptr->address;

    offset_and_write_addr(offset_addr1, offset_addr2, offset1, offset2, deref1,
                          deref2, ptr);
  } else {
    fprintf(stderr,
            "Asm Error: invalid memory-to-memory operation or bad syntax! "
            "(e.g., mov [var1], [var2] is not supported by x86 hardware)\n");
    exit(1);
  }
}
```

Yep, this was my global parser. Now making a micro-op was a thousand times easier and we just eliminated copy pasting A TON of code.


# Immediate values

Immediate values are essential to any language. Having to define a variable for every value you're going to use - nuh huh, waste of time.

I wrote a small little thing to eliminate if there's a full int as an operand in our parsing.

```c
void parser(char *string, int *offset, bool *deref, char *var_name) {
  int j = 0;
  bool numeric = true;
  while (string[j] != '\0') { // main thing
    if (!isdigit((unsigned char)string[j])) {
      numeric = false;
    }
    j++;
  }
  if (!numeric) { // check if its numeric
    int len = strlen(string);
    int start_idx = 0;

    if (string[0] == '[' && string[len - 1] == ']') {
      *deref = true;
      start_idx = 1;
    } else {
      *deref = false;
      start_idx = 0;
    }

    char name[10] = {0};
    int i;
    int name_ptr = 0;

    for (i = start_idx;
         i < len && string[i] != '+' && string[i] != '-' && string[i] != ']';
         i++) {
      if (name_ptr < 9) {
        name[name_ptr++] = string[i];
      }
    }
    name[name_ptr] = '\0';
    strcpy(var_name, name);

    if (i < len && (string[i] == '+' || string[i] == '-')) {
      int multiplier = (string[i] == '-') ? -1 : 1;
      i++;

      char offset_str[10] = {0};
      int offset_ptr = 0;

      for (; i < len && string[i] != ']'; i++) {
        if (offset_ptr < 9) {
          offset_str[offset_ptr++] = string[i];
        }
      }
      offset_str[offset_ptr] = '\0';

      *offset = atoi(offset_str) * multiplier;
    } else {
      *offset = 0;
    }
  } else {
    strcpy(var_name, string);
  }
}
```
And now this looks epic.

# The Stack

>The main thing about the stack is that it's extremely lenient with it's use, and normally you're the one deciding how to use it.

Generally speaking if you're not sure how a stack works, here's a quick sneaky peeky - 
```
       MEMORY DIRECTION           STACK LAYOUT             OFFSETS & REGISTERS
       
     (High Addresses)       +-----------------------+ 

            |               |  Function Parameter 2 |  [EBP + 12]
            |               +-----------------------+ 

            |               |  Function Parameter 1 |  [EBP + 8]
            |               +-----------------------+ 

            |               |    Return Address     |  [EBP + 4]  (Pushed by 'CALL')
            |               +-----------------------+ 

            |               |   Saved Caller EBP    |  <--- EBP Points Here [EBP + 0]
            |               +-----------------------+ 

            |               |    Local Variable 1   |  [EBP - 4]
            |               +-----------------------+ 

            |               |    Local Variable 2   |  [EBP - 8]
     GROWTH DIRECTION        +-----------------------+ 
            v               |   Temporary Storage   |  <--- ESP Points Here (Top of Stack)
     (Low Addresses)        +-----------------------+

```

Your ebp register is sitting right between your newest function call and your esp register is sitting right at the top of the stack

The ebp by itself is not doing much by itself, and esp is only tracking the top section whenever it moves due to pop or push, or manual allocation say `sub esp, 12`


Another thing - the stack grows downward from the bottom of our entire memory section.

>Why?

Well back in the 90s, you couldn't exactly afford healthy 4096 bits of RAM and have money left over to feed your family.

So they did their best with the limitations and instead of crying about it, actually did something kinda neat

```

                    |               |                    |
  STACK GROWS       |               v                    |
    DOWNWARD        :                                    :
                    :         Unallocated Memory         :  (Prevents collisions)
                    :                                    :
   HEAP GROWS       |               ^                    |
     UPWARD         |               |                    |
       ^            |                HEAP                |

```

So if the heap was larger than the stack, it would use memory more, and nothing would crash into each other. The more i think about it, there were probably more cases where this approach was sufficiently much better at saving and using most of memory more efficiently instead of being limited to even more severe circumstances. Developers of that time were pretty happy how this stuff was taking place.

My code was somewhat like this - 

```c
#include <compiler.h>
#include <globals.h>
#include <microops.h>
#include <stdbool.h>
#include <stdint.h>
#include <string.h>

void pop_(char *data) {
  int address;
  memcpy(&address, esp.value, 4); // put esp.value into address

  int offset = 0;
  bool deref = false;
  char name[10];
  parser(data, &offset, &deref, name);

  bool reg_mention = false;
  Register *reg;
  if (get_register(name) != NULL) {
    reg = get_register(name);
    reg_mention = true;
  }

  // popping
  unsigned char *actual_ptr = (unsigned char *)(address + memory.data);
  if (reg_mention) {
    memcpy(reg->value, actual_ptr, 4);
  } else {
    if (deref) {
      Variable *ptr_to_var = get_var(name, vartoaddr, &memory, var_ptr);
      memcpy(ptr_to_var->address, actual_ptr, 4);
    }
  }

  // do +4 on it and replace with 0's
  memset(actual_ptr, 0, 4);
  address += 4;
  memcpy(esp.value, &address, 4); // put updated esp value back into it
}
```

```c
#include <compiler.h>
#include <globals.h>
#include <microops.h>
#include <stdbool.h>
#include <stdint.h>
#include <string.h>
void push_(char *data) {

  int address;
  memcpy(&address, esp.value, 4); // put esp.value into address
  // do -4 on it
  address -= 4;

  unsigned char *actual_ptr =
      (unsigned char *)(address + memory.data); // actual ptr
  memcpy(esp.value, &address, 4); // put updated esp value back into it

  int offset = 0;
  bool deref = false;
  char name[10];
  parser(data, &offset, &deref, name);

  bool reg_mention = false;
  Register *reg;
  if (get_register(name) != NULL) {
    reg = get_register(name);
    reg_mention = true;
  }

  // if register
  if (reg_mention) {
    memcpy(actual_ptr, reg->value, 4);
  } else {
    // if variable
    Variable *ptr = get_var(name, vartoaddr, &memory, var_ptr);
    if (deref) {
      // read from the variable with offset
      memcpy(actual_ptr, ptr->address + offset, 4);
    } else {
      // read the address of the variable put in memory
      int addr = (int)(ptr->address + offset - memory.data);
      memcpy(actual_ptr, &addr, 4);
    }
  }
}
``` 

Essentially the push/pop instructions move the esp by 4.

>What's the point of ebp then?? 

Well, generally before a function call you push all your function arguments and then push your ebp's pointer address with it. This helps to trace back to many functions when something goes wrong and you can see exactly when it all nests there's generally a line of old EBP addresses that go back.

# CMP and JMP instruction

So right now, we do not have a turing complete language.

Our language can't do for or while loops, or anything fancy. Because we haven't essentially stored our instructions in memory yet. We've just been throwing instructions at our compiler in realtime to go process them. And while it has, it needs to be able to run through a SET of instructions at once without being told what to do from our code.

So we're going to use the .code section now

```c
#include <globals.h>
#include <stdio.h>
#include <string.h>
void add_ins(char *op, char *operand1, char *operand2) {
  Instruction ins;

  memset(&ins, 0, sizeof(Instruction));

  strncpy(ins.op, op, 4);
  strncpy(ins.operand1, operand1, 15);
  strncpy(ins.operand2, operand2, 15);

  memcpy(memory.code_ptr, &ins, sizeof(ins));

  memory.code_ptr += sizeof(ins);
}
```

This is going to make sure that we're going to have a proper EIP register that reads these instructions in order.

For reading an instruction we have this - 

```c
#include <globals.h>
#include <microops.h>
#include <stdio.h>
#include <string.h>
void read_ins(Instruction *ins) {

  printf("Instruction : %s %s, %s", ins->op, ins->operand1, ins->operand2);

  if (strcmp(ins->op, "add") == 0) {
    add_(ins->operand1, ins->operand2);
  }
  if (strcmp(ins->op, "mul") == 0) {
    mul_(ins->operand1, ins->operand2);
  }
  if (strcmp(ins->op, "sub") == 0) {
    sub_(ins->operand1, ins->operand2);
  }
  if (strcmp(ins->op, "cmp") == 0) {
    cmp_(ins->operand1, ins->operand2);
  }
  if (strcmp(ins->op, "and") == 0) {
    and_(ins->operand1, ins->operand2);
  }
  if (strcmp(ins->op, "not") == 0) {
    not_(ins->operand1);
  }
  if (strcmp(ins->op, "or") == 0) {
    or_(ins->operand1, ins->operand2);
  }
  if (strcmp(ins->op, "xor") == 0) {
    xor_(ins->operand1, ins->operand2);
  }

  if (strcmp(ins->op, "pop") == 0) {
    pop_(ins->operand1);
  }
  if (strcmp(ins->op, "push") == 0) {
    push_(ins->operand1);
  }
  if (strcmp(ins->op, "mov") == 0) {
    mov_(ins->operand1, ins->operand2);
  }

  if (strcmp(ins->op, "cmp") == 0) {
    cmp_(ins->operand1, ins->operand2);
  }

  if (strcmp(ins->op, "jmp") == 0) {
    jmp_(ins->operand1);
  }
  if (strcmp(ins->op, "je") == 0) {
    je_(ins->operand1);
  }
  if (strcmp(ins->op, "jne") == 0) {
    jne_(ins->operand1);
  }
  if (strcmp(ins->op, "jl") == 0) {
    jl_(ins->operand1);
  }
  if (strcmp(ins->op, "jg") == 0) {
    jg_(ins->operand1);
  }
  if (strcmp(ins->op, "ja") == 0) {
    ja_(ins->operand1);
  }
  if (strcmp(ins->op, "jb") == 0) {
    jb_(ins->operand1);
  }
}
```

I think that's enough?

```c
typedef struct {
  char op[4];
  char operand1[15];
  char operand2[15];
} Instruction;
```

# CMP

Now the CMP instruction does a very basic thing - takes 2 operands A and B and it subtracts them.

You're like "Wait so CMP doesnt Compare instructions??"

Yeah well it does. Technically speaking you could get the same effect with doing SUB.

>The only difference is neither of the operands get changed.

The thing what CMP pairs with is JMP, which explicitly allows us to change the flow of the program - which means turing complete language boom!!!!!!!!


```c
#include <compiler.h>
#include <globals.h>
#include <microops.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

static inline void op(int a, int b) {
  // simulating alu subtraction here cuz im too lazy to recreate the fking
  // alu right now
  int res = a - b;

  uint32_t ua = (uint32_t)a;
  uint32_t ub = (uint32_t)b;

  flags.zero_flag = (res == 0) ? 1 : 0;
  flags.sign_flag = (res < 0) ? 1 : 0;
  flags.carry_flag = (ua < ub) ? 1 : 0;
  if ((a > 0 && b < 0 && res < 0) || (a < 0 && b > 0 && res > 0)) {
    flags.overflow_flag = 1;
  } else {
    flags.overflow_flag = 0;
  }
}

static void offset_and_write_reg(unsigned char *offset_addr1,
                                 unsigned char *offset_addr2, int offset1,
                                 int offset2, bool deref1, bool deref2,
                                 Variable *ptr, Register *dest_reg) {

  if (ptr->is_immediate == true) {
    op(*(int *)dest_reg->value, immediate_val);
  } else {
    int addr_as_int = (int)(ptr->address - memory.data);

    if (deref1) {
      offset_addr1 = offset_addr1 + offset1;
    } else {
      op(*(int *)offset_addr2, addr_as_int);
      return;
    }

    if (deref2 && deref1) {
      fprintf(stderr, "Asm Error: mem to mem not allowed!");
      exit(1);
    }

    op(*(int *)offset_addr2, *(int *)offset_addr1);
  }
}
static void offset_and_write_addr(unsigned char *offset_addr1,
                                  unsigned char *offset_addr2, int offset1,
                                  int offset2, bool deref1, bool deref2,
                                  Variable *ptr) {

  if (deref2) {
    offset_addr2 = offset_addr2 + offset2;
  } else {
    offset_addr2 = ptr->address + offset2;
  }

  op(*(int *)offset_addr2, *(int *)offset_addr1);
}
static void offset_and_write_reg_from_reg(Register *dest_reg, Register *src_reg,
                                          int offset2, bool deref2) {
  if (deref2) { // has dest been dereferenced
    int calc = *(int *)(memory.data + *(int *)dest_reg->value + offset2);
    op(calc, *src_reg->value);
  } else {
    op(*dest_reg->value, *src_reg->value);
  }
}

void cmp_(char *dest_addr, char *src_addr) {
  operand_parse(dest_addr, src_addr, offset_and_write_reg,
                offset_and_write_addr, offset_and_write_reg_from_reg);
}
```

This is our cmp instruction, notice how im using a fake FLAGS register.

This is because everytime a subtraction happens in the ALU there's a bunch of FLAGS that try to tell the compiler before hand if something went wrong in operating or if there's a bunch of things you need to know about the result, like whether it's a zero (because counting starts from 1 in assembly for results), or overflow (because of 2 big operands exceeding the size limits).

So we have this - 

```c
typedef struct {
  int zero_flag;
  int carry_flag;
  int sign_flag;
  int overflow_flag;
} Flags;
```

The JMP instruction explicitly checks these flags via other instructions like JE (Jump Equals) or JNE (Jump Not Equals) by putting CMP and jump instructions next to each other you have something of a if statement in programming if you will.

```c
#include <compiler.h>
#include <globals.h>
#include <stdio.h>
#include <string.h>
void pass() {
  int addr;
  memcpy(&addr, eip.value, 4);
  addr += sizeof(Instruction);
  memcpy(eip.value, &addr, 4);
}
void jmp_(char *dest_addr) {
  int offset = 0;
  bool deref = false;
  char name[10];
  parser(dest_addr, &offset, &deref, name);

  bool destreg = false;

  Register *dest_reg;
  if (get_register(name) != NULL) {
    dest_reg = get_register(name);
    destreg = true;
  }

  if (destreg) {
    memcpy(eip.value, dest_reg->value, 4);
  } else {
    Variable *ptr = get_var(name, vartoaddr, &memory, var_ptr);
    // int addr;
    printf(" ptr addr -> %d \n", *(int *)ptr->address);
    memcpy(eip.value, ptr->address, 4);
  }
}

void je_(char *target) {
  if (flags.zero_flag == 1) {
    jmp_(target);
    return;
  }
  pass();
}
void jne_(char *target) {

  if (flags.zero_flag == 0) {
    jmp_(target);
    return;
  }
  pass();
}

void jl_(char *target) {
  if (flags.sign_flag != flags.overflow_flag) {
    jmp_(target);
    return;
  }
  pass();
}
void jg_(char *target) {
  if (flags.zero_flag == 0 && flags.sign_flag == flags.overflow_flag) {
    jmp_(target);
    return;
  }
  pass();
}

void jb_(char *target) {
  if (flags.carry_flag == 1) {
    jmp_(target);
    return;
  }
  pass();
}
void ja_(char *target) {
  if (flags.carry_flag == 0 && flags.zero_flag == 0) {
    jmp_(target);
    return;
  }
  pass();
}
```
JMP itself is an unconditional jump.

I remember in my binary expoitation days you could genuinely just edit the flags in realtime so you could control the flow of the program itself.

Anyways.

# The main loop

The final thing we need to do is that our compiler needs to be able to read off instructions and do JMPs when it's asked to. And halt when it's supposed to.

So here's what i wrote

```c
int curr_ins_address;
  memcpy(&curr_ins_address, eip.value, 4);
  unsigned char *actual_ptr;
  int cycle_count = 1;

  while (1){ // go ahead, keep updating EIP (instruction pointer) to go ahead
    actual_ptr = (unsigned char *)(curr_ins_address + memory.data);
    Instruction *ptr = (Instruction *)actual_ptr;

    if (ptr->op[0] != 'j') {
      curr_ins_address += sizeof(Instruction);
      memcpy(eip.value, &curr_ins_address, 4);
    } else {
      memcpy(&curr_ins_address, eip.value, 4);
    }
  }
```

Basically each iteration is one CPU cycle. Coordinating instruction requires a CPU cycle so that they don't overlap.

And our simple little program -

```c
add_var("my_count", &memory, sizeof(Instruction),
          (unsigned char *)&const_addr, &var_ptr, vartoaddr, INT);
add_ins("add", "eax", "10");
add_ins("sub", "ecx", "1");
add_ins("cmp", "ecx", "0");
add_ins("jne", "my_count", "");
add_ins("hlt", "", "");
```

It's a while loop that finishes when ECX hits 0, ECX is initially 3, and it keeps adding 10 to eax while it isn't finished.

# Conclusion

Well it took me well over an hour to write this so im just gonna say thank you for reading this


I definitely cant be missing first day of class im stupidim stupidim stupidim stupidim stupidim stupidim stupid


I'm gonna read up more on pipelining and kernel level stack then maybe ill touch these kinds of projects again

Later.
