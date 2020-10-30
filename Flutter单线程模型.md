Flutter 单线程模型保证UI运行流畅

Event Loop机制
Dart是单线程。意味着Dart代码是有序的，按照在main函数出现的次序一个接一个地执行，不会被其他代码打断。另外，作为支持Flutter这个UI框架的关键技术，Dart当然也支持异步。单线程和异步并不冲突。
单线程也可以异步
App绝大数时间都是在等待。比如，等用户点击，等网络请求返回，等文件IO结果，等等。这些等待行为并不是阻塞的。再如网络请求，Socket本身提供了select模型可以异步查询；而文件IO，操作系统也是基于事件的回调机制。所以基于这些特点，单线程模型可以在等待的过程中做别的事情，等真正需要相应结果了，再去做相应的处理。因为在等待过程并不是阻塞的，所以给我们的感觉就像是同时在做很多事情一样。但其实始终只有一个线程在处理事情。等待这个行为是通过Event Loop驱动的。事件队列Event Queue会把其他平行事件（如Socket）完成的，需要主线程相应的时间放入其中。与其他语言一样，Dart也是有一个巨大的事件循环，在不断的轮询事件队列，取出事件（比如，键盘事件，IO事件，网络事件等），在主线程同步执行其回调函数。如图所示：

异步任务
在Dart中，实际上有两个队列，一个事件队列（Event Queue），另一个则是微任务队列（Microtask Queue）。在每一次事件循环中，Dart总是先去第一个微任务队列中查询是否有可执行的任务，才会处理后续的事件队列的流程。Event Loop完整版的流程图，如下所示：

