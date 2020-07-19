# Project1 StandaloneKV 
+ 第一个项目实现的是一个单节点的键值存储，并且用的是`column family`。
+ 这个存储系统支持四个基本的操作：`Put/Delete/Get/Scan`。
+ 需要完成两个部分：
    1. 实现单节点的引擎
    2. 实现键值存储的处理函数

## 预备知识
+ 什么是`column family`？
```
简单来说就是`key`的命名空间，详细说明可以参考Google的论文`BigTable`(https://blog.csdn.net/harrytsz/article/details/82082320)
```

## 第一步：看代码
了解了要实现什么之后，就看是要简单了解一下已经有的项目代码。

`kv`目录下（Project1只需要关注下面的目录，其它的我也没仔细看=_=）：
+ `conf`：配置相关结构体
+ `server`：该目录下的`server.go`是需要补充的文件之一，`Server`结构体是与客户端链接的节点，
需要实现上面讲过的`Put/Delete/Get/Scan`这几个操作。还有`server_test.go`这个是测试文件，
错误都需要这找原因。
+ `storage`：主要关注`modify.go`、`storage.go`和`standalone_storage/standalone_storage.go`，
这也是上述提到的引擎部分，`Server`结构体中的`storage`也就是这里的`standalone_storage`
+ `util`：重点关注`engine_util`下的几个文件，这里面的几个函数需要用到。

## 第二步：跑代码
```
make project1
```
你会发现第一个测试就panic了，以第一个测试为例：
```go
func TestRawGet1(t *testing.T) {
	conf := config.NewTestConfig()
	s := standalone_storage.NewStandAloneStorage(conf)
	s.Start()
	server := NewServer(s)
	defer cleanUpTestData(conf)
	defer s.Stop()

	cf := engine_util.CfDefault
	Set(s, cf, []byte{99}, []byte{42})

	req := &kvrpcpb.RawGetRequest{
		Key: []byte{99},
		Cf:  cf,
	}
	resp, err := server.RawGet(nil, req)
	assert.Nil(t, err)
	assert.Equal(t, []byte{42}, resp.Value)
}

func Set(s *standalone_storage.StandAloneStorage, cf string, key []byte, value []byte) error {
	return s.Write(nil, []storage.Modify{
		{
			Data: storage.Put{
				Cf:    cf,
				Key:   key,
				Value: value,
			},
		},
	})
}
```
可以看出这个函数分为几步：
1. 获取配置
2. new一个standalone_storage
3. new一个server
4. `Set`方法调用的是standalone_storage的`Write`方法
5. 构建一个request然后测试server的`RawGet`方法
重点关注一下standalone_storage的`Write`方法，看下该函数：
```go
func (s *StandAloneStorage) Write(ctx *kvrpcpb.Context, batch []storage.Modify) error {
```
这个`Modify`长这样：
```go
// Modify is a single modification to TinyKV's underlying storage.
type Modify struct {
	Data interface{}
}

type Put struct {
	Key   []byte
	Value []byte
	Cf    string
}

type Delete struct {
	Key []byte
	Cf  string
}
```
可以看出`Modify`里的`Data`有两种情况，一种是`Put`，一种是`Delete`。
上面的`Set`函数就是构造了一个`Put`传进去，因此在实现`Write`方法是需要分两种情况处理。

