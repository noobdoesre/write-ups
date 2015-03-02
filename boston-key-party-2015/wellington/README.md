**I'm a noob trying to get better with re and pwn. I wrote this write-up just not to lose the knowledge I got and share it with u, folks \\|_|/**

Description of this 250 pt worth re task claims:

```
If you had the code, you'd see that the program is calling 
`decrypt("[QeZag^VQZShWQgeWVQSe]ZW^^Q[`efWSV", X). 
Unfortunately, you don't have it, HAHAHAHAHAHA. 
Ho, and by the way, the flag ends with a dot.
```

We are given elf for x86-64: 

```bash
$ file troll_log.4643d195d55746aa180abf7144909677 
troll_log.4643d195d55746aa180abf7144909677: ELF 64-bit LSB  executable, x86-64, 
...
```

Let's try to launch it:

```bash
$ ./troll_log.4643d195d55746aa180abf7144909677 
BKP 2015 - Troll_log
Password: pass.
LOSE
```

Let's go deeper:

```bash
$ strings troll_log.4643d195d55746aa180abf7144909677 | grep -i copyright
prolog_copyright
prolog_copyright
Copyright (C) 1999-2007 Daniel Diaz
   linedit %-25s Copyright (C) 1999-2007 Daniel Diaz

```

So, prolog is involved somehow (if you are familiar with prolog, you could guess earlier - ``the flag ends with a dot.``, but I've never had a deal with this lanuage).

Digging binary is totally the wrong way - for some reason we are told that

``decrypt("[QeZag^VQZShWQgeWVQSe]ZW^^Q[`efWSV", X).``

should do something useful. It looks like prolog command, so let's try to spawn prolog shell: i tried many diffirent inputs, but then decided to check out how it handles dots, because they are used as 'terminators' 
in prolog:

```bash
$ ./troll_log.4643d195d55746aa180abf7144909677 
BKP 2015 - Troll_log
Password: .
.

system_error(cannot_catch_throw(error(syntax_error('user_input:1 (char:1) expression expected'),read/1)))
warning: /home/jvoisin/prez/BKP2015/main.pl:19: user directive failed
GNU Prolog 1.3.0
By Daniel Diaz
Copyright (C) 1999-2007 Daniel Diaz
| ?- 
```

Nice! Let's use given command (as far I understood there is a rule 'decrypt' which 'returns' boolean and prolog has a feature of enumerating all sets on which 'decrypt' returns True):

```bash
| ?- decrypt("[QeZag^VQZShWQgeWVQSe]ZW^^Q[`efWSV", X).

X = [105,95,115,104,111,117,108,100,95,104,97,118,101,95,117,115,101,100,95,97,115,107,104,101,108,108,95,105,110,115,116,101,97,100]
```

```bash
$ python
>>> X = [105,95,115,104,111,117,108,100,95,104,97,118,101,95,117,115,101,100,95,97,115,107,104,101,108,108,95,105,110,115,116,101,97,100]
>>> flag = [chr(x) for x in X]
>>> print(''.join(flag))
i_should_have_used_askhell_instead
```

So, the flag is ``i_should_have_used_askhell_instead.``

```bash
$ ./troll_log.4643d195d55746aa180abf7144909677 
BKP 2015 - Troll_log
Password: i_should_have_used_askhell_instead.
WiN
```
