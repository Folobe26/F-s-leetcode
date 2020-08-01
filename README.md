# RAC复习

## Intro

- RAC 用 Signal 代替 Mutable 的变量来表达过去和未来的值
- RAC 的目的是让代码变成声明式，而不是过程式的观察和更新
- RAC 解决的一个**核心痛点**是为异步处理提供一个单一通用的方法，目前iOS下有这些机制
  - delegate
  - callback block
  - target-action
  - notification
  - KVO
- RAC 提供了比较好的函数式编程体验
- 更少的状态，更少的模版代码，代码更集中，更好的代码语义性



### Signal

Signal 在时间维度上表达一切数据流，也可以表达诸如 button 点击之类的事件值。

Signal 可以去获取状态，用来创建类似计算属性的概念

```objective-c
//创建一个signal，signal每次的值从reduce block中产生，赋值到 self.createEnabled上
RAC(self.loginButton, userInteractionEnabled) = [RACSignal
	combineLatest:@[ RACObserve(self, password), RACObserve(self, passwordConfirmation) ]
	reduce:^(NSString *password, NSString *passwordConfirm) {
		return @([passwordConfirm isEqualToString:password]);
}];
```

使用signal的高阶操作来完成一些复杂的过程式逻辑

使用 merge 代替 dispatchGroup 等

```objective-c
//相比于使用GCD，使用signal更加优雅
[[RACSignal merge:@[ [client fetchUserRepos], [client fetchOrgRepos] ]]
						subscribeCompleted:^{
							NSLog(@"They're both done!");
	}];
```



通过类似 Masonry 的方式避免嵌套地狱

```objective-c
[[[[client	logInUser]
      flattenMap:^(User *user) {
        // Return a signal that loads cached messages for the user.
        return [client loadCachedMessagesForUser:user];
      }]
      flattenMap:^(NSArray *messages) {
        // Return a signal that fetches any remaining messages.
        return [client fetchMessagesAfterMessage:messages.lastObject];
      }]
      subscribeNext:^(NSArray *newMessages) {
        NSLog(@"New messages: %@", newMessages);
      } completed:^{
        NSLog(@"Fetched all messages.");
}];
```



### Command

主要用来把target-action模式转换为RAC的signal的形式。

- 使用**RACCommand**来描述网络请求

```
self.loginCommand = [[RACCommand alloc] initWithSignalBlock:^(id sender) {
	return [client logIn];
}];

[self.loginCommand.executionSignals subscribeNext:^(RACSignal *loginSignal) {
	[loginSignal subscribeCompleted:^{
		NSLog(@"Logged in successfully!");
	}];
}];

self.loginButton.rac_command = self.loginCommand;
```

  - 使用**RACCommand**用来表达 UI 控件的动作

```objective-c
self.button.rac_command = [[RACCommand alloc] initWithSignalBlock:^(id _) {
	NSLog(@"button was pressed!");
	return [RACSignal empty];
}];
```



## RAC Excels At

RAC的几个优秀的应用场景

### 处理异步或者事件驱动的data source

使用 **RACSignal** 对【**UI 回调**、**网络回调**、**KVO通知**】进行抽象。

```objective-c
	@weakify(self);

	RAC(self.logInButton, enabled) = [RACSignal
		combineLatest:@[
			self.usernameTextField.rac_textSignal,
			self.passwordTextField.rac_textSignal,
			RACObserve(LoginManager.sharedManager, loggingIn),
			RACObserve(self, loggedIn)
		] reduce:^(NSString *username, NSString *password, NSNumber *loggingIn, NSNumber *loggedIn) {
			return @(username.length > 0 && password.length > 0 && !loggingIn.boolValue && !loggedIn.boolValue);
		}];

	[[self.logInButton rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(UIButton *sender) {
		@strongify(self);

		RACSignal *loginSignal = [LoginManager.sharedManager
			logInWithUsername:self.usernameTextField.text
			password:self.passwordTextField.text];

			[loginSignal subscribeError:^(NSError *error) {
				@strongify(self);
				[self presentError:error];
			} completed:^{
				@strongify(self);
				self.loggedIn = YES;
			}];
	}];

	RAC(self, loggedIn) = [[NSNotificationCenter.defaultCenter
		rac_addObserverForName:UserDidLogOutNotification object:nil]
		mapReplace:@NO];
```



