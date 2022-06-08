+++
title = "Exploring the ELF file structure"
date = "2020-04-14"
author = "vlaghe"
authorTwitter = "vlagh3" #do not include @
cover = ""
tags = ["linux", "c"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = true
hideComments = false
+++


Hello folks, hope you are all doing well because today we are going to talk about ELF files, but don’t be scared. Even if this topic offers a huge amount of information, I will try to make it as simple and fun as I can. With that being said, let’s talk a bit about the structure of this article. It will be split into the following parts:

```
General Information
The anatomy of an ELF file
  Missconsceptions
  Linking and executing
  ELF Header
  Program headers
  Section headers
Creating your own readelf
Conclusion
```

 
# General Information
So first of all, why would you want to bother learning about a specific file format that was adopted as a system default in UNIX almost 20 years ago?

- Well, this will help you understand better the inner workings of your operating system, thus giving you the answer to the known questions like why/what happened.
- You will be able to research ELF files, thus helping you in forensics.
- For a better understanding while developing.
- If you want to dive into reverse engineering and exploitation, this will come in handy.
- You will have another tool to play around with and maybe discover new creative ways to reap the benefit out of it.

I hope that I got you excited, so let’s jump into it.

 
## How a process is created
So whatever OS you are using, the idea is basically the same. In order to execute commands, the CPU, needs some specific language , also known as assembly/machine code. The OS is making this possible by translating common functions *( as simple as printing something to the screen )* in assembly language. So, instead of talking directly to the CPU, we use a programming language, with internal functions. A compiler then translates these internal functions into object code.

The object code is then linked *(with the help of a linker tool)* into a full program. Thus, resulting a binary file, which can be executed on the specific platform and CPU type.

 
# The anatomy of an ELF file

ELF stands for Executable and Linkable Format and it’s a common standard file format in UNIX systems. It is very flexible and extensible, for example it doesn’t exclude any particular CPU or ISA, allowing it to be adopted by many different hardware platforms and OS. A lot of people think that ELF files are just binaries and executables, but as discussed earlier we have already seen that it was used as an object file. Also, it can be used for shared libraries, core dumps and even for kernel modules !

Ok, so let’s start with a general layout on how a typical ELF file is structured:


```
   Linking View            Execution View

+-----------------+     +-----------------+
|  ELF header     |     |  ELF header     |
+-----------------+     +-----------------+
|  Program header |     |  Program header |
|  table (opt.)   |     |  table          |
+-----------------+     +-----------------+
|    Section 1    |     |                 |
+-----------------+     |    Segment 1    |
|       ...       |     |                 |
+-----------------+     +-----------------+
|    Section n    |     |                 |
+-----------------+     |    Segment 2    |
|       ...       |     |                 |
+-----------------+     +-----------------+
|       ...       |     |       ...       |
+-----------------+     +-----------------+
|  Section header |     |  Section header |
|  table          |     |  table (opt.)   |
+-----------------+     +-----------------+
```



As you can see, an ELF file has two different views and both of them have always 2 permanent headers:

1. ELF header *(ELF32_Ehdr/ELF64_Ehdr)* and the section header *(Elf32_Shdr/Elf64_Shdr)*
2. ELF header *(ELF32_Ehdr/ELF64_Ehdr)* and the program header *(Elf32_Phdr/Elf64_Phdr)*

The first one *(Linking View)* is divided into sections and it’s used when linking of a library or program takes place. The sections contain data about object files like: instructions, debugging information, symbols or relocation information.

The second one *(Execution view)* is divided into segments and it’s used during program execution.
We will discuss more about them later. For now, let’s focus on the ELF header.

## The ELF Header
The header that is always present in both states it’s the ELF header and it’s defined as follows:

{{< image src="/img/struct_ehdr.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

This structure is not very difficult to understand. It contains all the information for the binary within its very first bytes. Let’s quickly go through the first 6 attributes of the structure:

    E_indent : initial magic bytes that provide an answer for the OS on how to interpret and decode the files content.
    E_type   : identifies the object file type (executable, shared object, relocatable, etc)
    E_machine: specifies the required architecture
    E_version: specifies the current version (usually that’s 1)
    E_ehsize : specifies the ELF header’s size
    E_entry  : specifies the virtual address at which the system transfers the control first, thus starting the process. You can think of it as the main function, even if it’s more complicated than that.

The next values all specify certain offsets, size, address values for the section header and program header values which I will come back later.
The program header


## The program header

The program header describes segments within the binary that are necessary for program loading.
These segments contain one or more sections describing the memory layout, on the disk, of an executable and how it should be loaded into memory.
Since they are managing the creation of the process image, a program header becomes mandatory for executable files, but it’s optional for shared objects. That’s because a relocatable or shared object file(\*.o) is meant to be linked into an executable, but not meant to be loaded into memory.

The program header structure looks like this:

{{< image src="/img/struct_phdr.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

    P_type   : This attribute is specifying the type of the segment (e.g dynamic linking tables)
    P_offset : Gives the offset from the beginning of the file at which the first byte of the segment resides
    P_vaddr  : Gives the virtual address at which the segment will be loaded into memory
    P_paddr  : For systems which physical addressing is relevant, this member is reserved for the segment’s physical address.
    P_filesz : Contains the size of the file image of the segment
    P_memsz  : Contains the number of bytes in the memory image of the segment
    P_flags  : This attribute gives flags relevant to the segment (e.g read, write, execute)
    P_align  : Some alignment bytes with the power of 2

Since a program has multiple program segments, the ELF gives us all the information about where and how many of them exist in the ELF header:

- `E_phoff    `: Offset to the start of the program header table
- `E_phentsize`: Contains the size of a single program header table entry
- `E_phnum    `: Contains the total number of entries in the program header table

 

### Common program headers:
- `PT_LOAD`: This is always present in an executable and there will be at least one of these. It’s describing a loadable segment which is mapped into memory.
Generally a basic dynamically linked ELF executable will have 2 of these segments:
  * one for the text segment *(the actual program code)*
  * another one for global vars, data segment and other dynamic linking information
  * the memory alignment will be specified by the `p_align` member

- `PT_NOTE`: This type of segment it’s used by specific vendors/systems for marking an object file with special information that other programs will check for conformance, compatibility, etc.
  - it can hold any amount of entries.
  - each of the entries are an array of 4-byte words with the processor specific endianess in mind.

- `PT_INTERP` This program header element it’s useful for the system at execution time. The program retrieves the path name from it and creates the initial process image. Then is the interpreter’s responsibility to receive control from the system and provide an environment for the application program.
  - from my local `/bin/ps` we can find where the `PT_INTERP` segment is located with the help of `readelf –l /bin/ps`. After, we can use `hexdump` with the offset retrieved earlier to see its content.
  {{< image src="/img/iterp.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}
  - as you can see, in the example above the linker used is the ld-linux-x86-64, since our executable is 64-bit.

- `PT_PHDR`: This one holds the location and size of the program header table itself.
- `GNU_EH_FRAME`: This is a sorted queue used by the GNU C compiler. It stores exception handlers. So, when something goes wrong, this area can be used to deal with it correctly.
- `GNU_STACK`: It’s used to store stack information. This is a somewhat important segment for exploit development, because if the stack is executable and you apply a little memory manipulation magic, it may bring to some really serious security problems. So, if the GNU_STACK segment is not available, then usually it’s used an executable stack.

## Section headers

As we have spoken earlier, the segments from the program header table are necessary for an executable to run. Each of those segments have sections that are needed during linking time. The sections can be found in the section header table *(this is basically an array of Elf32_Shdr/Elf64_Shdr structures)* and each of these store information needed for the dynamic linker such as: symbols, global variables, etc. Also, they are referencing the size and location of them. They are not needed for correct program execution whereas program headers are.

This is possible because they are not helping to load and map any memory layout into the binary. Thus, you can strip off the sections, but the executable will be much harder to reverse/debug.
Let’s see how the `Elf32_Shdr/Elf64_Shdr` structure look:

  {{< image src="/img/struct_shdr.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}


    Sh_name    : This member holds an offset to the name of the section. It’s value is an index into the .shstrtab (Section Header String Table).
    Sh_type    : This member categorizes the sections’s content and semantics. (e.g  SHT_RELA=holds relocation entries,  SHT_SYMTAB=holds a symbol table)
    Sh_flags   : Specifies a 1-bit flag such as: SHF_WRITE(writable), SHF_ALLOC(occupies memory during execution)
    Sh_addr    : If the section will be in the memory image of the process, this member is holding the address where it will reside.
    Sh_offset  : Is holding the offset from the beginning of the file to the section.
    Sh_size    : Is holding the size of the section in bytes
    Sh_link    : Points to another section
    Sh_info    : This member holds extra information about the section.
    Sh_entsize : Contains the size of each entry, for sections that contain fixed-sized entries.

### Common sections

- `.text`: Contains the program code, this is usually packed within a segment with read and execute permissions *(`PT_LOAD`)*. Also it will be loaded only once, since the content will not change.
- `.data`: This section resides in the data segment and it contains initialized data that will contribute to the program’s memory image. (e.g initialized global variables)
- `.bss`: Same as the .data section but with uninitialized data.
- `.dynsym`: Holds the dynamic linking symbols table imported from shared libraries (e.g exit from libc) that are dynamically loaded at runtime.
- `.debug`: Holds information for symbolic debugging
- `.dynamic`: Holds dynamic linking information
- `.plt`: Holds the procedure linkage table. This is used to call functions from used shared libraries.
- `.got.plt`: This goes hand in hand with .plt section to dynamically resolve and guide the program to the correct address of the imported shared libraries functions.
- `.rodata`: Contains read-only data such as strings from code that look like this perror(“Error occurred !”).
- `.hash`: This section holds a symbol hash table.
- `.symtab`: The .symtab section contains all symbols from .dynsym as well as local symbols for the executable. This is mainly used for debugging.
- `.strtab`: Contains the symbol string table that is references by an entry within `ElfN_Sym` structs.
- `.shstrtab`: Contains the section header string table that is used to resolve names for each section. More precise, in here are the string values for the `sh_name` field from the section header struct. They can be accessed via an index/offset added on the `sh_offset` of this section.
- `.ctors/.dtors`: The .ctors *(constructors)* and .dtors *(destructors)* sections contain function pointers to initialization and finalization code that is to be executed before and after the actual `main()` body of program code.

The ELF header provides the needed information to find the location and the number of sections (same as the program header)

      E_shoff     : Offset to the start of the section header table
      E_shnum     : Specifies how many section headers are in the section program header
      E_shentsize : Specifies the size of one section header.

> **NOTE**: You can calculate the total size of the program header/section header by multypling `E_shnum/E_phnum * E_shentsize/E_phentsize`

# Creating your own readelf

Now, that we have covered all the information needed, you are probably thinking what can we do with that. The first thing that popped into my mind was to recreate the functionality of readelf. Basically, the tool is just inspecting the bytes from the ELF Header, Section Header and Program Header and it’s interpreting the data in a human readable form.
So, I started to build my own one that is doing exactly the same thing in order to understand even better all the structures and the format itself.

Let’s start by interpreting the theory that we have learned and get a better visual representation of it.
I wanted to make things easy, so we will take a look at the `/bin/ps` as our experimental rat. And it’s looking like that:
{{< image src="/img/file_ps.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

We are dealing with a 64-bit binary, so we need to keep in mind the appropriate data types and the overall size of each structure!
The ELF header is going to be 64 bytes long. With that knowledge, we can display the first 64 bytes with `hexdump` and see the following:
{{< image src="/img/elf_header.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

If you go back and look at the `Elf64_Ehdr` structure we can easily translate the bytes to the corresponding attributes:
{{< image src="/img/script_elf_header.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

> **NOTE**: the `e_ident` attribute is having a padding of  8 bytes that are unused. I will let you try to think of methods how to use this in your advantage. 😛

What I did here was just to make some dictionaries according to the [ELF paper](https://www.cs.cmu.edu/afs/cs/academic/class/15213-f00/docs/elf.pdf). Then create an `elf_header` class that has methods for every attribute to retrieve the information based on the file that I opened. We can do this same approach for the program headers and section headers within the binary. The only difference being that there are multiple of those.
So, if we go to the specified offset retrieved from the attribute `e_phoff` from the elf header, we can go to the start of the program header table and match the bytes again based on the `Elf64_Phdr` struct.
{{< image src="/img/Program_header.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

Last but not least, we do the same thing with the section header table.
{{< image src="/img/section_header.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

All of the code used for my own readelf ca be found [@ my github](https://github.com/vlagh3/interelf). Also, if you want to go in depth on this topic, you can have a look at the contents in the [elf.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/elf.h)

 

# Conclusion
**What are the take aways from everything we have done here?**
Never be afraid of testing, playing around and messing up with things. That’s how you learn after all!
This is a simple example I gave you, but with this knowledge, you can come with really creative projects. I already have some ideas that we will discuss in the next articles.

That’s it folks ! I hope you’ve learned something and developed a better understanding of the ELF format and the linux world.
By no means, I’m an expert in this field and the information I detailed in this article is pure self researching, so any productive criticism is always welcomed.
I’m really curious how you will use what you have learned, so let me know by commenting or contacting me. I’m always excited to talk on different subjects and maybe develop a new project we can work on.
Thanks for reading and I hope you’ve got a great time !
