|作者|版本号|时间|
|:-|:-|:-|
|Coordinate35| v1.0.0| 2019-08-28 |

## 单元测试是什么？

首先需要明确的就是，单元是什么？是一个函数？一个接口？还是一个模块？

这个可能每个人心中都用不同的定义。我比较赞同观点是：单元是指一段逻辑。

因此，单元测试就是对一段代码逻辑的正确性进行校验进行测试

## 单元测试的意义

从我自己的切身体会上来说，单元测试意义在于：

1. 最基本的，保证代码逻辑上的正确性
2. 能够进行回归，避免修改导致把以前的代码改挂
3. 迫使自己写出好的程序
4. 开发的时候能够快速的把代码跑起来，验证效果（在程序体量特别大，构建一次成本特别高；或者程序整体运行条件比较苛刻的时候会更明显）
5. 单测即文档

## 明确自己到底测什么

明确好单元的定义后，写单元测试前必须明确的一个点就是：到底需要测什么？也就是这个单元的边界在哪里。

例如下面这段代码：

```go
func (d *DeploymentModel) DeployConfirm(user string, deployID int64) error {
  var deploy Deploy
  if err := d.db.Find(&deploy, "id = ?", deployID).Error; err != nil {
		return err
  }
  
  if deploy.GetConfirm() == DEPLOY_UNCONFIRMED {
    cfm := &confirmer{} 
    if err := cfm.ConfirmDeployInNeed(deploy.Id, user); err != nil {
      return err
    }
  }
  
  return nil
}
```

这是一个使用了 gorm 作为数据库驱动的 web 项目中的一段代码。d.db 是 gorm 中的 *gorm.DB 对象。这个函数干了这么一件事：

1. 根据 deployID 从数据库中拿出对应的记录
2. 如果这个记录的状态是没有被确认，就去确认一下否则之间返回。

我们要写一个 TestDeployConfirm 函数：

1. 如果我们仅仅是测这个函数的逻辑，那我们不应该将 Find 函数、ConfirmDeployInNeed 这个函数真正的执行一遍。否则到底是测 DeployConfirm 还是测 Find / ConfirmDeployInNeed?
2. 如果是测 DeployConfirm 这个接口，那么我们确实应该真正的将数据库查询/ConfirmDeployInNeed执行

可见，不同的单元划分，写出来的单测是不一样的。通常，我们以：

1. web服务的接口（常见业务代码）
2. 模块的接口/模块的接口（团队合作开发）
3. 一个函数（个人日常开发）

着三种粒度作为一个单元。

明确自己要干的事情之后，自然而然的，我们就从代码开始着手。

## 有好代码，才能写出测试 

### 一个有问题的例子

刚开始写单元测试的时候，遇到的最大的问题应该就是，根本写不出来。构造一个 case 的实在是太困难了。这背后的原因，是因为代码的耦合度过高。还是一上面那段代码为例。这段代码问题很多，先只聚焦单测相关的内容。如果我们把 DeployConfirm 这个函数作为一个单元，我们看看这个单测应该怎么写：

1. d.db.Find 是一个肯定会访问数据库的动作。这意味着我们得事先准备好一个数据准备好了，可连接的数据库。同时，得构造出 gorm 访问数据库所需要的上下文。

2. 同时还得准备好 confirmer 相关的代码。由于直接使用的是 confirmer 这个具体实现，还得把 ConfirmDeployInNeed 这个函数实现好。甚至如果 ConfirmDeployInNeed 里面还有其他的硬依赖，那还得去实现里面的依赖。

是不是有点晕了？走到这一步的时候，可能就已经放弃了单元测试这个选项了。

最关键的，我们本质上是想测试 DeployConfirm 这个函数的逻辑，但是 ConfirmDeployInNeed 如果有问题，会导致我的数据库失败。数据库连接有问题/里面的数据有问题，都会导致我们测试不通过。

因此，能不能写出单元测试，取决于我们有没有写好被测试的代码。反过来，如果发现单元测试写起来很麻烦，首先要考虑代码是否写的合理。

### 可测试的代码应该是怎么样的

写代码是单元和分离的艺术。做好单元和分离，管理好抽象与实现，代码就可测试。具体大概的是以下几点：

#### 有状态与无状态分离

