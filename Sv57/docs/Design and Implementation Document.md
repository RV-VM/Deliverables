# Design and Implementation Document

## Goals for Sv57

- Support the 57-bit virtual address specified by Sv57

- Traverse a 5-level page table
- Support 256TB, 512GB, 1GB, 2MB super page

## Design ideas

- Modify the configuration module of Nutshell. Modify the virtual address from 39 bits to 57 bits, and modify the physical address from 32 bits to 64 bits to support a larger address width.

- Modify the number of levels of PTW to be able to traverse the 5-level page table.
- Modify the page mask. Newly support 256TB, 512GB super page.

## Code

Modify parameters in the TLB.scala.


```scala
sealed trait Sv57Const extends HasNutCoreParameter{
  val Level = 5
  val offLen  = 12
  val ppn0Len = 9
  val ppn1Len = 9
  val ppn2Len = 9
  
  val ppn3Len = PAddrBits - offLen - ppn0Len - ppn1Len - ppn2Len  // 0
  val ppnLen = ppn3Len + ppn2Len + ppn1Len + ppn0Len
  val vpn4Len = 9
  val vpn3Len = 9
  val vpn2Len = 9
  val vpn1Len = 9
  val vpn0Len = 9
  val vpnLen = vpn4Len + vpn3Len + vpn2Len + vpn1Len + vpn0Len
    
  
    
  def maskPaddr(ppn:UInt, vaddr:UInt, mask:UInt) = {
    //MaskData(vaddr, Cat(ppn, 0.U(offLen.W)), Cat(Fill(ppn2Len, 1.U(1.W)), mask, 0.U(offLen.W)))
    MaskData(vaddr, Cat(ppn, 0.U(offLen.W)), Cat(mask, 0.U(offLen.W)))
  }

  def MaskEQ(mask: UInt, pattern: UInt, vpn: UInt) = {
    //(Cat("h1ff".U(vpn2Len.W), mask) & pattern) === (Cat("h1ff".U(vpn2Len.W), mask) & vpn)
    (mask & pattern) === (mask & vpn)
  }
}
```


Modify PAddtBits to 56


```scala
trait HasNutCoreParameter {
  // General Parameter for NutShell
  val XLEN = if (Settings.get("IsRV32")) 32 else 64
  val HasMExtension = true
  val HasCExtension = Settings.get("EnableRVC")
  val HasDiv = true
  val HasIcache = Settings.get("HasIcache")
  val HasDcache = Settings.get("HasDcache")
  val HasITLB = Settings.get("HasITLB")
  val HasDTLB = Settings.get("HasDTLB")
  val AddrBits = 64 // AddrBits is used in some cases
  val VAddrBits = if (Settings.get("IsRV32")) 32 else 57 // VAddrBits is Virtual Memory addr bits
  val PAddrBits = 56 // PAddrBits is Phyical Memory addr bits
  val AddrBytes = AddrBits / 8 // unused
  val DataBits = XLEN
  val DataBytes = DataBits / 8
  val EnableVirtualMemory = if (Settings.get("HasDTLB") && Settings.get("HasITLB")) true else false
  val EnablePerfCnt = true
  // Parameter for Argo's OoO backend
  val EnableMultiIssue = Settings.get("EnableMultiIssue")
  val EnableOutOfOrderExec = Settings.get("EnableOutOfOrderExec")
  val EnableMultiCyclePredictor = false // false unless a customized condition branch predictor is included
  val EnableOutOfOrderMemAccess = false // enable out of order mem access will improve OoO backend's performance
}
```


Modify the width of the interface. Modify addrBits to 64.


```scala
class SimpleBusReqBundle(val userBits: Int = 0, val addrBits: Int = 64, val idBits: Int = 0) extends SimpleBusBundle
```

Modify EmbeddedTLB.scala, TLB.scala. Modify the initial value of level to 5. Index the first-level PTE through the ppn field of satp and vpn4 to support the traversal of the five-level page table.


```scala
is (s_idle) {
      when (!ioFlush && hitWB) {
        state := s_write_pte
        needFlush := false.B
        alreadyOutFire := false.B
      }.elsewhen (miss && !ioFlush) {
        state := s_memReadReq
        //raddr := paddrApply(satp.ppn, vpn.vpn2) 
        raddr := paddrApply(satp.ppn, vpn.vpn4)
        level := Level.U
        needFlush := false.B
        alreadyOutFire := false.B
      }
}
is (s_memReadResp) { 
      val missflag = memRdata.flag.asTypeOf(flagBundle)
      when (io.mem.resp.fire()) {
        when (isFlush) {
          state := s_idle
          needFlush := false.B
        }.elsewhen (!(missflag.r || missflag.x) && (level===5.U || level===4.U || level===3.U || level===2.U)) {          
            state := s_memReadReq
            raddr := paddrApply(memRdata.ppn, Mux(level === 5.U, vpn.vpn3, Mux(level === 4.U, vpn.vpn2, Mux(level === 3.U, vpn.vpn1, vpn.vpn0))
          }
        }.elsewhen (level =/= 0.U) { //TODO: fix needFlush  
         
          missMask := Mux(level===5.U, 0.U(maskLen.W), Mux(level===4.U, "hffff8000000".U(maskLen.W), Mux(level===3.U, "hffffffc0000".U(maskLen.W), Mux(level===2.U, "hffffffffe00".U(maskLen.W), "hfffffffffff".U(maskLen.W)))))      
          missMaskStore := missMask
        }
        level := level - 1.U
      }
    }
```

Modify the value of mask in missMask，to support 256TB, 512GB, 1GB, 2MB four super pages and 4KB page

| level | mask         | page size |
| ----- | ------------ | --------- |
| 5     | 0            | 256TB     |
| 4     | hffff8000000 | 512GB     |
| 3     | hffffffc0000 | 1GB       |
| 2     | hffffffffe00 | 2MB       |
| 1     | hfffffffffff | 4KB       |

## Appendix

nemu

https://github.com/Ziyue-Zhang/nemu-rv64

NutShell

https://github.com/Ziyue-Zhang/nutshell
