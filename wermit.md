# Baby's first C debug experience


## problem
```
kermit(47037,0x1fe398f40) malloc: Corruption of free object 0x133e096e0: msizes 2/11778 disagree
kermit(47037,0x1fe398f40) malloc: *** set a breakpoint in malloc_error_break to debug
```
F*ck

and I'm on macos 14.6 on apple silicon... i have no clue what I'm doing here... lets just hop into linux and throw valgrind at this.

```
$ valgrind kermit
==34728== Memcheck, a memory error detector
==34728== Copyright (C) 2002-2024, and GNU GPL'd, by Julian Seward et al.
==34728== Using Valgrind-3.24.0 and LibVEX; rerun with -h for copyright info
==34728== Command: kermit
==34728== 
==34728== Invalid write of size 1
==34728==    at 0x1638FE: ??? (in /usr/bin/kermit)
==34728==    by 0x2056B0: ??? (in /usr/bin/kermit)
==34728==    by 0x16066B: ??? (in /usr/bin/kermit)
==34728==    by 0x5132CA7: (below main) (libc_start_call_main.h:58)
==34728==  Address 0x542e424 is 0 bytes after a block of size 4 alloc'd
==34728==    at 0x4844818: malloc (vg_replace_malloc.c:446)
==34728==    by 0x20674E: ??? (in /usr/bin/kermit)
==34728==    by 0x16066B: ??? (in /usr/bin/kermit)
==34728==    by 0x5132CA7: (below main) (libc_start_call_main.h:58)
==34728== 
==34728== Invalid write of size 1
==34728==    at 0x1638C0: ??? (in /usr/bin/kermit)
==34728==    by 0x2056B0: ??? (in /usr/bin/kermit)
==34728==    by 0x16066B: ??? (in /usr/bin/kermit)
==34728==    by 0x5132CA7: (below main) (libc_start_call_main.h:58)
==34728==  Address 0x542e474 is 0 bytes after a block of size 4 alloc'd
==34728==    at 0x4844818: malloc (vg_replace_malloc.c:446)
==34728==    by 0x20674E: ??? (in /usr/bin/kermit)
==34728==    by 0x16066B: ??? (in /usr/bin/kermit)
==34728==    by 0x5132CA7: (below main) (libc_start_call_main.h:58)
...
```
thats uh, not great that looks like Problems infact (though maybe not quite inline with what macos `malloc` was  yelling about) lets start here.

I've got some addresses that mean nothing to me, in a binary I know nothing about, time for `CFLAGS = -g`, we only have to go down to line `6491` of the `makefile`... I'm already hating this... 

BUT, that gets us somewher!

```
==23285== Invalid write of size 1
==23285==    at 0x15C70E: ckstrncpy (ckclib.c:147)
==23285==    by 0x1F0CBE: cmdini (ckuus5.c:1322)
==23285==    by 0x15B5FB: main (ckcmai.c:3155)
==23285==  Address 0x4b82424 is 0 bytes after a block of size 4 alloc'd
==23285==    at 0x4844818: malloc (vg_replace_malloc.c:446)
==23285==    by 0x1F0BAA: cmdini (ckuus5.c:1242)
==23285==    by 0x15B5FB: main (ckcmai.c:3155)
...
```

debug information is useful!

so `cmdini()` is doing a malloc, and is later calling `ckstrncpy()` which does Bad Things to the allocated space... let's take a look at that malloc in `cmdini`, 

```C
...
        int j, k, m = 0, n;             /* Create sorted keyword table */
        char buf[16];
        char * p;
        if ((spdtab =
             (struct keytab *) malloc(sizeof(struct keytab) * ss[0]))) {
            for (i = 1; i <= ss[0]; i++) { /* ss[0] = number of elements */
                if (ss[i] < 1L) break;     /* Shouldn't happen */
                buf[0] = NUL;              /* Make string */
                sprintf(buf,"%ld",ss[i]);  /* SAFE */
                if (ss[i] == 8880L)
                  ckstrncpy(buf,"75/1200",sizeof(buf));
                if (ss[i] == 134L)
                  ckstrncat(buf,".5",16);
                n = strlen(buf);
                if ((n > 0) && (p = (char *)malloc(n+1))) {
...
```

