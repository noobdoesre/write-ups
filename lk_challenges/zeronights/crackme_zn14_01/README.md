I really liked this challenge because it teaches a very important approach of reverse engeneering: don't dig every instruction - sometimes it's better to see a big picture.

####Before we start
The binary is packed with upx. After ``upx.exe -d crackme_zn14_01.exe`` we need to disable aslr for this binary. Through ``cff explorer`` open ``File Header -> Characteristics -> check Relocation info...`` (it's easier than using emet).

####Big Picture
Look at @doHugeCheck: there is alot of stuff done with email but we do not need to reverse it! All we should know is - email is transformed to some 512 byte array, which has standalone non-null bytes and sequences of null bytes. What really matters is what's happening with password: 

```python
password -> passToBin() -> passRLE -> decodePassRLE() -> pass512ByteArray
email -> blackBox() -> email512ByteArray
if pass512ByteArray == email512ByteArray:
  win()
else:
  loose()
```

####Let's go deeper
How does decodePassRLE() work? 

The passRLE is treated as a bitstream. We need to understand what are the control bits. By the way writes should take turns - so you can't write two sequences of null-bytes or vice versa.

Let's see how null bytes are written (mad skillzz):

![When it happens](http://habrastorage.org/files/7dc/bcc/7bc/7dcbcc7bcda64e5085e5b1130cd0ad2e.png)
![How length is calculated](http://habrastorage.org/files/aa0/5fe/298/aa05fe2984e74bed9cf870e0b86e5851.png)

The format is following:

(_10_) initiates write of one null byte

(_11_) initiates write of more null bytes. The size depends on how many (_?1_) are met. (_?0_) ends the sequence:
```
size = 1;

size = size * 2 + (0|1)

size = size * 2 + (0|1)
```

For example:
```
Write 12 bytes:

  (11)(11)(01)(00)

  1->3->6->12

Write 2 bytes:

  (11)(00)

  1->2

Write 1 byte:

  (10)
```

Writing non null byte is done much easier: after writing zeroes decodePassRLE() picks up next 8 bits and writes them. 

Such encoding is called RLE (Run-length encoding).

We can construct correct passRLE (correctRLE) because we know email512ByteArray.

The last step is to construct password from correctRLE. Here is the pseudocode:

```python
alphabet = '0123456789ABCDEFGHJKMNPQRSTVWXYZ'

8bitBlocks <- correctRleBitstream
reversed8BitBlocks <- reverse(8bitBlocks)
bitstream <- reversed8BitBlocks
5bitBlocks <- bitstream
for 5bitBlock in 5bitBlocks:
  password.append(alphabet[int(5bitBlock)])
password = reverse(password)
```
####Why don't you attach the code?!
The code I wrote was very buggy - it damaged the last part of password so I had to calculate last two letters of password by hands :)