### 把不相关的事件串联起来【模块之间的截耦合和组合】

```
[client logInWithSuccess:^{
	[client loadCachedMessagesWithSuccess:^(NSArray *messages) {
		[client fetchMessagesAfterMessage:messages.lastObject success:^(NSArray *nextMessages) {
			NSLog(@"Fetched all messages.");
		} failure:^(NSError *error) {
			[self presentError:error];
		}];
	} failure:^(NSError *error) {
		[self presentError:error];
	}];
} failure:^(NSError *error) {
	[self presentError:error];
}];
```



### 并发执行不相关的事件

前述代替GCD group的例子



### 简化集合的处理

```
RACSequence *results = [[strings.rac_sequence
	filter:^ BOOL (NSString *str) {
		return str.length >= 2;
	}]
	map:^(NSString *str) {
		return [str stringByAppendingString:@"foobar"];
	}];
```





# RACSignal

最基础的概念之一。

头文件分了5个部分：

- RACSignal (RACStream)

- RACSignal (RACStreamOperations)

- RACSignal (Subscription)

- RACSignal (Debugging)

- RACSignal (Testing)

RACSignal是RACStream的子类，自然有一部分是Override的，也就是(RACStream)部分，除了 `CreateSignal:`方法。



RACSignal 是 push-driven 的 Stream，意味着数据流由我们主动 push，也就是sendNext，sendComplete，sendError。

是一个 Passive 的Stream。

每一个订阅，都会触发didSubScribe block，在block里对subscriber进行发送值的操作。

订阅不产生订阅和被订阅者的持有关系。



常见用法：

#### 单元信号

最简单的信号是单元信号，有 4 种：

```objectivec
// return信号：被订阅后，立马产生一个值事件，然后产生一个完成事件
RACSignal *signal1 = [RACSignal return:someObject];
// error信号：被订阅后，立马产生一个错误事件
RACSignal *signal2 = [RACSignal error:someError];
// empty信号：被订阅后，立马产生一个完成事件
RACSignal *signal3 = [RACSignal empty];
// never信号：永远不产生事件
RACSignal *signal4 = [RACSignal never];
```

#### 动态信号

```objectivec
RACSignal *signal5 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"1"];
    [subscriber sendNext:@"2"];
    [subscriber sendCompleted];
    return [RACDisposable disposableWithBlock:^{
        
    }];
}];
```

#### Cocoa 桥接

RAC 为大量的 Cocoa 类型提供便捷的信号桥接工具，如下是一些常见的桥接方式：

```objectivec
RACSignal *signal6 = [object rac_signalForSelector:@selector(setFrame:)];
RACSignal *signal7 = [control rac_signalForControlEvents:UIControlEventTouchUpInside];
RACSignal *signal8 = [object rac_willDeallocSignal];
RACSignal *signal9 = RACObserve(object, keyPath);
```

#### 信号变换

```objectivec
RACSignal *signal10 = [signal1 map:^id(id value) {
    return someObject;
}];
```

#### 序列变换

```objectivec
RACSignal *signal11 = sequence.signal;
```



### 订阅信号

- 通过`subscribeNext:error:completed:`方法订阅
- RAC 宏绑定
  - 直接把signal的value赋值到对应的快捷属性上

```objectivec
RAC(view, backgroundColor) = signal10;
// 每当signal10产生一个值事件，就将view.backgroundColor设为相应的值
```

- Cocoa 桥接

```
[object rac_liftSelector:@selector(someSelector:) withSignals:signal1, signal2, nil];
[object rac_liftSelector:@selector(someSelector:) withSignalsFromArray:@[signal1, signal2]];
[object rac_liftSelector:@selector(someSelector:) withSignalOfArguments:signal1];
```



## createSignal:

一个很简单的方法：

1. 创建RACDynamicSignal实体
2. copy didSubscribe block
3. set name



