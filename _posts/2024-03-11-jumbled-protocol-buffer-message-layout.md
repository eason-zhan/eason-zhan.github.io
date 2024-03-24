---
layout: post
title: Jumbled Protocol Buffer Message Layout
date: 2024-03-10 10:37 +0800
categories: [Programming, C++]
tags: [Protocol Buffer, performance]
image:
  path: /assets/img/decoys/brick_layout.jpeg
  alt: Brick Layout
---
### TL; DR;

When using `protoc` (the protocol buffer compiler) to compile a protobuf message into a C++ class, field reordering is taken based on the [`OptimizeLayoutHelper::Family`](https://github.com/protocolbuffers/protobuf/blob/e41ffb01a25632a6a207cb62fad913c2e35e316c/src/google/protobuf/compiler/cpp/padding_optimizer.cc#L66-L84). In some circumstances, this reordering ruins unintentionally your well-designed struct cache line alignment and impedes your program performance.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fprotocolbuffers%2Fprotobuf%2Fblob%2Fe41ffb01a25632a6a207cb62fad913c2e35e316c%2Fsrc%2Fgoogle%2Fprotobuf%2Fcompiler%2Fcpp%2Fpadding_optimizer.cc%23L66-L84&style=github&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

### Optimizing in the wrong way.

An online advertising system had a `Stats` message with a timestamp and some statistics data fields: 

```protobuf
message Stats {
    int64 ts                   = 1;
    int64 show                 = 2;
    int64 click                = 3;
    int64 cost                 = 4;
    int64 hour_show            = 5;
    int64 hour_click           = 6;
    int64 hour_cost            = 7;
    int64 acc_show             = 8;
    int64 acc_click            = 9;
    int64 acc_cost             = 10;
    repeated int64 bucket      = 11;
    repeated int64 hour_bucket = 12;
    repeated int64 acc_bucket  = 13;
}
```
One day, I need to optimize this message and delete its hour statistics data `hour_*` fields. I did expect that these deletions would be a memory and CPU save operation. However, the server has a CPU hotspot for accessing the stats message.

```protobuf
// After remove the `hour_*` fields
message StatsOpt {
    int64 ts                   = 1;
    int64 show                 = 2;
    int64 click                = 3;
    int64 cost                 = 4;
    int64 acc_show             = 5;
    int64 acc_click            = 6;
    int64 acc_cost             = 7;
    repeated int64 bucket      = 8;
    repeated int64 acc_bucket  = 9;
}
```
After hours of analysis, I found the culprit of this deterioration is the `OptimizeLayoutHelper::Family` reordering. According to the `OptimizeLayoutHelper::Family` layout ordering, we can draw the 64-byte cache line layout of these two message-compiled objects.

The object layout of message `Stats`:

```
+------ 16 BYTE ------+- 8 BYTE -+------ 16 BYTE ------+- 8 BYTE -+------ 16 BYTE ------+
|    (11)bucket       | (11)size |      (12)hour       | (12)size |  (13)acc_bucket     |
+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+
| (13.a)   |   (1)ts  |  (2)show | (3)click |  (4)cost |    (5)   |    (6)   | (7)      |
+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+
|    (8)   |    (9)   |   (10)   |     *    |     *    |     *    |     *    |     *    |
+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+
```

And, the object layout of message `StatsOpt`:

```
+------ 16 BYTE ------+- 8 BYTE -+------ 16 BYTE ------+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+
|    (8)bucket        | (8)size  |      (9)hour        | (9)size  | (13.a)   |   (1)ts  |
+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+
|  (2)show | (3)click |  (4)cost |    (5)   |    (6)   | (7)      |     *    |     *    |
+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+- 8 BYTE -+
```

`protoc` using two class members of a total of 24 Bytes to represent a repeated field, 16 Bytes for the `RepeatedField<T> f_` part, and 8 Bytes for the `atomic<int> _size` part.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Feason-zhan%2Fdsy%2Fblob%2Fmain%2Fdemos%2F03_protobuf_layout%2Fproto%2Fstats.pb.h%23L635-L649&style=github&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

As the ad server always accesses the stats `ts` and `cost` fields sequentially, for the `Stats` generated objects, the `ts` and `cost` are packed in the same cache line, however in the `StatsOpt` generated object, the `ts` and `cost` are packed in the different cache line. This is the wrongdoer who breaks the spatial locality.

### Summary

Using protocol buffer as a server communication protocol is a good idea, it is competent at serialization, deserialization, and initialization, but its layout is not optimal when using its generated objects in a high-performance server, it deserves some acumen to design your messages to avoid spatial locality break.
