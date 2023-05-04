---
layout: single
title: RocksDB options management
date: 2023-04-30 00:00:00 +0800
categories: 源码
tags: RocksDB
---

## RocksDB options

RockDB里最基础的两种配置就是`DBOptions`和`ColumnFamilyOptions`, 这两个类构成了打开RocksDB时需要传入的`Options`. 相关类的继承关系如下:

```cpp

struct Options : public DBOptions, public ColumnFamilyOptions {
  // Create an Options object with default values for all fields.
  Options() : DBOptions(), ColumnFamilyOptions() {}

  Options(const DBOptions& db_options,
          const ColumnFamilyOptions& column_family_options)
      : DBOptions(db_options), ColumnFamilyOptions(column_family_options) {}

  // ...
};

struct DBOptions {
  // ...
};

struct ColumnFamilyOptions : public AdvancedColumnFamilyOptions {
  // ...
};

struct AdvancedColumnFamilyOptions {
  // ...
};
```

只需要指定好`DBOptions`和`ColumnFamilyOptions`, 就能够构造出`Options`, 从而完成RocksDB的初始化. 常见的指定`DBOptions`和`ColumnFamilyOptions`的方法有两类:

- 直接在代码里构造`DBOptions`和`ColumnFamilyOptions`, 然后进行配置的修改. 这是绝大多数使用RocksDB的方式, 只不过每次修改配置, 都需要直接修改代码或者通过额外的gflag或者全局变量之类的进行控制.
- 从一个unordered_map或者string中解析并构造出`DBOptions`和`ColumnFamilyOptions`, 这篇文章主要是从这个方面入手, 分析这些配置是如何在RocksDB中进行管理的.

## 解析options的方法

RocksDB提供了以下办法从string或者一个unordered_map解析出相应的配置, 这些方法都在`include/rocksdb/convenience.h`中

- `GetDBOptionsFromString`
- `GetDBOptionsFromMap`
- `GetColumnFamilyOptionsFromString`
- `GetColumnFamilyOptionsFromMap`
- `GetBlockBasedTableOptionsFromString`
- `GetBlockBasedTableOptionsFromMap`

以GetDBOptionsFromString为例看下调用路径

```cpp
GetDBOptionsFromString
	-> GetDBOptionsFromMap
    -> ConfigureFromMap<DBOptions> // 模版函数
      -> Configurable::ConfigureFromMap
        -> Configurable::ConfigureOptions // 在这个map中, 尝试解析出已经注册的option
				  -> ConfigurableHelper::ConfigureOptions // 实际的解析工作
            -> ConfigurableHelper::ConfigureSomeOptions
							-> ConfigurableHelper::ConfigureOption // 解析单个配置
                // 下面是不同类型的配置构造方法
								* ConfigurableHelper::ConfigureCustomizableOption
								* Configurable::ParseOption
```

### **Configurable**

首先关注一个关键的类, `Configurable`, 它提供了一些获取和更新配置的接口, 比如

- GetOptions: 获取某个配置的值
- ConfigureFromMap: 从一个unordered_map<string, string>构造配置
- ConfigureFromString: 从一个string构造配置
- ConfigureOption: 进行单个配置的解析
- ValidateOptions: 验证传入的配置是否合法
- RegisterOptions: 模版函数, 将不同类型的配置名以及配置的详细信息进行注册. 详细信息包括类型, 在结构体中的偏移量, 构造的方式等等, 参见下文.

可以看到db_options.cc和cf_options.cc中, 分别对`DBOptions`和`ColumnFamilyOptions`进行了注册. 整个Configurable可以理解为一张注册表, 里面注册了不同种类的配置, 并且标注了每个配置的详细信息. 最终解析完的每个配置及对应的值也是保存在这个注册表中, 下面会不断分析这个注册表中的内容. `Configure`这个类的数据结构如下, 这其中就是保存了若干个`RegisteredOptions`, 每个`RegisteredOptions`是一种类型的配置注册表. (后面会分析注册配置的流程, 这里先跳过)

```cpp
class Configurable {
 protected:
  friend class ConfigurableHelper;
  struct RegisteredOptions {
    // The name of the options being registered
    std::string name;
    // Pointer to the object being registered
    void* opt_ptr;
#ifndef ROCKSDB_LITE
    // The map of options being registered
    const std::unordered_map<std::string, OptionTypeInfo>* type_map;
#endif
  };

  // ...

private:
  // Contains the collection of options (name, opt_ptr, opt_map) associated with
  // this object. This collection is typically set in the constructor of the
  // Configurable option via
  std::vector<RegisteredOptions> options_; // 所谓的注册表
};
```

