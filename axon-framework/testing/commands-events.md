# 命令 / 事件

CQRS的好处之一，尤其是事件源的好处之一是可以纯粹根据事件和命令来表达测试。事件和命令都是功能组件，对领域专家或业务所有者而言具有明确的含义。这不仅意味着以事件和命令表示的测试具有明确的功能含义，还意味着它们几乎不依赖于任何实现方案。

本章中描述的功能需要`axon-test`模块，该模块可以通过配置maven依赖项（使用`<artifactId>axon-test</artifactId>`和`<scope>test</scope>`）或从完整的软件包中获得。

本章描述的fixtures可与任何测试框架一起使用，例如JUnit和TestNG。

## 命令模型测试

命令处理组件通常是任何基于CQRS的架构中包含最复杂的组件。比其他组件更为复杂，这也意味着此组件还有与测试相关的额外要求。

尽管更为复杂，但命令处理组件的API相当简单。 它有一个命令传入，事件消亡。 在某些情况下，命令执行中可能会有一个查询。 除此之外，命令和事件是API的唯一组成部分。 这意味着可以根据事件和命令完全定义测试方案。 通常，形式为：

* 鉴于过去的某些事件，
* 执行此命令时，
* 预计这些事件将被发布和/或存储

Axon Framework提供了一个测试fixture，可让您精确地执行此操作。 `AggregateTestFixture`允许您配置由必要的命令处理程序和存储库组成的特定基础结构，并根据“given-when-then”事件和命令来表达方案。

> **测试 Fixtur 的重点**
>
> 由于此处的测试单元是聚合，因此`AggregateTestFixture`仅用于测试一个聚合。因此，`when` \(或 `given`\)子句中的所有命令都旨在针对测试fixture下的聚合。同样，所有`given`和`expected`的事件都应从测试fixture下的聚合中触发。

