---
layout: single
title: RocksDB SecondaryIndex
date: 2025-08-07 00:00:00 +0800
categories: 源码
tags: RocksDB
---

RocksDB在今年的版本，引入了新的实验性功能二级索引，这两天花了点时间看了看相关实现，整体功能还不是非常强大，只能算搭好了一个架子，为之后的扩展留下了一些空间。

### Usage Example

结合RocksDB单测说明二级索引的使用方式：

1. 通过在`Open`时的参数，创建二级索引实例并配置到事务数据库选项中。比如下面例子中，会对`kDefaultWideColumnName`，也就是默认的匿名列创建二级索引。
2. 在写入的过程中，任何写入的数据只要包含这个匿名列，就会自动维护对应的二级索引。比如下面例子中`key2`和`key3`的就会在二级索引中有对应数据，而`key1`因为没有写入匿名列，所以不存在对应二级索引数据。
3. 读取的时候通过`SecondaryIndexIterator`进行迭代查找即可，比如要查找所有对应匿名列value为`bar`的key，只需要对`SecondaryIndexIterator`进行`Seek("bar")`之后，既可获取。

```cpp
TEST_P(TransactionTest, SecondaryIndexPutDelete) {
  const TxnDBWritePolicy write_policy = std::get<2>(GetParam());
  if (write_policy != TxnDBWritePolicy::WRITE_COMMITTED) {
    ROCKSDB_GTEST_BYPASS("Test only WriteCommitted for now");
    return;
  }

  txn_db_options.secondary_indices.emplace_back(
      std::make_shared<SimpleSecondaryIndex>(
          kDefaultWideColumnName.ToString()));

  ASSERT_OK(ReOpen());

  ColumnFamilyOptions cf1_opts;
  ColumnFamilyHandle* cfh1 = nullptr;
  ASSERT_OK(db->CreateColumnFamily(cf1_opts, "cf1", &cfh1));
  std::unique_ptr<ColumnFamilyHandle> cfh1_guard(cfh1);

  ColumnFamilyOptions cf2_opts;
  ColumnFamilyHandle* cfh2 = nullptr;
  ASSERT_OK(db->CreateColumnFamily(cf2_opts, "cf2", &cfh2));
  std::unique_ptr<ColumnFamilyHandle> cfh2_guard(cfh2);

  auto& index = txn_db_options.secondary_indices.back();
  index->SetPrimaryColumnFamily(cfh1);
  index->SetSecondaryColumnFamily(cfh2);

  {
    std::unique_ptr<Transaction> txn(db->BeginTransaction(WriteOptions()));

    // Default CF => OK but not indexed
    ASSERT_OK(txn->Put(db->DefaultColumnFamily(), "key0", "foo"));

    // CF1 but no default column => OK but not indexed
    ASSERT_OK(txn->PutEntity(cfh1, "key1", {{"hello", "world"}}));

    // CF1, "bar" in the default column => OK and indexed
    ASSERT_OK(txn->Put(cfh1, "key2", "bar"));

    // CF1, "baz" in the default column => OK and indexed
    ASSERT_OK(txn->Put(cfh1, "key3", "baz"));

    ASSERT_OK(txn->Commit());
  }

  // Expected keys: "key0" in the default CF; "key1", "key2", "key3" in CF1;
  // secondary index entries for "key2" and "key3" in CF2
  {
    // Query the secondary index
    std::unique_ptr<Iterator> underlying_it(
        db->NewIterator(ReadOptions(), cfh2));
    auto it = std::make_unique<SecondaryIndexIterator>(
        index.get(), std::move(underlying_it));

    it->Seek("bar");
    ASSERT_TRUE(it->Valid());
    ASSERT_OK(it->status());
    ASSERT_EQ(it->key(), "key2");
    ASSERT_TRUE(it->value().empty());

    it->Seek("baz");
    ASSERT_TRUE(it->Valid());
    ASSERT_OK(it->status());
    ASSERT_EQ(it->key(), "key3");
    ASSERT_TRUE(it->value().empty());

  }
  
  // ...
}
```

> 另外RocksDB可以支持数据和对应的索引保存在不同的column family中，即上面例子中的`SetPrimaryColumnFamily`和`SetSecondaryColumnFamily`
> 

