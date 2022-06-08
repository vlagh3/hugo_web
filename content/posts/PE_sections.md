+++
title = "PE Sections"
date = "2020-03-20"
author = "vlaghe"
authorTwitter = "vlagh3" #do not include @
cover = ""
tags = ["windows", "c"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = true
hideComments = false
+++

Last time I have introduced you to the PE file format and its structure. Today we will continue discussing about the sections and their importance. So, let’s dive directly into it !

## The Data Directory
Some data structures need to be quickly located within an executable *(e.g the base relocations, imports, etc)*. So, Windows , provides a way to find those well-known structures in a consistent manner, more explicitly into the `DataDirectory` attribute of the Optional Header.
This attribute is just an array of 16 structures. Each array holds an `IMAGE_DATA_DIRECTORY` structure. This structure is very simple, and has just 2 elements:
1. DWORD VirtualAddress *(This is actually a RVA)*
2. DWORD Size *(stores the size of the pointed structure)*

So you can calculate the address ,at which each of the entries reside, with the `VirtualAddress` element and get its size by accessing the `Size` element. To access those elements you need to specify the correct index in the array. For example if you want to access the `Export` table, you will need to specify the index 0, since it’s the first element in the `DataDirectory` array.
Below I’ll include all the entries according to Microsoft:
{{< image src="/img/Microsoft.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

As you can see, a lot of these entries actually “point” to a section. But why? Why would they make this DataDirectory and not create a section with a Section Header that can provide the information needed to access it?
Well, that’s’ what I asked also, and the answer is pretty simple.

The linker often merge 2 sections together like the `.bss` section and the `.data` section.  Thus, you need a way to access them.

You will see that usually in the Section Header, the file size is smaller than the memory size. Some sections store data that are not necessarily needed to be stored on disk *(like the `.bss` one)*, and when the executable is loaded into memory, it will actually allocate *(for some sections)* more bytes than needed, to store the sections that are useless to store on disk.
To visualize this, I made the following schema:
{{< image src="/img/Schema.png" alt="Hello Friend" position="center" style="border-radius: 8px;" >}}

To sum it up, the sections that are not needed to be on disk ,and only in memory, will be merged ,with other sections, in the extra allocated space of another section.

 
## Sections
As said earlier, the sections in a PE file are kind of jumping around, depending in which state they are *(disk, memory)*.

Before I start describing each section, I want to specify the following. Any code or data that might be needed by either the program ,or the operating system, gets its own section. So different files can differ in what kind of sections they have. For example, those are the most typical sections for a EXE file: .text, .bss, .rdata, .data, .idata, .reloc. And for a typical OBJ file: .drectve, .debug$S, .data, .text, .debug$T .

Ok, now let’s see what kind of sections we have to deal with and what is the role of each one.

- `.text`: The default section where the actual executable code of the file is stored.
- `.data`: The default read/write data section. Global variables typically go here.
- `.rdata`: Read-only data, typically strings go here.
- `.bss`: Stores unitialized data. Seems to be merged to .data section by the linker, with binaries I looked at.
- `.rsrc`: The resources are stored in here. This section is read-only. This is a quite interesting sections ,because of course here can be loaded icons and other stuff, but you can also store embedded binaries. Also this section has structures organizating it, some kind of a filesystem.
- `.reloc`: Base relocations. Relocation information is helping to modify some hardcoded addresses that assume that the code was loaded at its preffered base address in memory. Those are generally needed for DLLs and not EXEs. You can remove them when linking with /FIXED switch.
- `.crt`: Data added for supporting the C++ runtime (CRT). A good example is the function pointers that are used to call the constructors and destructors of static C++ objects.
- `.sdata`: “Short” read/write data that can be addressed relative to the global pointer. Used for the IA-64 and other architectures that use a global pointer register.
- `.didat`: Delayload import data. Found in executables built in nonrelease mode. In release mode, the delayload data is merged into another section.
- `.debug`: The .debug section is used in object files to contain compiler-generated debug information and in image files to contain all of the debug information that is generated.
- `.drectve`: Contains linker directives and is only found in OBJs. Directives are ASCII strings that could be passed on the linker command line
- `.edata`: Contains information about symbols that other images can access through dynamic linking. Exported symbols are generally found in DLLs, but DLLs can also import symbols.
- `.idata`: All image files that import symbols, including virtually all executable (EXE) files, have an .idata section.
- `.pdata`: The exception table. Contains an array of `IMAGE_RUNTIME_FUNCTION_ENTRY` structures, which are CPU-specific. Pointed to by the `IMAGE_DIRECTORY_ENTRY_EXCEPTION` slot in the `DataDirectory`.
- `.tls`: Data for supporting thread local storage variables declared with `__declspec(thread)`. This includes the initial value of the data, as well as additional variables needed by the runtime.
- `.cormeta`: CLR metadata is stored in this section. It is used to indicate that the object file contains managed code.
- `.sxdata`: The valid exception handlers of an object are listed in the .sxdata section of that object.

That was a lot of information, I know, but it’s important to know ,at least, the important sections. Besides that, you can always use this as a reference.

In the final part we will take a look at the import table, export table and some others. Then with all this knowledge, I will show some of the most common types of exploiting the format, and hopefully after all of that , we can get into the really amazing and fun stuff.

 
# Conclusion
Today we talked a little bit more about sections, and we made a high step in understanding this format even better. I really enjoy writing these articles, so I hope you enjoy reading them as much as me writing it :D. Don’t forget to check the resources and also google about this topic and the very interesting ways of taking advantage of this knowledge.

## Resources
- https://docs.microsoft.com/en-us/windows/desktop/Debug/pe-format
- https://msdn.microsoft.com/en-us/library/ms809762.aspx
