# Svnapot

Task group name: Virtual Memory

Extension or extension group name: Svnapot

Component names: Svnapot

Group contributor name: CAS

Task group liaison email: dlustig@nvidia.com (Dan Lustig)

Development partner: liaison 

email: xuyinan@ict.ac.cn (Yinan Xu)  lixin212@mails.ucas.ac.cn(Xin Li)

Creation date: 2021-07-07

Last Modified Date: 2021-08-18

Version: 0.1


**Requirements (prose describing what has to be done and what the end result is):**


Implement the Svnapot extension on NutShell to verify the physical implementability of the extension.

According to the specification : https://github.com/riscv/virtual-memory/blob/main/specs/628-Svnapot.pdf

**Deliverables (bullet list of components and the changes expected):**

1. The final code : https://github.com/RV-VM/NutShell/tree/svnapot
2. Design and implementation documentation (which describes our design in detail and includes a link to our repository)
3. Functional testing and Analysis documentation (detailing our testing methods and final test results)
4. Physical back-end report (describes the physical implementation overhead of the extension)
5. Problem summary document (catalogs some typical implementation issues and points out some common issues to be aware of when implementing extensions)

**Acceptance criteria (bullet list with measurable results defined):**

NutShell passed CoreMark Dhrystone Microbench and Linux kernel(booting) with Svnapot extension enabled.


**Projected timeframe (best guess):**

5 months


**Sign off dates (done for every version> 1.0):**

**Task group liaison sign-off date:**

**Group contributor sign-off date:**  