创建的信号在第一次subscribe的时候执行 didSubscribe block。



## subscribe:

1. 创建 RACCompoundDisposable 对象：disposable
2. 重新包装subscriber

```objective-c
subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];
```

3. 使用 RACScheduler 执行 didSubscribe block
   1. 生成一个schedulingDisposable
4. 把schedulingDisposable 加入 disposable 



这里涉及三个点：

- 什么是RACCompoundDisposable
- 什么是RACScheduler

接着往下看

# RACDisposable

一个Disposable对象cover了清除一个subscription所需要的工作。

impl里全是骚代码，一个个看

### alloc

```objective-c
- (instancetype)init {
	self = [super init];

	_disposeBlock = (__bridge void *)self;
	OSMemoryBarrier();

	return self;
}

- (instancetype)initWithBlock:(void (^)(void))block {
	NSCParameterAssert(block != nil);

	self = [super init];

	_disposeBlock = (void *)CFBridgingRetain([block copy]); 
	OSMemoryBarrier();

	return self;
}
```



这个MemoryBarrier觉得有人会alloc以后分两个线程对这个地址进行init吗？【手动迷惑】



### dealloc

```
- (void)dealloc {
	if (_disposeBlock == NULL || _disposeBlock == (__bridge void *)self) return;

	CFRelease(_disposeBlock);
	_disposeBlock = NULL;
}
```



_disposeBlock有可能是self【参考init】，所以需要手动解除retain cycle



### dispose

```objective-c
- (void)dispose {
	void (^disposeBlock)(void) = NULL;

	while (YES) {
		void *blockPtr = _disposeBlock;
		if (OSAtomicCompareAndSwapPtrBarrier(blockPtr, NULL, &_disposeBlock)) {
			if (blockPtr != (__bridge void *)self) {
				disposeBlock = CFBridgingRelease(blockPtr);
			}

			break;
		}
	}

	if (disposeBlock != nil) disposeBlock();
}
```

先使用一个局部变量避免多线程竞争，再使用【原子化的指针比较函数】进行比较和赋值

CFBridgeRelease的作用是把blockPtr指向的void *交还给ARC管理，是为了能顺利释放



前面都是在检查和处理



函数的主要功能是执行disposeBlock



# RACScheduler

类似于NSOpertation，略过不表。



# 热信号 RACSubject

冷信号是被动的（passive），之所以说被动，是指信号和信号的alue的产生是被动的。

只会在被订阅时向订阅者发送values（触发didSubscribed block），values是在didSubscribed block中实时计算的。

热信号是主动的，它会在任意时间发出某个value，与订阅者的订阅行为无关。value也不是由signal本身计算的，而是外部丢进来的，就像冷信号丢给subscriber一样。基于这种抽象，可以认为热信号就是一个subscriber，同时被其他subscriber订阅。于是，热信号RACSubject，也是遵循subscriber协议（`<RACSubscriber>`）的。



RACSubject是RACSignal的子类

**实现了RACSubscriber协议，让我们可以fully control it，实际上它是一个信号源和订阅者的集合体**

**只有订阅者可以接受信号**



那么如果信号源想动态的响应其他地方的行为，也就是动态的接受外界信息，这部分行为可以被同化为一个【订阅者】，【信号源也是订阅者】的概念也就随之产生。



参考工具线现有的viewmodel中的用法，通常我们内部声明一个RACSubject，在viewmodel内部控制其信号的next error complete，在外部声明一个RACSignal，来表达【readonly】，不允许外部send next等。



RACSubject 的订阅相比于RACSignal多了一个add subscriber的操作：

```objective-c
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	NSMutableArray *subscribers = self.subscribers;
	@synchronized (subscribers) {
		[subscribers addObject:subscriber];
	}

	[disposable addDisposable:[RACDisposable disposableWithBlock:^{
		@synchronized (subscribers) {
			NSUInteger index = [subscribers indexOfObjectWithOptions:NSEnumerationReverse passingTest:^ BOOL (id<RACSubscriber> obj, NSUInteger index, BOOL *stop) {
				return obj == subscriber;
			}];

			if (index != NSNotFound) [subscribers removeObjectAtIndex:index];
		}
	}]];

	return disposable;
}
```

