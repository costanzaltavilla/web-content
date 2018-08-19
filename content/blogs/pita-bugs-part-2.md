---
title:       "PITA bugs part 2"
author:      "Z98"
date:        2013-06-15
aliases:     [ "pita-bugs-part-2", "node/607" ]
---

<p>Weclome to the second part of the series where I discuss bugs that were truly irritating to debug. In general, fixing a bug is usually simple. Finding the bug is where the actual work is involved.</p><p>As I mentioned before, I currently work as an intern at a company that does development work on a UEFI BIOS. BIOS development is difficult because we are often so close to the hardware that we often do not have the convenience of things like a debugger. Debuggers do exist, but setting them up is a bit more work than simply starting a program in debug mode in Visual Studio. Sometimes however even a debugger does not work and the problem could genuinely be hardware related. At that point, it is time to pull out the oscilliscope and look at the signal, assuming you can find exposed terminals. This particular issue was nowhere as bad, but it was bad enough to require going back and forth with debug statements to try and figure out what was going wrong.</p><p>Unlike HTCondor, the code I work with now is not under an open source license and so I cannot talk directly about the bug or the code. I can however talk broadly about the cause of the bug, which is an example of an array/pointer mixup. Simplistically speaking, there is an array of X elements that need to be written into a linked list that has variable sized data. The header for the linked list thus must indicate the size of the data as well, otherwise you might accidentally over read. What happened was for some reason, the ID that identified a block of data had somehow been corrupted, suggestioning that the entry before it had accidentally written over its allocated block of memory. The previous entry was represented as an array in the code, so it was entirely conceivable that an index out of bounds error had occurred when working on the array. After considerable amount of checking and doublechecking as well as a few code reviews and discussions with the original author of the code, we noticed something curious when printing out the distance between the elements of the array. Each array element was basically supposed to be an integer in size, but the distance between each element was at least triple that. Looking at the way the array was declared, it became obvious why this was happening.</p><p>The original code had been defined as something like this:</p><p><code>typedef { int x } buffer[3];<br />buffer *bufferArray;</code></p><p>The intention was to have a pointer that could access the three ints in buffer. What ended up happening was the typedef resulted in a variable type that was basically three ints packed together. A pointer to this type however was not a pointer to the three ints inside of the original array, but a pointer to the entire packed variable. People familiar with pointer operations will know that incrementing or decrementing a pointer or using indices on a pointer will cause the program to move along memory based on the size of the variable type. This time, the type ended up being three ints instead of just one. As such, write operations to something like bufferArray[1] or bufferArray[2] would be 12 or 24 bytes away from bufferArray[0]. As space had only be allocated for three ints in the above mentioned list, the additional writes would cross into the next element&#39;s header/data, resulting in the corruption I encountered. Fixing this can be done by either dropping the [3] in the buffer typedef or just using an array of ints. This was an example of trying to be overly clever with typedefs to make code more compact resulting in a buffer overflow. There are however even worse issues which I will cover in the future.</p>