---
layout: post
title: Unobfuscating C --- The Partisan 1-liner
---

Since 1984, a contest known as the [International Obfuscated C Code Contest](http://www.ioccc.org/index.html) (IOCCC) has been held over several years. Contestants aim to submit C programs with the most confusing, obscure, and of course, obfuscated code. Some examples of programs include a flight simulator, a program that that calculates pi by looking at an ASCII art circle in its own source code, and even computer hardware emulators. I decided to attempt the unobfuscation of a recent 2013 entry consisting of a one line program that can determine the party of a US president. What makes this program special is that it does this *without using any conditional statements or a lookup table*. Please note that I do not guarantee that my explanation and analysis of this program is correct, although I have tried to be as correct as possible. Feel free to contact me if you notice any errors.

Note: I recommend that you know the basics of the C programming language before moving on (this includes pointers). If you don't, "goto" the last paragraph of this article for my insights on unobfuscating this program. Also, this program only works properly on little-endian machines (x86) for reasons covered in the analysis, so I give my apologies for anyone trying this on a decade-old Mac (PowerPC), Raspberry Pi or most mobile devices (ARM).

This program ([cable1](http://www.ioccc.org/2013/cable1/hint.html)) is meant to take in three arguments: a name, a first party and second party. For example,

{% highlight bash %}
./cable1 obama republican democrat
{% endhighlight %}

 writes "democrat" to standard output, and 

{% highlight bash %}
./cable1 bush republican democrat
{% endhighlight %}

writes "republican" instead.

You can try with other types of names and parties as well. For example, running

{% highlight bash %}
./cable1 Gates Mac PC
{% endhighlight %}

outputs "PC". This program seems fairly smart for a one liner. However,

{% highlight bash %}
./cable1 Cook Mac PC
./cable1 Nadella Mac PC
{% endhighlight %}

output "PC" and "Mac" respectively. Are the CEOs of Apple and Microsoft secretly in love with their competition? Or is this program not as smart as it seems?

This is the entire source of the program:

{% highlight c %}
main(int riguing,char**acters){puts(1[acters-~!(*(int*)1[acters]%4796%275%riguing)]);}
{% endhighlight %}

First of all, let's add some much needed whitespace to this code:

{% highlight c %}
main(int riguing, char **acters) {
    puts(1[acters-~!(*(int*)1[acters]%4796%275%riguing)]);
}
{% endhighlight %}

That helps a bit, but it still isn't very readable. If you are a keen C programmer, you may know that arrays can be declared like 10[a] instead of a[10] (I got this information from [Stack Overflow](http://stackoverflow.com/questions/1995113/strangest-language-feature)). Thus, we can rewrite the second line as so:

{% highlight c %}
main(int riguing, char **acters) {
    puts((acters-~!(*(int*)acters[1]%4796%275%riguing))[1]);
}
{% endhighlight %}

The variable acters[1] is the first argument passed in (i.e. the name of a president). Now, \*(int\*)acters[1] has the highest precedence. This statement will result in an integer, which is then modified by a succession of remainder operations based on two magic divisors and the number of arguments passed in (i.e. riguing, which is 3 in this case). But there is a secret hidden in this statement that explains why this program is so "smart".

This statement results in an integer based on acters[1]. It just so happens that [weird things](http://stackoverflow.com/questions/17260527/casting-pointers-in-c) happen during this statement. Since integers on most systems are 4 bytes long, the type conversion to int ends up doing this calculation (where acters[1] == "obama"):

'o' + 'b'\*2^8 + 'a'\*2^16 + 'm'\*2^24

Using the ASCII decimal values of the letters and simplifying the powers of two, we get:

111 + 98\*256 + 97\*65536 + 109\*16777216 = 1835098735

The secret is that the ASCII values of the first four letters of the first argument determines whether the second or third argument is listed. This also explains why the program won't work properly on non-little-endian machines. For example, big-endian machines will do a different calculation:

'o'\*2^24 + 'b'\*2^16 + 'a'\*2^8 + 'm'

So why do names shorter than four letters long (e.g. fdr) still work in this program? I do not know for sure, but I presume that the magic divisors work properly regardless of what the arbitrary values of the missing letters are.

We will call the result of this RESULT. Note that these remainder operations make RESULT a non-zero integer for republicans and 0 for democrats.

{% highlight c %}
main(int riguing,char**acters) {
    puts((acters-~!(RESULT))[1]);
}
{% endhighlight %}

The statement acters-~!(RESULT) now has the highest precedence. RESULT is logically negated, then converted into its bitwise one's complement. We will call this result MODIFIED_RESULT. Due to the behaviour of RESULT, MODIFIED_RESULT appears to be -1 for republicans and -2 for democrats.

{% highlight c %}
main(int riguing,char**acters) {
    puts((acters-MODIFIED_RESULT)[1]);
}
{% endhighlight %}

Since acters is a char \*\* and MODIFIED_RESULT is an integer, if acters is set to the address 0x1000, then acters-MODIFIED_RESULT results in a char \*\* value of 1000-(MODIFIED_RESULT\*INT_SIZE). On a 64-bit machine, INT_SIZE is equal to 8, thus if MODIFIED_RESULT is -1 (republican) or -2 (democrat), the result of 1000-(MODIFIED_RESULT\*INT_SIZE) would be 1008 for republicans (1016 for democrats), and thus 0x1008 for republicans (0x1010 for democrats) in hexadecimal. Since acters-MODIFIED_RESULT is located either at 0x1008 or 0x1010, and due to the placement of arguments in acters being set serially in memory set 8 bytes apart (on a 64-bit machine) starting from 0x1000, (acters-MODIFIED_RESULT)[1] returns either the second argument at 0x1008 ("republican") or the first argument at 0x1016 ("democrat"). This is printed to the console via puts().

So how is this program smart enough to know which presidents belong to what parties? It appears that the creator of this program discovered a way of determining a value based on the value of the first argument and specific magic divisors. These divisors provide the correct value for any US president name. This value is then converted in a very non-intuitive way to assign the proper address to the "republican" or "democrat" arguments in memory.

An unobfuscated (but still unintuitive) version of this program is shown here:

{% highlight c %}
#include <stdio.h>

int main(int argc, char **argv) {
    enum {
        MAGIC_DIVISOR_1 = 4796,
        MAGIC_DIVISOR_2 = 275
    };
    /* 
     * If argv[1] is a republican, i != 0;
     * else if argv[1] is a democrat, i == 0
     */
    int i = *(int *)argv[1] % MAGIC_DIVISOR_1 % MAGIC_DIVISOR_2 % argc;
    /*
     * If i != 0, ~!i == -1 and set c sizeof(int) bytes greater than argv;
     * else if i == 0, ~!i == -2 and set c sizeof(int)*2 bytes greater than
     * argv
     */
    char **c = (argv - ~!i);
    /*
     * If c > argv by sizeof(int) bytes, c[1] returns the first argument
     * ("republican"); else if c > argv by sizeof(int)*2 bytes, c[1] returns
     * the second argument ("democrat")
     */
    puts(c[1]);
    return 0;
}
{% endhighlight %}

Solving this program may seem like a waste of time, but I found that it helped me to further learn and understand C's typing and memory pointer rules. There are many more IOCCC programs out there. Will you be the next person to unobfuscate them?