这里需要注意的是，在disposeblock里，RAC使用了in time search，这里理论上会进一步降低效率

```objective-c
- (void)sendNext:(id)value {
	[self enumerateSubscribersUsingBlock:^(id<RACSubscriber> subscriber) {
		[subscriber sendNext:value];
	}];
}

- (void)sendError:(NSError *)error {
	[self.disposable dispose];
	
	[self enumerateSubscribersUsingBlock:^(id<RACSubscriber> subscriber) {
		[subscriber sendError:error];
	}];
}

- (void)sendCompleted {
	[self.disposable dispose];
	
	[self enumerateSubscribersUsingBlock:^(id<RACSubscriber> subscriber) {
		[subscriber sendCompleted];
	}];
}
```

这部分代码完整表达了RACSubject如何充当一个信号源和订阅者双重角色。



## RACBehaviorSubject

可以理解为 capacity 为1的RACReplaySubject



## RACReplaySubject

内部构建一个buffer，每次订阅的时候都会重播buffer中的value

- **@property** (**nonatomic**, **assign**, **readonly**) NSUInteger capacity;
- **@property** (**nonatomic**, **strong**, **readonly**) NSMutableArray *valuesReceived;// This property should only be modified while synchronized on self.

```objective-c
for (id value in self.valuesReceived) {
				if (compoundDisposable.disposed) return;

				[subscriber sendNext:(value == RACTupleNil.tupleNil ? nil : value)];
			}

			if (compoundDisposable.disposed) return;

			if (self.hasCompleted) {
				[subscriber sendCompleted];
			} else if (self.hasError) {
				[subscriber sendError:self.error];
			} else {
				RACDisposable *subscriptionDisposable = [super subscribe:subscriber];
				[compoundDisposable addDisposable:subscriptionDisposable];
			}
```

但是性能低下

```objective-c
- (void)sendNext:(id)value {
	@synchronized (self) {
		[self.valuesReceived addObject:value ?: RACTupleNil.tupleNil];
		
		if (self.capacity != RACReplaySubjectUnlimitedCapacity && self.valuesReceived.count > self.capacity) {
			[self.valuesReceived removeObjectsInRange:NSMakeRange(0, self.valuesReceived.count - self.capacity)];
		}
		
		[super sendNext:value];
	}
}
```



# RACMulticastConnection

在介绍RACCommand前需要看下这个东西，Command依赖了这个东西。

一个很不错的blog：https://draveness.me/racconnection/



简单说，这个类是一个冷信号RACSignal动态转换为热信号后的结果。



`RACMulticastConnection` 封装了将一个RACSubject，去订阅冷信号，同时供多个其他对象订阅。

它的每一个对象都持有两个 `RACSignal`。一个是私有的源信号 `sourceSignal`，另一个是用于广播的信号 `signal`，其实是`RACSubject` 对象，不过对外只提供 `RACSignal` 接口，用于使用者通过 `-subscribeNext:` 等方法进行订阅。

`RACMulticastConnection` 不能直接初始化，因为它是用来把冷的RACSignal转换为热信号用的，所以需要对一个RACSignal使用 publish 或者 multicast 方法。

```objective-c
- (RACMulticastConnection *)publish {
	RACSubject *subject = [RACSubject subject];
	RACMulticastConnection *connection = [self multicast:subject];
	return connection;
}

- (RACMulticastConnection *)multicast:(RACSubject *)subject {
	RACMulticastConnection *connection = [[RACMulticastConnection alloc] initWithSourceSignal:self subject:subject];
	return connection;
}
```



`RACMulticastConnection` 有一个connect方法需要调用，它的实际意义是，让热信号RACSubject B 开始订阅 我们需要转换的 RACSignal A。



也就是说从connect以后才订阅multicastConnection的热信号的subscriber有可能无法接收到信号值

也可以使用autoConnect自动建立connect。