注册完成之后, 就可以从`Configurable`的所有已经注册过的参数中, 进行配置解析. 无论是从string还是从map进行解析, 最终都会调用到这个`ConfigurableHelper::ConfigureOptions`这个函数. 相关参数含义如下:

- `config_options`: 保存如何进行配置解析, 比如遇到未知的配置名怎么办, 遇到过期的配置怎么办等等
- `configurable`: 配置的注册表
- `opts_map`: 用户传进来的配置名和对应的值

整个过程就是遍历不同类型的配置表, 然后在`ConfigureSomeOptions`函数中, 对每个`opts_map`中传入的参数名和值, 尝试进行解析, 最终的参数也会更新到`Configurable`中.

```cpp
Status ConfigurableHelper::ConfigureOptions(
    const ConfigOptions& config_options, Configurable& configurable,
    const std::unordered_map<std::string, std::string>& opts_map,
    std::unordered_map<std::string, std::string>* unused) {
  std::unordered_map<std::string, std::string> remaining = opts_map;
  Status s = Status::OK();
  if (!opts_map.empty()) {
#ifndef ROCKSDB_LITE
    for (const auto& iter : configurable.options_) {
      if (iter.type_map != nullptr) {
        s = ConfigureSomeOptions(config_options, configurable, *(iter.type_map),
                                 &remaining, iter.opt_ptr);
        if (remaining.empty()) {  // Are there more options left?
          break;
        } else if (!s.ok()) {
          break;
        }
      }
    }
#else
    (void)configurable;
    if (!config_options.ignore_unknown_options) {
      s = Status::NotSupported("ConfigureFromMap not supported in LITE mode");
    }
#endif  // ROCKSDB_LITE
  }
  if (unused != nullptr && !remaining.empty()) {
    unused->insert(remaining.begin(), remaining.end());
  }
  if (config_options.ignore_unknown_options) {
    s = Status::OK();
  } else if (s.ok() && unused == nullptr && !remaining.empty()) {
    s = Status::NotFound("Could not find option: ", remaining.begin()->first);
  }
  return s;
}
```

### Immutable and mutable options

为了能表示清楚, 在打开RocksDB后, 哪些参数可以修改, 哪些不可以修改. RocksDB又进一步细分出了mutable option和immutable option.

`Configure`类会派生出若干对象, 他们分别用来管理`MutableDBOptions`, `ImmutableDBOptions`, `MutableCFOptions`和`ImmutableCFOptions`这些不同类型的配置.

在`options/db_options.cc`和`options/cf_options.cc`注册配置的入口如下, 调用`RegisterOptions`函数后就会在`Configurable`中完成注册.

> 代码命名风格有些不一致, 不用纠结
> 

```cpp
class MutableDBConfigurable : public Configurable {
 public:
  explicit MutableDBConfigurable(
      const MutableDBOptions& mdb,
      const std::unordered_map<std::string, std::string>* map = nullptr)
      : mutable_(mdb), opt_map_(map) {
    RegisterOptions(&mutable_, &db_mutable_options_type_info);
  }

protected:
  MutableDBOptions mutable_;
  const std::unordered_map<std::string, std::string>* opt_map_;
};

class DBOptionsConfigurable : public MutableDBConfigurable {
 public:
  explicit DBOptionsConfigurable(
      const DBOptions& opts,
      const std::unordered_map<std::string, std::string>* map = nullptr)
      : MutableDBConfigurable(MutableDBOptions(opts), map), db_options_(opts) {
    // The ImmutableDBOptions currently requires the env to be non-null.  Make
    // sure it is
    if (opts.env != nullptr) {
      immutable_ = ImmutableDBOptions(opts);
    } else {
      DBOptions copy = opts;
      copy.env = Env::Default();
      immutable_ = ImmutableDBOptions(copy);
    }
    RegisterOptions(&immutable_, &db_immutable_options_type_info);
  }

private:
  ImmutableDBOptions immutable_;
  DBOptions db_options_;
};
```

