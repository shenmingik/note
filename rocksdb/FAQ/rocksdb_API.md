- [Open&Close 数据库](#openclose-数据库)
- [Write 数据库](#write-数据库)
  - [Put 单次写](#put-单次写)
  - [WriteBatch 批量写](#writebatch-批量写)
  - [Sync 同步写](#sync-同步写)
- [Read 数据库](#read-数据库)
  - [Get 单次读](#get-单次读)
  - [MutiGet](#mutiget)
- [ColumnFamily 操作](#columnfamily-操作)
  - [创建列族](#创建列族)
  - [删除列族](#删除列族)
  - [以列族的形式打开DB](#以列族的形式打开db)
  - [列族读写](#列族读写)
- [Merge](#merge)
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
# Write 数据库
## Put 单次写
```cpp
#include <rocksdb/db.h>


int main() {
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
  //Put 接口
  for (int i = 0; i < AUTO_PRODUCE; i++) {
    if (status.ok()) {
      rocksdb::WriteOptions write_opt;
      string value = to_string(random() % 100000);
      status = db->Put(rocksdb::WriteOptions(), to_string(i), value);
      cout << "write key: " << to_string(i) << "     value: " << value << endl;
    }
  }

 // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
}
```
## WriteBatch 批量写
```cpp
#include <rocksdb/db.h>
#include <rocksdb/write_batch.h>

int main() {
  
  //Write 接口
  if(status.ok())
  {
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
  }
}
```
## Sync 同步写
```cpp
#include <rocksdb/db.h>
#include <rocksdb/write_batch.h>

int main() {
  
  //Write 接口
  //默认情况下rocksdb的写操作是异步的，如果想要同步写，需要指定写选项为sync
  rocksdb::WriteOptions write_opt;
  write_opt.sync = true;
  string key = "11";
  string value = to_string(random() % 100000);
  status = db->Put(write_opt, key, value);
}
```

# Read 数据库
## Get 单次读
```cpp
#include <rocksdb/db.h>

int main() {
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
  //Write 接口
  string key = "1";
  if(status.ok())
  {
      status = db->Get(rocksdb::ReadOptions(), key, &new_val);
      cout << "read key: " << key << "     value: " << new_val << endl;
  }

 // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
}
```
## MutiGet
```cpp
#include <rocksdb/db.h>


int main() {
  
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

}
```
# ColumnFamily 操作
## 创建列族
```cpp
#include <rocksdb/db.h>

int main() {
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
  //创建新列族
  if(status.ok())
  {
    rocksdb::ColumnFamilyHandle *handle;
    status = db->CreateColumnFamily(rocksdb::ColumnFamilyOptions(), "new_cf", &handle);
  }

  //记得要关闭列族资源
  status = db->DestroyColumnFamilyHandle(handle);

  // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
}
```
## 删除列族
```cpp
#include <rocksdb/db.h>

int main() {
  status = db->DropColumnFamily(handles[1]);
}
```
## 以列族的形式打开DB
```cpp
#include <rocksdb/db.h>

int main() {
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
}
```
## 列族读写
```cpp
#include <rocksdb/db.h>

int main() {
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
}
```


# Merge
