# xv6_lazy_page_allocation
2nd/2 Assignment of the "Operating Systems" course (Winter Semester 2022/2023 - NKUA). Making changes to 'xv6 educational OS' of "6.828: Operating System Engineering" MIT course, in order to achieve 'Lazy page allocation' for address page tables. Some simple page table printing at the beginning helps for debugging in the next steps. Info in README.

**DESCRIPTION OF FILES/FUNCTIONS**
-------------------------------

The source and header files of xv6 that were created or modified for the purposes of the assignment are:

1) kernel/vm.c/vmprint() -> This is the requested function for Step 1, responsible for printing the pagetable. 
              							The `vmprint()` function does not contain the code for this specific functionality 
              							but simply calls the recursive function `pteprinter()`. Additionally, the prototype 
              							of `vmprint()` is declared in `defs.h` (line 176).

2) kernel/vm.c/pteprinter() -> This is a recursive function used for traversing and printing each table and 
              							   sub-table of the parent pagetable. Starting from level=0, we are at the 'title' 
              							   of the table, and when we encounter the presence of the next level in the structure 
              							   (higher level corresponds to the existence of pagetable entries in some 'level' sub-table), 
              							   we recursively call `pteprinter()`, and so on. Moreover, depending on the level we are in, 
              							   we print the dots ('.. ') or ('..') in the correct format to indicate the hierarchy of levels.

3) kernel/sysproc.c/sys_sbrk() ->  The functionality of real memory allocation for a process requesting memory from the 
                  								 operating system, which may never be utilized (ensuring better resource management), 
                  								 has been removed from this function. There are two cases distinguished: when n>0 
                  								 (i.e., the process requests memory expansion), and when n<0 (i.e., the process requests 
                  								 memory shrinkage = memory deallocation). According to the assignment, when n>0, the p->sz 
                  								 (process memory size) is simply modified by increasing it by n, so that the process 'sees' 
                  								 (through its struct fields) that it has been granted additional n bytes of memory, but we 
                  								 do not actually allocate memory for the reasons explained. We wait until the process 
                  								 actually needs this additional memory, and then we allocate the required memory. 
                  								 When n<0, the function of the original `vm.c/uvmdealloc()` is preserved, which deallocates 
                  								 memory immediately, without being limited to virtual deallocation only. This makes sense as 
                  								 we prefer to deallocate memory as soon as possible but to be frugal in allocating memory that 
                  								 might never be utilized, depriving other processes. After this modification, we can also 
                  								 comment out the `proc.c/growproc()` function since we don't need it for n>0, and for n<0, 
                  								 we directly call `vm.c/uvmdealloc()` that was previously invoked by `proc.c/growproc()`. 
                  								 The intentional comments left visible to demonstrate the original version of xv6.


4) kernel/proc.c/growproc() -> For the reasons mentioned, it can be (and was) commented out.

5) kernel/trap.c/usertrap() -> After the previous changes, the following error message appeared (also mentioned in the assignment):


									init: starting sh
									$ echo hi
									usertrap(): unexpected scause 0x000000000000000f pid=3
									sepc=0x0000000000001258 stval=0x0000000000004008
									va=0x0000000000004000 pte=0x0000000000000000
									panic: uvmunmap: not mapped


								This error print is contained in the last `else{}` case of `usertrap()`, which checks for unexpected user-level states. 
								The error occurs because the process attempts to read, at some point, the physical address corresponding to a virtual 
								address. Considering the changes made so far in the previous functions, we have managed to allocate 'virtual/fake' 
								memory to a process, but not actual (physical) memory. Thus, it is evident that this physical address will not exist 
								(will be zero). To avoid immediate panic by the operating system, we introduce another `else if{}` case just before the 
								final `else{}`. In this case, we check whether the system error is due to a page fault read (`r_scause()=13`) or a page 
								fault write (`r_scause()=15`). In this scenario, the system should not panic but handle the lack of physical addresses 
								by allocating new pages.

6) kernel/vm.c/newpage_alloc() -> This function implements the allocation of a new page in case of a page fault 
                									error 13 or 15. Within the function, it first checks if the virtual address is within the memory boundaries of 
                									the process. Then, it rounds the virtual address to the beginning of a page and allocates memory (new page) using 
                									`kalloc()`. If successful, it initializes the memory using `memset()` and then maps the pages to establish a 
                									virtual-to-physical address mapping. Finally, it returns the memory address of the new page. In case of failure, 
                									it sets `p->killed = 1;` to kill the process.

7) kernel/vm.c/include section -> Following the guidelines, the header files 'spinlock.h' and 'proc.h' were included.

8) kernel/vm.c/mappages() ->  Line 198 ('panic("mappages: remap");') was commented out to bypass the particular panic and avoid it 
                							from appearing as an error during the process of mapping new allocated pages (which causes remap of 
                							already existing pages, but is not a practical problem).

9) kernel/vm.c/uvmunmap() ->  The calls to panic within the first two if structures of the for loop were commented out. Due to 
                							workload in other subjects, I did not have enough time to perfect the assignment. 
                							Therefore, I am not sure if further stages require commenting out all the panic calls inside uvmunmap().

							
10) kernel/vm.c/uvmcopy() ->  In this particular function, the calls to panic were commented out within the if structures to 
                							overcome the `FAIL` results in usertests that were related to the memory copy between parent and child processes. 
                							Now, in case of an absence of a physical address mapped to a virtual address, the function proceeds with `kalloc()` 
                							to allocate a memory page and then performs mapping of the new addresses.

11) kernel/vm.c/walkaddr() -> This function was causing the execution of usertests.c to terminate because it checks certain 
                							conditions: if a page table entry (pte) is 0 or if the memory pointed to by the pte does not have a
                              valid bit (PTE_V) or is not accessible by the user (PTE_U), the process terminates. However, by
                              introducing the following if structure, we can handle all three error cases by allocating a new
                              memory page through the `newpage_alloc()` function. Otherwise, we follow the
                              original process using the command 'pa = PTE2PA(*pte);' to find the physical address in the specific pte.


									if ((pa = newpage_alloc(myproc(), va)) <= 0){
										pa = 0;
									}
									return pa;


13) kernel/exec.c/exec() -> Here, a call to `vmprint()` was added before the last return statement of the function, but only for the 
							              first user process (p->pid == 1 in the `if{}`).



**Log Files**
---------

In the xv6.out file, which is created during code execution, the pagetable print from Step 1 and the test results 
(PASSES or FAILURES) are displayed.


**Makefile and Code Execution**
----------------------------

Using the command "make grade," all source files are compiled, and all test sections are executed through user/usertests.c.
Using the command "make qemu," all source files are compiled, and you can execute each section separately by typing 'echo hi' 
for 'lazytests' or 'usertests'.
