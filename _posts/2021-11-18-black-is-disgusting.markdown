---
layout: post
title: Why I don't like the black code formatter
date: 2021-11-18 12:06:02 +0200
categories: jekyll update
---

First off I understand the need for a tool to avoid teammates bickering with each other, and if I joined a team using black I would follow their rules.

However:

1. ' is cleaner than "

To me, ' has less visual noise than ". It's a single stroke rather than two. Additionally, on US keyboards ' does not require pressing shift, " does.

2. Black loves newlines, this leads to too much whitespace and makes skimming code harder

Take a look at this image.

<img src="{{site.url}}/images/blackbloat_args.png" style="display: block; margin: auto;" />

This could take half the space, and then I could view twice as many arguments in one go.

Black favoring to break functions with arguments up like this
```
function(
    argument
)
```

This generally leads to one seeing much less code on one's screen than originally. This worsens readability. I can see the argument for doing this
for a function with ten arguments, but one? Nah.

I might think of more issues this is it for now.
