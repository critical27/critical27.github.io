---
layout: single
title: "rocksdb prefix seek"
date: 2021-07-28 00:00:00 +0800
categories: 实践
tags: RocksDB
---

正确使用Prefix Seek的姿势。

## Prefix Seek

```c++
Options options;

// <---- Enable some features supporting prefix extraction
options.prefix_extractor.reset(NewFixedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

Iterator* iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"
iter->Next(); // Find next key-value pair inside prefix "foo"
```

When `options.prefix_extractor` is not nullptr and with default ReadOptions, iterators are not guaranteed a total order of all keys, but only keys for the same prefix. When doing Iterator.Seek(lookup_key), RocksDB will extract the prefix of lookup_key. If there is one or more keys in the database matching prefix of lookup_key, RocksDB will place the iterator to the key equal or larger than lookup_key of the same prefix, as for total ordering mode. If no key of the prefix equals or is larger than lookup_key, or after calling one or more Next(), we finish all keys for the prefix, **we might return Valid()=false**, or **any key (might be non-existing) that is larger than the previous key**. Setting `ReadOptions.prefix_same_as_start=true` guarantees the first of those two behaviors.

**也就是说在配置了`prefix_extractor`的前提下, rocksdb是prefix模式**, 当`ReadOptions.prefix_same_as_start=true`时，超出范围一定会返回`Valid()=false`.

而我们目前版本中没有设置`prefix_same_as_start=true`, 因此在还需要手动判断prefix是否匹配

```c++
class RocksPrefixIter : public KVIterator {

    bool valid() const override {
        return !!iter_ && iter_->Valid() && (iter_->key().starts_with(prefix_));
    }

}
```

