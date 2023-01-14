---
layout: single
title: thrift compact protocol
date: 2022-05-28 00:00:00 +0800
categories: 学习
tags: C++ thrift fbthrift
---

我们代码里之前升级了fbthrift，客户端使用的channel类型从`HeaderClientChannel`换成了`RocketClientChannel`，默认是使用`CompactProtocol`，然后通过tcpdump抓包看了眼RPC层发送的数据，发现已经完全看不懂了，正好借此机会梳理一下。

## Thrift compact protocol encoding

我们先简单了解下thrift的编码格式，在thrift中主要使用的类型是`BinaryProtocol`和`CompactProtocol`，我们这篇中主要介绍`CompactProtocol`。需要说明的是在fbthrift中已经不是完全兼容原始的apache thrift的编码格式了，做了一些扩展。

- Integer

先zigzag变换，再转换为varint (UNSIGNED LEB128) 我们简单说下编码的流程，假设待编码的值为`3600 * 24 * 1000 = 86400000`，也就是一天有多少毫秒。

1. zigzag转换

所谓zigzag转换主要是为了让正数和负数的有符号数都映射到映射到无符号数空间，之所以这么做主要是为了避免后面varint对于负数编码效率过低的问题，比如想要将int32类型的-1经过varint编码结果是`ff ff ff ff 0f`，这对于经常使用的-1这种数而言比较浪费。所以zigzag转换做的事情就是将有符号数`0, 1, -1, 2, -2...`依次映射为无符号数`0, 1, 2, 3, 4...`。这样在后续做varint编码时能够有效降低编码出来的字节数。

int32的zipzag转换代码如下，86400000经过转换得到172800000

```cpp
constexpr inline uint32_t i32ToZigzag(const int32_t n) {
  return (static_cast<uint32_t>(n) << 1) ^ static_cast<uint32_t>(n >> 31);
}

int32_t zigzagToI32(uint32_t n) {
  return (n & 1) ? ~(n >> 1) : (n >> 1);
}
```

2. 然后把zigzag转换的结果进行varint编码

varint主要是为了减少整数占用的字节数（我们通常使用的数都比较小，没有必要使用定长编码），var会把int32_t编码为1-5个字节，int64_t会编码为1-10个字节。varint有个性质是如果当前字节不是这个数的最后一个字节，那么当前字节的最高位为1，如果当前字节是这个数的最后一个字节，那么最高位为0。编码大致流程如下

1. 转换为二进制形式的大端的无符号数
2. 分成7个bits一组，最高位的一组不组7位的补0
3. 然后在每组7bit左侧加0或者1 （第一组加0 其余都加1 为了满足varint性质）
4. 进行大小端转换（也是为了满足varint性质），此时所得结果就是varint编码

我们用50399为例，编码如下

```
50399 = 1100010011011111            (二进制大端)
      =       11  0001001  1011111  (分成7bits一组)
      =  0000011  0001001  1011111  (第一组不够7个bits 补0)
      = 00000011 10001001 11011111  (第一组最前面补0 其余补1)
      = 11011111 10001001 00000011  (大小端转换)
      = 0xDF 0x89 0x03

```

同理，如果要对第一步zigzag转换得到的结果172800000进行varint编码过程如下：

```
172800000 = 1010010011001011100000000000
          =  1010010  0110010  1110000  0000000 (分成7bits一组 已经是7的倍数 不需要补0了)
          = 01010010 10110010 11110000 10000000 (第一组最前面补0 其余补1)
          = 10000000 11110000 10110010 01010010 (大小端转换)
          = 0x80 0xF0 0xB2 0x52                 (最终结果)
```

> 具体可以参照[LEB128 - Wikipedia](https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128)

- Enum

按Int32编码

- Binary

```text
Binary protocol, binary data, 1+ bytes:
+--------+...+--------+--------+...+--------+
| byte length         | bytes               |
+--------+...+--------+--------+...+--------+
```

`byte length`是实际binary的长度，按varint编码，`bytes`是实际数据

- String