```cpp
class ConfigurableMutableCFOptions : public Configurable {
 public:
  explicit ConfigurableMutableCFOptions(const MutableCFOptions& mcf) {
    mutable_ = mcf;
    RegisterOptions(&mutable_, &cf_mutable_options_type_info);
  }

 protected:
  MutableCFOptions mutable_;
};

class ConfigurableCFOptions : public ConfigurableMutableCFOptions {
 public:
  ConfigurableCFOptions(const ColumnFamilyOptions& opts,
                        const std::unordered_map<std::string, std::string>* map)
      : ConfigurableMutableCFOptions(MutableCFOptions(opts)),
        immutable_(opts),
        cf_options_(opts),
        opt_map_(map) {
    RegisterOptions(&immutable_, &cf_immutable_options_type_info);
  }

private:
  ImmutableCFOptions immutable_;
  ColumnFamilyOptions cf_options_;
  const std::unordered_map<std::string, std::string>* opt_map_;
};
```

`RegisterOptions`的过程本质就是保存了`MutableDBOptions`, `ImmutableDBOptions`, `MutableCFOptions`和`ImmutableCFOptions`这几类配置中具体配置项的详细信息

```cpp
template <typename T>
void Configurable::RegisterOptions(T* opt_ptr, const std::unordered_map<std::string, OptionTypeInfo>* opt_map) {
  RegisterOptions(T::kName(), opt_ptr, opt_map);
}

void Configurable::RegisterOptions(
    const std::string& name, void* opt_ptr,
    const std::unordered_map<std::string, OptionTypeInfo>* type_map) {
  RegisteredOptions opts;
  opts.name = name;
#ifndef ROCKSDB_LITE
  opts.type_map = type_map;
#else
  (void)type_map;
#endif  // ROCKSDB_LITE
  opts.opt_ptr = opt_ptr;
  options_.emplace_back(opts);
}
```

再具体看下每个类型的配置注册了些什么东西进去. 在`db_options.cc`和`cf_options.cc`定义了所有mutable和immutable参数的相关信息. 包括类型, 变量名, 在结构体中的偏移量, mutable/immutable等等, 保存在`OptionTypeInfo`这个类中. 以`CFOptions`为例, 几乎所有类型的配置都能够支持. 常见类型有:

- bool和整形
- `prefix_extractor`和`compaction_filter`这种配置需要传入一个`shared_ptr`的类型的自定义类型
- 最复杂的应该是`comparator`和`table_factory`

```cpp
static std::unordered_map<std::string, OptionTypeInfo>
    cf_mutable_options_type_info = {
        {"report_bg_io_stats",
         {offsetof(struct MutableCFOptions, report_bg_io_stats),
          OptionType::kBoolean, OptionVerificationType::kNormal,
          OptionTypeFlags::kMutable}},
        {"disable_auto_compactions",
         {offsetof(struct MutableCFOptions, disable_auto_compactions),
          OptionType::kBoolean, OptionVerificationType::kNormal,
          OptionTypeFlags::kMutable}},

        // ...

        {"level0_file_num_compaction_trigger",
         {offsetof(struct MutableCFOptions, level0_file_num_compaction_trigger),
          OptionType::kInt, OptionVerificationType::kNormal,
          OptionTypeFlags::kMutable}},
        {"level0_slowdown_writes_trigger",
         {offsetof(struct MutableCFOptions, level0_slowdown_writes_trigger),
          OptionType::kInt, OptionVerificationType::kNormal,
          OptionTypeFlags::kMutable}},
        {"level0_stop_writes_trigger",
         {offsetof(struct MutableCFOptions, level0_stop_writes_trigger),
          OptionType::kInt, OptionVerificationType::kNormal,
          OptionTypeFlags::kMutable}},

        // ...
        {"prefix_extractor",
         OptionTypeInfo::AsCustomSharedPtr<const SliceTransform>(
             offsetof(struct MutableCFOptions, prefix_extractor),
             OptionVerificationType::kByNameAllowNull,
             (OptionTypeFlags::kMutable | OptionTypeFlags::kAllowNull))},
};
```

