RISC-V Emulator 

This is written as a personal project, with a view to learning more about computer architecture. This is being implemented following the RISCV specs


Computer Organization

We have the following basic parts of a risc-v cpu:
1. CPU / core :The cpu contains the registers, program counter and the arithmetic logic unit (ALU) that performs all the operations.

2. DRAM

3. BUS :The bus is the data travel path between the cpu, dram and all other peripheral components.

Writing a DRAM struct

The memory for our emulator is simply an array of 64-bit variables, to store the 64-bit values
size of the memory by the variable DRAM_SIZE
start address of the memory in DRAM_BASE : start address higher than 0x0

same address space is shared by both the memory and the I/O devices

In a QEMU VM, the lower addresses are used for I/O ports therefore DRAM memory starts from the address 0x800000000

DRAM has two basic functions/operations  : reading memory → dram_load()
Writing memory → dram_store()


dram_load(): takes pointer to the dram to be read from, the address of data to be read and size of data to be read: 
8 bits → LB
16 bits → LH
32 bits → LW
64 bits→ LD

  // dram.h
uint64_t dram_load(DRAM* dram, uint64_t addr, uint64_t size);

The dram_store(), takes the same args as the load functions, plus the value arg, which contains the data to be written to the given address of the given dram.

void dram_store(DRAM* dram, uint64_t addr, uint64_t size, uint64_t value);


load specified number of bits, 8, 16, 32, and 64 from the DRAM. 
We note here that, due to use of memory mapped I/O, the address DRAM_BASE : memory[0].

To access data at given addr we subtract DRAM_BASE → mem[addr-DRAM_BASE]

While reading and storing values take endianess in account : 
Little-endian is an order in which the “little end” (least significant value in the sequence) is stored first, that is the least significant bytes are stored in the lower addresses

So while loading, we read the lower address values first into the bus by returning, and then, left shifting by 8 bits by 0xff to get the lower byte only and clear all the higher bytes

//dram.c

Uint64_t dram_load_32(DRAM*dram, uint64_t addr){
return (uint64-t) dram → mem[addr-DRAM_BASE]
    |	(uint64-t) dram → mem[addr-DRAM_BASE+1]<<8
    |	(uint64-t) dram → mem[addr-DRAM_BASE+2]<<16
    |	 (uint64-t) dram → mem[addr-DRAM_BASE +3 ] <<24
    }


For dram_load_64 we add mem[addr-DRAM_BASE + n] for more than 3 at at max till 7 





FOR darm_store();

we store the least significant byte first(little endian), then right shift by a byte to store the higher bytes. dram_store_16 and dram_store_64 are shown below.

For storing/writing value we don’t need to return anything 

void dram_store_16(DRAM*dram, uint64_t addr, uint64_t value ){
dram → mem[addr-DRAM_BASE] = (uint8_t) ((value >> 8) & 0xff);
dram → mem[addr-DRAM_BASE+1] = (unit8_t)((value >> 16) & 0xff);
}

Similarly for dram_store_64 till value>>56 


Writing a BUS struct


Bus provides a data path for data transfer across the various component of a computer. 
For RISC emulator the address and data bus is single 64 bit wide for 64 bit implementation and single 32 bit wide for 32 bit implementation 

In our case the BUS connects CPU and the DRAM 
So we write bus structure with DRAM objects to which it is connected to 

##  typedef : provide existing datatype with new name here int is aliased as Integer
 
typedef int Integer;

