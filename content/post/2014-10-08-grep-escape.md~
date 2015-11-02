---
layout: post
title: About Grep Escape
category: [tech]
tags: [shell]
---
What's the difference with `grep '$id\$'` and `grep \$id\$`?

The first will match string $id$, and the second will match the string ends with $id.

It turns out that, there're two level of escape here.

The first is shell quote, as `\` is also special characters in shell,  
it will be interprated as an escapce character, and turns `\$` into $ *BEFORE* it is sent to grep,
so grep will actually get $id$ as input.  
To it hehaves as we intended, we should use double slash `\\`, shell will remove the first slash, and send the
second to grep.  
To simplify things, use single quote is better choise, `'$id$'` will tell shell that `$id$` should be treated
as raw string.

The second is escape of grep itself. $ is special only if it resides in the end of string. So the first $ will be interprated
literally, the second should be escaped. To make grep treat input as fixed string, -F option is a better choice.

All the above considerations sum up, we get the final command:

    grep -F '$id$'