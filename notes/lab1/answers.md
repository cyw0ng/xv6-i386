# Lab01

## Part 2: The boot loader

- At what point does the processor start executing 32-bit code?

After the long jumping to the $protcseg instruction segment:
```asm
ljmp    $PROT_MODE_CSEG, $protcseg
```

Acording to the last instruction:

```
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
```

The cr0 control register has been turned into real mode enabled.

- What exactly causes the switch from 16- to 32-bit mode?

By setting the PE(Protection mode) bit of cr0 register to 1.

- What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?

The last instruction of the bootloader should be calling the first instruction of the loaded kernel at the section e_entry:
```C
((void (*)(void)) (ELFHDR->e_entry))();
```

According to the elf standard, this section serves as the entrypoint of the target elf image.

- Where is the first instruction of the kernel?

According to the `kernel.ld` lds script file, the entry point should be `_start`, and the `_start` section should be mapped into `entry`:

```
ENTRY(_start)

.globl		_start
_start = RELOC(entry)

.globl entry
entry:
	movw	$0x1234,0x472			# warm boot
    movl	$(RELOC(entry_pgdir)), %eax
	movl	%eax, %cr3
	movl	%cr0, %eax
	orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
	movl	%eax, %cr0

	mov	$relocated, %eax
	jmp	*%eax
relocated:
	movl	$0x0,%ebp			# nuke frame pointer
	movl	$(bootstacktop),%esp
	call	i386_init
```

So the first instruction here shoule be a `movw` which serves as a common procedure to jump over the memory testing:

```
/* Write 0x1234 to absolute memory location 0x472.  The BIOS reads this on booting to tell it to "Bypass memory test (also warm boot)".  This seems like a fairly standard thing that gets set by REBOOT.COM programs, and the previous reset routine did this too. */
```

Then we start to load the paging system, by loading the predefined entry_pgdit to cr3 control register:

```
__attribute__((__aligned__(PGSIZE)))
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0] = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,

	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
    [KERNBASE>>PDXSHIFT] = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};
```

Then turn on paging by setting `PG` bit to 1. Then jumping into `$relocated` section to start C format stacking based calling mode and setting the endpoint of the stack:

```
movl	$0x0,%ebp			# nuke frame pointer
movl	$(bootstacktop),%esp
call	i386_init
```

- How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

By inspecting the e_phnum of the kernel elf image:

```
e_phnum
    This member holds the number of entries the program header table. Thus the product of e_phentsize a e_phnum gives the table's size in bytes.  If a file has program header, e_phnum holds the value zero.
```

First, it reads the fixed length of ELFHDR into memory, then, relocates to program segment and map each section into memory.

- What if changing the start address of the text section

```
    // Move the 0x7c00 to 0x0000 as the example
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x0000 -o $@.out $^
```

Comparing the normal and the abnormal one, we can find that the jumping segment is not the same, and the relocated address is not the same too:
```
(gdb) si
[f000:cf59]    0xfcf59: ljmpw  $0xf,$0xcf61

(gdb) si
[   0:7c2d] => 0x7c2d:  ljmp   $0xb866,$0x80032
```

So the first instruction to see whether the entry point address is wrong is the first long jump which may change the segment.

But why the machine seems running as an endless loop:

Since first we just map the 4MB memory from virtual to physical, the other parts of the memory is not paged, so visiting these address will cause an error to break down the machine, but the qemu may reboot automatically when the virt. machine is down on time, so there comes an endless loop.

- How does the memory virt-phys mapping happens?

Set a break point at `0x7C2A` since it is the point where the system switch from non-paging to paging mode:
```
(gdb) b *0x7c2a
(gdb) set $mem_s = 0x100000
(gdb) set $mem_e = 0xF0100000
```

Since the page mapping is set, every memory request to the VA address will be translated into its corresponding physical address automatically.