我们写代码的时候，都应该尽可能的编写纯函数。因为对于同一个输入有固定的输出，才是纯逻辑，才能保证测试的结果幂等。将代码有状态的部分/和没有状态的部分分离，有状态的部分尽可能的少。我们常见的 MVC 模式，把 Model 层隔离出来就是这么个思想。例如上面这个函数，正确的做法应该是：

将“根据 deployID 从数据库中拿出对应的记录” 这个操作放在 model， 其余逻辑相关的东西都应该拆到 Controller 里面去，那我们在 Controller 里面的函数就是纯逻辑，单测就好写了。 

#### 通过接口隔离

接口是抽象，我们可以构造自己的 mock 的实现来替换原有的实现。在编写代码的时候，引用其他的包的都应该调用接口。还是以上面的代码为例：

首先要定义一个 interface:

```go
type IConfirmer interface {
  ConfirmDeployInNeed(deployID int64, user string) error
}
```

然后，任何对象都应该通过构造函数去实现

```go
func NewConfirmer() IConfirmer {
  ...
}
```

使用的时候：

```go
var confirmer IConfirmer
confirmer := NewConfirmer()
```

这样，我们就能做到只通过抽象耦合，而不是与实现耦合。我们就能用我们 Mock 的一个 confirmer（也实现了这个抽象），去替代原有的实现，从而达到随心所欲的操纵 ConfirmDeployInNeed 这个方法的返回值的目的。

#### 尽可能少的硬耦合

通过上面两个方法，我们已经能非常容易的去控制外部依赖了。还有一个问题没解决 NewConfirmer 这个函数是一个别的包的引过来的，我们好像没办法替换 NewConfirmer() 的返回值呀？

这就是硬耦合（直接在函数里面去 New 一个对象）带来麻烦。

我们在写代码的时候，都应该尽可能的将外部依赖通过函数传参进入到一段逻辑之内, 来避免硬耦合。常见的做法有：

如果 confirmer 只在当前方法里面用到，则直接作为参数传入当前函数：

```go
func (d *DeploymentModel) DeployConfirm(user string, deployID int64, cfm IConfirmer) error {
  ...
  cfm.ConfirmDeployInNeed(deployID, user)
}
```

如果 confirmer 在多处用到，则在构造函数初始化：

```go
func NewDeploymentModel(cfm IConfirmer) {
  ...
}

// or

func NewDeploymentModel() {
  return &DeploymentModel {
    confirmer: NewConfirmer(),
  } 
}
```

这种方式，我们就能将初始化和逻辑分开.

满足单测编写的前提后，我们可以开始怎么构造单测用例。

## 单测用例构造的思路

### 原则

通常，我们使用代码覆盖率来衡量单测是否写的全面。然而，如果我们的单测做到了代码覆盖率 100% ，真的就意味着我们的测试是完备的吗？

还是一上面的这段代码为例子，如果我们构造了两个单测的用例：

1. 这两个用例 GetConfirm() 的结果都等于 DEPLOY_UNCONFIRMED
2. 一个用例执行 ConfirmDeployInNeed 这个函数抛出异常；另一个用例执行ConfirmDeployInNeed 这个函数没有抛出异常

那么这两个 case 一跑下来，确实所有的代码都执行到了，代码覆盖率100%。但是我们的用例并没有考虑当  GetConfirm() 不等于 DEPLOY_UNCONFIRMED 的情况。因此我认为这样的测试也是不完备的。