### 写入流程

仍以前面的示例为例，我们使用为默认匿名列创建二级索引，primary column和对应二级索引列的数据如下所示。实际在二级索引中，所有数据都是存在key中的，这里是为了方便理解，仍然以映射形式展示。

```cpp
# primary column
key2 -> bar
key3 -> baz
```

```cpp
# secondary index
bar -> key2
baz -> key3
```

假设现在要把`key2`对应的value从`bar`改为`foo`，我们看下二级索引的修改逻辑。核心函数是`SecondaryIndexMixin::PutWithSecondaryIndicesImpl`，`主要步骤为`：

1. `GetPrimaryEntryForUpdate`
2. `RemoveSecondaryEntries`
3. `UpdatePrimaryColumnValues`
4. `AddPrimaryEntry`
5. `AddSecondaryEntries`

我们一步一步看下具体流程。

`GetPrimaryEntryForUpdate`：在primary column中，读取写入的key是否有已有的数据。在我们的例子中会读到`bar`。

```cpp
SecondaryIndexMixin::GetPrimaryEntryForUpdate
-> PessimisticTransaction::GetEntityForUpdate
  -> TransactionBaseImpl::GetEntityForUpdate
    -> TransactionBaseImpl::GetEntityImpl
      -> WriteBatchWithIndex::GetEntityFromBatchAndDB
        -> WriteBatchWithIndexInternal::GetEntityFromBatch
        -> DBImpl::GetImpl
```

> `SecondaryIndexMixin`中有个模版参数Txn，目前二级索引只支持`PessimisticTransaction`。
> 

最终调用到`WriteBatchWithIndex::GetEntityFromBatchAndDB`，这里面会先调用`GetEntityFromBatch`尝试从当前事务的写入中读取这个key（即read-own-write），如果读不到，则会调用`DBImpl::GetImpl`走正常的读逻辑。

`RemoveSecondaryEntries`：根据原有数据，删掉老的二级索引值，即在二级索引中删掉`bar`对应的数据。

```cpp
  Status RemoveSecondaryEntry(const SecondaryIndex* secondary_index,
                              const Slice& primary_key,
                              const Slice& existing_primary_column_value) {
    assert(secondary_index);

    std::variant<Slice, std::string> secondary_key_prefix;

    {
      const Status s = secondary_index->GetSecondaryKeyPrefix(
          primary_key, existing_primary_column_value, &secondary_key_prefix);
      if (!s.ok()) {
        return s;
      }
    }

    {
      const Status s =
          secondary_index->FinalizeSecondaryKeyPrefix(&secondary_key_prefix);
      if (!s.ok()) {
        return s;
      }
    }

    const std::string secondary_key =
        SecondaryIndexHelper::AsString(secondary_key_prefix) +
        primary_key.ToString();

    return Txn::SingleDelete(secondary_index->GetSecondaryColumnFamily(),
                             secondary_key);
  }
```

其中`GetSecondaryKeyPrefix`和`FinalizeSecondaryKeyPrefix`会确定二级索引编码中的前缀：

- 对于最普通的二级索引即`SimpleSecondaryIndex`来说，`GetSecondaryKeyPrefix`返回的就是primary column中的值。（目前rocksdb也支持了faiss的向量索引，也是通过二级索引来实现的，之类行为会有所区别）
- 之后在`FinalizeSecondaryKeyPrefix`中确定要删除二级索引的数据的前缀，实际就是一个 `VarInt32`加上当前二级索引对应的priamry column的值，即`VarInt32 + bar`。对应我们的例子中，`bar`的长度为3，`VarInt32`也是3，十六进制下的二级索引前缀就是就是`03626172`。
    
    ```cpp
    Status SimpleSecondaryIndex::FinalizeSecondaryKeyPrefix(
        std::variant<Slice, std::string>* secondary_key_prefix) const {
      assert(secondary_key_prefix);
    
      std::string prefix;
      PutLengthPrefixedSlice(&prefix,
                             SecondaryIndexHelper::AsSlice(*secondary_key_prefix));
    
      *secondary_key_prefix = std::move(prefix);
    
      return Status::OK();
    }
    ```
    