```cpp
static std::unordered_map<std::string, OptionTypeInfo>
    cf_immutable_options_type_info = {
        {"level_compaction_dynamic_level_bytes",
         {offsetof(struct ImmutableCFOptions,
                   level_compaction_dynamic_level_bytes),
          OptionType::kBoolean, OptionVerificationType::kNormal,
          OptionTypeFlags::kNone}},

        // ...

        {"num_levels",
         {offsetof(struct ImmutableCFOptions, num_levels), OptionType::kInt,
          OptionVerificationType::kNormal, OptionTypeFlags::kNone}},

        // ...

        {"comparator",
         OptionTypeInfo::AsCustomRawPtr<const Comparator>(
             offsetof(struct ImmutableCFOptions, user_comparator),
             OptionVerificationType::kByName, OptionTypeFlags::kCompareLoose)
             .SetSerializeFunc(
                 // Serializes a Comparator
                 [](const ConfigOptions& opts, const std::string&,
                    const void* addr, std::string* value) {
                   // it's a const pointer of const Comparator*
                   const auto* ptr =
                       static_cast<const Comparator* const*>(addr);
                   // Since the user-specified comparator will be wrapped by
                   // InternalKeyComparator, we should persist the
                   // user-specified one instead of InternalKeyComparator.
                   if (*ptr == nullptr) {
                     *value = kNullptrString;
                   } else if (opts.mutable_options_only) {
                     *value = "";
                   } else {
                     const Comparator* root_comp = (*ptr)->GetRootComparator();
                     if (root_comp == nullptr) {
                       root_comp = (*ptr);
                     }
                     *value = root_comp->ToString(opts);
                   }
                   return Status::OK();
                 })},

        {"memtable_factory",
         {offsetof(struct ImmutableCFOptions, memtable_factory),
          OptionType::kCustomizable, OptionVerificationType::kByName,
          OptionTypeFlags::kShared,
          [](const ConfigOptions& opts, const std::string&,
             const std::string& value, void* addr) {
            std::unique_ptr<MemTableRepFactory> factory;
            auto* shared =
                static_cast<std::shared_ptr<MemTableRepFactory>*>(addr);
            Status s =
                MemTableRepFactory::CreateFromString(opts, value, shared);
            return s;
          }}},
        {"memtable",
         {offsetof(struct ImmutableCFOptions, memtable_factory),
          OptionType::kCustomizable, OptionVerificationType::kAlias,
          OptionTypeFlags::kShared,
          [](const ConfigOptions& opts, const std::string&,
             const std::string& value, void* addr) {
            std::unique_ptr<MemTableRepFactory> factory;
            auto* shared =
                static_cast<std::shared_ptr<MemTableRepFactory>*>(addr);
            Status s =
                MemTableRepFactory::CreateFromString(opts, value, shared);
            return s;
          }}},
        {"table_factory",
         OptionTypeInfo::AsCustomSharedPtr<TableFactory>(
             offsetof(struct ImmutableCFOptions, table_factory),
             OptionVerificationType::kByName,
             (OptionTypeFlags::kCompareLoose |
              OptionTypeFlags::kStringNameOnly |
              OptionTypeFlags::kDontPrepare))},
        {"block_based_table_factory",
         {offsetof(struct ImmutableCFOptions, table_factory),
          OptionType::kCustomizable, OptionVerificationType::kAlias,
          OptionTypeFlags::kShared | OptionTypeFlags::kCompareLoose,
          // Parses the input value and creates a BlockBasedTableFactory
          [](const ConfigOptions& opts, const std::string& name,
             const std::string& value, void* addr) {
            BlockBasedTableOptions* old_opts = nullptr;
            auto table_factory =
                static_cast<std::shared_ptr<TableFactory>*>(addr);
            if (table_factory->get() != nullptr) {
              old_opts =
                  table_factory->get()->GetOptions<BlockBasedTableOptions>();
            }
            if (name == "block_based_table_factory") {
              std::unique_ptr<TableFactory> new_factory;
              if (old_opts != nullptr) {
                new_factory.reset(NewBlockBasedTableFactory(*old_opts));
              } else {
                new_factory.reset(NewBlockBasedTableFactory());
              }
              Status s = new_factory->ConfigureFromString(opts, value);
              if (s.ok()) {
                table_factory->reset(new_factory.release());
              }
              return s;
            } else if (old_opts != nullptr) {
              return table_factory->get()->ConfigureOption(opts, name, value);
            } else {
              return Status::NotFound("Mismatched table option: ", name);
            }
          }}},
 
        {"compaction_filter",
         OptionTypeInfo::AsCustomRawPtr<const CompactionFilter>(
             offsetof(struct ImmutableCFOptions, compaction_filter),
             OptionVerificationType::kByName, OptionTypeFlags::kAllowNull)},
        {"compaction_filter_factory",
         OptionTypeInfo::AsCustomSharedPtr<CompactionFilterFactory>(
             offsetof(struct ImmutableCFOptions, compaction_filter_factory),
             OptionVerificationType::kByName, OptionTypeFlags::kAllowNull)},
        {"merge_operator",
         OptionTypeInfo::AsCustomSharedPtr<MergeOperator>(
             offsetof(struct ImmutableCFOptions, merge_operator),
             OptionVerificationType::kByNameAllowFromNull,
             OptionTypeFlags::kCompareLoose | OptionTypeFlags::kAllowNull)},

};
```

