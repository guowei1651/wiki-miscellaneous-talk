# \[翻译\]面向领域的可观测性

原文为martin fowler的个人网站中的：[Domain-Oriented Observability](https://martinfowler.com/articles/domain-oriented-observability.html)

## 译者序

对于一个开发人员来说，应该关注的并不只是实现过程以及实现内容。更应该关注软件运行期的内容。可能是因为现在的开发人员的代码根本活不到软件运行期所以大家都不关注这部分的内容。不过随着国内软件水平的持续提升以及最终用户方对软件运行期要求的提高会逐渐倒逼开发人员开始关注运行期的内容。所以，提前一步进行知识的储备以及学习是非常有必要的。

一般情况下我们所说的指标监控都是在说明技术指标，技术上标识服务的可用性，可靠性的内容。对于一个软件系统来说，技术指标只能部分体现业务的感受。不能全面的体现出业务的情况以及业务发展趋势等内容。对于业务来说没有办法深入了解软系统运行情况。这对于业务决策、业务规划都是大问题，而本文为我们提供一种业务指标监控的方法论，帮助软件系统更好的了解用户感受，为之后的用户感受优化做出指导。

本文的作者依然是之前翻译过的[《功能切换（又称功能标志）》](https://www.jianshu.com/p/f79e6aedf407)的作者[**Pete Hodgson**](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.thepete.net%2Fabout%2F)。

## 正文

在我们的软件系统中，可观测性一直都是很有价值的，在这个云和微服务的时代，可观测性变得更加重要。然而，我们添加到系统中的可观察性在本质上往往是相当低的级别和技术性的，而且常常需要向各种日志、工具和分析框架的粗略、冗长的调用乱扔代码库。本文描述了一种清理这种混乱的模式，它允许我们以一种干净的、可测试的方式添加与业务相关的可观察性。

由于当前的趋势，如微服务和云，现代软件系统正变得更加分布式，运行在不太可靠的基础设施上。在我们的系统中建立可观测性一直是必要的，但这些趋势使得它比以往任何时候都更加重要。同时，DevOps运动意味着监控生产的人员比以往任何时候都更有可能在运行的系统中实际添加自定义的检测代码，而不是不得不将可观察性固定在一边。

但是，我们如何将可观测性添加到我们最关心的业务逻辑中，而不让检测细节阻塞我们的代码库呢？而且，如果这个工具很重要，我们如何测试我们是否正确实现了它？在本文中，我将通过将以业务为中心的可观测性视为我们代码库中的一个一流概念，来演示面向领域的可观测性哲学与一种称为领域探测的实现模式的结合是如何提供帮助的。

### 目录

* **观察什么**
* **可观测性问题**
* **收拾残局**
  * 域探测
* **可观测性测试**
  * 测试Instrumentation代码是一种痛苦
  * 域探测支持更清晰、更集中的测试
  * 包括执行上下文
  * 元数据类型
* **注入元数据**
  * 正在收集检测上下文
  * 域探测的范围
* **替代实施**
  * 基于事件的可观测性
  * 面向方面编程
* **何时应用面向领域的可观测性？**
  * 重新拟合现有的代码库

### 观测什么

“可观察性”的范围很广，从低级技术指标到高级业务关键绩效指标（kpi）。在技术方面，我们可以跟踪诸如内存和CPU利用率、网络和磁盘I/O、线程计数和垃圾收集（GC）暂停等内容。在业务方面，我们的业务/域度量可以跟踪购物车放弃率、会话持续时间或支付失败率等。

因为这些更高级别的指标是特定于每个系统的，所以它们通常需要手动编写的检测逻辑。这与较低级别的技术指标形成了对比，后者更通用，而且通常在启动时除了注入某种监视代理之外，不需要对系统的代码库进行太多修改就可以实现。

还需要注意的是更高级别的、面向产品的度量更具价值，因为根据定义，它们更紧密地反映了系统正在朝着其预期的业务目标运行。

我们可以通过添加跟踪有价值度量的工具实现面向领域的可观测性。

### 可观测性问题

因此，面向领域的可观测性是很有价值的，但它通常需要手工滚动的检测逻辑。这种定制的工具与我们系统的核心域逻辑并驾齐驱，在这里，清晰、可维护的代码至关重要。不幸的是，业务指标插装代码对于领域来说往往是嘈杂的，如果我们不小心，它可能会导致阅读代码时分散注意力，造成领域混乱。

让我们看一个引入检测代码可能导致的混乱的例子。在添加任何可观察性之前，下面是一个假设的电子商务系统（有些天真）折扣代码逻辑：

```java
class ShoppingCart…

  applyDiscountCode(discountCode){

    let discount; 
    try {
      discount = this.discountService.lookupDiscount(discountCode);
    } catch (error) {
      return 0;
    }

    const amountDiscounted = discount.applyToCart(this);
    return amountDiscounted;
  }
```

我想说我们这里有一些明确表达的领域逻辑。我们根据折扣代码查找折扣，然后将折扣应用于购物车。最后，我们返回已贴现的金额。如果我们找不到折扣，我们什么也不做，早就走了。

对购物车应用折扣是一个关键特性，因此良好的可观测性在这里很重要。让我们添加一些代码：

```java
class ShoppingCart…

  applyDiscountCode(discountCode){
    // 监控代码
    this.logger.log(`attempting to apply discount code: ${discountCode}`);

    let discount; 
    try {
      discount = this.discountService.lookupDiscount(discountCode);
    } catch (error) {
      // 监控代码
      this.logger.error('discount lookup failed',error);
      this.metrics.increment(
        'discount-lookup-failure',
        {code:discountCode});
      return 0;
    }
    // 监控代码
    this.metrics.increment(
      'discount-lookup-success',
      {code:discountCode});

    const amountDiscounted = discount.applyToCart(this);

    // 监控代码
    this.logger.log(`Discount applied, of amount: ${amountDiscounted}`);
    this.analytics.track('Discount Code Applied',{
      code:discount.code, 
      discount:discount.amount, 
      amountDiscounted:amountDiscounted
    });

    return amountDiscounted;
  }
```

除了执行查找和应用折扣的实际业务逻辑外，我们现在还调用各种仪表系统。我们正在为开发人员记录一些诊断，我们正在为生产中操作此系统的人员记录一些度量，我们还将事件发布到我们的分析平台中，供产品和营销人员使用。

不幸的是，增加可观测性已经把我们漂亮的、干净的域逻辑弄得一团糟。现在，我们的applyDiscountCode方法中只有25%的代码用于查找和应用折扣。我们刚开始使用的干净的业务逻辑没有改变，并且保持清晰和简洁，但是在现在占据方法大部分的低级工具代码中，它已经丢失了。此外，我们在域逻辑的中间引入了代码复制和魔术字符串。

简而言之，我们的工具代码对于任何试图阅读此方法并查看其实际功能的人来说都是一个巨大的干扰。

### 收拾残局

让我们看看是否可以通过重构我们的实现来清理这些混乱。首先，让我们将这个讨厌的低级检测逻辑提取到单独的方法中：

```java
  class ShoppingCart {
    applyDiscountCode(discountCode){
      // 监控代码
      this._instrumentApplyingDiscountCode(discountCode);

      let discount; 
      try {
        discount = this.discountService.lookupDiscount(discountCode);
      } catch (error) {
        // 监控代码
        this._instrumentDiscountCodeLookupFailed(discountCode,error);
        return 0;
      }
      // 监控代码
      this._instrumentDiscountCodeLookupSucceeded(discountCode);

      const amountDiscounted = discount.applyToCart(this);
      // 监控代码
      this._instrumentDiscountApplied(discount,amountDiscounted);
      return amountDiscounted;
    }

    // 监控代码
    _instrumentApplyingDiscountCode(discountCode){
      this.logger.log(`attempting to apply discount code: ${discountCode}`);
    }
    // 监控代码
    _instrumentDiscountCodeLookupFailed(discountCode,error){
      this.logger.error('discount lookup failed',error);
      this.metrics.increment(
        'discount-lookup-failure',
        {code:discountCode});
    }
    // 监控代码
    _instrumentDiscountCodeLookupSucceeded(discountCode){
      this.metrics.increment(
        'discount-lookup-success',
        {code:discountCode});
    }
    // 监控代码
    _instrumentDiscountApplied(discount,amountDiscounted){
      this.logger.log(`Discount applied, of amount: ${amountDiscounted}`);
      this.analytics.track('Discount Code Applied',{
        code:discount.code, 
        discount:discount.amount, 
        amountDiscounted:amountDiscounted
      });
    }
  }
```

这是一个好的开始。我们将插装细节提取到集中的插装方法中，在每个插装点使用一个简单的方法调用来保留业务逻辑。现在，各种检测系统的分散注意力的细节已经被下推到那些 \_instrument...方法中，阅读和理解applyDiscountCode变得更容易了。

然而，ShoppingCart现在有一堆完全专注于工具的私有方法，而这些工具并不是ShoppingCart的责任，这似乎是不对的。类中与该类的主要职责无关的功能集群通常表示有一个新类正在尝试出现。

让我们按照这个提示，收集这些检测方法，并将它们移到自己的DiscountInstrumentation类中：

```java
class ShoppingCart…

  applyDiscountCode(discountCode){
    this.instrumentation.applyingDiscountCode(discountCode);

    let discount; 
    try {
      discount = this.discountService.lookupDiscount(discountCode);
    } catch (error) {
      this.instrumentation.discountCodeLookupFailed(discountCode,error);
      return 0;
    }
    this.instrumentation.discountCodeLookupSucceeded(discountCode);

    const amountDiscounted = discount.applyToCart(this);
    this.instrumention.discountApplied(discount,amountDiscounted);
    return amountDiscounted;
  }
```

我们不会对方法进行任何更改；我们只是使用适当的构造函数将它们移到自己的类中：

```java
class DiscountInstrumentation {
  constructor({logger,metrics,analytics}){
    this.logger = logger;
    this.metrics = metrics;
    this.analytics = analytics;
  }

  applyingDiscountCode(discountCode){
    this.logger.log(`attempting to apply discount code: ${discountCode}`);
  }

  discountCodeLookupFailed(discountCode,error){
    this.logger.error('discount lookup failed',error);
    this.metrics.increment(
      'discount-lookup-failure',
      {code:discountCode});
  }

  discountCodeLookupSucceeded(discountCode){
    this.metrics.increment(
      'discount-lookup-success',
      {code:discountCode});
  }

  discountApplied(discount,amountDiscounted){
    this.logger.log(`Discount applied, of amount: ${amountDiscounted}`);
    this.analytics.track('Discount Code Applied',{
      code:discount.code, 
      discount:discount.amount, 
      amountDiscounted:amountDiscounted
    });
  }
}
```

我们现在有了一个很好的、清晰的职责分离：ShoppingCart完全专注于应用折扣等领域概念，而我们的新DiscountInstrumentation类封装了应用折扣过程的所有细节。

#### 域探针（Domain Probe）

DiscountInstrumentation是我称之为域探针的模式的一个例子。域探针提供了一个面向域语义的高级检测API，封装了实现面向域可观测性所需的低级检测管道。这使我们能够在仍然使用域语言的情况下为域逻辑添加可观测性，从而避免了插装技术中令人分心的细节。在前面的示例中，ShoppingCart通过报告正在应用的域观测折扣代码和折扣代码查找失败到DiscountInstrumentation探测而实现了可观测性，而不是直接在技术领域编写日志条目或跟踪分析事件。这似乎是一个微妙的区别，但是让域代码集中于域在保持代码基可读、可维护和可扩展方面有着丰厚的回报。

### 测试可观测性

很少能看到仪表逻辑的良好测试覆盖率。我并不经常看到自动测试来验证操作失败时是否记录了错误，或者在转换发生时是否发布了包含正确字段的分析事件。这可能部分是由于可观测性在历史上被认为不太有价值，但也因为为低级检测代码编写良好的测试是一件痛苦的事情。

#### 测试Instrumentation代码是一种痛苦

为了演示，让我们看看假设的电子商务系统的不同部分的一些工具，并看看如何编写一些测试来验证工具代码的正确性。

ShoppingCart有一个addToCart方法，该方法当前使用对各种可观测系统的直接调用（而不是使用域探测）进行检测：

```java
class ShoppingCart…

  addToCart(productId){
    this.logger.log(`adding product '${productId}' to cart '${this.id}'`);

    const product = this.productService.lookupProduct(productId);

    this.products.push(product);
    this.recalculateTotals();

    this.analytics.track(
      'Product Added To Cart',
      {sku: product.sku}
    );
    this.metrics.gauge(
      'shopping-cart-total',
      this.totalPrice
    );
    this.metrics.gauge(
      'shopping-cart-size',
      this.products.length
    );
  }
```

让我们来看看如何开始测试这个检测逻辑： shoppingCart.test.js

```javascript
  const sinon = require('sinon');

  describe('addToCart', () => {
    // ...

    it('logs that a product is being added to the cart', () => {
      const spyLogger = {
        log: sinon.spy()
      };
      const shoppingCart = testableShoppingCart({
        logger:spyLogger
      });

      shoppingCart.addToCart('the-product-id');

      expect(spyLogger.log)
        .calledWith(`adding product 'the-product-id' to cart '${shoppingCart.id}'`);
    });
  });
```

在这个测试中，我们设置了一个购物车进行测试，与一个间谍日志连接起来（一个“间谍”是一种双重测试，用于验证我们的测试对象是如何与其他对象交互的）。如果您想知道，testableShoppingCart只是一个小的帮助函数，它创建了ShoppingCart的一个实例，默认情况下具有伪造的依赖项。当我们的间谍就位后，我们调用shopping cart.addToCart（…），然后检查购物车是否使用记录器来记录适当的消息。

如前所述，此测试确实提供了合理的保证，即当产品添加到购物车时，我们正在进行日志记录。但是，它与日志记录的细节非常相关。如果我们决定在将来某个时候更改日志消息的格式，我们将毫无理由地中断此测试。这个测试不应该关注记录内容的确切细节，而应该关注使用正确上下文数据记录的内容。

我们可以尝试通过匹配正则表达式（regex）而不是精确的字符串来减少测试与日志消息格式细节的耦合程度。然而，这会使验证有点不透明。此外，创建一个健壮的regex所需的工作通常是一个很差的时间投资。

此外，这只是一个简单的测试如何记录的例子。更复杂的场景（例如，日志异常）更让人头疼日志框架的api和它们的同类在被模仿时不容易验证。

让我们继续看另一个测试，这次验证我们的分析集成：

```text
shoppingCart.test.js

  const sinon = require('sinon');

  describe('addToCart', () => {
    // ...

    it('publishes analytics event', () => {
      const theProduct = genericProduct();
      const stubProductService = productServiceWhichAlwaysReturns(theProduct);  ➋

      const spyAnalytics = {
        track: sinon.spy()
      };

      const shoppingCart = testableShoppingCart({
        productService: stubProductService,  ➊
        analytics: spyAnalytics  ➌
      });


      shoppingCart.addToCart('some-product-id');


      expect(spyAnalytics.track).calledWith(  ➍
        'Product Added To Cart',
        {sku: theProduct.sku}
      );
    });
  });
```

