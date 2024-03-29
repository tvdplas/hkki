# Installation
On any Ubuntu based operating system, run the following: 
```
sudo apt install git
git clone https://github.com/this/repo

sudo apt install libgtk2.0-dev
sudo apt install make
cd hkki
make
./hkki
```
This should install all dependencies, build, and run the executable.

# Hakuoki script file format(uncompressed)

General weirdness:
-----------------
This is not a typical script file as in most NDS games. This is a fully 
interpreted bytecode language which I suspect, is the compiled version 
of an internal scripting language. It contains not only the spoken text 
and the speakers name but commands for all that is going on.
Completely reverse engineering the virtual machine on which these 
scripts are run would enable us to change virtually anything in the 
game. From different effects to adding totally new scenes. But reversing 
a bytecode language isn't all that easy and it's going to take a while 
till I get somewhere with that ;)

Anyway, as interesting as the whole custom bytecode VM thing is. It's a 
lot of trouble if you want to edit them. You have to take care of the 
same annoying things as when you edit other executable formats. Global 
references are used throughout the file, these have to be adjusted if 
you change the size of instructions or add instructions. Additional to 
the global references there is an export table (usually at the end of 
the file, more on that later), which may also need to be adjusted.

The header of the STCM2L format as it seems to be called is pretty 
useless. As far as I can tell there is no table that contains 
information on the start of the different sections of the script. So 
you'll have to look for 'CODE_START_' manually.

All Strings are 4byte aligned. This is achieved by adding 0x00 bytes to 
the end of the string until the 4byte boundary is reached.

Oh yea, all of this is little endian, obviously.

CODE section:
-------------

So this is the code body. It's the same thing as the .text section in an 
PE file for example.
The sections starts with the string 'CODE_START_', right after that the 
code starts.

Instructions are in the following format:
    32bit: is this instruction a call?
    32bit: opcode
    32bit: amount of parameters
    32bit: length of instruction block (in bytes)

Following that are the specified amount of parameter blocks, these 
blocks look like this:
    32bit: p0
    32bit: p1
    32bit: p2

Sorry for the wierd naming scheme but there isn't much logic in all of 
this, at least none that I have found. Parameters need some more 
explaining as they are totally terrible.

Normal paramters are usually all numbers with 0xff as their 
most significant byte of p0(little-endian that is, yes I know that it's 
weird).
Now the really interesting values are pointers, which come in 2 flavours 
in this case. I call one local pointers and the others global pointers.

Global pointers are a bit easier so I'll start with those. These 
pointers point to some other place in the file. Usually to another 
instruction. They are easily noticable as their p0 equals 0xffffff41, 
always. The global pointer is then located in p1.

Local pointers... I feel stupid explaining it because finding them is a 
bit hackish. These are the pointers which point to a location inside 
the instruction. Usually they point to a string which is passed to the 
instruction (which is normally the case with text that somebody speaks 
in a game). I believe that it's usually distinguished only by the VM 
who knows the types in the parameter list for every instruction.

However, we don't know the types so we have to do something a little 
different. First of all, local pointers are values without 0xff as their 
most significant byte. Sadly not all values are marked by the 0xff. So 
we can't assume that every parameter that is not led by a 0xff is a 
local pointer. To check if a value is a local pointer we check if the 
value that could be a local pointer points inside the instruction block. 
This test may ofc fail, due to obvious reasons. But in practice it works 
fine as values without the 0xff marking tend to be quite low.
Note that local pointers only occur in p0 and they are absolute 
addresses.

Because this is bloody complicated and I'm not a great writer here's a 
snippet of C code showing how to parse parameters for now.

for(int i=0; i<paramcount; ++i){
    parameter* p = new parameter;
    fread(&p->val0, 4, 1, f);
    fread(&p->val1, 4, 1, f);
    fread(&p->val2, 4, 1, f);

    if( ((p->val0>>24) & 0xff)!=0xff && 
            p->val0 > old_addr && p->val0 < old_addr+length){
        p->type = PTYPE_LOCALP;
        p->relative_addr = p->val0 - old_addr;
     }else if(p->val1 == 0xffffff41){
        global_val = p->val1;
        p->type = PTYPE_GLOBALP;
     }else{
        p->type = PTYPE_VALUE;
     }
}


* Calls:
    Remember the first value in the instruction? Yes, it specifies if 
    something is a global call. If this is set to 1 instead of the usual 0 
    this means that this instruction is in fact only a call to another 
    piece of code. In this case the opcode is the absolute global 
    address to the instruction that will be called.
* Strings:
    Local pointers usually point to larger blocks of data that you want 
    to pass. This most likely will be a string. A string type in STCM2L 
    looks like this:
        32 bit: zero (0x00000000)
        32 bit: amount of 4 byte blocks used (strlen/4)
        32 bit: one (0x00000001)
        32 bit: string length
        followed by the string


Export table:
-----------
At least this one is quite straightforward.
Comes after the code segment and is marked by the string 'EXPORT_DATA'
Format for entries:
    32 bit: zero
    32 byte: null terminated string. Rest of space is padded with zeros
    32 bit: address of instruction that is exported

