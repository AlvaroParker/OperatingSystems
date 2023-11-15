# Beyond Physical Memory - Mechanisms 
We've assumed that every address space of every running process fits into memory, we will now relax this assumptions, and assume that we wish to support many concurrently running large address space. 

To support large address spaces, the OS needs a place to stash away portions of address spaces that currently aren't in great demand, currently the most used place to stash this is the hard drive, we call this stash portion the swap space. 

## Swap space
A space on the disk for moving pages back and fort, we swap pages out of memory to this space, and we swap pages into memory from this space. 
### Example 
We have a 3 page physical memory and an 8-page swap space. We also have 3 processes (Proc 0, Proc 1 and Proc 2) and they are actively sharing physical memory, there's also a 4th process that has all his pages on the swap space, hence is not running. 

![](swap_example1.png)

## The present bit
On our page table entry, we need to ad a bit entry that represents if the desired page is either on swap space or in physical memory.
- If the present bit is set to one, it means the page is present in physical memory and everything goes as "normal"
- If the present bit is set to zero, the page is not in memory, but rather on disk somewhere, this causes a page fault. On page fault, the OS needs to handle this exception via a page-fault handler. 

## The page fault
There are two type of systems on TLB misses: 
- Hardware managed TLBs
- Software managed TLBs
On both of them, there OS has a **page-fault handler**, this will swap the page into memory in order to service the page fault. 
### How does the OS finds the pages on swap space
- The OS could use the bits in the PTE normally used for data such as the PFN of the page for a disk address
- When the OS finds a page-fault, it looks for the in the PTE to find the address on disk and issues the request to disk to fetch the page into memory
- When the I/O completes, the OS updates the page table to mark the page as present, it updates the PFN field on the PTE to the proper in memory location and retry the instruction

## What if memory is full? 
When the memory is full, the OS might choose to kick out some pages to swap space in order to free up some space on memory. The policy to do this is known as **page-replacement policy**

## Page fault control flow
What the hardware does during translation
```pseudocode
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
(Success, TlbEntry) = TLB_Lookup(VPN) 
if (Success == True) // TLB Hit (no need to translate by accessing memory) 
	if (CanAccess(TlbEntry.ProtectBits) == True) 
		Offset = VirtualAddress & OFFSET_MASK
		PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
		Register = AccessMemory(PhysAddr)
	else
		RaiseException(PROTECTION_FAULT)
else  // TLB Miss
	PTEAddr = PTBR + (VPN * sizeof(PTE))
	PTE = AccessMemory(PTEAddr)
	if (PTE.Valid == False)
		RaiseException(SEGMENTATION_FAULT)
	else
		if (CanAccess(PTE.ProtectBits) == False)
			RaiseException(PROTECTION_FAULT)
		else if (PTE.Present == True)
			// assuming hardware-managed TLB
			TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
			RetryInstruction()
		else if (PTE.Present == False)
			RaiseException(PAGE_FAULT)
```

What the OS does upon a page fault
```pseudocode
PFN = FIndFreePhysicalPage()
if (PFN == -1)              // No free page found
	PFN = EvictPage()
DiskRead(PTE.DiskAddr, PFN) // Sleep (waiting for I/O)
PTE.present = True          // Update page table with present bit and translation (PFN)
PTE.PFN = PFN
RetryInstruction()          // Retry instruction
```