```objective-c
RACSignal *sourceSignal = [RACSignal createSignal:...];
RACMulticastConnection *connection = [sourceSignal multicast:[RACReplaySubject subject]];
[connection.signal subscribeNext:^(id  _Nullable x) {
    NSLog(@"product: %@", x);
}];
[connection connect];
[connection.signal subscribeNext:^(id  _Nullable x) {
    NSNumber *productId = [x objectForKey:@"id"];
    NSLog(@"productId: %@", productId);
}];
```





# RACCommand : NSObject

回忆一下，RACCommand是为了把target-action模式处理为Reactive的signal而设计的。



一定注意，command不是一个signal或者subject，是一个全新的类，有一点manager的感觉，下面有4个只读信号源属性。也是RAC里不按FRP设计的一个类，是完全OOP的。



RACCommand 会创建并订阅一个信号来响应一定的行为，用来做一些 sides-effecting 的事情。



一个简单的理解是：

 `RACCommand` 将外部的变量 `InputType` 转换成了使用 `RACSignal` 包裹的 `ValueType` 对象。



先看使用

```objectivec
RACCommand *command = [[RACCommand alloc] initWithSignalBlock:^RACSignal * _Nonnull(NSNumber * _Nullable input) {
    return [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        NSInteger integer = [input integerValue];
        for (NSInteger i = 0; i < integer; i++) {
            [subscriber sendNext:@(i)];
        }
        [subscriber sendCompleted];
        return nil;
    }];
}];
[[command.executionSignals switchToLatest] subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@", x);
}];

[command execute:@1];
[RACScheduler.mainThreadScheduler afterDelay:0.1
                                    schedule:^{
                                        [command execute:@2];
                                    }];
[RACScheduler.mainThreadScheduler afterDelay:0.2
                                    schedule:^{
                                        [command execute:@3];
                                    }];
```

输出显然是 0 0 1 0 1 2



接下里开始解析这个类



先看头文件：

- **@property** (**nonatomic**, **strong**, **readonly**) RACSignal<RACSignal<ValueType> *> *executionSignals;
- **@property** (**nonatomic**, **strong**, **readonly**) RACSignal<NSNumber *> *executing;
- **@property** (**nonatomic**, **strong**, **readonly**) RACSignal<NSNumber *> *enabled;
- **@property** (**nonatomic**, **strong**, **readonly**) RACSignal<NSError *> *errors;

内部：

- **@property** (**nonatomic**, **strong**, **readonly**) RACSubject *addedExecutionSignalsSubject;

- **@property** (**nonatomic**, **strong**, **readonly**) RACSubject *allowsConcurrentExecutionSubject;

- **@property** (**nonatomic**, **strong**, **readonly**) RACSignal *immediateEnabled;

- **@property** (**nonatomic**, **copy**, **readonly**) RACSignal * (^signalBlock)(**id** input);



_addedExecutionSignalsSubject是最重要的信号，承载了外部行为来临时，向订阅者发布新值的职责。

外部的行为对应 `execute` 方法，那么我们看下内部如何执行。

```objective-c
- (RACSignal *)execute:(id)input {
//先检查immediateEnabled的值，这个值是可靠的，被禁用的时候会被立即更新
//immediateEnabled受addedExecutionSignalsSubject影响，所以在主线程多次执行 execute 不会成功。
	BOOL enabled = [[self.immediateEnabled first] boolValue];
	if (!enabled) {
		NSError *error = [NSError errorWithDomain:RACCommandErrorDomain code:RACCommandErrorNotEnabled userInfo:@{
			NSLocalizedDescriptionKey: NSLocalizedString(@"The command is disabled and cannot be executed", nil),
			RACUnderlyingCommandErrorKey: self
		}];

		return [RACSignal error:error];
	}
//执行signal block 获得一个signal
	RACSignal *signal = self.signalBlock(input);
	NSCAssert(signal != nil, @"nil signal returned from signal block for value: %@", input);

	// We subscribe to the signal on the main thread so that it occurs _after_
	// -addActiveExecutionSignal: completes below.
	//
	// This means that `executing` and `enabled` will send updated values before
	// the signal actually starts performing work.
	RACMulticastConnection *connection = [[signal
		subscribeOn:RACScheduler.mainThreadScheduler]
		multicast:[RACReplaySubject subject]];
	
	[self.addedExecutionSignalsSubject sendNext:connection.signal];

	[connection connect];
	return [connection.signal setNameWithFormat:@"%@ -execute: %@", self, RACDescription(input)];
}
```





