# Sagas

类似于命令处理组件，sagas具有明确定义的界面：它们仅响应事件。另一方面，sagas通常具有时间概念，并且可能在事件处理过程中与其他组件交互。 Axon框架的测试支持模块包含可帮助您编写Sagas测试的fixture。

每个测试fixture包含三个阶段，类似于前一节中描述的命令处理组件fixture。

* 给定某些事件(来自某些聚合)，
* 当事件发生或时间流逝时，
* 预期某种行为或状态。

“given”和“when”阶段都接受事件作为它们交互的一部分。在“given”阶段，如果可能，所有副作用(如生成的命令)都将被忽略。另一方面，在“when”阶段，将记录saga生成的事件和命令，并可以进行验证。

下面的代码示例展示了如何使用fixture来测试saga，如果发票在30天内未支付，则会发送通知:

```java
FixtureConfiguration<InvoicingSaga> fixture = new SagaTestFixture<>(InvoicingSaga.class);
fixture.givenAggregate(invoiceId).published(new InvoiceCreatedEvent()) 
       .whenTimeElapses(Duration.ofDays(31)) 
       .expectDispatchedCommandsMatching(Matchers.listWithAllOf(aMarkAsOverdueCommand())); 
       // 或者，仅与命令消息的payload匹配
       .expectDispatchedCommandsMatching(Matchers.payloadsMatching(Matchers.listWithAllOf(aMarkAsOverdueCommand())));
```

Sagas可以使用回调发送命令，以便通知命令处理结果。由于在测试中没有实际的命令处理，行为是使用`CallbackBehavior`对象定义的。该对象在fixture上使用`setCallbackBehavior()`注册，并定义在发送命令时是否以及如何必须调用回调。

除了直接使用`CommandBus`，您还可以使用命令网关。请参阅下面如何指定它们的行为。

通常，sagas会与资源进行交互。这些资源不是saga状态的一部分，而是在saga加载或创建后注入的。测试装置允许您注册saga中需要注入的资源。要注册一个资源，只需以资源作为参数调用`fixure.registerResource(Object)`方法。该fixture将检测saga上适当的setter方法或字段(带有`@Inject`注解)，并使用可用资源调用它。

> **提示**
>
> 它可以非常有用的注入模拟对象(例如。Mockito或Easymock)进入您的Saga。它允许您验证saga与您的外部资源是否正确交互。

命令网关为sagas提供了一种更简单的调度命令的方法。使用自定义命令网关还可以更容易地创建模拟或存根来定义其在测试中的行为。然而，在提供模拟或存根时，实际的命令可能不会被发送，从而无法在测试fixture中验证发送的命令。

因此，fixture提供了两个方法，允许您注册命令网关和可选的定义其行为的模拟: `registerCommandGateway(Class)`和`registerCommandGateway(Class, Object)`。两个方法都返回给定类的一个实例，该类表示要使用的网关。这个实例也被注册为资源，以使其符合资源注入的条件。

当`registerCommandGateway(Class)`被用来注册网关时，它会将命令发送给由fixture管理的CommandBus。网关的行为主要由设备上定义的`CallbackBehavior`定义。如果没有提供显式的`CallbackBehavior`，回调函数将不会被调用，这使得它不可能为网关提供任何返回值。

当`registerCommandGateway(Class, Object)`被用来注册网关时，第二个参数被用来定义网关的行为。

测试治具试图消除可能的系统时间浪费。
测试fixture试图消除可能的系统时间浪费。这意味着在执行测试时，它看起来没有时间浪费，除非您使用`whenTimeElapses()`显式地声明。所有事件都将具有测试fixture创建时的时间戳。

在测试期间停止时间可以更容易预测将在什么时间发布事件。如果您的测试用例验证了某个事件计划在30秒内发布，则该事件将保留30秒，无论实际调度和测试执行之间花费了多长时间。

> **注意**
>
> fixture为基于时间的活动使用`StubScheduler`，例如调度事件和推进时间。fixture将发送到Saga实例的任何事件的时间戳设置为该调度程序的时间。这意味着时间在fixture启动时就被“停止”了，并且可以使用`whenTimeAdvanceTo`和`whenTimeElapses`方法显式地提前。

如果需要测试事件调度，也可以独立于测试fixture使用`StubEventScheduler`。此`EventScheduler`实现允许您验证在哪个时间安排哪些事件，并提供操作时间进度的选项。您可以使用指定的`Duration`提前时间，将时钟移动到特定的日期和时间，或者将时间提前到下一个计划的事件。所有这些操作都将返回进度间隔内计划的事件。
