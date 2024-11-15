## ***`errorGroup`***

[^Go常用协程并发库]: 

* ​	**`sync.WaitGroup`模式,通过`wg. Done`,`wg.Wait`,**``wg.Add`**实现如下案列：但是当在项目中涉及大量使用时就显得不那么方便**.

```go
var wg sync.WaitGroup
	for _, id := range ids {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			data, err := queryDatabase(id)
			if err != nil {
				fmt.Printf("Error querying ID %d: %v\n", id, err)
				return
			}
			resultsChan <- data
		}(id)
	}
    wg.Wait()
```

​	**对比使用errorGroup处理的并发如下,对此errorGroup进行了简单的封装，我们不必考虑Done和Add的使用。该Done和Add的地方都替我们处理完成了.**

* **线程安全**
  1. **数组与切片**：使用锁或 Channel 来同步访问。
  2. **Map（字典）**：使用 `sync.Mutex` 或 `sync.Map` 来保证线程安全。

```go
var (
    wg    errgroup.Group
    users []*model.User
    mutex sync.Mutex
)
for _, uid := range userIds {
    uid := uid
    wg.Go(func() error {
       user, err := l.svcCtx.UserModel.FindOne(ctx, uid)
       if err != nil {
          return err
       }
       mutex.Lock()
       defer mutex.Unlock()
       users = append(users, user)
       return nil
    })
}
wg.Wait()
```

## `mr.MapReduce`

* go-zero中主要用来对批量数据进行并发的处理，以此来提升服务的性能

***generate data***

```go
type (
	// GenerateFunc is used to let callers send elements into source.
	GenerateFunc func(source chan<- interface{})
)
```

***mapper  处理 generate 生产的数据进行处理***

```go
 
type (
	// MapFunc is used to do element processing and write the output to writer.
	MapFunc func(item interface{}, writer Writer)
	// MapperFunc is used to do element processing and write the output to writer,
	// use cancel func to cancel the processing.
	MapperFunc func(item interface{}, writer Writer, cancel func(error))
)
```

***reducer 对mapper处理后的数据做聚合返回***

```go
 
type (
	// ReducerFunc is used to reduce all the mapping output and write to writer,
	// use cancel func to cancel the processing.
	ReducerFunc func(pipe <-chan interface{}, writer Writer, cancel func(error))
	// VoidReducerFunc is used to reduce all the mapping output, but no output.
	// Use cancel func to cancel the processing.
	VoidReducerFunc func(pipe <-chan interface{}, cancel func(error))
)
```

* ***由于刚接触mr包，不涉及其实现原理 做一点使用上的心得,这是我使用mr的一点小案列，个人简单总结为***

* ***1.`mr.MapReduce` *定义参数，以下面代码举列,第一个泛型参数，表示每次调用 Map 阶段时传递给每个 goroutine 的数据类型。***

* ***2.第二，三泛型参数为mapper函数和result函数的返回结果***

* ### `mr.MapReduce` 的工作流程

  1. **Mapper 阶段**：对于每个 `userId`，都会启动一个并发的 goroutine 来执行与 `id` 相关的任务（比如从数据库获取用户信息）。这时，`int64`（`userId`）是每个并发任务的输入。
  2. **Reducer 和 Collect 阶段**：Map 阶段完成后，`Reducer` 负责聚合从 `Mapper` 输出的 `*model.User` 数据（通过 `writer.Write(user)`）。最后，`Collect` 阶段会把所有的用户信息合并成一个 `[]*model.User`😘

```go
func (l *MapReduceLogic) MapreduceSpend(ctx context.Context, userIds []int64) ([]*model.User, error) {
	users, err := mr.MapReduce[int64, *model.User, []*model.User](func(source chan<- int64) {
		for _, uid := range userIds {
			source <- uid
		}
	}, func(id int64, writer mr.Writer[*model.User], cancel func(err error)) {
		user, err := l.svcCtx.UserModel.FindOne(ctx, id)
		if err != nil {
			cancel(err)
			return
		}
		writer.Write(user)
	}, func(pipe <-chan *model.User, writer mr.Writer[[]*model.User], cancel func(err error)) {
		var users []*model.User
		for user := range pipe {
			users = append(users, user)
		}
		writer.Write(users)
	},
	)
	if err != nil {
		return nil, err
	}
	return users, err
}
```

