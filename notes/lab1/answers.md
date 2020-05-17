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

# Formatted Printing to the Concole

- Explain the interface between printf.c and console.c. Specifically, what function does console.c export? How is this function used by printf.c?

At the very top interface is the function interface cprintf, which reads in a format string and some number of vars, by disassembly, this series of function calls is far easy to understand:

```
// cprintf -> vcprintf -> vfprintmt

24: cprintf (int32_t arg_8h, int32_t arg_ch);
; arg int32_t arg_8h @ ebp+0x8
; arg int32_t arg_ch @ ebp+0xc
// no-op
0xf0100a8d      endbr32
// into function frame
0xf0100a91      pushl   %ebp
0xf0100a92      movl    %esp, %ebp
// leave area for next function call
0xf0100a94      subl    $0x10, %esp
// load address of arg_ch to $eax
0xf0100a97      leal    arg_ch, %eax
// call vcprintf and return the $eax of that result as this result
0xf0100a9a      pushl   %eax       ; int32_t arg_ch
0xf0100a9b      pushl   arg_8h     ; int32_t arg_8h
0xf0100a9e      calll   vcprintf   ; sym.vcprintf
0xf0100aa3      leave
0xf0100aa4      retl

59: vcprintf (int32_t arg_8h, int32_t arg_ch);
; var int32_t var_ch @ ebp-0xc
; var int32_t var_4h @ ebp-0x4
; arg int32_t arg_8h @ ebp+0x8
; arg int32_t arg_ch @ ebp+0xc
0xf0100a52      endbr32
0xf0100a56      pushl   %ebp
0xf0100a57      movl    %esp, %ebp
0xf0100a59      pushl   %ebx
0xf0100a5a      subl    $0x14, %esp
// PIC, get address of next instruction to ebx, then find the address of putch
0xf0100a5d      calll   __x86.get_pc_thunk.bx ; sym.__x86.get_pc_thunk.bx
0xf0100a62      addl    $0x108a6, %ebx
// save args, make function call
// init local var var_ch
0xf0100a68      movl    $0, var_ch
0xf0100a6f      pushl   arg_ch
0xf0100a72      pushl   arg_8h
0xf0100a75      leal    var_ch, %eax
0xf0100a78      pushl   %eax
0xf0100a79      leal    -0x108dc(%ebx), %eax
0xf0100a7f      pushl   %eax
0xf0100a80      calll   vprintfmt  ; sym.vprintfmt
// set result to var_ch
0xf0100a85      movl    var_ch, %eax
0xf0100a88      movl    var_4h, %ebx
0xf0100a8b      leave
// return var_ch
0xf0100a8c      retl
```

But vprintfmt is too big and too hard to analyze by disassemble:

