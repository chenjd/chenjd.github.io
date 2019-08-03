---
layout:     post
title:      "Recommend a very interesting website to .Net/Unity programmers: sharplab.io"
subtitle:   ""
date:       2019-08-03 12:00:00
author:     "chenjd"
header-img: "img/2019-08-03/title.png"
tags:
    - Unity
    - .NET
---

>This post was first published at [Medium](https://medium.com/@chen_jd/using-the-geometry-shader-to-achieve-model-explosion-effect-cf6d5ec03020)



## 0x00 Description & Implementation & Conclusion

I found a very interesting website today:

[SharpLab](https://sharplab.io/)

The pages of the website are not complicated, and the functions can be summarized in the title image. But I found this website by chance. It was only until today that I found it on Twitter, which was sent by my colleague who was 'spoiled' by this website that could be quick online-try-it-out.

![](/img/2019-08-03/twitter.png)

Simply put, this site can display the intermediate process and results of .Net code (such as c#) compilation.

Because .Net has many different implementations, the site offers a variety of different versions.

![](/img/2019-08-03/1.png)

As for decompilation, you can view the  `IL` code compiled from source code, or view the `"source code"` that is decompiled back or even `JIT Asm`. You can also view the `Syntax Tree` in the compilation. In the words of the author, "SharpLab allows you to see the code as compiler sees it, and get a better understanding of .NET languages."
![](/img/2019-08-03/2.png)
![](/img/2019-08-03/3.png)
And you can also switch between Debug/Release configurations.

Of course, in addition to viewing the compilation process and results, the website also supports viewing the result of the code. Since it provides a quick online-try-it-out approach, it is convenient to see C# compilation process and results.

And most importantly, the site itself is open source. And hosted on Github.

[ashmind/SharpLab](https://github.com/ashmind/SharpLab)

Welcome to star this project.