and for my own sanity lets check what the size of this malloc is with a quick 'n' dirty `printf("buf = %s, n = %d\n", buf, n);` 

```C
buf = 50, n = 2
buf = 75, n = 2
buf = 110, n = 3
buf = 134.5, n = 5
buf = 150, n = 3
buf = 200, n = 3
buf = 300, n = 3
buf = 600, n = 3
buf = 1200, n = 4
buf = 1800, n = 4
buf = 2400, n = 4
buf = 4800, n = 4
buf = 9600, n = 4
buf = 19200, n = 5
buf = 38400, n = 5
buf = 57600, n = 5
buf = 115200, n = 6
buf = 230400, n = 6
buf = 460800, n = 6
buf = 921600, n = 6
buf = 1000000, n = 7
buf = 1152000, n = 7
buf = 1500000, n = 7
buf = 2000000, n = 7
buf = 2500000, n = 7
buf = 3000000, n = 7
buf = 3500000, n = 7
buf = 4000000, n = 7
```
cool those all look good, but it hasnt crashed here... it crashes shortly after this and with no more calls of that `printf`.

lets move along the trace, `ckstrncpy` is where shit is getting broken, so lets see what it does and  we are doing with it:

what it does:
```C
/*  C K S T R N C P Y */

/*
  Copies a NUL-terminated string into a buffer whose total length is given,
  ensuring that the result is NUL-terminated even if it has to be truncated.

  Call with:
    dest = pointer to destination buffer
    src  = pointer to source string
    len  = length of destination buffer (the actual length, not one less).

  Returns:
    int, The number of bytes copied, 0 or more.

  NOTE: This is NOT a replacement for strncpy():
   . strncpy() does not require its source string to be NUL-terminated.
   . strncpy() does not necessarily NUL-terminate its result.
   . strncpy() treats the length argument as the number of bytes to copy.
   . ckstrncpy() treats the length argument as the size of the dest buffer.
   . ckstrncpy() doesn't dump core if given NULL string pointers.
   . ckstrncpy() returns a number.

  Use ckstrncpy() when you want to:
   . Copy a NUL-terminated string into a buffer without overrun, truncating 
      it if necessary to fit in the buffer, and null-terminating it.
   . Get the length of the string back.

  Use strncpy() when you want to:
   . Copy a piece of a string.
*/
```

cool, I _probably_ dont have to worry about its implementation, but of note is that its args are sane, `(dest,src,len)`.

what we are doing with it:
```C
        for (i = 0; i < nspd; i++) {
            ckstrncpy(spdtab[i].kwd, tmp[i].kwd,n);
            spdtab[i].flgs = tmp[i].flgs;
            spdtab[i].kwval = tmp[i].kwval;
        }
```

this looks fine, though I wonder what this `tmp` thing is and what `nspd` is....

looking around nearby we find that this broader section of code is creating a table of possible serial speeds, explains why the context of `buf` was all common tty baud rates I guess.

<sup><sub> if you have guesses on whats broken keep them to yourself... or don't... I can't tell you what to do </sub></sup>

I scroll up a few lines

```C
/* Create new sorted array of structs */
```

oh... I uh... surely not... lets throw another `printf` at this... 

```C
        for (i = 0; i < nspd; i++) {     /* i = index to sorted speeds */
            for (k = 0; k < nspd; k++) { /* k = index to original list */
                if (!strcmp(spdtab[k].kwd, speeds[i])) {
                    ckstrncpy(tmp[i].kwd,spdtab[k].kwd,n);
                    tmp[i].flgs = spdtab[k].flgs;
                    tmp[i].kwval = spdtab[k].kwval;
                    break;
                }
            }
            printf("i = %d, k = %d, speed = %s\n", i, k, speeds[i]);
        }
```

we see the issue now right?

the output might make it more clear:

```C
i = 0, k = 22, speed = 50
i = 1, k = 25, speed = 75
i = 2, k = 1, speed = 110
i = 3, k = 5, speed = 134.5
i = 4, k = 6, speed = 150
i = 5, k = 10, speed = 200
i = 6, k = 15, speed = 300
i = 7, k = 24, speed = 600
i = 8, k = 4, speed = 1200
i = 9, k = 8, speed = 1800
i = 10, k = 13, speed = 2400
...
```