方法不多，初始化方法：

```objective-c
- (instancetype)initWithEnabled:(nullable RACSignal<NSNumber *> *)enabledSignal signalBlock:(RACSignal<ValueType> * (^)(InputType _Nullable input))signalBlock;
```

初始化一个有条件enable的RACCommand。

- enabledSignal用来设置这个RACCommand的enable属性。enable属性default是YES
- signalBlock的input是传递到execute方法中的值，map这个值到一个signal上，返回这个signal。
  - 返回的signal会被发送到一个replaySubject上

代码比较多：

```objective-c
	_addedExecutionSignalsSubject = [RACSubject new];
//订阅 executionSignal 后的subject
	_allowsConcurrentExecutionSubject = [RACSubject new];
//允许RACCommand并发执行【后面解释】
	_signalBlock = [signalBlock copy];
//创建signal的block


	_executionSignals = [[[self.addedExecutionSignalsSubject
		map:^(RACSignal *signal) {
			return [signal catchTo:[RACSignal empty]];
		}]
		deliverOn:RACScheduler.mainThreadScheduler]
		setNameWithFormat:@"%@ -executionSignals", self];
	
//error需要做一个多播，能够记录所有的 executionSignal 的error
//主要是为了一个execution已经开始执行以后的订阅仍然有效。

	RACMulticastConnection *errorsConnection = [[[self.addedExecutionSignalsSubject
		flattenMap:^(RACSignal *signal) {
			return [[signal
				ignoreValues]
				catch:^(NSError *error) {
					return [RACSignal return:error];
				}];
		}]
		deliverOn:RACScheduler.mainThreadScheduler]
		publish];
//创建error的 multicast

	_errors = [errorsConnection.signal setNameWithFormat:@"%@ -errors", self];
	[errorsConnection connect];

//直接执行 executing signal
	RACSignal *immediateExecuting = [[[[self.addedExecutionSignalsSubject
		flattenMap:^(RACSignal *signal) {
			return [[[signal
				catchTo:[RACSignal empty]]
				then:^{
					return [RACSignal return:@-1];
				}]
				startWith:@1];
		}]
		scanWithStart:@0 reduce:^(NSNumber *running, NSNumber *next) {
			return @(running.integerValue + next.integerValue);
		}]
		map:^(NSNumber *count) {
			return @(count.integerValue > 0);
		}]
		startWith:@NO];

	_executing = [[[[[immediateExecuting
		deliverOn:RACScheduler.mainThreadScheduler]
		startWith:@NO]
		distinctUntilChanged]
		replayLast]
		setNameWithFormat:@"%@ -executing", self];

//允许并发
	RACSignal *moreExecutionsAllowed = [RACSignal
		if:[self.allowsConcurrentExecutionSubject startWith:@NO]
		then:[RACSignal return:@YES]
		else:[immediateExecuting not]];
	
	if (enabledSignal == nil) {
		enabledSignal = [RACSignal return:@YES];
	} else {
		enabledSignal = [enabledSignal startWith:@YES];
	}
	
	_immediateEnabled = [[[[RACSignal
		combineLatest:@[ enabledSignal, moreExecutionsAllowed ]]
		and]
		takeUntil:self.rac_willDeallocSignal]
		replayLast];
	
	_enabled = [[[[[self.immediateEnabled
		take:1]
		concat:[[self.immediateEnabled skip:1] deliverOn:RACScheduler.mainThreadScheduler]]
		distinctUntilChanged]
		replayLast]
		setNameWithFormat:@"%@ -enabled", self];

	return self;
}
```





# RACStream 

RACStream 是一个 Monad 模型，Monad 模型涉及函数式编程的核心理念，后面单开一个章节讲述。从结果上讲，RACStream具有一个 `bind:` 和一个`return`方法，可以完成函数式链接编程，是所有数据流动（Any stream of values）的抽象类型。

