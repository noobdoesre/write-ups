**I'm a noob trying to get better with re and pwn. I wrote this write-up just not to lose the knowledge I got and share it with u, folks \\|_|/**

I really liked this challenge because it teaches a very important approach of reverse engeneering: don't dig every instruction - sometimes it's better to see a big picture.

####What are we doing?
This is the first task from lk re challenge. There are 3 tasks in total - I solved only first and second challenge, though the last one is the most difficult and fascination one - to solve it you will have to reverse affect of crypto-algorithm. This challenge ended at December, 5th. Our task is to find correct key for our email. After you find it, you register email+key on lk site to get into table of participants. First 30 people who solve most challenges are supposed to be rewarded.

####Before we start
You can obtain the binary for this task at [lk site](http://www.kaspersky.ru/crackme/zero_nights).

The binary is packed with upx. After ``upx.exe -d crackme_zn14_01.exe`` we need to disable aslr for this binary, because otherwise it will use hardcoded offsets -> will access invalid memory. Through ``cff explorer`` open ``File Header -> Characteristics -> check Relocation info...``.

I attached already unpacked binary that should work on your system. Also attached .idb (Ida 6.5) which you should view while reading this write-up.

####Big Picture
Look at @doHugeCheck in attached .idb: there is alot of stuff done with email but we do not need to reverse it! All we should know is - email is transformed to some 512 byte array, which has standalone non-null bytes and sequences of null bytes. What really matters is what's happening with password: 

```python
password -> passToBin() -> passRLE -> decodePassRLE() -> pass512ByteArray
email -> blackBox() -> email512ByteArray
if pass512ByteArray == email512ByteArray:
  win()
else:
  loose()
```

####Let's go deeper
How does @decodePassRLE() work? 

The passRLE is treated as a bitstream. We need to understand what are the control bits. By the way, writes should take turns - so you can't write two sequences of null-bytes or vice versa.

Let's see how null bytes are written (mad skillzz):

![When it happens](http://habrastorage.org/files/7dc/bcc/7bc/7dcbcc7bcda64e5085e5b1130cd0ad2e.png)
![How length is calculated](http://habrastorage.org/files/aa0/5fe/298/aa05fe2984e74bed9cf870e0b86e5851.png)

Viewing at how length of null-byte-sequence is calculated, we can play with diffirent bitstreams - patch memory through debugger and feed it to @decodePassRLE(). After spending some time with notepad and a pen I came to the conclusion, that the format is following:

(_10_) initiates write of one null byte

(_11_) initiates write of more than one null bytes. The size depends on how many (_?1_) are met. (_?0_) ends the sequence:
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
This piece of code encodes any null-byte-sequence to rle. Launch it like ``python bla-bla.py %number of null bytes in sequence%``:
```python
import sys

binaryRle = []
tempList = []
zerosNumber = int(sys.argv[1])
if zerosNumber == 1:
    binaryRle.append(1)
    binaryRle.append(0)
else:
    binaryRle.append(1)
    binaryRle.append(1)
    while zerosNumber != 1:
        if zerosNumber % 2 == 0:
            tempList.append(0)
            zerosNumber /= 2
        else:
            tempList.append(1)
            zerosNumber -= 1
            zerosNumber /= 2
    tempList = tempList[::-1]
    for i in tempList:
        binaryRle.append(i)
        binaryRle.append(1)
    binaryRle[len(binaryRle) - 1] = 0

print(binaryRle)
```
```
python rle_test.py 12
[1, 1, 1, 1, 0, 1, 0, 0]
```
Writing non null byte is done much easier: after writing zeroes decodePassRLE() picks up next 8 bits and writes them. 

Such encoding is called [RLE](http://en.wikipedia.org/wiki/Run-length_encoding) (Run-length encoding).

We can construct correct passRLE (correctRLE) because we know email512ByteArray.

The last step is to construct password from correctRLE (it is the reverse action of @passToBin). Here is the pseudocode:

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

####Why don't you attach the full code?!
The code I wrote was very buggy - it damaged the last part of password so I had to calculate last two letters of password by hands :)