下面的示例显示了`GiftCard`集合\([如前面所述](../axon-framework-commands/modeling/aggregate.md#basic-aggregate-structure)\)上的JUnit 4与“given-when-then”测试fixture的用法：

```java
import org.axonframework.test.aggregate.AggregateTestFixture;
import org.axonframework.test.aggregate.FixtureConfiguration;

public class GiftCardTest {

    private FixtureConfiguration<GiftCard> fixture;

    @Before
    public void setUp() {
        fixture = new AggregateTestFixture<>(GiftCard.class);
    }

    @Test
    public void testRedeemCardCommand() {
        fixture.given(new CardIssuedEvent("cardId", 100))
               .when(new RedeemCardCommand("cardId", "transactionId", 20))
               .expectSuccessfulHandlerExecution()
               .expectEvents(new CardRedeemedEvent("cardId", "transactionId", 20));
        /*
        这四行定义了实际方案及其预期结果。
        第一行定义了过去发生的事件。
        这些事件定义了测试中的聚合的状态。
        实际上，这些事件存储在加载聚合时返回的事件。

        第二行定义了我们希望对系统执行的命令。

        最后，我们还有两个定义预期行为的方法。
        在本例中，我们使用推荐的void返回类型。
        最后一个方法定义，我们预期命令执行的结果是单个事件。
        */
    }
}
```

“given-when-then”测试fixture定义了三个阶段：配置，执行和验证。这些阶段的每个阶段均由不同的接口表示：分别为`FixtureConfiguration`，`TestExecutor`和`ResultValidator`。

> **Fluent 接口**
>
> 为了更好地利用这些阶段之间的迁移，最好使用这些方法提供的fluent接口，如上面的示例所示。

### 测试设置

在配置阶段（即在提供第一个“given”之前），要提供执行测试所需的构件。默认情况下，事件总线，命令总线和事件存储的特殊版本作为fixture的一部分提供。有适当的访问器方法可获取对它们的引用。任何未直接在聚合上注册的命令处理程序都需要使用`registerAnnotatedCommandHandler`方法进行显式配置。除了带注解的命令处理程序之外，您还可以注册各种组件和设置，这些组件和设置定义应如何设置围绕聚合测试的基础结构，包括以下内容：

* `registerRepository`:

  注册自定义的聚合[`Repository`](commands-events.md)。

* `registerRepositoryProvider`:

  注册用于[产生新聚合](../axon-framework-commands/modeling/aggregate-creation-from-another-aggregate.md)的RepositoryProvider。

* `registerAggregateFactory`:

  注册自定义[`AggregateFactory`](commands-events.md)。

* `registerAnnotatedCommandHandler`:

  注册带注解的命令处理程序对象。

* `registerCommandHandler`:

  注册`CommandMessage`的`MessageHandler`。

* `registerInjectableResource`:

  注册可以注入消息处理成员的资源。

* `registerParameterResolverFactory`:

  将[`ParameterResolverFactory`](commands-events.md)注册到测试fixture。

  此方法用于通过自定义`ParameterResolver`补充默认的`ParameterResolver`。

* `registerCommandDispatchInterceptor`:

  注册[`MessageDispatchInterceptor`](commands-events.md)命令。

* `registerCommandHandlerInterceptor`:

  注册[`MessageHandlerInterceptor`](commands-events.md)命令。

* `registerDeadlineDispatchInterceptor`:

  注册 [`DeadlineMessage`](commands-events.md) `MessageDispatchInterceptor`.

* `registerDeadlineHandlerInterceptor`:

  注册 [`DeadlineMessage`](commands-events.md) `MessageHandlerInterceptor`.

* `registerFieldFilter`:

  注册在“then”阶段中比较对象时使用的`Field`过滤器。

* `registerIgnoredField`:

  注册执行状态相等时对于给定类应忽略的字段。

* `registerHandlerDefinition`:

  将自定义[`HandlerDefinition`](../../appendices/message-handler-tuning/handler-enhancers.md)注册到测试fixture。

* `registerHandlerEnhancerDefinition`:

  将自定义[`HandlerEnhancerDefinition`](../../appendices/message-handler-tuning/handler-enhancers.md)注册到测试ficture。

  此方法用于通过自定义`HandlerEnhancerDefinition`补充默认的`HandlerEnhancerDefinition`。

* `registerCommandTargetResolver`:

  将`CommandTargetResolver`注册到测试fixture。

一旦配置了fixture，就可以定义“given”事件。测试fixture会将这些事件包装为`DomainEventMessages`。如果“given”事件实现了`Message`，则该消息的payload和元数据将包含在`DomainEventMessage`中，否则将给定事件用作。 DomainEventMessage的序列号是连续的，从0开始。如果不需要先前的活动，则可以将`namedNoPriorActivity()`用作起点。

或者，您也可以提供“given”方案的命令。在这种情况下，由这些命令生成的事件将在执行实际的被测命令时用作集合的来源。使用 "`givenCommands(...)`" 方法提供命令对象。

“given”阶段的最后一个选项是直接提供聚合状态。不建议在事件源的情况下使用此方法，仅在无法根据命令或事件重建聚合或使用[状态存储聚合](../axon-framework-commands/modeling/state-stored-aggregates.md)的情况下才建议这样做。使用`fixture.givenState(() -> new GiftCard())`定义初始状态。

### 测试执行阶段

执行阶段允许您有两个进入[验证阶段](commands-events.md#validation-phase)的入口点。首先，您可以提供要针对命令处理组件执行的命令。与给定事件类似，如果提供的命令为`CommandMessage`类型，则将按原样调度。监视调用的处理程序的行为（无论是在总体上还是作为外部处理程序），并将其与验证阶段中注册的预期值进行比较。

其次，可以使用`whenThenTimeElapses(Duration)`和`whennnTimeAdvancesTo(Instant)`处理一定的时间跨度。这些支持测试`DeadlineMessages`的发布，[这](../deadlines/)将在本章中进一步定义。

注意，只有在测试执行阶段发生的活动才会被监控。验证阶段不考虑“given”阶段产生的任何事件或副作用。

> **非法状态变化检测**
>
> 在测试执行期间，Axon尝试检测被测聚合中的任何非法状态更改。它通过将命令执行后的聚合状态与来自所有“given”和存储事件的聚合状态进行比较来实现这一点。如果该状态不相同，则表示状态更改发生在聚合的事件处理程序方法之外。静态和临时字段在比较中被忽略，因为它们通常包含对资源的引用。
>
> 您可以使用`setReportIllegalStateChange()`方法在fixture的配置中切换检测。

### 验证阶段

最后一个阶段是验证阶段，它允许您检查命令处理组件的活动。这通常是纯粹根据返回值和事件来完成的。

#### 验证命令结果

测试fixture允许您验证命令处理程序的返回值。您可以显式定义预期的返回值，或仅要求该方法成功返回。您还可以表达您希望CommandHandler抛出的任何异常。

以下方法可用于验证命令结果：

* `fixture.expectSuccessfulHandlerExecution()`:

  验证处理程序是否返回未标记为异常响应的常规响应。

  不评估确切的响应。

* `fixture.expectResultMessagePayload(Object)`:

  验证处理程序返回的成功响应，且有payload等于给定的payload。

* `fixture.expectResultMessagePayloadMatching(Matcher)`:

  验证处理程序返回了成功的响应，并且payload与给定的Matcher相匹配

* `fixture.expectResultMessage(CommandResultMessage)`:

  验证收到的`CommandResultMessage`具有与给定消息相同的payload和元数据。

* `fixture.expectResultMessageMatching(Matcher)`:

  验证`CommandResultMessage`与给定的Matcher相匹配。

* `fixture.expectException(Matcher)`:

  验证命令处理结果是否为异常结果，以及异常是否与给定的`Matcher`相匹配。

* `fixture.expectException(Class)`:

  验证命令处理结果是否为具有给定异常类型的异常结果。

* `fixture.expectExceptionMessage(String)`:

  验证命令处理结果是否为异常结果，以及异常消息是否等于给定消息。

* `fixture.expectExceptionMessage(Matcher)`:

  验证命令处理结果是否为异常结果，以及异常消息是否与给定的Matcher匹配。

#### 验证发布的事件

The other component is validation of published events. There are two ways of matching expected events.
另一个组件是已发布事件的验证。有两种方式可以匹配预期事件。

第一种方法是传入需要与实际事件进行字面值比较的事件实例。将预期事件的所有属性与其在实际事件中的对应项进行比较(使用`equals()`)。如果其中一个属性不相等，则测试将失败，并生成大量错误报告。

表达预期的另一种方式是使用“Matchers”（由Hamcrest库提供）。`Matcher`是一个接口，它规定了两个方法：`matches(Object)`和`describeTo(Description)`。第一个返回一个布尔值来指示匹配器是否匹配。第二种可以让你表达你的预期。例如，“GreaterThanTwoMatcher”可以将“值大于2的任何事件”附加到描述中。描述允许创建关于测试用例失败原因的表达性错误消息。

为事件列表创建匹配器是一项乏味且容易出错的工作。为了简化操作，Axon提供了一组匹配器，允许您提供一组特定于事件的匹配器，并告诉Axon它们应该如何与列表匹配。这些匹配器通过抽象的`Matchers`实用程序类静态获得。

以下是可用事件列表匹配器及其用途的概述：

* **列出所有内容**: `Matchers.listWithAllOf(event matchers...)`

  如果所有提供的事件匹配器都与实际事件列表中的至少一个事件匹配，则此匹配器将成功。

  不关注多个匹配器是否与同一事件匹配，也不管列表中的某个事件与任何匹配器都不匹配。

* **列出任何一项**: `Matchers.listWithAnyOf(event matchers...)`

  如果提供的一个或多个事件匹配器与实际事件列表中的一个或多个事件匹配，则此匹配器将成功。

  有些匹配器甚至根本不匹配，而另一些匹配器则针对多个其他匹配器。

* **事件序列**: `Matchers.sequenceOf(event matchers...)`
  
  使用此匹配器验证实际事件与提供的事件匹配程序的顺序是否相同。如果每个匹配器匹配前一个匹配器所匹配的事件之后的事件，则它将成功。这意味着可能会出现不匹配事件的“间隔”。

  如果在评估事件之后，有更多的匹配器可用，那么它们都将根据“`null`”进行匹配。这由事件匹配器决定他们是否接受。

* **事件的确切顺序**: `Matchers.exactSequenceOf(event matchers...)`

  “事件序列”匹配器的变体，不允许出现不匹配事件的间隔。

  这意味着每个匹配器都必须在先前匹配器所匹配的事件之后直接与该事件匹配。

为了方便起见，提供了一些通常需要的事件匹配器。它们与单个事件实例匹配：

* **Equal 事件**: `Matchers.equalTo(instance...)`

  验证给定对象在语义上是否等于给定事件。

  该匹配器将使用null安全的equals方法比较实际对象和预期对象字段中的所有值。

  这意味着即使事件未实现equals方法，也可以对其进行比较。

  使用equals对给定参数字段中存储的对象进行比较，要求它们正确实现一个对象。

* **No more 事件**: `Matchers.andNoMore()` 或 `Matchers.nothing()`

  只与 `null` 值匹配。

  可以将此匹配器作为最后一个匹配器添加到事件匹配器的确切序列中，以确保没有剩余的不匹配事件。

* **Predicate Matching**: `Matchers.matches(Predicate)` or `Matchers.predicate(Predicate)`

  创建一个与指定`Predicate`定义的值匹配的匹配器。

  可以在`Predicate` API提供更好的方法来验证结果的情况下使用。

由于向匹配器传递了事件消息列表，因此有时您只想验证消息的payload。有匹配器可以帮助您：

* **Payload matching**: `Matchers.messageWithPayload(payload matcher)`

  验证消息的payload是否与给定的payload匹配器匹配。

* **Payloads matching**: `Matchers.payloadsMatching(list matcher)`

  验证消息的payloads是否匹配给定的匹配器。

  给定的匹配器必须与包含每个消息payload的列表匹配。

  payloads匹配匹配器通常用作外部匹配器，以防止payload匹配器重复。

下面是显示这些匹配器用法的一个代码示例。在此示例中，我们预计将发布两个事件。第一个事件必须是“ThirdEvent”，第二个事件必须是“ aFourthEventWithSomeSpecialThings”。可能没有第三事件，因为这将对“andNoMore”匹配器失败。

```java
import org.axonframework.test.aggregate.FixtureConfiguration;

import static org.axonframework.test.matchers.Matchers.andNoMore;
import static org.axonframework.test.matchers.Matchers.equalTo;
import static org.axonframework.test.matchers.Matchers.exactSequenceOf;
import static org.axonframework.test.matchers.Matchers.messageWithPayload;
import static org.axonframework.test.matchers.Matchers.payloadsMatching;

class MyCommandModelTest {

    private FixtureConfiguration<MyCommandModel> fixture;

    public void testWithMatchers() {
        fixture.given(new FirstEvent(), new SecondEvent())
               .when(new DoSomethingCommand("aggregateId"))
               .expectEventsMatching(exactSequenceOf(
                   // 我们只能针对payload进行匹配：
                   messageWithPayload(equalTo(new ThirdEvent())),
                   // 这将与一条消息匹配
                   aFourthEventWithSomeSpecialThings(),
                   // 这将确保没有其他事件
                   andNoMore()
               ));

               //或者如果我们只想匹配payloads：
               .expectEventsMatching(payloadsMatching(
                   exactSequenceOf(
                       // 我们只有payloads，因此我们可以直接equalTo
                       equalTo(new ThirdEvent()),
                       // 现在，此匹配器也与payload匹配
                       aFourthEventWithSomeSpecialThings(),
                       // 这仍然需要没有其他事件
                       andNoMore()
                   )
               ));
   }
}
```

#### 验证聚合状态

在某些情况下，可能需要验证测试后剩余聚合的状态。尤其是在given-when-then场景中，given也表示初始状态，使用[状态存储聚合](../axon-framework-commands/modeling/state-stored-aggregates.md)时也是如此。

fixture提供了一种方法，该方法允许验证在[执行阶段](commands-events.md#test-execution-phase)之后剩余的聚合状态（例如，when状态）得到验证。

```java
fixture.givenState(() -> new GiftCard())
       .when(new RedeemCardCommand())
       .expectState(state -> {
           // perform assertions
       });
```

`ExpectState`方法采用Aggregate类型的使用者。使用测试框架提供的常规断言来断言给定聚合的状态。任何（运行时）异常或错误将相应地使测试用例失败。

> **事件源的聚合状态验证**
>
> 测试事件源聚合的状态验证被认为是不好的做法。理想情况下，聚合状态对于测试代码是完全不透明的，因为仅应验证行为。通常，验证状态的愿望表明测试套件中缺少某个测试方案。

#### 验证 Deadlines

验证阶段还提供选项来验证给定的聚合实例的计划和[deadlines](../deadlines/)。您可以通过`Duration`或`Instant`，使用显式的equals、`Matcher`或仅使用一个deadline类型来验证deadline消息。

以下方法可用于验证Deadlines：

* `expectScheduledDeadline(Duration, Object)`:

  显式地预期在指定的`Duration`之后安排一个给定的`deadline`。

* `expectScheduledDeadlineMatching(Duration, Matcher)`:

  预期在指定的`Duration`之后安排与`Matcher`匹配的deadline。

* `expectScheduledDeadlineOfType(Duration, Class)`:

  预期在指定的`Duration`之后安排与给定类型匹配的deadline。

* `expectScheduledDeadlineWithName(Duration, String)`:

  预期在指定的`Duration`之后安排与给定deadline名称匹配的deadline。

* `expectScheduledDeadline(Instant, Object)`:

  显式地预期在指定的`Instant`安排给定的`deadline`。

* `expectScheduledDeadlineMatching(Instant, Matcher)`:

  预期在指定的`Instant`安排与`Matcher`匹配的deadline。

* `expectScheduledDeadlineOfType(Instant, Class)`:

  预期在指定的`Instant`安排与给定类型匹配的deadline。

* `expectScheduledDeadlineWithName(Instant, String)`:

  预期在指定的`Instant`安排与给定deadline名称匹配的deadline。

* `expectNoScheduledDeadlines()`:

  预期没有deadlines被安排。

* `expectNoScheduledDeadlineMatching(Matcher)`:

  预期没有匹配`Matcher`的deadline被安排。

* `expectNoScheduledDeadlineMatching(Duration, Matcher)`:

  预期匹配`Matcher`的deadline在指定的`Duration`之后被安排。

* `expectNoScheduledDeadline(Duration, Object)`

  显式地预期没有给定的`deadline`被安排在指定的`Duration`之后。

* `expectNoScheduledDeadlineOfType(Duration, Class)`

  预期在指定的`Duration`之后安排不匹配给定类型的deadline。\ '

* `expectNoScheduledDeadlineWithName(Duration, String)`

  预期在指定的`Duration`之后安排不匹配给定deadline名称的deadline。\ '

* `expectNoScheduledDeadlineMatching(Instant, Matcher)`:

  预期没有匹配`Matcher`的deadline被安排在指定的`Instant`。

* `expectNoScheduledDeadline(Instant, Object)`

  显式地预期没有给定的`deadline`被安排在指定的`Instant`。

* `expectNoScheduledDeadlineOfType(Instant, Class)`

  预期没有匹配给定类型的deadline被安排在指定的`Instant`。

* `expectNoScheduledDeadlineWithName(Instant, String)`

  预期没有匹配给定deadline名称的deadline被安排在指定的`Instant`。

* `expectDeadlinesMet(Object...)`:

  显式地预期一个或几个已经完成的`deadline`。

* `expectDeadlinesMetMatching(Matcher<List<DeadlineMessage>>)`:

  预期一个或几个匹配的deadline已经满足。