## Bind

该方法是RACStream必须实现的方法，对一个RACStream里的values们lazily bind一个block。只能被用来提前终止一个bind或者在某些状态下关闭该Stream。

参考RACSignal bind方法需要做以下事情：

1. 对Stream原来的value进行订阅
2. 当Stream发出新的value时，使用bind的block对value进行变换
3. 如果block返回了一个信号，订阅这个信号，并把所有的值再次发送给subscriber
4. 如果block要求结束信号，则把原信号结束掉
5. 当所有的signals都complete的时候，发送结束到subscriber
6. 如果出现error信号，发送到subscriber





# RACDelegateProxy

介绍了RACCommand，磨平了了target-action后，我们再来抹平RACDelegateProxy。

但是delegate磨平还是比较困难，rac的处理性能也比较低。这部分就不介绍了，delegate的时候还是用delegate，不要强行用rac。



# RACChannel

RACChannel是用来做【双向绑定】的。即，一方的值改变，另一方也改变。

实际进行双向绑定的时候使用的是子类而不是直接使用RACChannel，RACChannel为大部分UIKit组件进行了扩展。



目前RAC官方支持了的控件有：

- NSUserDefaults
- UIControl
- UIDatePicker
- UISegmentControl
- UISwitch
- UITextField



内部存放了两个 `RACChannelTerminal` 表达绑定的双方。分别是 leadingTerminal  followingTerminal

RACChannelTerminal 代表 RACChannel 的两端，是RACSignal的子类，并且遵循RACSubscriber，跟RACSubject一样，是一个热信号，区别：

- 设置新的F（followingTerminal）的时候，会把L（leadingTerminal）的值发送给F
- F只会把 _future_ 的值发送给leading

当然，作为 `RACSignal` 的子类，`RACChannelTerminal` 必须覆写 `-subscribe:` 方法：

```objectivec
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	return [self.values subscribe:subscriber];
}
```



RACChannel另一个点是，如果一方error或者complete，另一方也error或者complete，因此，error或者complete需要replay。

```objective-c
//RACChannel 的 init
- (instancetype)init {
	self = [super init];

	RACReplaySubject *leadingSubject = [RACReplaySubject replaySubjectWithCapacity:0];
	RACReplaySubject *followingSubject = [RACReplaySubject replaySubjectWithCapacity:1];

	[[leadingSubject ignoreValues] subscribe:followingSubject];
	[[followingSubject ignoreValues] subscribe:leadingSubject];

	_leadingTerminal = [[RACChannelTerminal alloc] initWithValues:leadingSubject otherTerminal:followingSubject];
	_followingTerminal = [[RACChannelTerminal alloc] initWithValues:followingSubject otherTerminal:leadingSubject];

	return self;
}
```

可以看出，RACChannelTerminal本身是个subject，包装的signal也是subject，因为希望能重播error或者complete。





双向绑定：

使用`RACKVOChannel` 来高效地完成双向绑定：

```objective-c
RACChannelTo(view, property) = RACChannelTo(model, property);
```

简单讲RACChannel的扩展是使用下标语法，下标语法会被自动解析为getter或者setter。【所以这里不存在给右值赋值的问题】

```objective-c
//RACChannelTo宏展开
[[RACKVOChannel alloc] initWithTarget:self keyPath:@"integerProperty" nilValue:@42][@"followingTerminal"];
```

![image-20200801225235513](/Users/liyifan/Library/Application Support/typora-user-images/image-20200801225235513.png)

使用RACKVOChannel的好处是没有暴露leadingTerminal

> `RACChannel` 非常适合于视图和模型之间的双向绑定，在对方的属性或者状态更新时及时通知自己，达到预期的效果；我们可以使用 ReactiveCocoa 中内置的很多与 `RACChannel` 有关的方法，来获取开箱即用的 `RACChannelTerminal`，当然也可以使用 `RACChannelTo` 通过 `RACKVOChannel` 来快速绑定类与类的属性。













