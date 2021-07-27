

# **Functional testing and analysis documentation**

## 1. Test framework 

The DiffTest framework was used as the test framework.The framework contains a simulator with correct behavior :NEMU, the processor core to be tested: Nutshell, the simulated peripheral devices: SDRAM, UART, etc. After Nutshell executes an instruction, Nutshell will wake up NEMU to execute an instruction, and compare the states of the two, the state continues consistent, the inconsistency error, and the error field is printed.

## 2. Method

Fill a SV57's initial level 5 table table in the same time before starting the test, where the RAM is simulated by an array. It is necessary to fill the contents of the page table to the corresponding array. The address assignment information of the page table in the RAM is as follows.

| Name    | Level                   | Starting Address | Size  |
| ------- | ----------------------- | ---------------- | ----- |
| ptedev  | fifth level page table  | 0x87eca000       | 512kb |
| pdedev  | fourth level page table | 0x87f3a000       | 4kb   |
| ptemmio | fifth level page table  | 0x87f3b000       | 512kb |
| pdemmio | fourth level page table | 0x87fbb000       | 4kb   |
| pdddde  | first level page table  | 0x87fbc000       | 4kb   |
| pddde   | second level page table | 0x87fbd000       | 4kb   |
| pdde    | third level page table  | 0x87fbe000       | 4kb   |
| pde     | fourth level page table | 0x87fbf000       | 4kb   |
| pte     | fifth level page table  | 0x87fc000        | 256kb |