先转换为UTF8，然后按Binary编码，不包含Null终止符（所以我们代码中是不推荐使用string的 转换为UTF8没必要）

- Double

Values of type double are first converted to an int64 according to the IEEE 754 floating-point "double format" bit layout.

> 摘自[thrift/thrift-compact-protocol.md at master · apache/thrift (github.com)](https://github.com/apache/thrift/blob/master/doc/specs/thrift-compact-protocol.md)，存疑的是为什么要先转换为int64，再按浮点数编码

另外fbthrift中额外支持了Float

- Message

> 对于Message，fbthrift代码中的编码和thrift有所不同，下面显示的仍是thrfit的格式

```text
Compact protocol Message (4+ bytes):
+--------+--------+--------+...+--------+--------+...+--------+--------+...+--------+
|pppppppp|mmmvvvvv| seq id              | name length         | name                |
+--------+--------+--------+...+--------+--------+...+--------+--------+...+--------+

Where:
- `pppppppp` is the protocol id, fixed to `1000 0010`, 0x82.
- `mmm` is the message type, an unsigned 3 bit integer.
- `vvvvv` is the version, an unsigned 5 bit integer, fixed to `00001`.
- `seq id` is the sequence id, a signed 32 bit integer encoded as a var int.
- `name length` is the byte length of the name field, a signed 32 bit integer encoded as a var int (must be >= 0).
- `name` is the method name to invoke, a UTF-8 encoded string.

Message types are encoded with the following values:
- Call: 1
- Reply: 2
- Exception: 3
- Oneway: 4
```

- Struct

结构体中包含0个或多个字段，结构体结束时会编码一个终止符(stop-field, 编码为0x00)，每个字段使用其对应的编码格式。另外union和exception的编码方式和struct相同。结构体的BNF如下:

```
struct        ::= ( field-header field-value )* stop-field
field-header  ::= field-type field-id
```

定义在IDL中的struct的每个字段都有FieldId，这个id会被编码进数据中，但每个字段名不会。所以只要在修改thrift结构时，每个字段的名字其实是可以重命名，只要不修改id就不会影响前向后向兼容。另外只要不是required的字段，是不需要编码进数据中的，所以如果对于下面的struct，如果一个字段都不设置值，在最终编码时候就只有一个终止符。

```text
struct TestStruct {
    1: i64 xxx,
    2: binary yyy,
    3: optional list<bool> zzz,
}
```

struct编码格式如下

```text
Compact protocol field header (short form) and field value:
+--------+--------+...+--------+
|ddddtttt| field value         |
+--------+--------+...+--------+

Compact protocol field header (1 to 3 bytes, long form) and field value:
+--------+--------+...+--------+--------+...+--------+
|0000tttt| field id            | field value         |
+--------+--------+...+--------+--------+...+--------+

Compact protocol stop field:
+--------+
|00000000|
+--------+

- `dddd` is the field id delta, an unsigned 4 bits integer, strictly positive.
- `tttt` is field-type id, an unsigned 4 bit integer.
- `field id` the field id, a signed 16 bit integer encoded as zigzag int.
- `field-value` the encoded field value.
```

这里可以看到`CompactProtocol`里面对于每个字段编码时候可能有两个方式，主要原因在于编码时候不要求每个字段按照FieldId的顺序（当然生成的中间代码可以按FieldId顺序进行编码，但不是必须的），所以为了支持乱序编码各个字段，引入了两个格式。如果当前`编码的字段FieldId - 上一个编码的FieldId < 16`，就可以使用short form，所以在short form中对于FieldId是使用了delta encoding，而如果不满足这个条件的时候，就可以使用long form。

说完了FieldId，FieldType就比较简单了

```c++
// BOOLEAN_TRUE, encoded as 1
// BOOLEAN_FALSE, encoded as 2
// I8, encoded as 3
// I16, encoded as 4
// I32, encoded as 5
// I64, encoded as 6
// DOUBLE, encoded as 7
// BINARY, used for binary and string fields, encoded as 8
// LIST, encoded as 9
// SET, encoded as 10
// MAP, encoded as 11
// STRUCT, used for both structs and union fields, encoded as 12
// FLOAT, encoded as 13 (只有fbthrift支持)
```

至于字段具体值，就会使用对应类型的编码格式。

- List and Set

```text
Compact protocol list header (1 byte, short form) and elements:
+--------+--------+...+--------+
|sssstttt| elements            |
+--------+--------+...+--------+

Compact protocol list header (2+ bytes, long form) and elements:
+--------+--------+...+--------+--------+...+--------+
|1111tttt| size                | elements            |
+--------+--------+...+--------+--------+...+--------+

- `ssss` is the size, 4 bit unsigned int, values `0` - `14` (只能用在[0-14]大小 15被long form使用了)
- `tttt` is the element-type, a 4 bit unsigned int (类型和上面FieldType基本一样 除了BOOL统一按2)
- `size` is the size, a var int (int32), positive values `15` or higher
- `elements` are the encoded elements
```

- Map

```text
Compact protocol map header (1 byte, empty map):
+--------+
|00000000|
+--------+

Compact protocol map header (2+ bytes, non empty map) and key value pairs:
+--------+...+--------+--------+--------+...+--------+
| size                |kkkkvvvv| key value pairs     |
+--------+...+--------+--------+--------+...+--------+

- `size` is the size, a var int (int32), strictly positive values
- `kkkk` is the key element-type, a 4 bit unsigned int
- `vvvv` is the value element-type, a 4 bit unsigned int
- `key value pairs` are the encoded keys and values
```

## Show me the code

先看一个fbthrift的UT中的例子(不是实际抓包的，但可以通过这个例子了解其编码细节)，具体可参见RocketClientChannelTest，我们简化下其中的thrift接口，用如下的接口做个简单测试(忽略糟糕的命名，这个sendResponse接口实际就是个echo)

```thrift
service TestService {
  string sendResponse(1: string str);
}
```

使用如下代码从客户端发送一个RPC请求

```cpp
TEST_F(RocketClientChannelTest, SyncThread) {
  folly::EventBase evb;
  auto client = makeClient(evb);

  std::string response;
  client.sync_sendResponse(response, "doodle");
  EXPECT_EQ("doodle", response);
}
```

在UT的server端解析出来的request如下所示

```text
00000000  00 00 65 00 00 00 00 05  00 00 01 00 00 7f ff ff  |..e.............|
00000010  ff 7f ff ff ff 0a 74 65  78 74 2f 70 6c 61 69 6e  |......text/plain|
00000020  0a 74 65 78 74 2f 70 6c  61 69 6e 00 00 3a f0 9f  |.text/plain..:..|
00000030  9a 80 35 0c 15 10 5c 18  17 52 6f 63 6b 65 74 43  |..5...\..RocketC|
00000040  6c 69 65 6e 74 43 68 61  6e 6e 65 6c 2e 63 70 70  |lientChannel.cpp|
00000050  18 14 76 65 73 6f 66 74  2d 31 39 32 2d 31 36 38  |..vesoft-192-168|
00000060  2d 38 2d 32 31 31 00 00  00 00 2a 00 00 00 01 11  |-8-211....*.....|
00000070  00 00 00 18 15 04 18 0c  73 65 6e 64 52 65 73 70  |........sendResp|
00000080  6f 6e 73 65 15 00 25 80  f0 b2 52 00 18 06 64 6f  |onse..%...R...do|
00000090  6f 64 6c 65 00 00 00 00  00 00 00 00 00 00 00 00  |odle............|
000000a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
```

不难发现这里面包含了类似HTTP Header的东西，这其中包含了一个`SetupFrame`和RPC的payload，从0x73字节开始才是真正的RPC请求。

这个包至少有两部分组成`SetupFrame`和RPC请求，我们先忽略前面的`SetupFrame`，在实际抓包中没有抓到跟`SetupFrame`相关的TCP包，应该是在client和server第一次请求时才需要进行相应的初始化，有空后续再研究下。我们把RPC请求部分单独提出来进行详细分析(从0x73字节的18开始)

```text
00000070  00 00 00 18 15 04 18 0c  73 65 6e 64 52 65 73 70  |........sendResp|
00000080  6f 6e 73 65 15 00 25 80  f0 b2 52 00 18 06 64 6f  |onse..%...R...do|
00000090  6f 64 6c 65 00 00 00 00  00 00 00 00 00 00 00 00  |odle............|
```

RPC的请求也是由两部分组成，RequestRpcMetadata和RPC接口中的参数，在客户端发送请求的代码中我们可以看到这两部分是如何编码的。

### RequestRpcMetadata

填写`RequestRpcMetadata`代码路径如下

```
RequestChannel::clientSendT
	-> RequestChannel::sendRequestAsync
		-> RocketClientChannel::sendRequestResponse
			-> RocketClientChannel::sendThriftRequest
				-> apache::thrift::detail::makeRequestRpcMetadata (准备Metadata)
				-> RocketClientChannel::sendSingleRequestSingleResponse (准备rpc request)
					-> apache::thrift::rocket::pack (代码在thrift/lib/cpp2/transport/rocket/PayloadUtils.h)
						-> apache::thrift::rocket::makePayload (实际打包)
							-> RequestRpcMetadata::write(CompactProtocolWriter*) (最终调用ProtocolWriter写入Metadata)
```

对于上面的例子中Metadata字段如下所示，这部分代码都是通过IDL层生成的，所以各个接口的MetaData可能不同。需要注意的是，只有最后一列isSet为true时，对应的字段才会进行编码，这和实际RPC参数的处理是一样的。由于生成的中间代码的编码次序是按照Field依次写入（再次强调，依次写入不是必须的，参考上面对Struct编码的说明），所有字段都是有就设置，没有就不设置。

| 字段 | 作用 | 类型 | isSet |
| --- | --- | --- | --- |
| __fbthrift_field_protocol | 编码协议，RocketClientChannel默认使用apache::thrift::ProtocolId::COMPACT | int32 | Y |
| __fbthrift_field_name | RPC函数名 | string (size + data) | Y |
| __fbthrift_field_kind | RPC的类型，上面代码中是apache::thrift::RpcKind::SINGLE_REQUEST_SINGLE_RESPONSE | int32 | Y |
| __fbthrift_field_clientTimeoutMs | 客户端设置的超时时间 默认60s | int32 | Y |
| __fbthrift_field_queueTimeoutMs | 排队超时时间 默认是0 （无超时） |  | N |
| __fbthrift_field_priority | RPC的优先级(thrift接口中可定义) |  | N |
| __fbthrift_field_otherMetadata | 剩余的Metadata 以map<string, string>形式保存 |  | N |
| __fbthrift_field_crc32c | CRC32 默认不填 |  | N |
| __fbthrift_field_loadMetric | 没研究作用 |  | N |
| __fbthrift_field_compression | NONE 压缩算法 |  | N |
| __fbthrift_field_compressionConfig | 压缩配置 |  | N |
| __fbthrift_field_interactionId | 没研究作用 |  | N |
| __fbthrift_field_interactionCreate | 没研究作用 |  | N |
| __fbthrift_field_serviceTraceMeta | 没研究作用 |  | N |
| frameworkMetadata | 框架级别的Metadata |  | N |

> 上面的表格都是从生成的写入Metadata的中间文件中整理出来的

在编码的时候，除了要编码字段的值，还需要需要把每个字段的id和类型也编码进去，和RPC实际参数也是一样的。根据上面的表，写入的字段就是`__fbthrift_field_protocol`，`__fbthrift_field_name`，`__fbthrift_field_kind`和`__fbthrift_field_clientTimeoutMs`。对应上面的struct的short form编码格式，我们详细看下是MetaData(也是个struct)是如何编码的:

```text
Compact protocol field header (short form) and field value:
+--------+--------+...+--------+
|ddddtttt| field value         |
+--------+--------+...+--------+
- `dddd` is the field id delta, an unsigned 4 bits integer, strictly positive.
- `tttt` is field-type id, an unsigned 4 bit integer.
```

这四个字段的FieldId依次为1,2,3,5(为啥跳过4不清楚，生成的中间文件中就是这样)，类型依次为Int32, Binary, Int32, Int32。

```cpp
// 第一个字段__fbthrift_field_protocol
// fieldId为1, type为I32即5, 所以编码为(1 << 4) | 5, 最终值为0x15
// 对应的值为COMPACT即2, 经过Int编码转换为i32ToZigzag(writeVarint(2)) = 4
0x15, 0x04,
// 第二个字段__fbthrift_field_name
// fieldId为2, previous fieldId为1, type为Binary即8, 所以编码为
// (2 - 1) << 4 | 8 = 0x18
// 然后由于类型为binary, 所以编码其长度为0x0c
// 然后剩下的就是实际值从0x73开始的12个字节对应的就是这个接口名sendResponse的ASCII值 
0x18, 0x0c, 0x73, 0x65, 0x6e, 0x64, 0x52, 0x65, 0x73, 0x70, 0x6f, 0x6e, 0x73, 0x65,
// 第三个字段__fbthrift_field_kind
// fieldId为3, previous fieldId为2, type为I32即5, 所以编码为
// (3 - 2) << 4 | 5 = 0x15
// 值为0 编码出来也是0
0x15, 0x00,
// __fbthrift_field_clientTimeoutMs
// 我在测试中把这个字段设置为了3600 * 24 * 1000 = 86400000 也就是一天的毫秒表示
// fieldId为5, previous fieldId为3, type为I32即5, 所以编码为
// (5 - 3) << 4 | 5 = 0x25
// 对应的值为86400000, 经过Int编码转换为writeVarint(i32ToZigzag(86400000)) = 0x80, 0xf0, 0xb2, 0x52
// 可以参考最上面介绍Integer编码部分
0x25, 0x80, 0xf0, 0xb2, 0x52,
// Metadata这个stuct的终止符
0x00
```

在fbthrift中对应写入FieldId和FieldType的代码如下

```cpp
uint32_t CompactProtocolWriter::writeFieldBeginInternal(
    const char* /*name*/,
    const TType fieldType,
    const int16_t fieldId,
    int8_t typeOverride,
    int16_t previousId) {
  DCHECK_EQ(previousId, lastFieldId_);
  uint32_t wsize = 0;

  // if there's a type override, use that.
  int8_t typeToWrite =
      (typeOverride == -1
           ? apache::thrift::detail::compact::TTypeToCType[fieldType]
           : typeOverride);

  // check if we can use delta encoding for the field id
  if (fieldId > previousId && fieldId - previousId <= 15) {
    // write them together
    wsize += writeByte(
        static_cast<int8_t>((fieldId - previousId) << 4) | typeToWrite);
  } else {
    // write them separate
    wsize += writeByte(typeToWrite);
    wsize += writeI16(fieldId);
  }

  lastFieldId_ = fieldId;
  return wsize;
}
```

到这里RequestRpcMetadata就编码完了。

### 实际RPC请求参数

对于我们这个比较简单的例子来说，编码RequestRpcMetadata完后剩余的设计RPC请求参数比较简单，只剩这些字节了

`18 06 64 6f 6f 64 6c 65`

```text
18中的高4位1代表和前面一个字段的id差为1，说明实际请求参数和RequestRpcMetadata被编码在了一个更大的结构体中
低4位8代表是binary
然后06代表binary的长度
剩下的6个字节刚好就是发送的参数"doodle"的ASCII
```

到这里相关的字节都解析完了，本来是想用实际tcpdump结果再分析下的，一下梳理了这么多，后续有空会把SetupFrame和实际tcpdump的结果再分析下~

## Reference

[thrift/thrift-compact-protocol.md at master · apache/thrift (github.com)](https://github.com/apache/thrift/blob/master/doc/specs/thrift-compact-protocol.md)

[LEB128 - Wikipedia](https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128)

[Variable-length quantity - Wikipedia](https://en.wikipedia.org/wiki/Variable-length_quantity)

[facebook/fbthrift: Facebook's branch of Apache Thrift, including a new C++ server. (github.com)](https://github.com/facebook/fbthrift)