```
void
vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)
{
	// stack vars, set volatile
	register const char *p;
	register int ch, err;
	unsigned long long num;
	int base, lflag, width, precision, altflag;
	char padc;

	while (1) {
		// if the char on this position is not a control sign, just print it, also get rid of it if reaches the end of string
		// according to the self inc instruction, the fmt is actually moved
		while ((ch = *(unsigned char *) fmt++) != '%') {
			if (ch == '\0')
				return;
			putch(ch, putdat);
		}

		// Process a %-escape sequence
		padc = ' ';
		width = -1;
		precision = -1;
		lflag = 0;
		altflag = 0;
	// switch a char, since the last move has changed the position, just come to the next chat right after control sign
	reswitch:
		switch (ch = *(unsigned char *) fmt++) {

		// flag enablement
		// flag to pad on the right
		case '-':
			padc = '-';
			goto reswitch;

		// flag enablement
		// flag to pad with 0's instead of spaces
		case '0':
			padc = '0';
			goto reswitch;

		// flag enablement, set width
		// width field
		case '1':
		case '2':
		case '3':
		case '4':
		case '5':
		case '6':
		case '7':
		case '8':
		case '9':
			// get precision to the whole number
			for (precision = 0; ; ++fmt) {
				precision = precision * 10 + ch - '0';
				ch = *fmt;
				if (ch < '0' || ch > '9')
					break;
			}
			goto process_precision;

		// get a var form va_arg as the length of int
		case '*':
			precision = va_arg(ap, int);
			goto process_precision;

		case '.':
			if (width < 0)
				width = 0;
			goto reswitch;

		case '#':
			altflag = 1;
			goto reswitch;

		process_precision:
			if (width < 0)
				width = precision, precision = -1;
			goto reswitch;

		// long flag (doubled for long long)
		case 'l':
			lflag++;
			goto reswitch;

		// get a var form va_arg as the length of char
		// character
		case 'c':
			putch(va_arg(ap, int), putdat);
			break;

		// special, map an int into error message
		// error message
		case 'e':
			err = va_arg(ap, int);
			if (err < 0)
				err = -err;
			if (err >= MAXERROR || (p = error_string[err]) == NULL)
				printfmt(putch, putdat, "error %d", err);
			else
				printfmt(putch, putdat, "%s", p);
			break;

		// string
		case 's':
			if ((p = va_arg(ap, char *)) == NULL)
				p = "(null)";
			if (width > 0 && padc != '-')
				for (width -= strnlen(p, precision); width > 0; width--)
					putch(padc, putdat);
			for (; (ch = *p++) != '\0' && (precision < 0 || --precision >= 0); width--)
				if (altflag && (ch < ' ' || ch > '~'))
					putch('?', putdat);
				else
					putch(ch, putdat);
			for (; width > 0; width--)
				putch(' ', putdat);
			break;

		// (signed) decimal
		case 'd':
			num = getint(&ap, lflag);
			if ((long long) num < 0) {
				putch('-', putdat);
				num = -(long long) num;
			}
			base = 10;
			goto number;

		// unsigned decimal
		case 'u':
			num = getuint(&ap, lflag);
			base = 10;
			goto number;

		// (unsigned) octal
		case 'o':
			// Replace this with your code.
			putch('X', putdat);
			putch('X', putdat);
			putch('X', putdat);
			break;

		// pointer
		case 'p':
			putch('0', putdat);
			putch('x', putdat);
			num = (unsigned long long)
				(uintptr_t) va_arg(ap, void *);
			base = 16;
			goto number;

		// (unsigned) hexadecimal
		case 'x':
			num = getuint(&ap, lflag);
			base = 16;
		number:
			printnum(putch, putdat, num, base, width, padc);
			break;

		// escaped '%' character
		case '%':
			putch(ch, putdat);
			break;

		// unrecognized escape sequence - just print it literally
		default:
			putch('%', putdat);
			for (fmt--; fmt[-1] != '%'; fmt--)
				/* do nothing */;
			break;
		}
	}
}
```

The major techincal here is va_args family, which convert a single number into its format:

```
typedef __builtin_va_list va_list;

#define va_start(ap, last) __builtin_va_start(ap, last)
#define va_arg(ap, type) __builtin_va_arg(ap, type)
#define va_end(ap) __builtin_va_end(ap)
```

But to the bottom half, the true print is carried out by callback __putch__:

```
// putch -> cputchar -> cons_putc
// -> serial_putc / lpt_putc / cga_putc
static void
putch(int ch, int *cnt)
{
	cputchar(ch);
	*cnt++;
}

// Output a char
static void
cons_putc(int c)
{
	serial_putc(c);
	lpt_putc(c);
	cga_putc(c);
}

static void
serial_putc(int c)
{
	int i;

	// while COM1+COM_LSR is not avail, spin wait
	for (i = 0;
	     !(inb(COM1 + COM_LSR) & COM_LSR_TXRDY) && i < 12800;
	     i++)
		delay();
	// write char
	outb(COM1 + COM_TX, c);
}

// Write to the parllel port
static void
lpt_putc(int c)
{
	int i;

	for (i = 0; !(inb(0x378+1) & 0x80) && i < 12800; i++)
		delay();
	outb(0x378+0, c);
	outb(0x378+2, 0x08|0x04|0x01);
	outb(0x378+2, 0x08);
}

// Set CGA view, handle special chars since they change the linear write mode
static void
cga_putc(int c)
{
	// if no attribute given, then use black on white
	if (!(c & ~0xFF))
		c |= 0x0700;

	switch (c & 0xff) {
	case '\b':
		if (crt_pos > 0) {
			crt_pos--;
			crt_buf[crt_pos] = (c & ~0xff) | ' ';
		}
		break;
	case '\n':
		crt_pos += CRT_COLS;
		/* fallthru */
	case '\r':
		crt_pos -= (crt_pos % CRT_COLS);
		break;
	case '\t':
		cons_putc(' ');
		cons_putc(' ');
		cons_putc(' ');
		cons_putc(' ');
		cons_putc(' ');
		break;
	default:
		crt_buf[crt_pos++] = c;		/* write the character */
		break;
	}

	// What is the purpose of this?
	if (crt_pos >= CRT_SIZE) {
		int i;

		memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
		for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
			crt_buf[i] = 0x0700 | ' ';
		crt_pos -= CRT_COLS;
	}

	/* move that little blinky thing */
	outb(addr_6845, 14);
	outb(addr_6845 + 1, crt_pos >> 8);
	outb(addr_6845, 15);
	outb(addr_6845 + 1, crt_pos);
}
```