上面提到过, 对于每个配置, 在`OptionTypeInfo`中则保存了配置的详细信息, 包括在结构体中的偏移量, 相应的转换函数指针, 配置的类型, 配置的flag等等.

```cpp
class OptionTypeInfo {
  // ...
private:
  // 结构体中的偏移量
  int offset_;

  // 根据string构造相应配置的转换函数
  // The optional function to convert a string to its representation
  ParseFunc parse_func_;

  // 从配置转换相应string表示的转换函数
  // The optional function to convert a value to its string representation
  SerializeFunc serialize_func_;

  // 判断两个配置是否是是相同的配置
  // The optional function to match two option values
  EqualsFunc equals_func_;

  // 配置的类型, 可以是布尔, 整形, 浮点型, 结构体, 数组, 自定义类型等等
  OptionType type_;
  // 验证参数是否合法的方式, 是否允许为null等
  OptionVerificationType verification_;
  // bitmap用来额外标记该配置的一些特性, 如不是mutable, 是不是shared_ptr, unique_ptr等
  OptionTypeFlags flags_;
};
```

到这里, 我们基本就能理解, RocksDB如何通过配置注册进行配置管理, 以及如何通过外部传入的string或者unordered_map进行配置解析. 这里有一部分逻辑缺失了, 就是RocksDB通过什么路径完成配置的注册, 我们会在下面验证options的这部分一并介绍.

## 验证options的方法

不管`DBOptions`和`ColumnFamilyOptions`是从其他地方解析出来, 还是说代码中直接指定, 当打开RocksDB时, 会调用`ValidateOptionsByTable`函数, 分别会对`DBOptions`和`ColumnFamilyOptions`进行校验. 调用路径如下:

```cpp
// db/db_impl/db_impl_open.cc
Status DBImpl::Open(const DBOptions& db_options, const std::string& dbname,
                    const std::vector<ColumnFamilyDescriptor>& column_families,
                    std::vector<ColumnFamilyHandle*>* handles, DB** dbptr,
                    const bool seq_per_batch, const bool batch_per_txn) {
  Status s = ValidateOptionsByTable(db_options, column_families);
  if (!s.ok()) {
    return s;
  }
  // ...
}

// db/db_impl/db_impl_open.cc
Status ValidateOptionsByTable(
    const DBOptions& db_opts,
    const std::vector<ColumnFamilyDescriptor>& column_families) {
  Status s;
  for (auto cf : column_families) {
    s = ValidateOptions(db_opts, cf.options);
    if (!s.ok()) {
      return s;
    }
  }
  return Status::OK();
}

// options/options_helper.cc
Status ValidateOptions(const DBOptions& db_opts,
                       const ColumnFamilyOptions& cf_opts) {
  Status s;
#ifndef ROCKSDB_LITE
  auto db_cfg = DBOptionsAsConfigurable(db_opts);
  auto cf_cfg = CFOptionsAsConfigurable(cf_opts);
  s = db_cfg->ValidateOptions(db_opts, cf_opts);
  if (s.ok()) s = cf_cfg->ValidateOptions(db_opts, cf_opts);
#else
  s = cf_opts.table_factory->ValidateOptions(db_opts, cf_opts);
#endif
  return s;
}
```

在调用到`options_helper`中的`ValidateOptions`函数时, 就会构造`DBOptions`和`ColumnFamilyOptions`相应的`Configurable`对象了, 也就是在这里完成了配置的注册. 最终调用`Configurable::ValidateOptions`函数进行校验, 检查参数是否合法.

## 不同options的构造和转换

最后, 我们提下不同options之间的构造和转换的方法, 这些方法散落在很多文件中.

- 能从`DBOptions`构造`ImmutableDBOptions`和`MutableDBOptions`, 反之则不行.

