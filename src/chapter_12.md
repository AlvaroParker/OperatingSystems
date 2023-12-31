# Paging - Faster Translations
On the previous chapter we saw that: 
- By  using paging, we require a large amount of mapping information
- The mapping information is usually stored in physical memory
- Paging, hence requires an extra memory lookup for each virtual address generated by the running program (we need to check the mapped information that's stored in memory )

## The solution: TLB
- To speed address translation we need help from the hardware
- We add what's called a **translation-lookaside buffer** (TLB) 
- TLB is part of the chips memory-management unit (MMU) 
- It's a hardware cache, but because is "closer" to the CPU, memory access is much faster

**What does it do?** 
- For each virtual memory reference, we check the TLB to see if the translation is there
- If it is, we perform the translation, without having to access physical memory


## TLB Basic algorithm
```
VPN = (VirtualAddress & VPN_MASK) >> SHIFT 
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True) // TLB Hit
	if (CanAccess(TlbEntry.ProtectBits) == True)
		Offset = VirtualAddress & OFFSET_MASK
		PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
		Register = AccessMemory(PhysAddr)
	else
		RaiseException(PROTECTION_FAULT)
else // TLB Miss
	PTEAddr = PTBR + (VPN * sizeof(PTE))
	PTE = AccessMemory(PTEAddr) // Bad performance, we are accessing memory :(
	if (PTE.Valid == False)
		RaiseException(SEGMENTATION_FAULT)
	else if (CanAccess(PTE.ProtectedBits) == False)
		RaiseException(SEGMENTATION_FAULT)
	else
		TLB_Insert(VPN, PTE.PFN, PTE.ProtectedBits)
		RetryInstruction()	
```

This algorithms works like this: 
1. Extract the virtual page number (VPN) from the virtual address 
2. Check if the TLB holds the translation for this VPN
3. If it does, then the TLB holds the translation, hence we can now:
	1. Extract the page frame number (PFN) from the TLB
	2. Concatenate the PFN onto the offset from the original virtual address, hence forming the desired physical address 
	3. Access memory, handle errors
4. If it doesn't find the translation in the TLB:
	1. Hardware accesses the page table to find the translation (this is costly)
	2. Assuming that the translation is valid, we update the TLB
	3. Once the TLB is updated, we hardware retries the instruction 


## TLB Miss handling
There are two ways to handle a TLB miss: Hardware and software wise.
### Hardware handling: 
To handle TLB miss, the hardware must know: 
- Exactly where the page tables are located in memory
- The exact format of the page tables
On miss, the hardware walks to the page table, find the correct page-table entry and extract the desired translation, updates the TLB with the translation and retry the instruction. 

### Software handling
**The basic process works like this** 
- On a miss, the hardware raises an exception
- The exception pauses the current instruction stream, raises the privilege level to kernel mode, jumps to to a trap handler
- The trap handler, is code within the OS that is written to handle the TLB misses
- When the trap handler runs, it will lookup the translation in the page table, updates the TLB and return from the trap
- The hardware retries the instruction 

Difference between the trap signal of the TLB miss and others we saw:
- The return from trap is different than the one we saw on system calls. On the case of system calls, we would simply resume execution after the trap into the OS, just like a return from a procedure call.
- In the case of the TLB trap signal, we need to resumen execution on the instruction that caused the exception (basically running the instruction again), this can cause an infinite loop, so the OS needs to be careful when raising this exception. 

## TLB Contents
A TLB entry might look like this: 
```
VPN | PFN | other bits
```
- Both VPN and PFN are present in each entry.
- A translation coudl end up in any of these locations (this is called a fully-associative cache)
- The hardware searches the entries in parallel to see if there is a match

The `other bits` might contain: 
- A **valid** bit: If the entry has a valid translation or not
- **Protection** bit: Determines how a page can be accessed (for example, code page might be marked *read and execute* only, heap might be marked *read and write*, etc.)
- **Address-space**, **identifier**, **dirty bit** are also common bits present in here
## TLB Issue: Context switches
When switching from one process to another, the hardware or OS must be careful to ensure that the about-to-be-run process does not accidentally use translations from some previously run process. 

**Example:** We have a running process (P1) that assumes that the TLB is caching translations that are valid for it, i.e, that come from P1's page table, let's assume that the 10th virtual page of P1 maps to hysical frame 100. Then a context switch happens to another process (P2), assume we also have a virtual page number for this process (P2) but instead it maps to physical frame 170, if entries of both process were in the TLB, the contents of the TLB would be something like this: 

VPN| PFN | valid | prot
---|---|---|---
10 | 100 | 1| rwx
10 | 170 | 1 | rwx

Here we have a problem, two VPNs with the same value map to a different PFN, each VPN is valid only to their process but we don't have a way to tell from which process is each VPN

### Possible solutions
#### Flush the TLB on context switch
- On software-based system: with an explicit hardware instruction
- On hardware-managed TLB: The flush could be enacted when the page-table base register is changed
- On either case, the flush operations sets all valid bits to 0, clearing the contents of the TLB
- **Cost:** Each time a process run, it must incur TLB misses as it touches its data and code pages, the more context switches are, the higher the cost. 

#### Address space identifier
- Hardware support to enable sharing of the TLB is added via address space identifier (ASID)
- The ASID if kinda like a process identifier, but it has fewer bits
- If we take our example above, we would have the following table

VPN| PFN | valid | prot | ASID
---|---|---|---|---
10 | 100 | 1| rwx | 1
10 | 170 | 1 | rwx | 2

- Here we can identify which VPN corresponds to which process using the ASID value. 

## TLB Issue: Replacement policy 
When we are adding an entry, we have to replace it with an old one, which one to replace? 
Typical policies are: 
- Least-recently-used (LRU)
- Random policy
