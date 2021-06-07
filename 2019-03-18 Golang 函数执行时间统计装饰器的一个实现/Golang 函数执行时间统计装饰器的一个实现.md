|作者|版本号|时间|
|:-|:-|:-|
|Coordinate35| v1.0.0| 2019-03-18|


## 背景

最近在搭一个新项目的架子，在生产环境中，为了能实时的监控程序的运行状态，少不了逻辑执行时间长度的统计。时间统计这个功能实现的期望有下面几点：

1. 实现细节要剥离：时间统计实现的细节不期望在显式的写在主逻辑中。因为主逻辑中的其他逻辑和时间统计的抽象层次不在同一个层级
2. 用于时间统计的代码可复用
3. 统计出来的时间结果是可被处理的。
4. 对并发编程友好

## 实现思路

### 统计细节的剥离

最朴素的时间统计的实现，可能是下面这个样子：

``` go
func f() {
  startTime := time.Now()
  logicStepOne()
  logicStepTwo()
  endTime := time.Now()
  timeDiff := timeDiff(startTime, endTime)
  log.Info("time diff: %s", timeDiff)
}
```

《代码整洁之道》告诉我们：一个函数里面的所有函数调用都应该处于同一个抽象层级。

在这里时间开始、结束的获取，使用时间的求差，属于时间统计的细节，首先他不属于主流程必要的一步，其次他们使用的函数 time.Now() 和 logicStepOne, logicStepTwo 并不在同一个抽象层级。

因此比较好的做法应该是把时间统计放在函数 f 的上层，比如：

```go
func doFWithTimeRecord() {
  startTime: = time.Now()
  f()
  endTime := Time.Now()
  timeDiff := timeDIff(startTime, endTime)
  log.Info("time diff: %s", timeDiff)
}
```

### 时间统计代码可复用&统计结果可被处理&不影响原函数的使用方式

我们虽然达成了函数内抽象层级相同的目标，但是大家肯定也能感受到：这个函数并不好用。

原因在于，我们把要调用的函数 f 写死在了 doFWithTimeRecord 函数中。这意味着，每一个要统计时间的函数，我都需要实现一个 doXXWithTimeRecord, 而这些函数里面的逻辑是相同的，这就违反了我们 DRY（Don't Repeat Yourself）原则。因此为了实现逻辑的复用，我认为装饰器是比较好的实现方式：将要执行的函数作为参数传入到时间统计函数中。

#### 举个网上看到的例子

实现一个功能，第一反应肯定是查找同行有没有现成的轮子。不过看了下，没有达到自己的期望，举个例子：

```go
type SumFunc func(int64, int64) int64

func timedSumFunc(f SumFunc) SumFunc {
  return func(start, end int64) int64 {
    defer func(t time.Time) {
      fmt.Printf("--- Time Elapsed: %v ---\n", time.Since(t))
    }(time.Now())
    
    return f(start, end)
  }
}
```

说说这段代码不好的地方：

1. 这个装饰器入参写死了函数的类型：

   ```go
   type SumFunc func(int64, int64) int64
   ```

   也就是说，只要换一个函数，这个装饰器就不能用了，这不符合我们的第2点要求

2. 这里时间统计结果直接打印到了标准输出，也就是说这个结果是不能被原函数的调用方去使用的：因为只有掉用方，才知道这个结果符不符合预期，是花太多时间了，还是正常现象。这不符合我们的第3点要求。

#### 怎么解决这两个问题呢？

这个时候，《重构，改善既有代码的设计》告诉我们：Replace Method with Method Obejct——以函数对象取代函数。他的意思是当一个函数有比较复杂的临时变量时，我们可以考虑将函数封装成一个类。这样我们的函数就统一成了 0 个参数。（当然，原本就是作为一个 struct 里面的方法的话就适当做调整就好了）

现在，我们的代码变成了这样：

```go
type TimeRecorder interface {
  SetCost(time.Duration)
  TimeCost() time.Duration
}

func TimeCostDecorator(rec TimeRecorder, f func()) func() {
  return func() {
    startTime := time.Now()
    f()
    endTime := time.Now()
    timeCost := endTime.Sub(startTime)
    rec.SetCost(timeCost)
  }
}
```

这里入参写成是一个 interface ，目的是允许各种函数对象入参，只需要实现了 SetCost 和 TimeCost 方法即可

## 对并发编程友好

最后需要考虑的一个问题，很多时候，一个类在整个程序的生命周期是一个单例，这样在 SetCost 的时候，就需要考虑并发写的问题。这里考虑一下几种解决方案：

1. 使用装饰器配套的时间统计存储对象，实现如下：

   ```go
   
   func NewTimeRecorder() TimeRecorder {
     return &timeRecorder{}
   }
   
   type timeRecorder struct {
     cost time.Duration
   }
   
   func (tr *timeRecorder) SetCost(cost time.Duration) {
     tr.cost = cost
   }
   
   func (tr *timeRecorder) Cost() time.Duration {
     return tr.cost
   }
   ```

2. 抽离出存粹的执行完就可以销毁的函数对象，每次要操作的时候都 new 一下

3. 函数对象内部对 SetCost 函数实现锁机制

这三个方案是按推荐指数从高到低排序的，因为我个人认为：资源允许的情况下，尽量保持对象不可变；同时怎么统计、存储使用时长其实是统计时间模块自己的事情。

## 单元测试

最后补上单元测试：

```go
func TestTimeCostDecorator(t *testing.T) {
  testFunc := func() {
    time.Sleep(time.Duration(1) * time.Second)
  }
  
  type args struct {
    rec TimeRecorder
    f func()
  }
  
  tests := []struct {
    name string
    args args
  }{
    {
      "test time cost decorator",
      args{
        NewTimeRecorder(),
        testFunc,
      },
    },
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      got := TimeCostDecorator(tt.args.rec, tt.args.f)
      got()
      if tt.args.rec.Cost().Round(time.Second) != time.Duration(1) * time.Second.Round(time.Second) {
        "Record time cost abnormal, recorded cost: %s, real cost: %s",
        tt.args.rec.Cost().String(),
        tt.Duration(1) * time.Second,
      }
    }) 
  }
}
```

测试通过，验证了时间统计是没问题的。至此，这个时间统计装饰器就介绍完了。如果这个实现有什么问题，或者大家有更好的实现方式，欢迎大家批评指正与提出～