- 此时可以得到最终二级索引中二级索引的key，即代码中的`secondary_key_prefix + primary_key`。
    
    ```
    03 62 61 72 6B 65 79 32
       b  a  r  k  e  y  2
    ```
    
- 调用SingleDelete删除二级索引中的key

在前两步`GetPrimaryEntryForUpdate`和`RemoveSecondaryEntries中`，已经删除了老的二级索引。之后就开始处理要写入的新数据，在`UpdatePrimaryColumnValues`中，根据要写入的值，确定会有哪些二级索引需要更新，并把每个二级索引需要写入的字段保存到`IndexData`中。

```cpp
  Status UpdatePrimaryColumnValues(ColumnFamilyHandle* column_family,
                                   const Slice& primary_key,
                                   WideColumns& primary_columns,
                                   autovector<IndexData>& applicable_indices) {
    assert(column_family);
    assert(applicable_indices.empty());

    // TODO: as an optimization, we can avoid calling SortColumns a second time
    // in WriteBatchInternal::PutEntity
    WideColumnsHelper::SortColumns(primary_columns);

    applicable_indices.reserve(secondary_indices_->size());

    for (const auto& secondary_index : *secondary_indices_) {
      assert(secondary_index);

      if (secondary_index->GetPrimaryColumnFamily() != column_family) {
        continue;
      }

      const auto it = WideColumnsHelper::Find(
          primary_columns.begin(), primary_columns.end(),
          secondary_index->GetPrimaryColumnName());
      if (it == primary_columns.end()) {
        continue;
      }

      applicable_indices.emplace_back(
          IndexData(secondary_index.get(), it->value()));

      auto& index_data = applicable_indices.back();

      const Status s = secondary_index->UpdatePrimaryColumnValue(
          primary_key, index_data.previous_column_value(),
          &index_data.updated_column_value());
      if (!s.ok()) {
        return s;
      }

      it->value() = index_data.primary_column_value();
    }

    return Status::OK();
  }
```

到这就可以开始写入新数据了。`AddPrimaryEntry`负责写入primary column的数据，流程比较简单：

```cpp
SecondaryIndexMixin::AddPrimaryEntry
-> PessimisticTransaction::PutEntity
  -> WriteCommittedTxn::PutEntityImpl
```

`AddSecondaryEntries`会根据之前准备好的`IndexData`写入二级索引的数据：

```cpp
  Status AddSecondaryEntry(const SecondaryIndex* secondary_index,
                           const Slice& primary_key,
                           const Slice& primary_column_value,
                           const Slice& previous_column_value) {
    assert(secondary_index);

    std::variant<Slice, std::string> secondary_key_prefix;

    {
      const Status s = secondary_index->GetSecondaryKeyPrefix(
          primary_key, primary_column_value, &secondary_key_prefix);
      if (!s.ok()) {
        return s;
      }
    }

    {
      const Status s =
          secondary_index->FinalizeSecondaryKeyPrefix(&secondary_key_prefix);
      if (!s.ok()) {
        return s;
      }
    }

    std::optional<std::variant<Slice, std::string>> secondary_value;

    {
      const Status s = secondary_index->GetSecondaryValue(
          primary_key, primary_column_value, previous_column_value,
          &secondary_value);
      if (!s.ok()) {
        return s;
      }
    }

    {
      const std::string secondary_key =
          SecondaryIndexHelper::AsString(secondary_key_prefix) +
          primary_key.ToString();

      const Status s =
          Txn::Put(secondary_index->GetSecondaryColumnFamily(), secondary_key,
                   secondary_value.has_value()
                       ? SecondaryIndexHelper::AsSlice(*secondary_value)
                       : Slice());
      if (!s.ok()) {
        return s;
      }
    }

    return Status::OK();
  }
```

这一步和前面删掉老的二级索引数据一样，需要先根据新的二级索引的值，确定二级索引的编码前缀：`VarInt32` + 二级索引对应的priamry column的新值。对于我们要写入的新数据`foo`来说就是`03 66 6F 6F`。而完整的新的二级索引key就是：

```cpp
03 66 6F 6F 6B 65 79 32
   f  o  o  k  e  y  2
```

最后调用`PessimisticTransaction::Put`完成写入。

## 读取流程

通过写入流程，我们可以理解完整的二级索引的编码为：