The page table can cover 0x30000000-0x3ffff, 0x40000000 - 0x4ffff, 0x40000000-0X4FFFFFF, and 0x80000000-0X8FFFFFF these three-segment physical address, where 0x30000000 - 0x4FFFFF corresponds to dev and MMIO. And use the PDDE [0] index PDEDEV, and PDEMMIo is indexed by PDDE [1], and PDE is indexed by PDDE [2]. The implementation of this page in nutshell /src/test/csrc/**ram.cpp ** is as follows

```scala
void addpageSv57() {
//three layers
//addr range: 0x0000000080000000 - 0x0000000088000000 for 128MB from 2GB - 2GB128MB
//the first layer: one entry for 1GB. (512GB in total by 512 entries). need the 2th entries
//the second layer: one entry for 2MB. (1GB in total by 512 entries). need the 0th-63rd entries
//the third layer: one entry for 4KB (2MB in total by 512 entries). need 64 with each one all  

#define PAGESIZE (4 * 1024)  // 4KB = 2^12B
#define ENTRYNUM (PAGESIZE / 8) //512 2^9
#define PTEVOLUME (PAGESIZE * ENTRYNUM) // 2MB
#define PTENUM (RAMSIZE / PTEVOLUME) // 128MB / 2MB = 64
#define PDDDDENUM 1
#define PDDDENUM 1
#define PDDENUM 1
#define PDENUM 1
#define PDDDDEADDR (0x88000000 - (PAGESIZE * (PTENUM + 4))) //0x88000000 - 0x1000*68
#define PDDDEADDR (0x88000000 - (PAGESIZE * (PTENUM + 3))) //0x88000000 - 0x1000*67
#define PDDEADDR (0x88000000 - (PAGESIZE * (PTENUM + 2))) //0x88000000 - 0x1000*66
#define PDEADDR (0x88000000 - (PAGESIZE * (PTENUM + 1))) //0x88000000 - 0x1000*65
#define PTEADDR(i) (0x88000000 - (PAGESIZE * PTENUM) + (PAGESIZE * i)) //0x88000000 - 0x100*64
#define PTEMMIONUM 128
#define PDEMMIONUM 1
#define PTEDEVNUM 128
#define PDEDEVNUM 1

  uint64_t pdddde[ENTRYNUM];
  uint64_t pddde[ENTRYNUM];
  uint64_t pdde[ENTRYNUM];
  uint64_t pde[ENTRYNUM];
  uint64_t pte[PTENUM][ENTRYNUM];
  
  //special addr for mmio 0x40000000 - 0x4fffffff
  uint64_t pdemmio[ENTRYNUM];
  uint64_t ptemmio[PTEMMIONUM][ENTRYNUM];
  
  // special addr for internal devices 0x30000000-0x3fffffff
  uint64_t pdedev[ENTRYNUM];
  uint64_t ptedev[PTEDEVNUM][ENTRYNUM];

  // dev: 0x30000000-0x3fffffff
  pdde[0] = (((PDDDDEADDR-PAGESIZE*(PDEMMIONUM+PTEMMIONUM+PDEDEVNUM)) & 0xfffff000) >> 2) | 0x1;

  for (int i = 0; i < PTEDEVNUM; i++) {
    pdedev[ENTRYNUM-PTEDEVNUM+i] = (((PDDDDEADDR-PAGESIZE*(PDEMMIONUM+PTEMMIONUM+PDEDEVNUM+PTEDEVNUM-i)) & 0xfffff000) >> 2) | 0x1;
  }

  for(int outidx = 0; outidx < PTEDEVNUM; outidx++) {
    for(int inidx = 0; inidx < ENTRYNUM; inidx++) {
      ptedev[outidx][inidx] = (((0x30000000 + outidx*PTEVOLUME + inidx*PAGESIZE) & 0xfffff000) >> 2) | 0xf;
    }
  }

  pdde[1] = (((PDDDDEADDR-PAGESIZE*1) & 0xfffff000) >> 2) | 0x1;

  for(int i = 0; i < PTEMMIONUM; i++) {
    pdemmio[i] = (((PDDDDEADDR-PAGESIZE*(PTEMMIONUM+PDEMMIONUM-i)) & 0xfffff000) >> 2) | 0x1;
  }
  
  for(int outidx = 0; outidx < PTEMMIONUM; outidx++) {
    for(int inidx = 0; inidx < ENTRYNUM; inidx++) {
      ptemmio[outidx][inidx] = (((0x40000000 + outidx*PTEVOLUME + inidx*PAGESIZE) & 0xfffff000) >> 2) | 0xf;
    }
  }
  
  //0x800000000 - 0x87ffffff
  pdddde[0] = ((PDDDEADDR & 0xfffff000) >> 2) | 0x1;
  //pdddde[511] = ((PDDDEADDR & 0xfffff000) >> 2) | 0x1;
  pddde[0] = ((PDDEADDR & 0xfffff000) >> 2) | 0x1;
  //pddde[511] = ((PDDEADDR & 0xfffff000) >> 2) | 0x1;
  pdde[2] = ((PDEADDR & 0xfffff000) >> 2) | 0x1;
  //pdde[510] = ((PDEADDR & 0xfffff000) >> 2) | 0x1;
  //pdde[2] = ((0x80000000&0xc0000000) >> 2) | 0xf;

  for(int i = 0; i < PTENUM ;i++) {
    pde[i] = ((PTEADDR(i)&0xfffff000)>>2) | 0x1;
    //pde[i] = (((0x8000000+i*2*1024*1024)&0xffe00000)>>2) | 0xf;
  }

  for(int outidx = 0; outidx < PTENUM; outidx++ ) {
    for(int inidx = 0; inidx < ENTRYNUM; inidx++ ) {
      pte[outidx][inidx] = (((0x80000000 + outidx*PTEVOLUME + inidx*PAGESIZE) & 0xfffff000)>>2) | 0xf;
    }
  }

  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDDDENUM+PDDDENUM+PDDENUM+PDENUM+PDEMMIONUM+PTEMMIONUM+PDEDEVNUM+PTEDEVNUM)),ptedev,PAGESIZE*PTEDEVNUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDDDENUM+PDDDENUM+PDDENUM+PDENUM+PDEMMIONUM+PTEMMIONUM+PDEDEVNUM)),pdedev,PAGESIZE*PDEDEVNUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDDDENUM+PDDDENUM+PDDENUM+PDENUM+PDEMMIONUM+PTEMMIONUM)),ptemmio, PAGESIZE*PTEMMIONUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDDDENUM+PDDDENUM+PDDENUM+PDENUM+PDEMMIONUM)), pdemmio, PAGESIZE*PDEMMIONUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDDDENUM+PDDDENUM+PDDENUM+PDENUM)), pdddde, PAGESIZE*PDDENUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDDENUM+PDDENUM+PDENUM)), pddde, PAGESIZE*PDDENUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDDENUM+PDENUM)), pdde, PAGESIZE*PDDENUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*(PTENUM+PDENUM)), pde, PAGESIZE*PDENUM);
  memcpy((char *)ram+(RAMSIZE-PAGESIZE*PTENUM), pte, PAGESIZE*PTENUM);
}
```

In Nutshell, hardcoding setting the SATP register initial value pointing to the starting position of the page table, set the initial value of SATP to 0xa000000000087fbc, where the mode bit is equal to 10, can support SV57. The PPN field is 0x87BFC, and the address of the fifth-level page table can be obtained from the left-left image of the address 0x87bfc000.

```scala
val satp = RegInit(UInt(XLEN.W), "ha000000000087fbc".U)
```

Finally, modify the condition of the address transformation in the Nutshell, so that the M mode is turned on. After that, Nutshell can directly perform address translation and access the page table when accessing the memory, without requiring the test program to maintain an additional page table, which reduces the difficulty of testing.As long as the test program can be run correctly, the behavior of the MMU is correct.

```scala
//val vmEnable = satp.asTypeOf(satpBundle).mode === 8.U && (io.csrMMU.priviledgeMode < ModeM)
val vmEnable = satp.asTypeOf(satpBundle).mode === 10.U
```

## 3. Results

Microbench, coremark, dhrystone and cputest tests can run correctly, indicating the correctness of the functionality.