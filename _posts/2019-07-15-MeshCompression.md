---
layout:     post
title:      "Mesh compression in Unity"
subtitle:   " \"Why is the memory of my game unchanged?\""
date:       2019-07-16 12:00:00
author:     "chenjd"
header-img: "img/post-mesh-compression.png"
tags:
    - Unity
---

>This post was first published at [Medium](https://medium.com/@chen_jd/mesh-compression-in-unity-why-is-the-memory-of-my-game-unchanged-76f5d4e8244e)


## 0x00 Preface

Recently, a developer friend asked me some questions about mesh compression in Unity. He found that after checking the Mesh Compression option on the Model Importer panel, there was no change in memory. In fact, it is a misunderstanding of the role of Mesh Compression after the Mesh Compression is turned on then the memory occupied by the Mesh is reduced.

I believe that many developers will have similar misunderstandings after seeing the name Mesh Compression. So this blog is about how to optimize Mesh in Unity to save memory, and why the Mesh Compression option is turned on, but it doesn't help memory.


## 0x01 Mesh compression in Unity

In Unity, there are three main ways to optimize the space overhead of mesh data.

They are, in the **Player Setting**:

- Vertex Compression

- Optimize Mesh Data

![](/img/2019-07-15/playersettings.jpg)

And in the **Model importer**:

- Mesh Compression

![](/img/2019-07-15/importer.jpg)

Among them, the implementation of **Vertex Compression** is to set the data format of the vertex channel to 16bit, so it can save runtime memory usage (float->half).

**Optimize Mesh Data** is mainly used to remove unwanted channel and exclude additional data. Because it has nothing to do with compression, this article does not discuss this option.

However, **Mesh Compression** uses a compression algorithm to compress the mesh data. As a result, the space occupied by the hard disk is reduced, but it is decompressed to the original precision data during the runtime.

The implementation of Vertex Compression has a greater impact on Runtime memory. And whether Vertex Compression can be performed depends on the import settings of the model and whether dynamic batching can be performed (in the build phase, it is mainly judged whether the number of vertices of the mesh meets the condition).

It can be summarized as follows:

1. Is it suitable for dynamic batching?

2. Is Read/Write Enabled turned on in Model Importer?

3. Is **Mesh Compression** turned on in the Model Importer?

The first point is whether the mesh is suitable for dynamic batching. This is not only related to whether dynamic batching is checked in the player setting, but also whether the number of vertices of the mesh exceeds 300. Of course, whether dynamic batching is feasible or not is mainly related to the number of vertex attributes, but for simplicity, during the build phase, the engine takes a common vertice with three vertex attributes, which is 300 vertices.

So if dynamic batching is enabled, the mesh will not be vertex-compressed when the number of vertices is less than 300.

Second, the Read/Write Enabled option also disables Vertex Compression. So in general, in order to save memory, it is better not to check this option, in addition to the inability to perform vertex compression, it will additionally retain a mesh in memory.

Again, it is also often overlooked that if **Mesh Compression** is turned on, it will override Vertex Compression and Optimize Mesh Data settings. Mesh Compression compresses the storage space of the mesh on the hard disk, but does not save memory during runtime.

## 0x02 Test

Below I will use a few small examples to visually check the memory usage of Mesh under various settings.

- Unity Version : 2018.2.1

- Platform : Windows Player
  
#### Case 1

In case 1, compression is not enabled (MeshCompression and vertex compression), and Read/Write Enabled is not enabled.

##### Model Importer:

Building_e:

MeshCompression Off

Read/Write Enabled Off

Rock_c:

MeshCompression Off

Read/Write Enabled Off

##### Player Setting:

Dynamic Batching Off

vertex compression:None

Optimize Mesh Data:Off

##### Result:

Building_e 0.5mb

Rock_c 3.3kb
![](/img/2019-07-15/case1.png)

> Note: If Dynamic Batching is enabled, the Rock_c model will have more memory usage and will reach 5.9kb even if Read/Write Enabled is not enabled. This is because we will keep a copy of its mesh data when the dynamic batch is turned on, which is the same as the effect of Read/Write Enabled.

#### Case 2

In case 2, I only enable **Vertex Compression**.

##### Model Importer:

Building_e:

MeshCompression Off

Read/Write Enabled Off

Rock_c:

MeshCompression Off

Read/Write Enabled Off

##### Player Setting:

Dynamic Batching Off

vertex compression:Everything

Optimize Mesh Data:Off

##### Result:

Building_e 379.8kb

Rock_c 2.6kb
![](/img/2019-07-15/case2.png)

The compression takes effect at this time.

#### Case 3

At this time, I enable **Vertex Compression** and **Dynamic Batching**.

##### Model Importer:

Building_e:

MeshCompression Off

Read/Write Enabled Off

Rock_c:

MeshCompression Off

Read/Write Enabled Off

##### Player Setting:

Dynamic Batching On

vertex compression:Everything

Optimize Mesh Data:Off

##### Result:

Building_e 379.8kb

Rock_c 5.9kb

![](/img/2019-07-15/case3.png)

As you can see, the memory usage of Rock_c has increased. This is because the number of vertices of Rock_c is only 40+, and the format of index buffer is 16bit. When building with dynamic batching turned on, it is considered to be in compliance with dynamic batch conditions, so the operation of vertex compression will not be performed at build time.

Therefore, when the dynamic batch is turned on, for some small models (vertex count<300) Vertex Compression are invalid. Also, such a model will remain in the memory accessible to the cpu for dynamic batching needs.

#### Case 4

At case4, I enable **Vertex Compression**, **Dynamic Batching** and **Read/Write Enabled**.

##### Model Importer:

Building_e:

MeshCompression Off

Read/Write Enabled On

Rock_c:

MeshCompression Off

Read/Write Enabled On

##### Player Setting:

Dynamic Batching On

vertex compression:Everything

Optimize Mesh Data:Off

##### Result:

Building_e 1.0mb

Rock_c 5.9kb

![](/img/2019-07-15/case4.png)

It shows that the memory of Building_e is not compressed at this time, but it is doubled due to Read/Write Enabled. Due to the reason mentioned before, the memory usage of Rock_c is doubled too.

Therefore, in order to ensure that vertex compression can be executed normally and reduce the memory usage of the mesh during runtime, do not enable Read/Write Enable.

#### Case 5

At case 5, I enable **Vertex Compression**, **Dynamic Batching** and **Mesh Compression**.

The most confusing thing I think is the **Mesh Compression** option. Literally, this option is turned on to compress the model. But actually turning this option on will only reduce the storage size of the mesh on the hard disk. The vertex's format at runtime is not changed, it is still Float. Therefore, it is impossible to reduce the memory at Runtime.
![](/img/post-mesh-compression.png)

##### Model Importer:

Building_e:

MeshCompression On

Read/Write Enabled Off

Rock_c:

MeshCompression On

Read/Write Enabled Off

##### Player Setting:

Dynamic Batching On

vertex compression:Everything

Optimize Mesh Data:Off

##### Result:

Building_e 0.5mb

Rock_c 5.9kb
![](/img/2019-07-15/case5.png)

As you can see, Vertex Compression has failed. After Mesh Compression is turned on, the value of memory returns to the value when there is no compression.

## 0x03 Conclusion

The reason why this result is obtained after checking **Mesh Compression** has been described a lot above.

Therefore, a small suggestion is that if you want to optimize the mesh memory overhead, do not enable **Mesh Compression** to avoid the failure of **Vertex Compression**.

> Note: The model used for the test came from Asset Store.