而不配置`prefix_extractor`的时候, rocksdb需要用户自己判断是否超出范围等等, 具体参考这里[Manual prefix iterating](https://github.com/facebook/rocksdb/wiki/Prefix-Seek#manual-prefix-iterating)

因为有这些行为:
> Some undefined result might include: deleted keys might show up, key ordering is not followed, or very slow queries. Also note that even data within the prefix range might not be correct if the iterator has moved out of the prefix range and come back again.

### How to ignore prefix bloom filters in read

> **`total_order_seek`实际是用来控制是否使用`prefix bloom filter`, 如果为真则不使用**

> `total_order_seek`应该还能保证数据有序

Users can only use prefix bloom filter to read inside a specific prefix. If iterator is outside a prefix, the feature needs to be disabled for specific iterators to prevent wrong results:

```c++
ReadOptions read_options;
read_options.total_order_seek = true;
Iterator* iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```

Putting `read_options.total_order_seek = true` will make sure the query returns the same result as if there is no prefix bloom filter.

### TLDR

核心结论:

1. prefix_same_as_start保证读出来的数据prefix和target相同。
2. **无论是BlockBasedTable还是PlainTable，如果target对应的prefix bloom filter没有被写入的话，当prefix_same_as_start = true时，Seek(target)是一定读不出数据的。** 而当prefix_same_as_start = false时，Seek(target)可以读到target之后的所有数据，不要求prefix和target相同。
3. PlainTable由于只支持`prefix-based Seek()`，同第2条结论。

### BlockBasedTable

当InDomain为true的key会写入一个prefix bloom filter。

```c++
/*
 * A SliceTransform is a generic pluggable way of transforming one string
 * to another. Its primary use-case is in configuring rocksdb
 * to store prefix blooms by setting prefix_extractor in
 * ColumnFamilyOptions.
 */
class SliceTransform {
  ...

  // Extract a prefix from a specified key. This method is called when
  // a key is inserted into the db, and the returned slice is used to
  // create a bloom filter.
  virtual Slice Transform(const Slice& key) const = 0;

  // Determine whether the specified key is compatible with the logic
  // specified in the Transform method. This method is invoked for every
  // key that is inserted into the db. If this method returns true,
  // then Transform is called to translate the key to its prefix and
  // that returned prefix is inserted into the bloom filter. If this
  // method returns false, then the call to Transform is skipped and
  // no prefix is inserted into the bloom filters.
  //
  // For example, if the Transform method operates on a fixed length
  // prefix of size 4, then an invocation to InDomain("abc") returns
  // false because the specified key length(3) is shorter than the
  // prefix size of 4.
  //
  // Wiki documentation here:
  // https://github.com/facebook/rocksdb/wiki/Prefix-Seek-API-Changes
  //
  virtual bool InDomain(const Slice& key) const = 0;

  ...
}
```

* FixedPrefixTransform / CappedPrefixTransform

省略若干

```c++
// 长度大于prefix_len_的key才会写入prefix bloom filter
class FixedPrefixTransform : public SliceTransform {
  Slice Transform(const Slice& src) const override {
    assert(InDomain(src));
    return Slice(src.data(), prefix_len_);
  }

  bool InDomain(const Slice& src) const override {
    return (src.size() >= prefix_len_);
  }
};

// 所有key都会写入bloom filter，prefix长度最大为cap_len_
class CappedPrefixTransform : public SliceTransform {

  Slice Transform(const Slice& src) const override {
    assert(InDomain(src));
    return Slice(src.data(), std::min(cap_len_, src.size()));
  }

  bool InDomain(const Slice& /*src*/) const override { return true; }

};
```

参考[DBIter](DBIter.md)里面`DBIter::Seek`和`DBIter::FindNextUserEntryInternal`部分

当`prefix_same_as_start_ = true`时, 首先会把`target`转换为`target_prefix`. 然后不断调用`FindNextUserEntry`进行查找，当前缀不匹配时完成。(`prefix_extractor_->Transform(ikey_.user_key).compare(*prefix) != 0`)

```c++
void DBIter::Seek(const Slice& target) {
  ...
  if (prefix_same_as_start_) {
    // The case where the iterator needs to be invalidated if it has exausted
    // keys within the same prefix of the seek key.
    assert(prefix_extractor_ != nullptr);
    Slice target_prefix = prefix_extractor_->Transform(target);
    FindNextUserEntry(false /* not skipping saved_key */,
                      &target_prefix /* prefix */);
    if (valid_) {
      // Remember the prefix of the seek key for the future Next() call to
      // check.
      prefix_.SetUserKey(target_prefix);
    }
  } else {
    FindNextUserEntry(false /* not skipping saved_key */, nullptr);
  }
  ...
}

bool DBIter::FindNextUserEntryInternal(bool skipping_saved_key,
                                       const Slice* prefix) {
  ...
  do {
    ...
    // compare with upper bound
    if (iterate_upper_bound_ != nullptr && iter_.MayBeOutOfUpperBound() &&
        user_comparator_.CompareWithoutTimestamp(
            ikey_.user_key, /*a_has_ts=*/true, *iterate_upper_bound_,
            /*b_has_ts=*/false) >= 0) {
      break;
    }

    assert(prefix == nullptr || prefix_extractor_ != nullptr);
    // if has prefix, compare with prefix
    if (prefix != nullptr &&
        prefix_extractor_->Transform(ikey_.user_key).compare(*prefix) != 0) {
      assert(prefix_same_as_start_);
      break;
    }

    ...
  }
  ...
}
```

测试代码如下

```c++
void test(DB* db, bool prefix_same_as_start) {
    Status s;
    s = db->Put(WriteOptions(), "AAAAB", "value0");
    assert(s.ok());
    s = db->Put(WriteOptions(), "AAABB", "value1");
    assert(s.ok());
    // 这条数据是用来对比FixedPrefixTransform和CappedPrefixTransform的
    /*
    s = db->Put(WriteOptions(), "AAB", "value2");
    assert(s.ok());
    */
    s = db->Put(WriteOptions(), "AABBB", "value3");
    assert(s.ok());
    s = db->Put(WriteOptions(), "ABBBB", "value4");
    assert(s.ok());
    s = db->Put(WriteOptions(), "BBBBB", "value5");
    assert(s.ok());
    s = db->Put(WriteOptions(), "C", "value6");
    assert(s.ok());

    // flush是为了确保从sst来读数据 否则有的memtable是有whole_key_filtering
    db->Flush(FlushOptions());

    ReadOptions read_options;
    read_options.prefix_same_as_start = prefix_same_as_start;

    cout << "Prefix seek of AAB:\n";
    Iterator *iter = db->NewIterator(read_options);
    iter->Seek("AAB");
    while (iter->Valid()) {
        cout << iter->key().ToString().c_str() << "\n";
        iter->Next();
    }

    cout << "Prefix seek of AABB:\n";
    iter = db->NewIterator(read_options);
    iter->Seek("AABB");
    while (iter->Valid()) {
        cout << iter->key().ToString().c_str() << "\n";
        iter->Next();
    }

    cout << "Prefix seek of AABBB:\n";
    iter = db->NewIterator(read_options);
    iter->Seek("AABBB");
    while (iter->Valid()) {
        cout << iter->key().ToString().c_str() << "\n";
        iter->Next();
    }

  cout << "Prefix seek of A:\n";
  iter = db->NewIterator(read_options);
  iter->Seek("A");
  while (iter->Valid()) {
    cout << iter->key().ToString().c_str() << "\n";
    iter->Next();
  }

  cout << "Get full key AABBB: ";
  std::string value;
  s = db->Get(ReadOptions(), rocksdb::Slice("AABBB"), &value);
  assert(s.ok());
  cout << value.c_str() << "\n";

    cout << "Get full key C: ";
    s = db->Get(ReadOptions(), rocksdb::Slice("C"), &value);
    if (!s.ok()) {
        cout << s.ToString() << "\n";
    } else {
        cout << value.c_str() << "\n";
    }

    delete iter;
}

int main()
{
    DB *db;
    Options options;
    options.create_if_missing = true;
    options.prefix_extractor.reset(NewFixedPrefixTransform(4));
    // options.prefix_extractor.reset(NewCappedPrefixTransform(4));
    // options.prefix_extractor.reset(new TestTransform(4));

    Status s = DB::Open(options, "/Users/doodle/Mine/rocksdb_test", &db);
    assert(s.ok());

    test(db, true);
    test(db, false);

    delete db;
    return 0;
}
```

插入的prefix bloom filter

| key   | prefix of FixedPrefixTransform(4) | prefix of CappedPrefixTransform(4) |
| ----- | --------------------------------- | ---------------------------------- |
| AAAAB | AAAA                              | AAAA                               |
| AAABB | AAAB                              | AAAB                               |
| AABBB | AABB                              | AABB                               |
| ABBBB | ABBB                              | ABBB                               |
| BBBBB | BBBB                              | BBBB                               |
| C     |                                   | C                                  |

查询的prefix bloom filter

| key   | prefix of FixedPrefixTransform(4) | prefix of CappedPrefixTransform(4) |
| ----- | --------------------------------- | ---------------------------------- |
| AAB   |                                   | AAB                                |
| AABB  | AABB                              | AABB                               |
| AABBB | AABB                              | AABB                               |

结果:

1. 当`prefix_same_as_start = true`时，用"AAB"来Prefix查找，一条都查不出来，看下面结果，因为"AAB"无论是用`FixedPrefixTransform`还是`CappedPrefixTransform`所Transform出来的prefix 都不满足下面条件(不在插入的prefix bloom filter中)，对于前缀"A"也是同理

`prefix_extractor_->Transform(ikey_.user_key).compare(*prefix) != 0`

2. 而当`prefix_same_as_start = false`时，则都可以读出来

3. 对于`BlockBasedTable`由于`whole_key_filtering`默认为true，所以使用Get读单个key的时候能够正常读出来，不受prefix bloom filter的影响

```c++
// FixedPrefixTransform / CappedPrefixTransform / TestTransform 结果一样
/*
// prefix_same_as_start = true 时，当prefix不匹配时Valid为false
// 当以AAB为前缀查询时，结果都为空
Prefix seek of AAB:
Prefix seek of AABB:
AABBB
Prefix seek of AABBB:
AABBB
Prefix seek of A:
Get full key AABBB: value3
Get full key C: value6

--------------------------
// prefix_same_as_start = false 时，即便当前key和prefix不匹配也继续查找
Prefix seek of AAB:
AABBB
ABBBB
BBBBB
C
Prefix seek of AABB:
AABBB
ABBBB
BBBBB
C
Prefix seek of AABBB:
AABBB
ABBBB
BBBBB
C
Prefix seek of A:
AAAAB
AAABB
AABBB
ABBBB
BBBBB
C
Get full key AABBB: value3
Get full key C: value6
*/
```

而当真的如果插入一条"AAB"数据后在，`FixedPrefixTransform`和`CappedPrefixTransform`结果就会不同

`FixedPrefixTransform`结果

```c++
/*
// prefix_same_as_start = true
// FixedPrefixTransform 读不出来Seek("AAB") 因为AAB的prefix bloom filter没有写进去
Prefix seek of AAB:
Prefix seek of AABB:
AABBB
Prefix seek of AABBB:
AABBB
Prefix seek of A:
Get full key AABBB: value3
Get full key C: value6

--------------------------
// prefix_same_as_start = false
Prefix seek of AAB:
AAB
AABBB
ABBBB
BBBBB
C
Prefix seek of AABB:
AABBB
ABBBB
BBBBB
C
Prefix seek of AABBB:
AABBB
ABBBB
BBBBB
C
Prefix seek of A:
AAAAB
AAABB
AAB
AABBB
ABBBB
BBBBB
C
Get full key AABBB: value3
Get full key C: value6
*/
```

`CappedPrefixTransform`结果

```c++
/*
// CappedPrefixTransform 就可以读到Seek("AAB")
Prefix seek of AAB:
AAB
Prefix seek of AABB:
AABBB
Prefix seek of AABBB:
AABBB
Prefix seek of A:
Get full key AABBB: value3
Get full key C: value6

--------------------------
// prefix_same_as_start = false
Prefix seek of AAB:
AAB
AABBB
ABBBB
BBBBB
C
Prefix seek of AABB:
AABBB
ABBBB
BBBBB
C
Prefix seek of AABBB:
AABBB
ABBBB
BBBBB
C
Prefix seek of A:
AAAAB
AAABB
AAB
AABBB
ABBBB
BBBBB
C
Get full key AABBB: value3
Get full key C: value6
*/
```

* 自己定义prefix_extractor

```c++
class TestTransform : public rocksdb::SliceTransform {
private:
    size_t prefixLen_;

public:
    explicit TestTransform(size_t prefixLen)
        : prefixLen_(prefixLen) {}

    const char* Name() const override { return "TestTransform"; }

    rocksdb::Slice Transform(const rocksdb::Slice& src) const override {
        return rocksdb::Slice(src.data(), prefixLen_);
    }

    bool InDomain(const rocksdb::Slice& key) const override {
        return key.size() >= prefixLen_;
    }
};
```

### PlainTable

PlainTable只支持`prefix-based Seek()`，如果prefix没有生成对应的bloom_filter，是读不到数据的。(以`AAB`为前缀时)

此外，由于PlainTable没有whole_key_filtering，如果生成对应的prefix bloom_filter没有生成，使用Get是无法读到结果的。(`FixedPrefixTransform`中直接读`C`是读不到的)

```c++
int main()
{
    DB *db;
    Options options;
    options.create_if_missing = true;
    options.prefix_extractor.reset(NewFixedPrefixTransform(4));
    // options.prefix_extractor.reset(NewCappedPrefixTransform(4));

    options.table_factory.reset(rocksdb::NewPlainTableFactory());

    Status s = DB::Open(options, "/Users/doodle/Mine/rocksdb_test", &db);
    assert(s.ok());

    test(db, true);
    test(db, false);

    delete db;
    return 0;
}
```

`FixedPrefixTransform`结果

```c++
/*
// C因为FixedPrefixTransform(4)没有生成prefix bloom filter, 所以读不到
Prefix seek of AAB:
Prefix seek of AABB:
AABBB
Prefix seek of AABBB:
AABBB
Prefix seek of A:
Get full key AABBB: value3
Get full key C: NotFound:

--------------------------
Prefix seek of AAB:
Prefix seek of AABB:
AABBB
ABBBB
BBBBB
C
Prefix seek of AABBB:
AABBB
ABBBB
BBBBB
C
Prefix seek of A:
Get full key AABBB: value3
Get full key C: NotFound:
*/
```

`CappedPrefixTransform`结果

```c++
Prefix seek of AAB:
AAB
Prefix seek of AABB:
AABBB
Prefix seek of AABBB:
AABBB
Prefix seek of A:
Get full key AABBB: value3
Get full key C: value6

--------------------------
Prefix seek of AAB:
AAB
AABBB
ABBBB
BBBBB
C
Prefix seek of AABB:
AABBB
ABBBB
BBBBB
C
Prefix seek of AABBB:
AABBB
ABBBB
BBBBB
C
Prefix seek of A:
Get full key AABBB: value3
Get full key C: value6
*/
```
