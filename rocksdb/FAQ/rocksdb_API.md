- [Open&Close 数据库](#openclose-数据库)
- [Options](#options)
  - [加载上次配置](#加载上次配置)
- [Write 数据库](#write-数据库)
  - [Put 单次写](#put-单次写)
  - [WriteBatch 批量写](#writebatch-批量写)
  - [Sync 同步写](#sync-同步写)
  - [批量导入至sst](#批量导入至sst)
    - [SST导入rocksdb](#sst导入rocksdb)
- [Read 数据库](#read-数据库)
  - [Get 单次读](#get-单次读)
  - [MutiGet](#mutiget)
- [ColumnFamily 操作](#columnfamily-操作)
  - [创建列族](#创建列族)
  - [删除列族](#删除列族)
  - [以列族的形式打开DB](#以列族的形式打开db)
  - [列族读写](#列族读写)
- [Merge](#merge)
- [迭代器](#迭代器)
  - [全部遍历](#全部遍历)
  - [部分遍历](#部分遍历)
  - [反向遍历](#反向遍历)
- [快照](#快照)
- [比较器](#比较器)
- [备份](#备份)
  - [生成备份](#生成备份)
  - [加载备份文件](#加载备份文件)
- [检查点](#检查点)
- [rocksdb Tool](#rocksdb-tool)
  - [数据访问和管理工具 —— lab](#数据访问和管理工具--lab)
  - [SST访问工具 —— SST_dump](#sst访问工具--sst_dump)
```cpp
```
# Open&Close 数据库
```cpp
#include <rocksdb/db.h>

int main() {
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
  ...

 // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
}
```
# Options
## 加载上次配置

```cpp
  rocksdb::DBOptions loaded_db_opt;
  std::vector<rocksdb::ColumnFamilyDescriptor> loaded_cf_descs;

  rocksdb::LoadLatestOptions("./test", rocksdb::Env::Default(), &loaded_db_opt,
                             &loaded_cf_descs);
  vector<rocksdb::ColumnFamilyHandle*> handler;

  /*
   * 以下用户自定义的函数和类型指针必须在初始化时默认指定
   * env
   * memtable_factory
   * compaction_filter_factory
   * prefix_extractor
   * comparator
   * merge_operator
   * compaction_filter
   * cache in BlockBasedTableOptions
   * table_factory other than BlockBasedTableFactory
   */

  //以下是一个例子
  loaded_cf_descs[0].options.compaction_filter = new MyCompactionFilter();

  //检查参数
  rocksdb::Status status = rocksdb::CheckOptionsCompatibility("./test", rocksdb::Env::Default(), loaded_db_opt, loaded_cf_descs);

  status = rocksdb::DB::Open(loaded_db_opt, "./test",
                                             loaded_cf_descs, &handler, &db);
```

# Write 数据库
## Put 单次写
```cpp
  //Put 接口
  for (int i = 0; i < AUTO_PRODUCE; i++) {
    if (status.ok()) {
      rocksdb::WriteOptions write_opt;
      string value = to_string(random() % 100000);
      status = db->Put(rocksdb::WriteOptions(), to_string(i), value);
      cout << "write key: " << to_string(i) << "     value: " << value << endl;
    }
  }
```
## WriteBatch 批量写
```cpp
#include <rocksdb/write_batch.h>


  //Write 接口
  string key = "1";
  string val = "4664";
  string new_val;
  rocksdb::WriteBatch write_batch;
  write_batch.Delete(key);
  write_batch.Put(key,val);
  status = db->Write(rocksdb::WriteOptions(), &write_batch);
  if(status.ok())
  {
    status = db->Get(rocksdb::ReadOptions(), key, &new_val);
    cout << "read key: " << key << "     value: " << new_val << endl;
  }
```
## Sync 同步写
```cpp
  //Write 接口
  //默认情况下rocksdb的写操作是异步的，如果想要同步写，需要指定写选项为sync
  rocksdb::WriteOptions write_opt;
  write_opt.sync = true;
  string key = "11";
  string value = to_string(random() % 100000);
  status = db->Put(write_opt, key, value);
```

## 批量导入至sst

```cpp
  //下面的代码会以最快的速度写入到一个sst文件，之后将这个sst导入rocksdb即可
  //注意以下两点：
  /*
   * 1. 比较器必须和已打开的数据库完全相同
   * 2. 行必须按顺序插入
   */
  rocksdb::Options opt;
  opt.create_if_missing = true;
  rocksdb::SstFileWriter sst_writer(rocksdb::EnvOptions(), opt);
  //此时需要确保这个文件是存在的，如果不存在手动在响应路径创建
  string file = "./file/file.sst";

  rocksdb::Status status = sst_writer.Open(file);
 
  for (int i = 0; i < AUTO_PRODUCE;i++)
  {
    sst_writer.Put(to_string(i), to_string(random() % 10000));
  }

  sst_writer.Finish();
```
### SST导入rocksdb

```cpp
  vector<string> files;
  files.push_back("./file/file.sst");

  status = db->IngestExternalFile(files, rocksdb::IngestExternalFileOptions());
```

# Read 数据库
## Get 单次读
```cpp

  std::string new_value;
  //Write 接口
  string key = "1";
  if(status.ok())
  {
      status = db->Get(rocksdb::ReadOptions(), key, &new_val);
      cout << "read key: " << key << "     value: " << new_val << endl;
  }

```
## MutiGet
```cpp
  //MultiGet 使用
  vector<string> str_keys;
  vector<rocksdb::Slice> keys;
  vector<string> values;
  for (int i = 0; i < AUTO_PRODUCE; i++) {
    str_keys.push_back(to_string(i));
  }

  for (int i = 0; i < AUTO_PRODUCE; i++)
  {
    keys.emplace_back(rocksdb::Slice(str_keys[i]));
  }


  if(status.ok())
  {
    db->MultiGet(rocksdb::ReadOptions(), keys,&values);
  }

  for(auto& str: values)
  {
    cout << str << " ";
  }
  cout << endl;
```
# ColumnFamily 操作
## 创建列族
```cpp
  //创建新列族
  if(status.ok())
  {
    rocksdb::ColumnFamilyHandle *handle;
    status = db->CreateColumnFamily(rocksdb::ColumnFamilyOptions(), "new_cf", &handle);
  }

  //记得要关闭列族资源
  status = db->DestroyColumnFamilyHandle(handle);
```
## 删除列族
```cpp
  status = db->DropColumnFamily(handles[1]);
```
## 以列族的形式打开DB
```cpp
  //创建“default”、“new_cf”这两种列族的数据库
  vector<rocksdb::ColumnFamilyDescriptor> vec_cf_desc;
  vec_cf_desc.push_back(rocksdb::ColumnFamilyDescriptor(
      ROCKSDB_NAMESPACE::kDefaultColumnFamilyName,
      rocksdb::ColumnFamilyOptions()));
  vec_cf_desc.push_back(rocksdb::ColumnFamilyDescriptor(
      "new_cf", rocksdb::ColumnFamilyOptions()));

  vector<rocksdb::ColumnFamilyHandle *> handles;
  status = rocksdb::DB::Open(opt, "./file/testdb", vec_cf_desc, &handles, &db);
  if (!status.ok()) {
    cerr << status.ToString() << endl;
    exit(0);
  }
```
## 列族读写
```cpp
  //创建“default”、“new_cf”这两种列族的数据库
  vector<rocksdb::ColumnFamilyDescriptor> vec_cf_desc;
  vec_cf_desc.push_back(rocksdb::ColumnFamilyDescriptor(
      ROCKSDB_NAMESPACE::kDefaultColumnFamilyName,
      rocksdb::ColumnFamilyOptions()));
  vec_cf_desc.push_back(rocksdb::ColumnFamilyDescriptor(
      "new_cf", rocksdb::ColumnFamilyOptions()));

  vector<rocksdb::ColumnFamilyHandle *> handles;
  status = rocksdb::DB::Open(opt, "./file/testdb", vec_cf_desc, &handles, &db);
  if (!status.ok()) {
    cerr << status.ToString() << endl;
    exit(0);
  }

  //读列族defalut之中key为1的value信息
  string key = "1";
  string value;
  status = db->Get(rocksdb::ReadOptions(), handles[0], key, &value);
  cout << "read key: " << key << "     value: " << value << endl;

  //往列族new_cf之中写入key和value
  status = db->Put(WriteOptions(), handles[1], Slice("key"), Slice("value"));
```


# Merge
```cpp
#include <rocksdb/merge_operator.h>
#include <rocksdb/db.h>

//对原有的值进行累加运算
class MyMergeOperator : public rocksdb::MergeOperator {
 public:
  virtual const char* Name() const override { return "MyMergeOperator"; }

  virtual bool FullMerge(const rocksdb::Slice& key,
                         const rocksdb::Slice* existing_value,
                         const std::deque<std::string>& operand_list,
                         std::string* new_value,
                         rocksdb::Logger* logger) const override {
    int current_val = atoi(existing_value->data());

    for (auto& oper : operand_list) {
      current_val += atoi(oper.c_str());
    }

    new_value->assign(to_string((current_val)));
    string log_str = "FullMerge: " + *new_value;
    rocksdb::Log(logger, log_str.c_str());
    return true;
  }

  virtual bool PartialMerge(const rocksdb::Slice& key,
                            const rocksdb::Slice& left_operand,
                            const rocksdb::Slice& right_operand,
                            std::string* new_value,
                            rocksdb::Logger* logger) const override {
    int left_value = atoi(left_operand.data());
    int right_value = atoi(right_operand.data());

    new_value->assign(to_string(left_value + right_value));

    string log_str = "PartialMerge: " + *new_value;
    rocksdb::Log(logger, log_str.c_str());
    return true;
  }
};

int main() {
  rocksdb::DB* db;
  rocksdb::Options opt;
  opt.create_if_missing = true;
  //一旦以merge_operator!=nullptr 的方式打开db，之后的操作就必须指定merge_operator，否则会影响遍历或者Get操作
  opt.merge_operator.reset(new MyMergeOperator());

  rocksdb::Status status = rocksdb::DB::Open(opt, "./test", &db);

  db->Put(rocksdb::WriteOptions(), "1", "100");
  std::string value;
  db->Get(rocksdb::ReadOptions(), "1", &value);
  cout << value << endl;  // 100

  db->Merge(rocksdb::WriteOptions(), "1", "1");
  db->Merge(rocksdb::WriteOptions(), "1", "2");
  db->Merge(rocksdb::WriteOptions(), "1", "3");
  db->Merge(rocksdb::WriteOptions(), "1", "4");
  value.clear();
  db->Get(rocksdb::ReadOptions(), "1", &value);
  cout << value << endl;  // 110

  db->Close();
  delete db;
}
```
# 迭代器

## 全部遍历
```cpp
  rocksdb::Iterator* iter = db->NewIterator(rocksdb::ReadOptions());
  for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
    cout << "key: " << iter->key().ToString() << " value: " << iter->value().ToString() << endl;
  }
```
## 部分遍历

```cpp
  rocksdb::Iterator* iter = db->NewIterator(rocksdb::ReadOptions());
  for (iter->Seek("0"); iter->Valid() && iter->key().ToString() <= "5";
       iter->Next()) {
    cout << "key: " << iter->key().ToString()
         << " value: " << iter->value().ToString() << endl;
  }
```
## 反向遍历

```cpp
  rocksdb::Iterator* iter = db->NewIterator(rocksdb::ReadOptions());
  for (iter->SeekToLast();iter->Valid();iter->Prev())
  {
    cout << "key: " << iter->key().ToString()
         << " value: " << iter->value().ToString() << endl;
  }
```
# 快照
```cpp
  rocksdb::ReadOptions readopt;
  //生成此时的一致性快照，此时key = 1,value = 886
  readopt.snapshot = db->GetSnapshot();
  rocksdb::Iterator* iter = db->NewIterator(readopt);
  db->Put(rocksdb::WriteOptions(), "1", "1433223");
  // 下面读到的仍是key = 1,value = 886
  for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
    cout << "key: " << iter->key().ToString()
         << " value: " << iter->value().ToString() << endl; 
  }
  //记得释放资源
  delete iter;
  db->ReleaseSnapshot(readopt.snapshot);
```

# 比较器
```cpp
class MyComparator : public rocksdb::Comparator {
 public:
  //按照逆字典序的方式排序key
  int Compare(const rocksdb::Slice& a, const rocksdb::Slice& b) const {
    int ai, bi;
    ai = atoi(a.data());
    bi = atoi(b.data());
    if (ai > bi) return 1;
    if (ai < bi) return -1;
    return 0;
  }

  // Ignore the following methods for now:
  const char* Name() const { return "MyComparator"; }
  void FindShortestSeparator(std::string*, const rocksdb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};

int main() {
  MyComparator my_comparator;
  rocksdb::DB* db;
  rocksdb::Options opt;
  opt.create_if_missing = true;
  opt.comparator = &my_comparator;

  //注意，不能在创建数据库之后再更改比较器
  rocksdb::Status status = rocksdb::DB::Open(opt, "./new_test", &db);
}
```

# 备份
## 生成备份
```cpp
#include <rocksdb/utilities/backup_engine.h>

  rocksdb::BackupEngine *backup_engine;
  //此时需要确保backup这个目录的存在
  status = rocksdb::BackupEngine::Open(
      rocksdb::Env::Default(), rocksdb::BackupEngineOptions("./backup/test"),
      &backup_engine);
  if (!status.ok()) {
    cerr << status.ToString() << endl;
  }
  //产生两个备份
  status = backup_engine->CreateNewBackup(db);
  db->Put(rocksdb::WriteOptions(), "21", "21");
  status = backup_engine->CreateNewBackup(db);

  vector<rocksdb::BackupInfo> backup_info_vec;
  backup_engine->GetBackupInfo(&backup_info_vec);
  for(auto& it:backup_info_vec)
  {
    cout << it.size << endl;
    //验证备份的合法性
    status = backup_engine->VerifyBackup(it.backup_id);
    if(!status.ok())
    {
      cerr << status.ToString() << endl;
    }
  }

  delete backup_engine;
```
## 加载备份文件

```cpp
  rocksdb::BackupEngineReadOnly *backup_engine;
  //此时需要确保backup这个目录的存在
  status = rocksdb::BackupEngineReadOnly::Open(
      rocksdb::Env::Default(), rocksdb::BackupEngineOptions("./backup/test"),
      &backup_engine);

  //加载备份文件(包括WAL)至./backup/test_restore
  status = backup_engine->RestoreDBFromBackup(1, "./backup/test_restore",
                                              "./backup/test_restore");

  delete backup_engine;
```
# 检查点

```cpp
//创建一个检查点对象
Status Create(DB* db, Checkpoint** checkpoint_ptr);

//创建检查点

Status CreateCheckpoint(const std::string& checkpoint_dir);
```

# rocksdb Tool

## 数据访问和管理工具 —— lab
```bash
# ldb一般在rocksdb的build目录中的tools目录下
# 基本格式 ./ldb --db=[DB_PATH] [COMMAND] xxx
```

```bash
# 读取
./ldb --db=/home/ik/workspace/lab/cpp/note/rocksdb/bin/test get 1
```

```bash
# 写入
./ldb --db=/home/ik/workspace/lab/cpp/note/rocksdb/bin/test put 15 123456789

# 批量写入
./ldb --db=/home/ik/workspace/lab/cpp/note/rocksdb/bin/test batchput 16 1 17 1 18 1
```

```bash
# scan 
./ldb --db=/home/ik/workspace/lab/cpp/note/rocksdb/bin/test scan 1 10
```

```bash
# compact
./ldb --db=/home/ik/workspace/lab/cpp/note/rocksdb/bin/test compact --compression_type=bzip2 --block_size=65536 --try_load_options=true --column_family=new_cf
```

```bash
# 修复DB，此时需要保证MANIFEST文件不存在
ldb repair --db=test
```

## SST访问工具 —— SST_dump
```bash
# 解析sst文件为txt格式
sst_dump --file=test/000093.sst --command=raw
```

```bash
# 打印这个sst文件的前五个key
sst_dump --file=test/000093.sst --command=scan --read_num=5
```

```bash
# 验证sst文件的合法性
sst_dump --file=test/000127.sst --command=check --verify_checksum
```

```bash
# 读取文件的元信息
sst_dump --file=test/000127.sst --show_properties
```