事件队列和微任务队列使用场景：
微任务队列顾名思义，表示一个短时间内就会完成的异步任务。从上图的流程可以看到，微任务在事件循环中的优先级是最高的，只要队列中有任务，就可以一直霸占着事件循环。微任务是由ScheduleMicroTask建立的。比如：scheduleMicrotask(() => print('This is a microtask'));一般的异步任务通常也很少必须要在事件队列前完成，所以也不需要太高的优先级。很少直接使用微任务队列。在Flutter内部，只有7处用户微任务队列（比如：手势识别，文本输入，滚动视图，保存页面效果等高优先执行的场景）。
异步任务使用最多的还是优先级更低的Event Queue。比如：IO，绘制，定时器这些异步事件，都是通过事件队列驱动主线程执行的。
Dart为Event Queue的任务建立提供了一层封装，叫座Future。它表示一个在未来时间才会完成的任务。把一个函数体放入Future，就完成了从同步任务到异步任务的包装。Future还提供了链式调用的能力。可以在异步任务执行完毕后一次执行链路上的其他函数体。
下面是具体代码，分别盛名两个异步任务，在下一个事件循环中输出一段字符串。其中第二个任务执行完毕后，还会继续输出另外两段字符串：
Future(() => print('Running in Future 1'));//下一个事件循环输出字符串
Future(() => print(‘Running in Future 2'))
.then((_) => print('and then 1'))
.then((_) => print('and then 2’));//上一个事件循环结束后，连续输出三段字符串
这两个Future异步任务的执行优先级都比微任务的优先级低。
正常情况下，一个Future异步任务的执行是相对简单的：声明一个Future时，Dart会将异步任务的函数执行体投入到事件队列，然后立即返回，后续的代码继续同步执行。而当同步执行的代码执行完毕后，事件队列会按照加入事件队列的顺序（即声明顺序），依次取出事件，最后同步执行Future的函数体及后续的then。这意味着，then与Future函数体共用一个事件循环。而如果Future有多个then，它们会按照链式调用的先后顺序同步执行。同样也会共用一个事件循环。如果Future执行体已经执行完毕，此时又有这个Future的引用，往后面加一个then方法体，面对这种情况，Dart会将后续加入的then方法体放入微任务队列，尽快执行。下面的代码演示Future的执行规则，即先加入事件队列，或者先声明的任务先执行；then在Future结束后立即执行。
在第一个例子中，由于f1比f2先声明，所以f1先被放入事件队列，所以f1比f2先执行。
//f1比f2先执行
Future(() => print('f1'));
Future(() => print('f2'))；
在第二个例子中，由于Future函数体与then共用同一个事件循环，因此f3执行后立刻同步执行then3；
//f3执行后会立刻同步执行then 3
Future(() => print('f3')).then((_) => print('then 3'));
最后一个例子中，Future函数体是null，这意味它不需要也不需要事件循环，因为后续的then也无法与它共享。在这种场景下，Dart会把后续的then放入微任务队列，在下一次事件循环中执行。
//then 4会加入微任务队列，尽快执行
Future(() => null).then((_) => print('then 4'));
下面的例子中，依次声明若干个异步任务Future，以及微任务。在其中的一些Future内部，又嵌入了Future和microtask的声明：
Future(() => print('f1'));//声明一个匿名Future
Future fx = Future(() =>  null);//声明Future fx，其执行体为null
//声明一个匿名Future，并注册了两个then。在第一个then回调里启动了一个微任务
Future(() => print('f2')).then((_) {
print('f3');
scheduleMicrotask(() => print('f4'));
}).then((_) => print('f5'));
//声明了一个匿名Future，并注册了两个then。第一个then是一个Future
Future(() => print('f6'))
.then((_) => Future(() => print('f7')))
.then((_) => print('f8'));
//声明了一个匿名Future
Future(() => print('f9'));
//往执行体为null的fx注册了了一个then
fx.then((_) => print('f10'));
//启动一个微任务
scheduleMicrotask(() => print('f11'));
print('f12');
运行一下，上述各个异步任务会依次打印其内部执行结果：
f12
f11
f1
f10
f2
f3
f5
f4
f6
f9
f7
f8
看到打印结果蒙了。Event Queue与Microtask Queue中的变化情况。依次分析一下它们的执行顺序是这样的。

因为其他语句都是异步任务，所以先打印f12
剩下的异步任务中，微任务队列优先级最高，因此随后打印f11；然后按照Future声明的先后顺序打印f1
随后到了fx，由于fx的执行体是null，相当于执行完毕，Dart将fx的then放入微任务队列，由于微任务队列的优先级高，因此fx的then还是会最先执行，打印f10
然后就是fx下面的f2，打印f2，然后执行then，打印f3。f4是一个微任务，要到下一个事件循环才执行，因此后续then继续不同步执行，打印f5.本次事件循环结束，下一个事件循环取出来f4这个微任务，打印f4.
然后到了f2下面的f6，打印f6，然后执行then。这里需要注意的是，这个then是一个Future异步任务，因此这个then，以及后续的then都被放入到事件队列中了。
f6下面还有f9，打印f9
最后一个事件循环，打印f7，以及后续的f8
上面的代码很烧脑，万幸平时开发Future时，一般不会遇到这么奇葩的写法。只需要记住一点：then会在Future函数体执行完毕后立刻执行，无论是共用同一个事件循环还是进入下一个微任务。
在深入理解Future异步任务执行的执行规则之后，接下来如何封装一个异步函数
异步函数
对于一个异步函数来说，其返回时内部执行动作并未结束，因此需要返回一个Future对象，供调用着使用。调用着根据Future对象，来决定：是在这个Future对象上注册一个then，等Future的执行体结束了以后再进行异步处理；还是一直同步等待Future执行体结束。对于异步函数返回的Future对象，如果调用着决定同步等待，则需要在调用使用await关键字，并且在调用处的函数体使用使用async关键字。下面的例子中，异步方法延迟3秒返回一个Hello 2019，在调用处我们使用await进行持续等待，等它返回了再打印：
//声明了一个延迟3秒返回Hello的Future，并注册了一个then返回拼接后的Hello 2019
Future<String> fetchContent() =>
Future<String>.delayed(Duration(seconds:3), () => "Hello")
.then((x) => "$x 2019");
main() async{
print(await fetchContent());//等待Hello 2019的返回
}
注意到在使用await进行等待的时候，在等待语句的调用上下文函数main加上了async关键字，为什么要加这个关键词呢？
因为Dart中的await并不是阻塞等待，而是异步等待。Dart会将调用体的函数也视为异步函数，在等待语句的上下文放入Event Queue中，一旦有了结果，Event Loop就会把它从Event Queue中取出，等待代码继续执行。下面的两个具体例子，来加深印象。第一例子，第二行的then执行体f2是一个Future，为了等它完成再进行下一步操作，使用了await，期望打印结果为f1，f2，f3，f4:
Future(() => print('f1'))
.then((_) async => await Future(() => print('f2'))) .then((_) => print('f3'));
Future(() => print('f4'));
实际上，运行代码，打印结果为f1，f4，f2，f3
分析代码的执行顺序：
按照任务的声明顺序，f1和f4被先后加入事件队列。
f1被取出并打印；然后到了then。then的执行体是个future f2，于世被放入Event Queue。然后把await也方法Event Queue。
Event Queue里面还有一个f4，await并不能阻塞f4的执行。因此，Event Loop先取出f4，打印f4；然后才能取出并打印f2，最后把等待的await取出，开始执行后面的f3.
由于await是采用事件队列的机制实现等待行为的，所以比它先在事件队列中的f4并不会被阻塞。
下面是另外一个例子：在主函数调用一个异步函数去打印一段话，而在这个异步函数中，我们使用await与async同步等待另一个异步函数返回字符串：
//声明了一个延迟2秒返回Hello的Future，并注册了一个then返回拼接后的Hello 2019
Future<String> fetchContent() =>
Future<String>.delayed(Duration(seconds:2), () => "Hello")
.then((x) => "$x 2019");
//异步函数会同步等待Hello 2019的返回，并打印
func() async => print(await fetchContent());
main() {
print("func before");
func();
print("func after");
}
最终的输出顺序其实是“func before”“func after”“hello 2019”。func函数中的等待语句似乎没起到作用。这是为什么呢？
分析代码的执行顺序如下：
首先，第一句代码是同步的，因此先打印“func before”
然后，进入func函数，func函数调用异步函数fetchContent，并使用await进行等待，把fetchContent，await语句的上下文函数func先后放入事件队列。
await的上下文函数并不包含调用栈，因此func后续代码继续执行，打印“func after”。
2秒后，fetchContent异步任务返回“Hello 2019”，于是func的await也被取出，打印“Hello 2019”。
通过分析，await和async只对调用上下文的函数有效，并不向上传递。因此对于上述例子，func是在异步等待。如果想在main函数中也同步等待，需要在调用异步函数时加上await，在main函数也加上async。
多线程实现并发
Isolate
尽快Dart是基于单线程模式的，为了进一步利用多核CPU，将CPU密集型运算进行隔离，Dart也提供了多线程机制，即Isolate。在Isolate中，资源隔离做的非常好，每个Isolate都有自己的Event Loop与Queue，Isolate之间不共享任何资源，只能依靠消息机制通信，因此也就没有资源抢占问题。
同其他语言一样，Isolate的创建很简单，只要给定一个函数入口，创建时再传入一个参数，就可以启动Isolate。如下面代码，声明一个Isolate的入口函数，然后在main函数中启动它，并传入一个字符串参数：
doSth(msg) => print(msg);
main() {
Isolate.spawn(doSth, "Hi");
...
}
更多情况下，并不会这么简单，不仅希望并发，还希望Isolate在并发执行的时候告诉主Isolate当前的执行结果。对于执行结果的告知，Isolate通过发送管道（SendPort）实现消息通信机制。可以在启动并发Isolate时将主Isolate的发动管道作为参数传递给它，这样并发Isolate就可以在任务执行完毕后利用这个管道给主Isloate发消息。
下面例子来说明：在主Isolate里，创建一个并发Isolate，在函数入口传入主Isloate的发送管道，然后等待并发Isloate的回传消息。在并发Isloate中，用这个管道给主Isolate发送一个字符串Hello：
Isolate isolate;
start() async {
ReceivePort receivePort= ReceivePort();//创建管道
//创建并发Isolate，并传入发送管道
isolate = await Isolate.spawn(getMsg, receivePort.sendPort);
//监听管道消息
receivePort.listen((data) {
print('Data：$data');
receivePort.close();//关闭管道
isolate?.kill(priority: Isolate.immediate);//杀死并发Isolate
isolate = null;
});
}
//并发Isolate往管道发送一个字符串
getMsg(sendPort) => sendPort.send("Hello");
需要注意：在Isolate中，发送管道是单向的，启动一个Isolate执行某项任务，Isoate执行完毕，发送消息告知主Isloate。如果Isloate执行任务时，需要依赖主Isolate给它发送的参数，执行完毕后再发送执行结果给主Isolate，这样双向通道。让并发Isolate也回传一个发送管道即可。
加下来的例子，实现一个并发计算阶乘，来说明如何实现双向通信：
创建了一个异步函数计算阶乘。在这个异步函数内，创建了一个并发 Isolate，传入主 Isolate 的发送管道；并发 Isolate 也回传一个发送管道；主 Isolate 收到回传管道后，发送参数 N 给并发 Isolate，然后立即返回一个 Future；并发 Isolate 用参数 N，调用同步计算阶乘的函数，返回执行结果；最后，主 Isolate 打印了返回结果：
//并发计算阶乘
Future<dynamic> asyncFactoriali(n) async{
final response = ReceivePort();//创建管道
//创建并发Isolate，并传入管道
await Isolate.spawn(_isolate,response.sendPort);
//等待Isolate回传管道
final sendPort = await response.first as SendPort;
//创建了另一个管道answer
final answer = ReceivePort();
//往Isolate回传的管道中发送参数，同时传入answer管道
sendPort.send([n,answer.sendPort]);
return answer.first;//等待Isolate通过answer管道回传执行结果
}
//Isolate函数体，参数是主Isolate传入的管道
_isolate(initialReplyTo) async {
final port = ReceivePort();//创建管道
initialReplyTo.send(port.sendPort);//往主Isolate回传管道
final message = await port.first as List;//等待主Isolate发送消息(参数和回传结果的管道)
final data = message[0] as int;//参数
final send = message[1] as SendPort;//回传结果的管道
send.send(syncFactorial(data));//调用同步计算阶乘的函数回传结果
}
//同步计算阶乘
int syncFactorial(n) => n < 2 ? n : n * syncFactorial(n-1);
main() async => print(await asyncFactoriali(4));//等待并发计算阶乘结果
没错，确实太繁琐了。在 Flutter 中，像这样执行并发计算任务可以采用更简单的方式。Flutter 提供了支持并发计算的 compute 函数，其内部对 Isolate 的创建和双向通信进行了封装抽象，屏蔽了很多底层细节，调用时只需要传入函数入口和函数参数，就能够实现并发计算和消息通知。
试着用 compute 函数改造一下并发计算阶乘的代码
//同步计算阶乘int syncFactorial(n) => n < 2 ? n : n * syncFactorial(n-1);//使用compute函数封装Isolate的创建和结果的返回main() async => print(await compute(syncFactorial, 4));
总结：
Dart 是单线程的，但通过事件循环可以实现异步。而 Future 是异步任务的封装，借助于 await 与 async，可以通过事件循环实现非阻塞的同步等待；Isolate 是 Dart 中的多线程，可以实现并发，有自己的事件循环与 Queue，独占资源。Isolate 之间可以通过消息机制进行单向通信，这些传递的消息通过对方的事件循环驱动对方进行异步处理。在 UI 编程过程中，异步和多线程是两个相伴相生的名词，也是很容易混淆的概念。对于异步方法调用而言，代码不需要等待结果的返回，而是通过其他手段（比如通知、回调、事件循环或多线程）在后续的某个时刻主动（或被动）地接收执行结果。因此，从辩证关系上来看，异步与多线程并不是一个同等关系：异步是目的，多线程只是实现异步的一个手段之一。而在 Flutter 中，借助于 UI 框架提供的事件循环，可以不用阻塞的同时等待多个异步任务，因此并不需要开多线程。一定要记住这一点。
