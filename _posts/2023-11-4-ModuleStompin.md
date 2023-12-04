---
layout: post
title: Module Stomping
subtitle: Who up stompin they modules
thumbnail-img: /assets/img/Stompin_thumb.jpg
share-img: /assets/img/Stompin_thumb.jpg
tags: [Evasion, Windows]
categories: Red_Team
comments: true
readtime: something lmao
---


<img src="../assets/img/Stompin_thumb.jpg" height="50%" width="50%" class="mx-auto d-block" unselectable="on"/><br/>

In my last blog post I talked about stack spoofing, a technique used to hide the origin of an API call so as to not point back to our implant in memory. The implementation I went over is a trade-off; at a glance, the stack is clean, but in reality, there are a myriad of IOCs. Namely, the use of gadgets and a misalignment: a stack frame's return address that lacks a preceding call instruction. There is an alternative to this: Module Stomping. The purpose of this post is to discuss and demonstrate both benefits and indicators of compromise (IOCs) that Module Stomping and some of its variants brings to our implants.

# Reviewing Module Stomping
Module Stomping works by loading or mapping a module into memory, overwriting it with our payload (preferrably an R/X section), then executing our payload. First, executing shellcode which calls `MessageBox` by directly executing the memory (function pointer), without stomping:


A basic example using `Chakra.dll` to execute shellcode which calls `MessageBox`:

<img src="../assets/img/Stomping_1.png" height="75%" width="75%" class="mx-auto d-block" unselectable="on"/><br/>

Notice how the call stack of the main thread reveals that `MessageBox` was executed by a private read/executable memory address. Since it is memory that does not pertain to an module mapped from disk, it may be subject to a scan which would reveal the shellcode's contents. However, if we were to execute the same shellcode with Module Stomping, this would be the result:

<img src="../assets/img/Stomping_2.png" height="75%" width="75%" class="mx-auto d-block" unselectable="on"/><br/>

<details style="background-color: #000000; color: #FFFFFF; border-left: 5px solid #00FF00; border-radius: 10px; padding: 10px;">
    <summary>Code Snippet</summary>
    <pre>
        <code style="color: white">
#include &#60;windows.h&#62;
#include &#60;stdio.h&#62;
void main() {

	unsigned char Shellcode[] = //msfvenom -p windows/x64/messagebox TEXT=hi TITLE=hi -f c EXITFUNC=thread

	PIMAGE_DOS_HEADER Dos	  = NULL;
	PIMAGE_NT_HEADERS Nt	  = NULL;
	PIMAGE_SECTION_HEADER Txt = NULL;

	PVOID pTxt     = NULL;
	DWORD SzTxt	   = 0;
	DWORD OldProt  = 0;
	
	Dos = ( PIMAGE_DOS_HEADER )  LoadLibraryExA( "Chakra.dll", NULL, DONT_RESOLVE_DLL_REFERENCES );
	Nt  = ( PIMAGE_NT_HEADERS )  ( ( ( PBYTE ) Dos ) + Dos->e_lfanew );
	Txt = IMAGE_FIRST_SECTION( Nt ); 

	pTxt  = ( ( PBYTE ) Dos ) + Txt->VirtualAddress;
	SzTxt = Txt->Misc.VirtualSize;

	VirtualProtect( pTxt, SzTxt, PAGE_EXECUTE_READWRITE, &OldProt );
	memcpy( pTxt, Shellcode, sizeof(Shellcode) );
	VirtualProtect( pTxt, SzTxt, PAGE_EXECUTE_READ, &OldProt );

	// Idk how to actually do a clean looking function pointer execute
	typedef void* (*vpexec)();
	vpexec exec = ( vpexec ) pTxt;
	exec();

}
        </code>
    </pre>
</details>

Our call stack does not show indications of unbacked R/X memory anymore; all return addresses are able to be resolved to an offset of a loaded module. However there is a significant IOC: checking the bytes between the module's sections in memory and on-disk would reveal that the bytes have been modified! PE-Sieve can help identify this; it compares the bytes of a module on disk to the module in memory. While it does this to detect inline hooks, this can double as a Module Stomping deteciton.

<img src="../assets/img/Stomping_3.png" height="75%" width="75%" class="mx-auto d-block" unselectable="on"/><br/>