so we are taking the `k`th item from spdtab, putting it into the `i`th position of tmp. and then later when we hit the issues with `ckstrncpy` we are taking the ith position of tmp and putting it in the ith position of `spdtab`.

in the debug output above I stopped at `speed = 2400` as that is where this scheme breaks...

the malloc for the 13th (k) position, 2400, was 4 bytes (plus the null), but we go to put it back into the 10th position... which was allocated for "200" plus its null, we are one char too long!

theres a few ways to fix this, making them all the same size seems like a good way, this is reinforced by the fact that in the sorting function we `malloc` for `tmp` with:

```C
tmp[i].kwd = malloc(maxspeedlen+2);
```

where 
```C 
int maxspeedlen = 20;
```

but I cant just use that because that only happens if we are sorting the list, theres cases where `spdtab` is not re sorted, and stays out of order.

so! 

## solution 
#### (update: this was not a good solution, much of the section stands though)

do I:
- move that "maxspeedlen" out and alloc everything with it
- realloc `spdtab[i].kwd` as it gets sorted

lets go for number 2, that seems like a more sane (and more memory efficient?) option, and given it only happens once(?) at startup, I'm not even gonna consider performance... performance in kermit isnt real...

```
$ valgrind ./wermit 
==34671== Memcheck, a memory error detector
==34671== Copyright (C) 2002-2024, and GNU GPL'd, by Julian Seward et al.
==34671== Using Valgrind-3.24.0 and LibVEX; rerun with -h for copyright info
==34671== Command: ./wermit
==34671== 
==34671== Conditional jump or move depends on uninitialised value(s)
==34671==    at 0x49CFFE5: srandom_r (random_r.c:210)
==34671==    by 0x49CFBF1: srand (random.c:211)
==34671==    by 0x15B932: main (ckcmai.c:3352)
==34671== 
```

one bug squashed, many to go. 


```diff
diff --git a/ckuus5.c b/ckuus5.c
index fdd286b..909b552 100644
--- a/ckuus5.c
+++ b/ckuus5.c
@@ -1319,6 +1319,7 @@ cmdini() {
             }
         }
         for (i = 0; i < nspd; i++) {
+            spdtab[i].kwd = realloc(spdtab[i].kwd, strlen(tmp[i].kwd) + 1); /* realloc to avoid *issues* */
             ckstrncpy(spdtab[i].kwd, tmp[i].kwd,n);
             spdtab[i].flgs = tmp[i].flgs;
             spdtab[i].kwval = tmp[i].kwval;
```

shrimple as.

or... it would be if this was in any way related to that macos `malloc` problem... which this wasnt... in fact this bug doesnt even exist in the version of kermit I'm running on macos... it was introduced in `C-Kermit 10.0 Beta.06` [here](https://github.com/KermitProject/ckermit/blame/main/ckuus5.c#L1268)

also I have zero clue if my realloc is portable across even a tiny subset of the shit kermit builds for, you may notice in that Blame that theres some code and comments about issues on vms, and with non-constant array dimensions being a C99 feature...

future me problem, or someone elses problem I guess.

## update:

scratch the above solution, it's stupid, I hadn't reallized the real issue, sorting was sort of happening Twice...

in `C-Kermit 7.0.197` there's a [chunk of code](https://github.com/KermitProject/ckermit/blob/d0f8b1da7aea5bf3912e4289bfb23c988b0e7c60/ckuus5.c#L805-L847) which purport to generate a sorted list of speeds, but it [apparently](https://github.com/KermitProject/ckermit/blob/main/NOTES.TXT#L118-L137) wasnt working.

I have now ripped out the ~1997 code which was playing a little fast and loose with its allocations and was hard to follow in its "sorting". This has made no changes to existing assumptions about sortedness, and the code now relies solely on the `#ifndef NOSORTSPEEDS` block for sorting of the speed list. 

### bonus note:

Since stripping that I have _not_ found the speeds list to be unsorted. I am electing to keep the `NOSORTSPEEDS` block for the moment as I do not know if there could be cases where the list is not sorted.

I hope to take a closer look and check if the explicit sorting is needed. But for now the patch can be found [here](./kermit-patch)