```cpp
struct ImmutableDBOptions {
  ImmutableDBOptions();
  explicit ImmutableDBOptions(const DBOptions& options);
  // ...
};

struct MutableDBOptions {
  MutableDBOptions();
  explicit MutableDBOptions(const DBOptions& options);
  // ...
};
```

- `ColumnFamilyOptions`也类似

```cpp
struct ImmutableCFOptions {
  ImmutableCFOptions();
  explicit ImmutableCFOptions(const ColumnFamilyOptions& cf_options);
  // ...
};

struct MutableCFOptions {
  MutableCFOptions();
  explicit MutableCFOptions(const ColumnFamilyOptions& options);
  // ...
};
```

- 一开始我们提到提供了从string或者一个unordered_map解析出相应的配置的方法. 实际上`DBOptions`/`ColumnFamilyOptions`都提供了和string或者unordered_map互相转换的函数:

```cpp
Status GetColumnFamilyOptionsFromMap(
    const ConfigOptions& config_options,
    const ColumnFamilyOptions& base_options,
    const std::unordered_map<std::string, std::string>& opts_map,
    ColumnFamilyOptions* new_options);

Status GetDBOptionsFromMap(
    const ConfigOptions& cfg_options, const DBOptions& base_options,
    const std::unordered_map<std::string, std::string>& opts_map,
    DBOptions* new_options);

Status GetColumnFamilyOptionsFromString(const ConfigOptions& config_options,
                                        const ColumnFamilyOptions& base_options,
                                        const std::string& opts_str,
                                        ColumnFamilyOptions* new_options);

Status GetDBOptionsFromString(const ConfigOptions& config_options,
                              const DBOptions& base_options,
                              const std::string& opts_str,
                              DBOptions* new_options);
```

- 对于mutable的options, 额外提供了和string互相转换的函数. 准确说能够从unordered_map构造相应的options对象, 而从options对象则只能转换成string.

```cpp
Status GetMutableDBOptionsFromStrings(
    const MutableDBOptions& base_options,
    const std::unordered_map<std::string, std::string>& options_map,
    MutableDBOptions* new_options) {
  assert(new_options);
  *new_options = base_options;
  ConfigOptions config_options;
  Status s = OptionTypeInfo::ParseType(
      config_options, options_map, db_mutable_options_type_info, new_options);
  if (!s.ok()) {
    *new_options = base_options;
  }
  return s;
}

Status GetStringFromMutableDBOptions(const ConfigOptions& config_options,
                                     const MutableDBOptions& mutable_opts,
                                     std::string* opt_string) {
  return OptionTypeInfo::SerializeType(
      config_options, db_mutable_options_type_info, &mutable_opts, opt_string);
}
```

```cpp
Status GetMutableOptionsFromStrings(
    const MutableCFOptions& base_options,
    const std::unordered_map<std::string, std::string>& options_map,
    Logger* /*info_log*/, MutableCFOptions* new_options) {
  assert(new_options);
  *new_options = base_options;
  ConfigOptions config_options;
  Status s = OptionTypeInfo::ParseType(
      config_options, options_map, cf_mutable_options_type_info, new_options);
  if (!s.ok()) {
    *new_options = base_options;
  }
  return s;
}

Status GetStringFromMutableCFOptions(const ConfigOptions& config_options,
                                     const MutableCFOptions& mutable_opts,
                                     std::string* opt_string) {
  assert(opt_string);
  opt_string->clear();
  return OptionTypeInfo::SerializeType(
      config_options, cf_mutable_options_type_info, &mutable_opts, opt_string);
}
```

- 在`options/options_helper.h`中还提供了一些辅助函数, 比如根据`ImmutableDBOptions`和`MutableDBOptions`构造`DBOptions`的方法.

```cpp
DBOptions BuildDBOptions(const ImmutableDBOptions& immutable_db_options,
                         const MutableDBOptions& mutable_db_options);

ColumnFamilyOptions BuildColumnFamilyOptions(
    const ColumnFamilyOptions& ioptions,
    const MutableCFOptions& mutable_cf_options);

void UpdateColumnFamilyOptions(const ImmutableCFOptions& ioptions,
                               ColumnFamilyOptions* cf_opts);
void UpdateColumnFamilyOptions(const MutableCFOptions& moptions,
                               ColumnFamilyOptions* cf_opts);
```
