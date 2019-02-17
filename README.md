# 35c3-arraymaster2-writeup

## Usage
Replace the libc I provide with the one of your system and run: `./exploit.py`

## Writeup

The bug is an integer overflow in the allocation of an int64 array. This gets you an OOB read/write.

With each array allocation two chunks get allocated on the heap. The first one contains the metadata
of the array (e.g. pointers, size...) and the second one contains the actual values stored in the array.
This is the layout of my arrays on the heap:
```
            chunk1 of array 1
0x00:      HEAP METADATA         <- Data used by the heap algorithm
0x08:      2305843009213693953   <- this is the size field of my first array
0x10:      0x40                  <- this is the type field 0x40 = 64 bc it's a int64 array
0x18:  +---index0 pointer        <- this points to the first index of ou
0x20:  |   int64_get pointer     <- this gets called whenever you call get on the array
0x28:  |   int64_set pointer     <- this gets called whenever you call set on the array
       |         chunk2 of array 1
0x30:  |   HEAP METADATA         <- Data used by the heap algorithm
0x38:  +-->0x0000000000000000    <- Index0
0x40:      0x0000000000000000    <- Index1
...
            chunk1 of array 2
HEAP METADATA         <- Data used by the heap algorithm
1                     <- this is the size field of my first array
0x40                  <- this is the type field 0x40 = 64 bc it's a int64 array
index0 pointer        <- this points to the first index of ou
int64_get pointer     <- this gets called whenever you call get on the array
int64_set pointer     <- this gets called whenever you call set on the array
            chunk2 of array 2
HEAP METADATA         <- Data used by the heap algorithm
0x4141414141414141    <- Index0
            chunk1 of array 3
HEAP METADATA         <- Data used by the heap algorithm
1                     <- this is the size field of my first array
0x40                  <- this is the type field 0x40 = 64 bc it's a int64 array
index0 pointer        <- this points to the first index of ou
int64_get pointer     <- this gets called whenever you call get on the array
int64_set pointer     <- this gets called whenever you call set on the array
            chunk2 of array 3
HEAP METADATA         <- Data used by the heap algorithm
0x4242424242424242    <- Index0
```
The integer overflow allows us to read and write data after our array which means that if we allocate a second array after 
the one with the out-of-bounds access, we can read and write all of the fields of the second one using `get <first array ID> <offset to element of the second array>` and `get <first array ID> <offset to element of the second array> data`. 
There are a few things we can do with this. First of all we can leak the PIE offset by simply reading either the 
`int64_get` or the `int64_set` pointer of the second array and subtracting the offset of either one of these functions in the binary. 

But whats even more interesting, is that we can build an arbitrary write/read primitive using the OOB read/write.
Every Array has an `Index0` pointer, which tells the `get` and `set` functions the address of the first actual element
of the array so that they know where to start reading from or writing to. If we use the OOB read/write primitive to manipulate 
the `Index0` pointer of another array to point to some address we want to read/write and then call `get/set` on the manipulated array, we are able to write and read at arbitrary addresses!


This is how the array needs too look like to get an arbitrary read/write:
```
chunk1 of array 1
0x00:      HEAP METADATA         
0x08:      2305843009213693953   
0x10:      0x40                  
0x18:  +---index0 pointer        
0x20:  |   int64_get pointer     
0x28:  |   int64_set pointer----+     
|         chunk2 of array1      |
0x30:  |   HEAP METADATA        | use the int64_set of the first array to write some address 
0x38:  +-->0x0000000000000000   | into the index0 pointer of the second array
            chunk1 of array 2   | 
0x00      HEAP METADATA         |
0x08      1                     |
0x10      0x40                  |
0x18  +---index0 pointer <------+   <- This needs to be set to the address we want to read from/write to
0x20  |   int64_get pointer  -\
      |                         > just use these functions to read/write 8 byte chunks
0x28  |   int64_set pointer  -/
      |           chunk2 of array 2
0x30  |   HEAP METADATA
0x38  |   0x4141414141414141    <- The actual content is irrelevant since the index0 pointer points somewhere else
     ...
 ...  |
      +-->   Some memory we want to read from/write to
```

Now that we've constructed our read/write primitive, there are multiple ways to get a shell. The simplest is to first use 
the read to leak the address of `puts@got` in order for us to calculate the libc base. With that we are able to overwrite 
the `int64_get` or `int64_set` pointers with the address of a oneshot gadget and then just call get/set. Another possibility 
would be to overwrite the `int64_get` pointer with `system()` and size field with `/bin/sh\x00`. 