int main() {
  
    // n is of type int, but we are using
      // alias Integer
    Integer n = 10;

## struct : it is a data type used to group items of possibly different data types into single data type struct. The item in structure are called member and they can be of any valid data type 

typedef struct Students {
    char name[50];
    char branch[50];
    int ID_no;
} stu;

int main() {
  
      // Using alias to define structure
    stu s;
    strcpy(s.name, "Geeks");
    strcpy(s.branch, "CSE");
    s.ID_no = 108;
  
##strcpy(); : used to copy one string to other 
    strcpy(s.name, "Geeks");
    printf("%s\n", s.name);  // o/p : Geeks 



//bus.h 

typedef struct BUS{
Struct DRAM dram 
} BUS;


We use to function bus_load and bus_store  which load or store values to bus from provided address in DRAM 

//bus.h

Uint64_t bus_load(BUS *bus, uint64_t addr, uint64_t size ){
return dram_load (&(bus→dram), addr, size );  
}

void bus_store(BUS * bus , uint64_t addr, uint64_t size, uint64_t value);{
	dram_store (&(bus→ dram ), addr, size, value); 
}


#Basic CPU struct 

Components : 
Registers : risc-v cpu has 32 registers each 64/32 bit wide  
register x0 is hardwired 0 
Unprivileged register pc which is the program counter, this register holds the address of the current instruction being executed   
We have system bus : bus that connects cpu to main mem 

#include <stdint.h>

typedef struct CPU {
    uint64_t regs[32];          // 32 64-bit registers (x0-x31)
    uint64_t pc;                // 64-bit program counter
    struct BUS bus;             // CPU connected to BUS
} CPU;
functions for each of tasks of cpu pipeline : 


//  includes/cpu.h

void cpu_init(struct CPU *cpu);
Initializes the provided cpu by pointer at 0, initializing all the 32 registers, and setting the program counter pc to the start of the memory .

uint32_t cpu_fetch(struct CPU *cpu);
Function reads from memory (DRAM) for execution, and stores it to the instruction variable :  inst 
 
int cpu_execute(struct CPU *cpu, uint32_t inst);
It is basically ALU and the ins decode combined (ALU + decoder)
Decodes instructions fetched from DRAM in the inst variable and execute 

void dump_registers(struct CPU *cpu);
Is debug function to view content of 32 reg when needed 


##cpu_init()
Initialize all the 32 64 bit registers 
The register x02, contain stack pointer SP which points towards top of the memory 
x02 = DRAM_SIZE + DRAM_BASE   
pc should point start of the mem which first instruction 
pc = DRAM_BASE 

// cpu.c 

void cpu_init(CPU * cpu){
cpu → regs[0] = 0x00;
cpu → regs[2] = DRAM_BASE+DRAM_SIZE;
cpu → pc =DRAM_BASE;
}

Note : cpu is a pointer to a CPU structure.

##cpu_fetch();

Fetches instruction pc adressing at dram.
Adressed data is put on bus from dram using dram_load()
Data at address given by the pc which points to instruction to be read 

//cpu.c

uint32_t cpu_fetch(CPU*cpu){
uint32_t inst = dram_load(&(cpu→bus), cpu→pc, 32);
return inst;
}

Private load/store functions 

// cpu.c

uint64_t cpu_load(CPU* cpu, uint64_t addr, uint64_t size) {
    return bus_load(&(cpu->bus), addr, size);
}

void cpu_store(CPU* cpu, uint64_t addr, uint64_t size, uint64_t value) {
    bus_store(&(cpu->bus), addr, size, value);
}

Instruction Decoding 


Instruction that we read from the dram for execution is 32-bit wide.

R-Type : Register Type instructions 
I-Type : Immediate type instructions 
S-Type : Store type ins
B-Type : Break type 
U-Type : Upper Immediate
J-Type: Jump type 



Opcode: It is part of machine instruction that tells which operation to perform by CPU, tells processor what action to execute such as addition, loading data . branching 




Example : 
For x86 asm 
MOV AX, BX → move data from BX to AX
ADD AX, 5 → add 5 to AX
For RIsc-V(RV32I)
ADD x1,x2,x3 → add x1=x2+x3
LW x5, 0(x10) → load word from address x10 to x5 

rd: gives address of destination register (4-bit) → result of operation is stored 

Exp ~ ADD x1,x2,x3 (x1=x2+x3)
x1 is rd
x2 is rs1
X3 is rs2 

Funct3 : specifies operation type within a given opcode (each type of instruction R,S,I etc. have unique opcodes)  {3bit value}


	Note : some instructions (like SUB vs. ADD) use funct7 in combination with funct3                 to differentiate them.

Funct7 : just like funct3, funct7 divides a group of same funct3 instruction into multiple instructions. 
Exmp : SR(right shift) has 2 ins : 
SRA (arithmetic shift right)
SRL(logical shift right)
For different funct7	

rs2 & rs2 : 4 bit value which gives address of source register 1 and 2 
Imm : it is constant value encoded directly in the instruction 

# imm values must fit into different bit sizes depending on the instruction type 

I-Type (12-bit)

Arithmetic with imm values and load instructions 
[11:0]

ADDI x5, x6, 10 
imm = 000000001010 (10 in bin, 12 bit signed)
	
		LW x5, 8(x6) {x5=x6+8}
imm = 000000000100 (8 bin)

S-Type (12-bit, split into two fields)
store instructions (SW, SB, SH) 
imm[11:5] and imm[4:0]
		
		SW x5, 4(x6)      // {mem[x6+4]=x5}
imm =000000000100 (Binary for 4)
imm[11:5] = 0000000, imm[4:0] = 00100


B-Type (12-bit, signed, split)
conditional branch instructions (BEQ, BNE, BLT, etc.)
Imm[12|10:5] and imm[4:1|11]

BEQ x5, x6, 8       // if (x5 == x6) jump to PC + 8
imm = 000000000100 (Binary for 8)
imm[12|10:5] = 0000000, imm[4:1|11] = 01000


J-Type (20-bit, signed, split)
jumping (JAL).
imm[20|10:1|11|19:12]

		JAL x5, 16    // x5 =PC + 4, jump to PC +16
imm = 00000000000100000000 (Binary for 16)
arranged out of order for hardware efficiency.

## why add PC+4 
When a function call is made using JAL, execution must return to the instruction after the JAL call, not to itself.
If we saved the PC, the function would jump back to itself, causing an infinite loop.
Since RISC-V instructions are always 4 bytes long, we save PC + 4, so execution resumes correctly after the call.


U-Type (20bit, signed)
load upper immediate (LUI) and add upper immediate (AUIPC).
imm[31:12]

LUI x5, 0x12345  //x5=0x12345 << 12
 imm = 00010010001101000101 (0x12345)
Loads a 20-bit immediate into the upper 20 bits of x5, while the lower 12 bits are set to 0
		
		AUIPC x5, 0x12345   // x5 = PC + (0x12345 << 12 )
Computes an address relative to the current PC.
Typically used for position-independent code.


