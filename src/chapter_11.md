# Paging - Introduction
*Basic definition: We chop up space into fixed-sized pieces, this is known as paging*
- We divide a process's address space into fixed-sized units, each of which we call a page
- Physical memory is view as an array of fixed-sized slots called page frames
- Each page frame contains a single virtual-memory page
## Example
Imagine we have a small address space of 64 bytes, with four 16-bytes pages. 

<center><img src="./images/pageas.png"></center>

<center><i>This could be the address space of a process, which is divided into 4 pages</i></center>

And that we have physical memory which consists of fixed-size slots, in this case 8 page frames.

<center><img src="./images/paginpm.png"></center>

Here we have:
- A 128 bytes physical memory
- 8 page frames (from 0 to 7)
- Each page of the address space mentioned at the beginning is located somewhere in memory (not necessarily in order) 

**Advantages of paging:** 
- Flexibility
- Simplicity: If the OS wishes to place to place our 64 byte address space into our 8-page physical memory, it simply finds four free pages, perhaps the OS keeps a free list and just grabs the first 4 free page that finds

Usually the OS keeps a per process data structure known as **page table:**
- It store the address translation for each of the virtual pages of the address space of the process
- This let us know where in physical memory a virtual page resides
- In our example the page table would be something like this: 
```
(Virtual Page 0 -> Physical Frame 3)
(Virtual Page 1 -> Physical Frame 7)
(Virtual Page 2 -> Physical Frame 5)
(Virtual Page 3 -> Physical Frame 2)
```

### Example memory access 
- We have the address space described at the beginning
- We want to perform a memory access: 
```S
movl <virtual address>, %eax
```
Lets focus on how to translate the virtual address: 
- We need to split it into two components:
	1. Virtual page number (VPN) (This will help us map our virtual page number to a physical frame number)
	2. The offset within the page

Since our virtual address space is 64 bytes, we need 6 bits total for our virtual address (2^6 = 64)
*This is because we can only reference 64 locations, and each location is 1 byte. We CAN'T reference a specific bit, the smallest addressable unit is a byte*

Our virtual address can be seeing as this: 

|Va5|Va4|Va3|Va2|Va1|Va0|
|---|---|---|---|---|---|

Here the highest order bit is `Va5` and the lowest order bit is `Va0`. Also we know that the page size is 16 bytes, hence we only need 4 bits to address the virtual memory inside our page (2^4=16), finally our 2 remaining highest order bits can be used to know which page number are we in. 

```
[Va5, Va4] -> VPN
[Va3, Va2, Va1, Va0] -> Offset
```

If we know want to load, for example, virtual address 21:
```S
movl 21, %eax
```
We know that 21 in binary is `010101`, hence we have the following virtual address: 

<center><img src="./images/vaddresstable.png"></center>

We know that our `VPN = 01 = 1` and that our `offset = 0101 = 5`, hence our virtual address is on the first virtual page with offset of 5 bytes. 

From our page table we know that virtual page `1` corresponds to `Physical Frame 7`, then we can replace the `VPN` with the `PFN` (Physical frame number) and then issue the load to physical memory: 

<center><img src="./images/translatepage.png"></center>

The offset stays the same because it tells us which byte within the page we want to address. 
## What's in the page table
- Page table is a data structure used to map virtual address to physical addresses, it a per process data structure, meaning each process has a page table
- Simplest form: Linear page table (Just an array) 
- Linear page table indexes the array by the virtual page number and looks up the page-table entry (PTE) at that index to find the physical frame number (PFN). 
- For example if I wanted to translate the virtual page number 3 to physical frame number, I would access `array[3]`  array index 3. 
### What's inside a Page-table entry (PTE)
- Each PTE has a number of different bits worth understanding at some level.

- **Valid bit**: Indicates if a translation is valid. When a process start running, it will have stack and code at one end of his address space, and heap on the other end. All the space in between is unused hence it will be marked invalid. By marking all the unused pages in the address space, we remove the need to allocate physical frames for those pages, thus saving memory. 
- **Protection bits**: Indicates if the page can be read, written or executed. 
- **Present bit**: Indicates if the page is on physical memory or disk (swap on Linux)
- **Dirty bit**: Indicates whether the page has been modified since it was brought into memory
- **Reference bit**: Track whether a page has been accessed, it helps us to determine which pages are "popular" and thus should be kept in memory. 

An example page-table entry would look like this for the x86 architecture: 
<center><img src="./images/pte_example.png"></center>

## The problem with paging: Too slow
Imagine we want to access memory `21` on our example's address space: 

```
movl 21, %eax
```

To do this we would need to translate virtual address space (21) into the physical address (117).
Hence, before fetching the data from address 117, our system needs to:
- Fetch the page table from the process's page table
- Perform the trasnlation
- Load the data from physical memory
To do so, the hardware must known where the page table is for the current process, let's imagines that this is on a cpu register, then our translate would look like this:
```
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
PTEAddr = PageTableBaseRegister + (VPN * sizeof(PTE))

offset = VirtualAddress & OFFSET_MASK
PhysAddr = (PFN << SHIFT) | offset
```
- On our first line we take our virtual address and apply a `VPN_MASK` to keep only the bits that represent or virtual page number, on our example, with virtual address `21` we would have: 
	- 21 in binary is `010101`
	- Our `VPN_MASK` would be `110000` (we only need our most significant bits)
	- `SHIFT` would be 4, because we want to move our bits 4 spaces to the right
	- `VPN = (010101 & 110000) >> 4 = (010000) >> 4 = (000001)`
- The second line would take the memory address of the page table and add the index (`VPN` multiplied by the size of the entries in the table `sizeof(PTE)`) to get the page table entry address
- Once the `PTEAddr` (page table entry address) is known, the hardware can load and extract the PFN (physical address of the page frame)
- On the last line, we use the `PFN` and left shift it and append the `offset` to get the final physical address space
- We can finally fetch the data from the physical address 

If you notice, we need to make one extra memory reference (we need to fetch the `PTE`) in order to get the initial memory address (`21` in this case).
