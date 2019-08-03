---
layout:     post
title:      "Using the geometry shader to achieve model explosion effect"
subtitle:   ""
date:       2019-07-31 12:00:00
author:     "chenjd"
header-img: "img/2019-07-31/title.png"
tags:
    - Unity
    - Shader
---

>This post was first published at [Medium](https://medium.com/@chen_jd/using-the-geometry-shader-to-achieve-model-explosion-effect-cf6d5ec03020)



## 0x00 Description

This article will show you how to use the geometry shader to turn the Stanford Armadillo model 
into sand.

## 0x01 Implementation

What you may be more familiar with is the "Stanford Bunny".![](/img/2019-07-31/bunny.png)
They came from models used by Greg Turk and Marc Levoy at Stanford. But the Bunny is so cute, therefore the Armadillo, which looks like a monster, is a good choice. What's more it has more vertices, so using it for sanding is better.
![](/img/2019-07-31/Armadillo.png)
As you can see, it has 106,289 vertices and 212,574 polygons.
![](/img/2019-07-31/mesh.png)

Ok, now let's import the Stanford Armadillo obj file into Unity and you can see that this monster is already in our scene. Next we will use the geometry shader to achieve the desired explosion visual effect.
![](/img/2019-07-31/scene.png)

When I mentioned the geometry shader, I used it to generate more new vertices and polygons to achieve the desired effect, such as using it to generate grass on the GPU for [real-time rendering of real grass](https://github.com/chenjd/Realistic-Real-Time-Grass-Rendering-With-Unity).

But the geometry shader can not only generate new primitives, but it also reduces the output of vertices and polygons to achieve some interesting effects, such as the example in this small article, using the geometry shader to achieve the monster's explosion and sanding effect.

![](/img/2019-07-31/pipeline.png)
And what we have to do is very simple, that is to change the input triangle composed of 3 vertices into a point composed of only one vertex in the geometry shader.

    float3 tempPos = (IN[0].vertex + IN[1].vertex + IN[2].vertex) / 3;
    o.vertex = UnityObjectToClipPos(tempPos);

In this way, the mesh that makes up the monster is changed from triangles to points, and the number of vertices is also reduced. As for the monster itself, it becomes like this:
![](/img/2019-07-31/title.png)

But the model at this time is static, therefore there is no effect of explosion. So next we have to let the monster's model move over time.

And a kinematic formula that everyone knows can be used to achieve this effect:
![](/img/2019-07-31/formula.png)

The `S` is the latest position of the vertex, the values of `v0` and `a` can be passed to the shader as a uniform variable, the direction of motion can be along the normal direction of the triangle, and the value of `t` is the y component of `_Time` which is a Unity built-in variable.

Now, we have the variables we need:

    //速度的值加速度的值
    float _Speed;
    float _AccelerationValue;
    float _StartTime

    //法线
    float3 v1 = IN[1].vertex - IN[0].vertex;
    float3 v2 = IN[2].vertex - IN[0].vertex;
    float3 norm = normalize(cross(v1, v2));
    ...
    //时间
    float realTime = _Time.y - _StartTime;

Then we just bring them in the kinematics formula:

    tempPos += norm * (_Speed * realTime + .5 * _AccelerationValue * pow(realTime, 2));

The final result will look like this:

![](https://camo.githubusercontent.com/5876549794f4cc79aea5facebf23f40890ab207f/687474703a2f2f696d616765732e636e626c6f67732e636f6d2f636e626c6f67735f636f6d2f6d75726f6e677869616f706966752f3636323039332f6f5f3230313731323031313134343331313531323134333037313339395f736d616c6c2e676966)

Or you can check the [youtube link](https://www.youtube.com/watch?v=ZyRmmJk84FU)

And you can find the demo here:

[Explosion and sand effect](https://github.com/chenjd/Unity-Miscellaneous-Shaders)

check it out!

## 0x02 Conclusion

Ok, the above is my blog this week and I hope everyone can gain something! 