All exported interface of `console.c` are:

```
// Console init
void cons_init(void);
// Get a char from console
int cons_getc(void);

void kbd_intr(void); // irq 1
void serial_intr(void); // irq 4

// Put a char
void	cputchar(int c);
int	getchar(void);
int	iscons(int fd);
```

2. Explain the following from console.c:
```
1      if (crt_pos >= CRT_SIZE) {
2              int i;
3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
5                      crt_buf[i] = 0x0700 | ' ';
6              crt_pos -= CRT_COLS;
7      }
```

This section controls whatif a new char breaks up the page, since the `CRT_SIZE = (CRT_ROWS * CRT_COLS)`, this means adding a new char will pass the max length, move the second line and more into the true crt_buf, and set the last line to void and move the crt_pos to the last line.

3. For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
Trace the execution of the following code step-by-step:

```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```
> In the call to cprintf(), to what does fmt point? To what does ap point?

fmt means the format string which describes the output format, ap ponits to the va_args which describes multiple or unknown-in-advance vars.

> List (in order of execution) each call to cons_putc, va_arg, and vcprintf. For cons_putc, list its argument as well. For va_arg, list what ap points to before and after the call. For vcprintf list the values of its two arguments.

```
int	vcprintf(const char *fmt, va_list);

(gdb) b vcprintf 
Breakpoint 1 at 0xf0100a52: file kern/printf.c, line 18.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100a52 <vcprintf>:       endbr32 

Breakpoint 1, vcprintf (fmt=0xf0101b77 "6828 decimal is %o octal!\n", ap=0xf010ffe4 "\254\032")
    at kern/printf.c:18
18      {
// Indeed, the ap points to the memory place of the var args
(gdb) x/u ap
0xf010ffe4:     6828
```

- Run the following code.
```
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
```
> What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise. Here's an ASCII table that maps bytes to characters.
> The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?
> Here's a description of little- and big-endian and a more whimsical description.

57616 = 0xE110, so the first result is e110, Follow the little endian format, the second should be char('72', '6c', '64', \0), is "rld"

- In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
```
    cprintf("x=%d y=%d", 3);
-> x=3 y=1600

(gdb) x/d ap
0xf010ffe4:     3
(gdb) x/d (int *)ap + 1
0xf010ffe8:     1600
```

Unknown, the value should be the value right after the position of that 3.

- Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last.

> How would you have to change cprintf or its interface so that it would still be possible to pass it a variable number of arguments?

According to the stack structure, the vars are:
```
var(n)
var(n - 1)
var(n - 2)
var(2)
var(1)
esp+1 -> var(0)
esp -> ret-address
```

If the gcc changes its order, the interface should change to always allow the pushing order is reversed to achive the same result.

- Exercise 9. Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

```
#define	KERNBASE	0xF0000000
// Kernel stack.
#define KSTACKTOP	KERNBASE
#define KSTKSIZE	(8*PGSIZE)   		// size of a kernel stack
#define KSTKGAP		(8*PGSIZE)   		// size of a kernel stack guard

[KERNBASE>>PDXSHIFT] = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
```

According to the paging configuration, the KERNBASE and 4MB below has been mapped into phys memory. The kernel stack should be 32k. Get back to the right start:

```
relocated:
	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer
	# Set the stack pointer
	movl	$(bootstacktop),%esp
	# now to C code
	call	i386_init
```

It just move 4k bootstacktop(KERNBASE) into $esp and call into the right 1st function of kernel.

- Exercise 12. Modify your stack backtrace function to display, for each eip, the function name, source file name, and line number corresponding to that eip.

> In debuginfo_eip, where do __STAB_* come from? 

They comes from the lds load script for the sections .stab and .stabstr:
```
	/* Include debugging information in kernel memory */
	.stab : {
		PROVIDE(__STAB_BEGIN__ = .);
		*(.stab);
		PROVIDE(__STAB_END__ = .);
		BYTE(0)		/* Force the linker to allocate space
				   for this section */
	}

	.stabstr : {
		PROVIDE(__STABSTR_BEGIN__ = .);
		*(.stabstr);
		PROVIDE(__STABSTR_END__ = .);
		BYTE(0)		/* Force the linker to allocate space
				   for this section */
	}
```

According to the standard defination of N_SLINE, it could be found by pairing n_value and extract the n_desc from found record:

```
Line number in text segment

.stabn N_SLINE, 0, desc, value
desc  -> line_number
value -> code_address (relocatable addr where the corresponding code starts)
For single source lines that generate discontiguous code, such as flow of control statements, there may be more than one N_SLINE stab for the same source line. In this case there is a stab at the start of each code range, each with the same line number.
```