## 第三步：写代码
只需要补充两个文件的代码：
1. standalone_storage.go
```go
type StandAloneStorage struct {
	// Your Data Here (1).
	engine *engine_util.Engines
	config *config.Config
}

func NewStandAloneStorage(conf *config.Config) *StandAloneStorage {
	// Your Code Here (1).
	dbPath := conf.DBPath
	kvPath := path.Join(dbPath, "kv")
	raftPath := path.Join(dbPath, "raft")

	kvEngine := engine_util.CreateDB(kvPath, conf)
	raftEngine := engine_util.CreateDB(raftPath, conf)

	return &StandAloneStorage{
		engine: engine_util.NewEngines(kvEngine, raftEngine, kvPath, ""),
		config: conf,
	}
}

func (s *StandAloneStorage) Start() error {
	// Your Code Here (1).
	return nil
}

func (s *StandAloneStorage) Stop() error {
	// Your Code Here (1).
	return s.engine.Close()
}

func (s *StandAloneStorage) Reader(ctx *kvrpcpb.Context) (storage.StorageReader, error) {
	// Your Code Here (1).
	return NewStandAloneReader(s.engine.Kv.NewTransaction(false)), nil
}

func (s *StandAloneStorage) Write(ctx *kvrpcpb.Context, batch []storage.Modify) error {
	// Your Code Here (1).
	for _, b := range batch {
		switch b.Data.(type) {
		case storage.Put:
			put := b.Data.(storage.Put)
			err := engine_util.PutCF(s.engine.Kv, put.Cf, put.Key, put.Value)
			if err != nil {
				return err
			}
		case storage.Delete:
			del := b.Data.(storage.Delete)
			err := engine_util.DeleteCF(s.engine.Kv, del.Cf, del.Key)
			if err != nil {
				return err
			}
		}
	}
	return nil
}

type StandAloneReader struct {
	kvTxn *badger.Txn
}

func NewStandAloneReader(kvTxn *badger.Txn) *StandAloneReader {
	return &StandAloneReader{
		kvTxn: kvTxn,
	}
}

func (s *StandAloneReader) GetCF(cf string, key []byte) ([]byte, error) {
	value, err := engine_util.GetCFFromTxn(s.kvTxn, cf, key)
	if err == badger.ErrKeyNotFound {
		return nil, nil
	}
	return value, err
}

func (s *StandAloneReader) IterCF(cf string) engine_util.DBIterator {
	return engine_util.NewCFIterator(cf, s.kvTxn)
}

func (s *StandAloneReader) Close() {
	s.kvTxn.Discard()
}
```
2. server.go
```go
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error) {
	// Your Code Here (1).
	reader, err := server.storage.Reader(req.Context)
	if err != nil {
		return &kvrpcpb.RawGetResponse{}, err
	}
	value, err := reader.GetCF(req.Cf, req.Key)
	if err != nil {
		return &kvrpcpb.RawGetResponse{}, err
	}
	resp := &kvrpcpb.RawGetResponse{
		Value:    value,
		NotFound: false,
	}
	if value == nil {
		resp.NotFound = true
	}
	return resp, nil
}

func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error) {
	// Your Code Here (1).
	put := storage.Put{
		Key:   req.Key,
		Value: req.Value,
		Cf:    req.Cf,
	}
	batch := storage.Modify{Data: put}

	err := server.storage.Write(req.Context, []storage.Modify{batch})
	if err != nil {
		return &kvrpcpb.RawPutResponse{}, err
	}
	return &kvrpcpb.RawPutResponse{}, nil
}

func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error) {
	// Your Code Here (1).
	del := storage.Delete{
		Key: req.Key,
		Cf:  req.Cf,
	}
	batch := storage.Modify{Data: del}

	err := server.storage.Write(req.Context, []storage.Modify{batch})
	if err != nil {
		return &kvrpcpb.RawDeleteResponse{}, err
	}
	return &kvrpcpb.RawDeleteResponse{}, nil
}

func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error) {
	// Your Code Here (1).
	reader, err := server.storage.Reader(req.Context)
	if err != nil {
		return &kvrpcpb.RawScanResponse{}, err
	}
	iter := reader.IterCF(req.Cf)
	iter.Seek(req.StartKey)
	var pairs []*kvrpcpb.KvPair
	limit := req.Limit
	for ; iter.Valid(); iter.Next() {
		item := iter.Item()
		val, _ := item.Value()
		pairs = append(pairs, &kvrpcpb.KvPair{
			Key:   item.Key(),
			Value: val,
		})

		limit--
		if limit == 0 {
			break
		}
	}
	resp := &kvrpcpb.RawScanResponse{
		Kvs: pairs,
	}
	return resp, nil
}
```

## 第四步：跑通测试
写代码的过程中难免会有错误，只需要通过测试不通过的部分进行debug就没啥问题了
```
$ make project1 
GO111MODULE=on go test -v --count=1 --parallel=1 -p=1 ./kv/server -run 1 
=== RUN   TestRawGet1
--- PASS: TestRawGet1 (0.82s)
=== RUN   TestRawGetNotFound1
--- PASS: TestRawGetNotFound1 (1.03s)
=== RUN   TestRawPut1
--- PASS: TestRawPut1 (0.84s)
=== RUN   TestRawGetAfterRawPut1
--- PASS: TestRawGetAfterRawPut1 (1.03s)
=== RUN   TestRawGetAfterRawDelete1
--- PASS: TestRawGetAfterRawDelete1 (0.82s)
=== RUN   TestRawDelete1
--- PASS: TestRawDelete1 (0.87s)
=== RUN   TestRawScan1
--- PASS: TestRawScan1 (1.04s)
=== RUN   TestRawScanAfterRawPut1
--- PASS: TestRawScanAfterRawPut1 (1.14s)
=== RUN   TestRawScanAfterRawDelete1
--- PASS: TestRawScanAfterRawDelete1 (0.76s)
=== RUN   TestIterWithRawDelete1
--- PASS: TestIterWithRawDelete1 (0.87s)
PASS
ok      github.com/pingcap-incubator/tinykv/kv/server   10.380s
```