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

[UPDATE 2 (a few days later): I FOUND IT](#update-2)

## update:

scratch the above solution, it's stupid, I hadn't reallized the real issue, sorting was sort of happening Twice...

in `C-Kermit 7.0.197` there's a [chunk of code](https://github.com/KermitProject/ckermit/blob/d0f8b1da7aea5bf3912e4289bfb23c988b0e7c60/ckuus5.c#L805-L847) which purport to generate a sorted list of speeds, but it [apparently](https://github.com/KermitProject/ckermit/blob/main/NOTES.TXT#L118-L137) wasnt working.

I have now ripped out the ~1997 code which was playing a little fast and loose with its allocations and was hard to follow in its "sorting". This has made no changes to existing assumptions about sortedness, and the code now relies solely on the `#ifndef NOSORTSPEEDS` block for sorting of the speed list. 

### bonus note:

Since stripping that I have _not_ found the speeds list to be unsorted. I am electing to keep the `NOSORTSPEEDS` block for the moment as I do not know if there could be cases where the list is not sorted.

I hope to take a closer look and check if the explicit sorting is needed. But for now the patch can be found [here](./kermit-patch)

## update 2

Lets get our environment set up in macos first. I'm running kermit from homebrew (my first mistake), it is installing version 9.0.302, the last stable release, from 2011... and it grabs that as a tarball from the kermit website:

```ruby
  url "https://www.kermitproject.org/ftp/kermit/archives/cku302.tar.gz"
  version "9.0.302"
  sha256 "0d5f2cd12bdab9401b4c836854ebbf241675051875557783c332a6a40dac0711"
```

id like to work in git, and there is a mirror on github so lets check if this exists over there:

[C-Kermit 9.0.302 2011/07/11 Unix, VMS](https://github.com/KermitProject/ckermit/commit/0fd869b4f726ad3042754bf3e45b1e68e38daba1)

this looks right, but we can do a quick check:
```sh
q3w3e3@Charlottes-MBP ckermit % git log --pretty=format:'%h' -n 1   
0fd869b
q3w3e3@Charlottes-MBP ckermit % ls -la                              
total 6224
drwxr-xr-x    4 q3w3e3  staff      128 Mar  3 23:53 .
drwxr-x---+ 114 q3w3e3  staff     3648 Mar  3 23:55 ..
drwxr-xr-x   14 q3w3e3  staff      448 Mar  3 23:53 .git
-rw-r--r--    1 q3w3e3  staff  3122219 Aug 21  2011 cku302.tar.gz
q3w3e3@Charlottes-MBP ckermit % tar xzf cku302.tar.gz                                               
q3w3e3@Charlottes-MBP ckermit % git diff
q3w3e3@Charlottes-MBP ckermit % 
```
(there might be an easier way to check but this works for meee)

but brew doesnt just do this... it turns out the formula also applies some patches

```rb  
  # Apply patch to fix build failure with glibc 2.28+
  # Apply patch to fix build failure on Sonoma (missing headers)
  # Will be fixed in next release: https://www.kermitproject.org/ckupdates.html
  patch :DATA
```

We can just yoink that patchset and apply it over the current state of the code:

```bash
q3w3e3@Charlottes-MBP ckermit % sed -e '1,/END/ d' /opt/homebrew/Library/Taps/homebrew/homebrew-core/Formula/c/c-kermit.rb | patch
patching file ckucmd.c
patching file ckcdeb.h
patching file ckcmai.c
patching file ckuusx.c
patching file ckwart.c
```

Where that regex just grabs me all the lines after the `__END__` in the ruby for the [c-kermit brew formula](https://github.com/Homebrew/homebrew-core/blob/0c08d2a9803fa70c55b62648442d9c94598a7fb1/Formula/c/c-kermit.rb)

Now we should be able to build kermit as brew distributes it, lets just sanity check that:

```
q3w3e3@Charlottes-MBP ckermit % ./wermit 
C-Kermit 9.0.302 OPEN SOURCE:, 20 Aug 2011, for macOS 14.6.1 (64-bit)
```
Cool its real.

Lets make sure the bug happens here too:

```
q3w3e3@Charlottes-MBP ckermit % ./wermit + ../1brc/1brc ../1brc/data/measurements.txt                            
wermit(42743,0x1fe398f40) malloc: Corruption of free object 0x1030040b0: msizes 5/11781 disagree
wermit(42743,0x1fe398f40) malloc: *** set a breakpoint in malloc_error_break to debug
zsh: abort      ./wermit + ../1brc/1brc ../1brc/data/measurements.txt
q3w3e3@Charlottes-MBP ckermit % 
```

Cool, definitely real.

Now we can rebuild with `-g -fsanitize=address`. Making sure to ALSO pass `-fsanitize=address` to the linker... I only wasted a small amount of time on builds failing with linker errors, ( promise)

```rust
q3w3e3@Charlottes-MBP ckermit % ./wermit + ../1brc/1brc ../1brc/data/measurements.txt                                             
=================================================================
==45765==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x000103a02e5f at pc 0x000100397728 bp 0x00016fa98920 sp 0x00016fa98918
WRITE of size 1 at 0x000103a02e5f thread T0
    #0 0x100397724 in zchko ckufio.c:2647
    #1 0x10043cbc8 in cmofi ckucmd.c:1580
    #2 0x10050fb1c in doopen ckuus6.c:9414
    #3 0x10058083c in docmd ckuusr.c:8775
    #4 0x1004ce4d0 in parser ckuus5.c:3099
    #5 0x1003663ec in docmdfile ckcmai.c:2485
    #6 0x1005cc7a4 in cc_execute ckusig.c
    #7 0x100368c04 in main ckcmai.c:3442
    #8 0x1961bf150  (<unknown module>)

0x000103a02e5f is located 1 bytes before 31-byte region [0x000103a02e60,0x000103a02e7f)
allocated by thread T0 here:
    #0 0x1016cb124 in wrap_malloc+0x94 (libclang_rt.asan_osx_dynamic.dylib:arm64e+0x53124)
    #1 0x100397028 in zchko ckufio.c:2561
    #2 0x10043cbc8 in cmofi ckucmd.c:1580
    #3 0x10050fb1c in doopen ckuus6.c:9414
    #4 0x10058083c in docmd ckuusr.c:8775
    #5 0x1004ce4d0 in parser ckuus5.c:3099
    #6 0x1003663ec in docmdfile ckcmai.c:2485
    #7 0x1005cc7a4 in cc_execute ckusig.c
    #8 0x100368c04 in main ckcmai.c:3442
    #9 0x1961bf150  (<unknown module>)
```

cool this should be fun, lets take a look at the malloc incase its clearly allocating too little space like the last one.

```c
/*
  zchkod is a global flag meaning we're checking not to see if the directory
  file is writeable, but if it's OK to create files IN the directory.
*/
    if (!zchkod && isdir(name)) {	/* Directories are not writeable */
	debug(F111,"zchko isdir",name,1);
	return(-1);
    }
    s = malloc(x+3);                    /* Must copy because we can't */
    if (!s) {                           /* write into our argument. */
        fprintf(stderr,"zchko: Malloc error 46\n");
        return(-1);
    }
    ckstrncpy(s,name,x+3);
```
time for some more `printf()` instead of "real" debuggers...

```
zchko: checking ../1brc/output/Oakham.txt
allocating 28 bytes (x+3) for copy of name
```
we have `"../1brc/output/Oakham.txt"`, which  is 25 chars, `s` gets allocated 28 bytes, and the ckstrncpy has a len set to 28 as well... this looks fine. Guess we have to look at `zchko()`, in particular line 2649 which is where this is falling over

```C
2636   if (itsadir && i > 0) {

....        ...

2648        } else {
2649            s[i++] = '.';                   /* Append "." to path. */
2650            s[i] = '\0';
2651        }
```

Why is this written like this... I mean, it should be fine... but thats kinda gross... lets check what `i` is, and check `itsadir`, as `../1brc/output/Oakham.txt` should not be a directory.

```
zchko: checking ../1brc/output/Oakham.txt
allocating 28 bytes for copy of name
itsadir=0 i=-1
```

what the fuck.

Well that explains why its _upset_ about this, `s[-1]` probably isnt where we should be shoving a `'.'`. but _why_ is `i=-1`... lets scroll up a bit

```C
#ifdef UNIX
#ifdef NOUUCP

...

fd = open(name,O_WRONLY,mode);	/* Must attempt to open it */
	debug(F111,"zchko open",name,fd); 
	if (fd > -1) {			/* to get a file descriptor */
	    if (isatty(fd))		/* for isatty() */
	      istty++;
	    debug(F111,"zchko isatty",name,istty);
	    fd = close(fd);
	    if (istty) {
		goto doaccess;
	    }
	} else {
	    debug(F101,"zchko open errno","",errno); 
	    x = -1;
	}
    }
#endif	/* NOUUCP */
#endif	/* UNIX */
    for (i = x; i > 0; i--) {           /* Strip filename from right. */
        if (ISDIRSEP(s[i-1])) {
            itsadir = 1;
            break;
        }
    }
```

So, I guess we are failing to open the file (and I have shoved a print in there to verify I'm hitting that else statement)... This is actually expected, as my kermit which is hitting this is intentionally trying to open files that don't exist. This should, per c-kermit docs and behaviour on other systems, create the file. But when I say "on other systems" I mean ones without "NOUUCP" set... macos is special. So we know what _is_ happening but do we know what _should be_ happening?

**IS:** file fails to open, `x` is set to `-1` for some reason (maybe we will come back to this), `i` is then set to `x` for a loop which works along the filename to work out if it is a directory, but this loop never executes as `i > 0` is `False`... so we move on... 

the next code that executes is at line `2636` which we saw above, where we drop into the `else` statement where our OOB access happens!

**SB:** file fails to open, we do nothing else except maybe log some errors, free some memory, and return something indicating failure

thankfully the code to do that exists down just a few lines:

```C
    if (x < 0)
      debug(F111,"zchko access failed:",s,errno);
    else
      debug(F111,"zchko access ok:",s,x);
    if (s) free(s);			/* Free temporary storage */

    return((x < 0) ? -1 : 0);           /* and return. */
```

This is where GOTO is our friend, we can just add a pair of lines...

One after `x=-1`, we will `GOTO fuck;` and one right before that `if (x < 0)` we can just add the label `fuck:`  or something more appropriate.

lets rebuild and re-test with that fixed:
```sh
?Write permission denied - ../1brc/output/Oakham.txt
```

well shit.

I'm guessing that this newly working `-1` return is breaking something elsewhere? 
lets take a look at where `zchko` was being called from... `cmofi ckucmd.c:1580` 

```C
if (strcmp(s,CTTNAM) && (zchko(s) < 0)) { /* OK to write to console */
    ...
    printf("?Write permission denied - %s\n",s);
    ...
```

that'll be it!

we know whats going on with `zchko`, and we know its returning `-1`, so that explains why we _are_ hitting this `if`, but not if we _should be_. So we can take a look at the first part of the statement `strcmp(s,CTTNAM)`, whats `CTTNAM`?

```C
#ifdef UNIX
#define CTTNAM "/dev/tty"
```

this really feels like we want to check if `s` is the same as `CTTNAM` and `zchko(s)` less than `0`... right now we can hit this codepath if `strcmp` returns `-1` and `zchko` returns `-1`... when the intent appears to be to only drop into this `if` statement if we are writing to the terminal device, as that will fail `zchko` but we can still shove stuff onto it.

let's make that quick change to `if ((strcmp(s,CTTNAM) == 0) && (zchko(s) < 0))` and try again:

```bash
q3w3e3@Charlottes-MBP ckermit % ./wermit + ../1brc/1brc ../1brc/data/measurements.txt
exit 0
```
bug squashed!!!

but, it turns out someone actually beat me to this... this bug is already fixed in much the same way in beta 10.0...

https://github.com/KermitProject/ckermit/blob/main/ckucmd.c#L1694-L1696
https://github.com/KermitProject/ckermit/blob/main/ckufio.c#L2746-L2748
https://github.com/KermitProject/ckermit/blob/main/ckufio.c#L2827

but... im on macos and using brew, and brew only allows stable releases, so ill take theis small set of changes and attempt to get them added to the patchset applied by the brew formula:

https://github.com/Homebrew/homebrew-core/pull/270495

Thanks for reading, sorry for rambling. I hope you have enjoyed baby's first real C debug adventures, I know I have.

Love you,<br>Charlotte
