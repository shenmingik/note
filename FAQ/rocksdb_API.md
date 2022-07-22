- [Open&Close 数据库](#openclose-数据库)
- [Write 数据库](#write-数据库)
  - [Put 单次写](#put-单次写)
  - [WriteBatch 批量写](#writebatch-批量写)
  - [Sync 同步写](#sync-同步写)
- [Read 数据库](#read-数据库)
  - [Get 单次读](#get-单次读)
  - [MutiGet](#mutiget)
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
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
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

 // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
}
```
## Sync 同步写
```cpp
#include <rocksdb/db.h>
#include <rocksdb/write_batch.h>

int main() {
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
  //Write 接口
  //默认情况下rocksdb的写操作是异步的，如果想要同步写，需要指定写选项为sync
  rocksdb::WriteOptions write_opt;
  write_opt.sync = true;
  string key = "11";
  string value = to_string(random() % 100000);
  status = db->Put(write_opt, key, value);

 // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
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
  // 打开一个数据库的句柄
  rocksdb::DB *db;
  rocksdb::Options opt;
  opt.create_if_missing = true;       //如果不存在就创建数据库
  
  rocksdb::Status status;
  status = rocksdb::DB::Open(opt, "./file/testdb", &db);
  
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

 // 关闭打开的句柄
  status = db->Close();
  if (status.ok()) {
    delete db;
  }
}
```
# Merge