真正完备的测试，应该是要达到测试的用例数等于单元的[圈复杂度]([https://baike.baidu.com/item/%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/828737?fr=aladdin](https://baike.baidu.com/item/圈复杂度/828737?fr=aladdin)),保证从逻辑的入口和出口之间的所有可能的代码运行路径都覆盖到。

### 实践

但是在日常的敏捷开发中，可能时间不允许/其实也没必要维护这么多的用例，来追求理论上完美的单元测试。这就要求我们做一定程度的取舍。

在大多数情况下，我们的程序都是：

1. 有一条正常运行的通路，分枝用于抛出/处理异常
2. 有的分叉口/边界。比如降级，或者确实就是得根据不同的情况做不同的处理。

这个时候我们要做的就是：

1. 一定要保证有一个用例覆盖了正常的通路。这保证了我们主流程的正确性，同时日后迭代也能回归到，增强迭代的信心
2. 覆盖常规或者比较关键分叉和边界。比如就是降级能不能正确处理。
3. 每次线上出问题的地方，补充上单测。
4. 极端情况下的边界可以视重要程度不考虑。比如某一次数据库访问挂了。

明确好构造用例的思路，我们就真正着手于编写单测带么了。

## 单测编写工具

单元测试也是代码，也需要人去编写和维护 ，编写的时候所要遵循的原则也是一样的。工欲善其事，必先利其器。因此，在实际编写时我倾向于使用一些现成的框架和库来协助开发与维护。

### 常规工具 —— VSCode+Go插件

VSCode 的插件有个非常好用的功能，就是一他能自动根据你的代码生成对应的单元测试的框架代码。生成出来的代码就像是个样子：

```go
func TestDeploymentModel_DeployConfirm(t *testing.T) {
  type fields struct {
    db *gorm.DB,
  }
  type args struct {
    user     string
    deployID int64
  }
  tests :=[]struct{
    name    string
    fields  fields
    args    args
    wantErr bool
  }{
    // todo: Add test case
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      db: tt.fields.db,
    })
    if err := m.DeployConfirm(tt.args.user, tt.args.deployID); (err != nil) != tt.wantErr {
      t.Errorf("DeploymentModel.DeployConfirm() error = %v, wantErr %v", err, tt.wantErr)
    }
  }
}
```

然后你只需要把用例有关的数据以结构体实例的方式补充到 tests 数组里面，每一个元素一个用例。我非常喜欢这种 generate code 的方式进行编程，这能让人更专注于真正有意义的事情上。

### 单测框架

如果不满足于 VSCode + Go 插件的表达能力（比如你希望对测试分组分类），那么可以考虑使用一些单元测试框架。比较值得推荐的单测框架就是 [GoConvey](https://github.com/smartystreets/goconvey) 和 [Testify](https://github.com/stretchr/testify)

#### GoConvey

##### 编码风格

GoConvey 提供了一种函数式编程风格的表达体验。利用他官方提供的例程:

```go
package package_name

import (
    "testing"
    . "github.com/smartystreets/goconvey/convey"
)

func TestSpec(t *testing.T) {

	// Only pass t into top-level Convey calls
	Convey("Given some integer with a starting value", t, func() {
		x := 1

		Convey("When the integer is incremented", func() {
			x++

			Convey("The value should be greater by one", func() {
				So(x, ShouldEqual, 2)
			})
		})
	})
}
```

可以看到，他利用闭包的方式，能无限的嵌套下去。能一步一步的构造自己的上下文，然后构造出自己的用例。

同时，GoConvey 也提供了一系列标准的[断言库](https://github.com/smartystreets/goconvey/wiki/Assertions) (如 例程代码中的 So 函数)。在上面 VSCode 生成的代码里面也可以看到， golang 原生式没有断言的，他需要我们自己实现具体的判断逻辑，然后通过 t.Errorf()把没有通过的结果抛出来。利用框架提供的断言能让我们找回原来断言式编程的熟悉感。

最后，我认为比较吸引人的一点是它提供了一个 web 服务：

1. 能监听本地文件的变化，在编码过程中，文件的变动都能自动触发单元测试。
2. 单元测试的报告能以一个相对美观的 web 页面呈现，提升编程体验。

#### Testify

Testify 提供了一种面向对象编程风格的表达体验。同样列出他的官方简化例程:

```go
import (
    "testing"
    "github.com/stretchr/testify/suite"
)

type ExampleTestSuite struct {
    suite.Suite
    VariableThatShouldStartAtFive int
}

func (suite *ExampleTestSuite) SetupTest() {
    suite.VariableThatShouldStartAtFive = 5
}

func (suite *ExampleTestSuite) TestExample() {
    suite.Equal(suite.VariableThatShouldStartAtFive, 5)
}

func TestExampleTestSuite(t *testing.T) {
    suite.Run(t, new(ExampleTestSuite))
}
```

丛代码可以看到, Testify 在测试的各个阶段设置了锚点（比如 SetupTest方法用于构造测试用例所需要的上下文），然后 结构体中的每一个 Test 开头的方法，都会被作为测试用例执行。完全的使用用例见这个[链接](https://github.com/stretchr/testify/blob/master/suite/suite_test.go)

同样的，Testify 也提供了一套标准的[断言库](https://godoc.org/github.com/stretchr/testify/assert)。

最后 testify 提供了用于构造 Mock 对象的库，这一点我们放在下一个话题。

### 外部对象的 mock

单元测试的框架我们有了，最关键的还是构造测试用例。在我们明确了要测试的单元的时候，构造用例的时候就涉及到要模拟的测试单元的外部依赖的返回值。这就两个我们可以使用的工具选项：[GoMock](https://github.com/golang/mock) 和 Testify 提供 Mock 模块。

#### GoMock

这个是 Google 官方提供的唯一的单元测试的工具。这个工具好用的地方在于，他能自动根据你的 interface 的定义生成 Mock 对象的代码。写单元测试的时候直接使用就可以了。由于生成的代码比较长，我就不贴在这里了，有兴趣的同学可以自己生成一个看看。

#### Testify 的 Mock

 Testify 框架本身也提供了 Mock 的库。但是从我个人的感觉来说，这个库的使用体验并不好。原因就在于，Mock 对象的实现得自己写。举个例子：

假设现在我们有这么一个接口：

```go
type IService interface {
  SendHTTPRequest(
    method string,
    link string,
    params map[string]string,
    headers map[string],
    body []byte,
  ) (
    service.IResponse,
    error,
  )
}
```

那关于这个接口的 Mock 对象就需要这么写：

```go
import（
	"github.com/stretchr/testify/mock"
）

type MockIService struct {
  mock.Mock
}

func (ms *MockIService) SendHTTPRequest(
  method string,
  link string,
  params map[string]string,
  headers map[string],
  body []byte,
) (
  service.IResponse,
  error,
) {
  args := ms.Called(method, link, params, headers, body, timeout)
  return args.Get(0).(service.IResponse), args.Error(1)
}
```

这些代码完全就是有规律的，完全可以使用机器去编写的。设想，如果一个接口有多个接口函数的签名，同时一个里面有无数个接口，这里面的无用的工作量是非常恐怖的。

### 给函数/变量打桩—— [GoStub](github.com/prashantv/gostub)

有的时候，我们使用的外部逻辑是一个简单的辅助函数/或者特定场景下使用了一个全局变量，这种时候 mock 就比较无力了。这个时候，我们就可以借助 GoStub 给一个变量打桩。具体的使用实践，可以参考[这篇文章](https://www.jianshu.com/p/70a93a9ed186)，在此不做详细的展开。

### 给方法打桩——[Monkey](https://github.com/bouk/monkey)

当你的一个方法依赖了当前结构体绑定的另外一个方法的时候，就涉及到了方法的打桩。方法的打桩可以使用 Monkey 这个工具，使用方法可以参考这个[实践](https://www.jianshu.com/p/2f675d5e334e),在这里就不展开了。

## 实践总结

### 框架的选择

我更倾向于 GoConvey + GoMock。原因是：

1. GoConvey 使用起来简单，功能够用，单测组织起来表达能力也强。
2. GoMock 比 Testify 的 mock 好用，并且 mock 是我目前写单测的主要依赖解决手段。

### 库的使用

在具体写代码的时候，如果没有管理好抽象和具体实现的的依赖关系，让单测变得异常复杂。

举个例子：如果一个包依赖了另一个包的具体实现，比如说示例代码中的 confirmer。这个时候你想要去写单测，很可能就使用了 Monkey 去给那个方法打桩。然后打桩还打不了，因为我们没办法把打过桩的这个对象注入到我们测试的单元中。这时候很有可能就会把 cfm 这个变量写成一个全局变量，然后想到用GoStub 去给这个全局变量打桩。代码这样去写很显然是不合理的。

因此，写单测的时候无论如何都应该优先使用 Mock（毕竟 Google 官方也只提供了 gomock  这个工具，说明官方认为这个就已经够用了）。当要用到 GoStub 和 Monkey 时，先考虑清楚是不是代码写的不合理，然后确认使用场景之后，再合理选择使用。

原文链接：https://blog.coordinate35.cn/html/article.html?type=3&article_id=18