# Advanced Module Stomping
Module Stomping was improved upon by different projects such as [ModuleShifting](https://github.com/naksyn/ModuleShifting), [Sweet Dreams](https://github.com/CognisysGroup/SweetDreams), and [Brute Ratel](https://bruteratel.com/release/2023/03/19/Release-Nightmare/)'s sleep implementation. The improved technique, which seems to be referred to as "Advanced Module Stomping" most commonly, restores the original contents of the stomped module after execution. By restoring our the original module's memory after execution, we no longer get flagged by PE-Sieve.

<img src="../assets/img/Stomping_4.png" height="75%" width="75%" class="mx-auto d-block" unselectable="on"/><br/>

GGEZ I'm going to bypass CS/ATP/Cylance/etc? Nah. There's actually another significant IOC, but before I talk about that, it's important to note that the code we've used thus far is simple; a small loader that runs simple shellcode. What if we wanted to go bigger -- an implant perhaps? There's a lot more complications that come with this. 2 glaring issues with implementing a loader which uses Advanced Module Stomping to hide implant call stacks:

* They are often reflectively loaded -- if our loader simply executes implant shellcode, the implant will actually just reflectively load itself into unbacked memory! There goes our hard work.  
* They persist in memory -- how can we restore a module's code without crashing the implant?  

So what now? These were questions I had at the beginning of this year, but then Brute Ratel's Nightmare release came out in March and was capable of overcoming these. Currently, to my awareness, there is no open source project that does this. This next phase of the blog will be focused around how I implemented Advanced Module Stomping in a reflective loader. If you're only interested in the IOCs of the technique, skip towards the bottom of this post.

# Credits
Before I go any further, major credit is due to Chetan, Kyle, and Austin. My POC is essentially AceLdr, with Advanced Module Stomping implemented via slight modifications to Foliage sleep obfuscation. This wouldn't be possible without [Kyle](https://github.com/kyleavery/AceLdr) and [Austin](https://github.com/realoriginal/titanldr-ng)'s work. Chetan's [blog post](https://bruteratel.com/release/2023/03/19/Release-Nightmare/) about the topic was also incredibly useful in troubleshooting a lot of issues I had. Also, massive thanks to [Bakki](https://twitter.com/shubakki) for helping me understand the structure of AceLdr so I could actually implement the technique.

# Foliage
The first issue isn't too much of an issue; if we're worried that the reflective loader of our implant will reflectively load itself into private memory, we can simply create and implement our own reflective loader and prepend it to an implant in DLL format. But what about the second issue? Sleep obfuscation techniques faced a similar issue: how do you encrypt a running implant in memory without crashing it? This was addressed by using timers and APCs in the form of [Ekko](https://github.com/Cracked5pider/Ekko/) and [Foliage](https://github.com/realoriginal/foliage). I ended up forking AceLdr, which implemented Foliage, and modified the thread contexts to support sleep obfuscation. Foliage was implemented via an Import Address Table (IAT) hook on the `Sleep` API, so an implant would use Foliage rather than a normal sleep call.

With that being said, a quick rundown on the Foliage sleep technique.

* Create a suspended thread (Thread A) with a comedically large stack. The thread start address will be on RtlUserThreadStart + 0x21
* Capture Thread A's context
* Copy that context into a comedic amount of thread contexts
* Change most of the contexts' RIP/rcx/rdx/etc to point to different API calls and argument values
    * With the large stack, we can offset each context to include room for stack args (5+ args)
    * Our contexts will make implant R/W => encrypt => spoof implant thread context => decrypt => fix context => make implant R/X => continue normal execution
* Queue up a buncch of APCs on Thread A to execute NtContinue, passing our contexts as the argument
* Signal Thread A to execute our APCs and magic happens

If you want more on sleep obfuscation, read my other [blog post](https://dtsec.us/2023-04-24-Sleep/) (shameless plug).

We can simply modify the contexts to restore a stomped module's .text section before rest, and restore implant after rest, rather than encrypting implant. However, it gets a little tricky because that means we need to back up the original .text section of the stomped module along with our implant. This is would done during reflective loading because by the time implant is running, the module would already stomped. This would mean our sleep IAT hook would have to be aware of the address of the backup, along with the size of the respective data. We'll have to mod AceLdr up a bit to make this work.

# Modding AceLdr
Understanding how to pass information gathered/created from the reflective loading process, to our implant (in this case, its IAT hooks) will be important. Shoutout again to bakki for helping me understand how AceLdr did this. AceLdr makes the linker compile code into a certain layout via a linker script. Note that only the .text and .rdata sections are linked because we want our reflective loader to be R/X, so there will be no writeable section.

```
SECTIONS
{
    .text :
    {
        *( .text$A )
        *( .text$B )
        *( .text$C )
        *( .text$D )
        *( .text$E )
        *( .rdata* )
        *( .text$F )
    }
}
```
To indicate which .text section our executable code will be compiled to, the `SECTION( [A-F] )` macro is used. Keep in mind this is specifically for the `mingw` compiler. For example, the following code for the `copyStub` function will be located at the "B" section of `.text`

```
SECTION( B ) VOID copyStub( PVOID buffer )
{   
    PVOID Destination   = buffer;
    PVOID Source        = C_PTR( OFFSET( Stub ) );
    DWORD Length        = U_PTR( G_END() - OFFSET( Stub ) );

    memcpy( Destination, Source, Length );
};
```

The way the code was set up results in the following organization:

* .text$A - Start (Shellcode entrypoint)
* .text$B - Reflective Loader functions
* .text$C - Stub (Also referred to as Table)
* .text$D - IAT Hooks, including Foliage
* .rdata  - yeah
* .text$F - GetIp

During reflective loading, AceLdr copies sections C to F into the memory it allocated, then it copies and maps the implant. The original instance of AceLdr is then freed from memory based on the Cobalt Strike profile. The IAT hooks written during the reflective loading process point to the the copied instance of section D. The Stub in section C is actually just empty data that gets populated during the copying process. This is how we can pass the location and size of our backup page to Foliage.