```cpp
VarInt32(primary_column_value) + primary_column_value + priamry_column_key
```

那么读取的逻辑也很简单，根据给定的`primary_column_value`进行迭代，即可找到哪些key对应的value为`primary_column_value`。主要逻辑都在`SecondaryIndexIterator`中，主要接口如下：

```cpp
// SecondaryIndexIterator can be used to find the primary keys for a given
// search target. It can be used as-is or as a building block. Its interface
// mirrors most of the Iterator API, with the exception of SeekToFirst,
// SeekToLast, and SeekForPrev, which are not applicable to secondary indices
// and thus not present. Querying the index can be performed by calling the
// returned iterator's Seek API with a search target, and then using Next (and
// potentially Prev) to iterate through the matching index entries. The iterator
// exposes primary keys, that is, the secondary key prefix is stripped from the
// index entries.

class SecondaryIndexIterator {
 public:
  SecondaryIndexIterator(const SecondaryIndex* index,
                         std::unique_ptr<Iterator>&& underlying_it);

  bool Valid() const;
  Status status() const;
  void Seek(const Slice& target);
  void Next();
  void Prev();

  // Returns the primary key from the current secondary index entry.
  Slice key() const;

  // Returns the value of the current secondary index entry.
  Slice value() const;

  // Returns the value of the current secondary index entry as a wide-column structure
  const WideColumns& columns() const;

  // ...

 private:
  const SecondaryIndex* index_;
  std::unique_ptr<Iterator> underlying_it_;
  Status status_;
  std::string prefix_;
};
```

可以看到`SecondaryIndexIterator`内部还用了一个普通的Iterator，接口普通Iterator也基本一致。主要的区别在于：

- `key`返回的是当前二级索引对应的`priamry_column_key`。举例说明，如果primary column中数据为`key2 → bar`，对应二级索引的编码为`\3barkey2`，此时`key()`返回的不是完整的key，而是对应primary column中的key，即`key2`。
- `value`对于普通的二级索引一直为空

相关实现都比较简单，不展开解释了。

```cpp
void SecondaryIndexIterator::Seek(const Slice& target) {
  status_ = Status::OK();

  std::variant<Slice, std::string> prefix = target;

  const Status s = index_->FinalizeSecondaryKeyPrefix(&prefix);
  if (!s.ok()) {
    status_ = s;
    return;
  }

  prefix_ = SecondaryIndexHelper::AsString(prefix);

  // FIXME: this works for BytewiseComparator but not for all comparators in
  // general
  underlying_it_->Seek(prefix_);
}

Slice SecondaryIndexIterator::key() const {
  assert(Valid());

  Slice key = underlying_it_->key();
  key.remove_prefix(prefix_.size());

  return key;
}

Slice SecondaryIndexIterator::value() const {
  assert(Valid());

  return underlying_it_->value();
}

const WideColumns& SecondaryIndexIterator::columns() const {
  assert(Valid());

  return underlying_it_->columns();
}
```

## 写在最后

通过wide-columns和二级索引，RocksDB逐渐开始支持结构化数据的功能。索引的本质在于排序，可以看到目前RocksDB的二级索引本质上也是对`primary_column_value`进行了排序，进而能够根据value找到对应key。然而，但目前默认的二级索引的编码格式其实只能支持给定前缀的查找。举例说明，假设要查询值范围在`[abc, bcd]`区间内的对应数据，从逻辑上说，这个区间内应该能查到值为`abcd`的数据。然而由于默认二级索引编码格式的最前端是一个变长长度，而`abcd`的长度和`abc`并不同，导致在二级索引编码上一个是`\3`开头，一个是`\4`开头，因此无法直接进行范围查询。

```cpp
VarInt32(primary_column_value) + primary_column_value + priamry_column_key
```

因此，目前默认的二级索引只能查询给定的值，而不能支持范围查询，这作为一个索引其实是不太合格的。好在RocksDB把二级索引相关行为封装在`SecondaryIndexMixin`中，如果需要定制二级索引的编解码等，可以通过实现`SecondaryIndex`的相关接口，实现自己的二级索引。比如RocksDB目前就实现了基于faiss和二级索引的向量索引`FaissIVFIndex`。