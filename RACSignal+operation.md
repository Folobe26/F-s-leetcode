# RACSignal操作大全

RACSignal+Operations.h内的操作大全

RAC的操作仅供查表参考，更规范的定义私以为应该以Rx系为准。

## doNext
//内部包装了一层signal，从而实现注入一个block，在next的时候执行

**\- (RACSignal<ValueType> *)doNext:(void (^)(ValueType _Nullable x))block RAC_WARN_UNUSED_RESULT;**


## doError
//类似于donext的原理

**\- (RACSignal<ValueType> *)doError:(void (^)(NSError * _Nonnull error))block RAC_WARN_UNUSED_RESULT;**


## doCompleted
//类似于donext原理

**\- (RACSignal<ValueType> *)doCompleted:(void (^)(void))block RAC_WARN_UNUSED_RESULT;**





```
[subscriber sendNext:@0];
        [subscriber sendNext:@1];
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
           [subscriber sendNext:@2];
        [subscriber sendNext:@3];
        [subscriber sendCompleted];         
});
```

## throttle
参考下面带predicate的

**\- (RACSignal<ValueType> *)throttle:(NSTimeInterval)interval RAC_WARN_UNUSED_RESULT;**



原信号发出第一个信号(@0)发出开始计时，在 `interval` 秒内如果第二个信号(@1) 符合 predicate 的判断条件，则该前一个信号会被忽略；如果2个信号间隔时间超过 `interval` 秒或者不满足 predicate 判断，则前一个信号会发给订阅者。

**\- (RACSignal<ValueType> *)throttle:(NSTimeInterval)interval valuesPassingTest:(BOOL (^)(id _Nullable next))predicate RAC_WARN_UNUSED_RESULT;**


## delay
方法是将原信号的next和complete事件延迟发送给订阅者，error事件不会delay

基于当前scheduler，如果当前没有，则创建一个后台的

**\- (RACSignal<ValueType> *)delay:(NSTimeInterval)interval RAC_WARN_UNUSED_RESULT;**


## repeat
在complete来后继续订阅，内部也是包装了一层signal

**\- (RACSignal<ValueType> *)repeat RAC_WARN_UNUSED_RESULT;**


## initially
///注入一个block，每次signal被订阅的时候都会触发这个block，也就是所谓的side effect

///  [[[[fileManager rac_createFileAtPath:path contents:data] initially:^{

///      // 2. Second, backup current file

///      [fileManager moveItemAtPath:path toPath:backupPath error:nil];

///    }]

///    initially:^{

///      // 1. First, acquire write lock.

///      [writeLock lock];

///    }]

///    finally:^{

///      [writeLock unlock];

///    }];

\- (RACSignal<ValueType> *)initially:(**void** (^)(**void**))block RAC_WARN_UNUSED_RESULT;


## finally
/// 注入一个在complete或者error的时候执行的blockt

\- (RACSignal<ValueType> *)finally:(**void** (^)(**void**))block RAC_WARN_UNUSED_RESULT;


## bufferWithTime
/// Divides the receiver's `next`s into buffers which deliver every `interval`

/// seconds.

///

/// interval - The interval in which values are grouped into one buffer.

/// scheduler - The scheduler upon which the returned signal will deliver its

///       values. This must not be nil or +[RACScheduler

///       immediateScheduler].

///

/// Returns a signal which sends RACTuples of the buffered values at each

/// interval on `scheduler`. When the receiver completes, any currently-buffered

/// values will be sent immediately.

\- (RACSignal<RACTuple *> *)bufferWithTime:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler;


## collect
/// Collects all receiver's `next`s into a NSArray. Nil values will be converted

/// to NSNull.

///

/// This corresponds to the `ToArray` method in Rx.

///

/// Returns a signal which sends a single NSArray when the receiver completes

/// successfully.

\- (RACSignal<NSArray<ValueType> *> *)collect RAC_WARN_UNUSED_RESULT;


## takeLast
在收到complete以后回放最后N个value

\- (RACSignal<ValueType> *)takeLast:(NSUInteger)count RAC_WARN_UNUSED_RESULT;


## combineLatestWith
/// Combines the latest values from the receiver and the given signal into

/// 2-tuples, once both have sent at least one `next`.

///

/// Any additional `next`s will result in a new RACTuple with the latest values

/// from both signals.

///

/// signal - The signal to combine with. This argument must not be nil.

///

/// Returns a signal which sends RACTuples of the combined values, forwards any

/// `error` events, and completes when both input signals complete.

\- (RACSignal<RACTwoTuple<ValueType, **id**> *> *)combineLatestWith:(RACSignal *)signal RAC_WARN_UNUSED_RESULT;



/// Combines the latest values from the given signals into RACTuples, once all

/// the signals have sent at least one `next`.

///

/// Any additional `next`s will result in a new RACTuple with the latest values

/// from all signals.

///

/// signals - The signals to combine. If this collection is empty, the returned

///      signal will immediately complete upon subscription.

///

/// Returns a signal which sends RACTuples of the combined values, forwards any

/// `error` events, and completes when all input signals complete.

\+ (RACSignal<RACTuple *> *)combineLatest:(**id**<NSFastEnumeration>)signals RAC_WARN_UNUSED_RESULT;



/// Combines signals using +combineLatest:, then reduces the resulting tuples

/// into a single value using -reduceEach:.

///

/// signals   - The signals to combine. If this collection is empty, the

///        returned signal will immediately complete upon subscription.

/// reduceBlock - The block which reduces the latest values from all the

///        signals into one value. It must take as many arguments as the

///        number of signals given. Each argument will be an object

///        argument. The return value must be an object. This argument

///        must not be nil.

///

/// Example:

///

///  [RACSignal combineLatest:@[ stringSignal, intSignal ] reduce:^(NSString *string, NSNumber *number) {

///    return [NSString stringWithFormat:@"%@: %@", string, number];

///  }];

///

/// Returns a signal which sends the results from each invocation of

/// `reduceBlock`.

\+ (RACSignal<ValueType> *)combineLatest:(**id**<NSFastEnumeration>)signals reduce:(RACGenericReduceBlock)reduceBlock RAC_WARN_UNUSED_RESULT;




## merge
调用下面的方法去merge

**\- (RACSignal *)merge:(RACSignal *)signal RAC_WARN_UNUSED_RESULT;**



merge后的信号都会成为一个signal去传递value，ABC merge 为D后，ABC任意新next value都会在D next中出现，但是只有ABC都complete以后才会触发D的complete，ABC任意的error都会触发D的error。

**\+ (RACSignal<ValueType> *)merge:(id<NSFastEnumeration>)signals RAC_WARN_UNUSED_RESULT;**


## flatten
/// Merges the signals sent by the receiver into a flattened signal, but only

/// subscribes to `maxConcurrent` number of signals at a time. New signals are

/// queued and subscribed to as other signals complete.

///

/// If an error occurs on any of the signals, it is sent on the returned signal.

/// It completes only after the receiver and all sent signals have completed.

///

/// This corresponds to `Merge<TSource>(IObservable<IObservable<TSource>>, Int32)`

/// in Rx.

///

/// maxConcurrent - the maximum number of signals to subscribe to at a

///         time. If 0, it subscribes to an unlimited number of

///         signals.

\- (RACSignal *)flatten:(NSUInteger)maxConcurrent RAC_WARN_UNUSED_RESULT;


## then
/// Ignores all `next`s from the receiver, waits for the receiver to complete,

/// then subscribes to a new signal.

///

/// block - A block which will create or obtain a new signal to subscribe to,

///     executed only after the receiver completes. This block must not be

///     nil, and it must not return a nil signal.

///

/// Returns a signal which will pass through the events of the signal created in

/// `block`. If the receiver errors out, the returned signal will error as well.

\- (RACSignal *)then:(RACSignal * (^)(**void**))block RAC_WARN_UNUSED_RESULT;


## concat
/// Concats the inner signals of a signal of signals.

\- (RACSignal *)concat RAC_WARN_UNUSED_RESULT;


## aggregateWithStart
/// Aggregates the `next` values of the receiver into a single combined value.

///

/// The algorithm proceeds as follows:

///

/// 1. `start` is passed into the block as the `running` value, and the first

///   element of the receiver is passed into the block as the `next` value.

/// 2. The result of the invocation (`running`) and the next element of the

///   receiver (`next`) is passed into `reduceBlock`.

/// 3. Steps 2 and 3 are repeated until all values have been processed.

/// 4. The last result of `reduceBlock` is sent on the returned signal.

///

/// This method is similar to -scanWithStart:reduce:, except that only the

/// final result is sent on the returned signal.

///

/// start    - The value to be combined with the first element of the

///        receiver. This value may be `nil`.

/// reduceBlock - The block that describes how to combine values of the

///        receiver. If the receiver is empty, this block will never be

///        invoked. Cannot be nil.

///

/// Returns a signal that will send the aggregated value when the receiver

/// completes, then itself complete. If the receiver never sends any values,

/// `start` will be sent instead.

\- (RACSignal *)aggregateWithStart:(**id**)start reduce:(**id** (^)(**id** running, **id** next))reduceBlock RAC_WARN_UNUSED_RESULT;



/// Aggregates the `next` values of the receiver into a single combined value.

/// This is indexed version of -aggregateWithStart:reduce:.

///

/// start    - The value to be combined with the first element of the

///        receiver. This value may be `nil`.

/// reduceBlock - The block that describes how to combine values of the

///        receiver. This block takes zero-based index value as the last

///        parameter. If the receiver is empty, this block will never be

///        invoked. Cannot be nil.

///

/// Returns a signal that will send the aggregated value when the receiver

/// completes, then itself complete. If the receiver never sends any values,

/// `start` will be sent instead.

\- (RACSignal *)aggregateWithStart:(**id**)start reduceWithIndex:(**id** (^)(**id** running, **id** next, NSUInteger index))reduceBlock RAC_WARN_UNUSED_RESULT;



/// Aggregates the `next` values of the receiver into a single combined value.

///

/// This invokes `startFactory` block on each subscription, then calls

/// -aggregateWithStart:reduce: with the return value of the block as start value.

///

/// startFactory - The block that returns start value which will be combined

///        with the first element of the receiver. Cannot be nil.

/// reduceBlock - The block that describes how to combine values of the

///        receiver. If the receiver is empty, this block will never be

///        invoked. Cannot be nil.

///

/// Returns a signal that will send the aggregated value when the receiver

/// completes, then itself complete. If the receiver never sends any values,

/// the return value of `startFactory` will be sent instead.

\- (RACSignal *)aggregateWithStartFactory:(**id** (^)(**void**))startFactory reduce:(**id** (^)(**id** running, **id** next))reduceBlock RAC_WARN_UNUSED_RESULT;



/// Invokes -setKeyPath:onObject:nilValue: with `nil` for the nil value.

///

/// WARNING: Under certain conditions, this method is known to be thread-unsafe.

///     See the description in -setKeyPath:onObject:nilValue:.

\- (RACDisposable *)setKeyPath:(NSString *)keyPath onObject:(NSObject *)object;



/// Binds the receiver to an object, automatically setting the given key path on

/// every `next`. When the signal completes, the binding is automatically

/// disposed of.

///

/// WARNING: Under certain conditions, this method is known to be thread-unsafe.

///     A crash can result if `object` is deallocated concurrently on

///     another thread within a window of time between a value being sent

///     on this signal and immediately prior to the invocation of

///     -setValue:forKeyPath:, which sets the property. To prevent this,

///     ensure `object` is deallocated on the same thread the receiver

///     sends on, or ensure that the returned disposable is disposed of

///     before `object` deallocates.

///     See https://github.com/ReactiveCocoa/ReactiveCocoa/pull/1184

///

/// Sending an error on the signal is considered undefined behavior, and will

/// generate an assertion failure in Debug builds.

///

/// A given key on an object should only have one active signal bound to it at any

/// given time. Binding more than one signal to the same property is considered

/// undefined behavior.

///

/// keyPath - The key path to update with `next`s from the receiver.

/// object  - The object that `keyPath` is relative to.

/// nilValue - The value to set at the key path whenever `nil` is sent by the

///      receiver. This may be nil when binding to object properties, but

///      an NSValue should be used for primitive properties, to avoid an

///      exception if `nil` is sent (which might occur if an intermediate

///      object is set to `nil`).

///

/// Returns a disposable which can be used to terminate the binding.

\- (RACDisposable *)setKeyPath:(NSString *)keyPath onObject:(NSObject *)object nilValue:(**nullable** **id**)nilValue;



/// Sends NSDate.date every `interval` seconds.

///

/// interval - The time interval in seconds at which the current time is sent.

/// scheduler - The scheduler upon which the current NSDate should be sent. This

///       must not be nil or +[RACScheduler immediateScheduler].

///

/// Returns a signal that sends the current date/time every `interval` on

/// `scheduler`.

\+ (RACSignal<NSDate *> *)interval:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler RAC_WARN_UNUSED_RESULT;



/// Sends NSDate.date at intervals of at least `interval` seconds, up to

/// approximately `interval` + `leeway` seconds.

///

/// The created signal will defer sending each `next` for at least `interval`

/// seconds, and for an additional amount of time up to `leeway` seconds in the

/// interest of performance or power consumption. Note that some additional

/// latency is to be expected, even when specifying a `leeway` of 0.

///

/// interval - The base interval between `next`s.

/// scheduler - The scheduler upon which the current NSDate should be sent. This

///       must not be nil or +[RACScheduler immediateScheduler].

/// leeway  - The maximum amount of additional time the `next` can be deferred.

///

/// Returns a signal that sends the current date/time at intervals of at least

/// `interval seconds` up to approximately `interval` + `leeway` seconds on

/// `scheduler`.

\+ (RACSignal<NSDate *> *)interval:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler withLeeway:(NSTimeInterval)leeway RAC_WARN_UNUSED_RESULT;



/// Takes `next`s until the `signalTrigger` sends `next` or `completed`.

///

/// Returns a signal which passes through all events from the receiver until

/// `signalTrigger` sends `next` or `completed`, at which point the returned signal

/// will send `completed`.

\- (RACSignal<ValueType> *)takeUntil:(RACSignal *)signalTrigger RAC_WARN_UNUSED_RESULT;



/// Takes `next`s until the `replacement` sends an event.

///

/// replacement - The signal which replaces the receiver as soon as it sends an

///        event.

///

/// Returns a signal which passes through `next`s and `error` from the receiver

/// until `replacement` sends an event, at which point the returned signal will

/// send that event and switch to passing through events from `replacement`

/// instead, regardless of whether the receiver has sent events already.

\- (RACSignal *)takeUntilReplacement:(RACSignal *)replacement RAC_WARN_UNUSED_RESULT;



/// Subscribes to the returned signal when an error occurs.

\- (RACSignal *)**catch**:(RACSignal * (^)(NSError * **_Nonnull** error))catchBlock RAC_WARN_UNUSED_RESULT;



/// Subscribes to the given signal when an error occurs.

\- (RACSignal *)catchTo:(RACSignal *)signal RAC_WARN_UNUSED_RESULT;



/// Returns a signal that will either immediately send the return value of

/// `tryBlock` and complete, or error using the `NSError` passed out from the

/// block.

///

/// tryBlock - An action that performs some computation that could fail. If the

///      block returns nil, the block must return an error via the

///      `errorPtr` parameter.

///

/// Example:

///

///  [RACSignal try:^(NSError **error) {

///    return [NSJSONSerialization JSONObjectWithData:someJSONData options:0 error:error];

///  }];

\+ (RACSignal<ValueType> *)**try**:(**nullable** ValueType (^)(NSError **errorPtr))tryBlock RAC_WARN_UNUSED_RESULT;



/// Runs `tryBlock` against each of the receiver's values, passing values

/// until `tryBlock` returns NO, or the receiver completes.

///

/// tryBlock - An action to run against each of the receiver's values.

///      The block should return YES to indicate that the action was

///      successful. This block must not be nil.

///

/// Example:

///

///  // The returned signal will send an error if data values cannot be

///  // written to `someFileURL`.

///  [signal try:^(NSData *data, NSError **errorPtr) {

///    return [data writeToURL:someFileURL options:NSDataWritingAtomic error:errorPtr];

///  }];

///

/// Returns a signal which passes through all the values of the receiver. If

/// `tryBlock` fails for any value, the returned signal will error using the

/// `NSError` passed out from the block.

\- (RACSignal<ValueType> *)**try**:(**BOOL** (^)(**id** **_Nullable** value, NSError **errorPtr))tryBlock RAC_WARN_UNUSED_RESULT;



/// Runs `mapBlock` against each of the receiver's values, mapping values until

/// `mapBlock` returns nil, or the receiver completes.

///

/// mapBlock - An action to map each of the receiver's values. The block should

///      return a non-nil value to indicate that the action was successful.

///      This block must not be nil.

///

/// Example:

///

///  // The returned signal will send an error if data cannot be read from

///  // `fileURL`.

///  [signal tryMap:^(NSURL *fileURL, NSError **errorPtr) {

///    return [NSData dataWithContentsOfURL:fileURL options:0 error:errorPtr];

///  }];

///

/// Returns a signal which transforms all the values of the receiver. If

/// `mapBlock` returns nil for any value, the returned signal will error using

/// the `NSError` passed out from the block.

\- (RACSignal *)tryMap:(**id** (^)(**id** **_Nullable** value, NSError **errorPtr))mapBlock RAC_WARN_UNUSED_RESULT;



/// Returns the first `next`. Note that this is a blocking call.

\- (**nullable** ValueType)first;



/// Returns the first `next` or `defaultValue` if the signal completes or errors

/// without sending a `next`. Note that this is a blocking call.

\- (**nullable** ValueType)firstOrDefault:(**nullable** ValueType)defaultValue;



/// Returns the first `next` or `defaultValue` if the signal completes or errors

/// without sending a `next`. If an error occurs success will be NO and error

/// will be populated. Note that this is a blocking call.

///

/// Both success and error may be NULL.

\- (**nullable** ValueType)firstOrDefault:(**nullable** ValueType)defaultValue success:(**nullable** **BOOL** *)success error:(NSError * **_Nullable** * **_Nullable**)error;



/// Blocks the caller and waits for the signal to complete.

///

/// error - If not NULL, set to any error that occurs.

///

/// Returns whether the signal completed successfully. If NO, `error` will be set

/// to the error that occurred.

\- (**BOOL**)waitUntilCompleted:(NSError * **_Nullable** * **_Nullable**)error;



/// Defers creation of a signal until the signal's actually subscribed to.

///

/// This can be used to effectively turn a hot signal into a cold signal.

\+ (RACSignal<ValueType> *)defer:(RACSignal<ValueType> * (^)(**void**))block RAC_WARN_UNUSED_RESULT;



/// Every time the receiver sends a new RACSignal, subscribes and sends `next`s and

/// `error`s only for that signal.

///

/// The receiver must be a signal of signals.

///

/// Returns a signal which passes through `next`s and `error`s from the latest

/// signal sent by the receiver, and sends `completed` when both the receiver and

/// the last sent signal complete.

\- (RACSignal *)switchToLatest RAC_WARN_UNUSED_RESULT;



/// Switches between the signals in `cases` as well as `defaultSignal` based on

/// the latest value sent by `signal`.

///

/// signal    - A signal of objects used as keys in the `cases` dictionary.

///         This argument must not be nil.

/// cases     - A dictionary that has signals as values. This argument must

///         not be nil. A RACTupleNil key in this dictionary will match

///         nil `next` events that are received on `signal`.

/// defaultSignal - The signal to pass through after `signal` sends a value for

///         which `cases` does not contain a signal. If nil, any

///         unmatched values will result in

///         a RACSignalErrorNoMatchingCase error.

///

/// Returns a signal which passes through `next`s and `error`s from one of the

/// the signals in `cases` or `defaultSignal`, and sends `completed` when both

/// `signal` and the last used signal complete. If no `defaultSignal` is given,

/// an unmatched `next` will result in an error on the returned signal.

\+ (RACSignal<ValueType> *)**switch**:(RACSignal *)signal cases:(NSDictionary *)cases **default**:(**nullable** RACSignal *)defaultSignal RAC_WARN_UNUSED_RESULT;



/// Switches between `trueSignal` and `falseSignal` based on the latest value

/// sent by `boolSignal`.

///

/// boolSignal - A signal of BOOLs determining whether `trueSignal` or

///        `falseSignal` should be active. This argument must not be nil.

/// trueSignal - The signal to pass through after `boolSignal` has sent YES.

///        This argument must not be nil.

/// falseSignal - The signal to pass through after `boolSignal` has sent NO. This

///        argument must not be nil.

///

/// Returns a signal which passes through `next`s and `error`s from `trueSignal`

/// and/or `falseSignal`, and sends `completed` when both `boolSignal` and the

/// last switched signal complete.

\+ (RACSignal<ValueType> *)**if**:(RACSignal<NSNumber *> *)boolSignal then:(RACSignal *)trueSignal **else**:(RACSignal *)falseSignal RAC_WARN_UNUSED_RESULT;



/// Adds every `next` to an array. Nils are represented by NSNulls. Note that

/// this is a blocking call.

///

/// **This is not the same as the `ToArray` method in Rx.** See -collect for

/// that behavior instead.

///

/// Returns the array of `next` values, or nil if an error occurs.

\- (**nullable** NSArray<ValueType> *)toArray;



/// Adds every `next` to a sequence. Nils are represented by NSNulls.

///

/// This corresponds to the `ToEnumerable` method in Rx.

///

/// Returns a sequence which provides values from the signal as they're sent.

/// Trying to retrieve a value from the sequence which has not yet been sent will

/// block.

**@property** (**nonatomic**, **strong**, **readonly**) RACSequence<ValueType> *sequence;



/// Creates and returns a multicast connection. This allows you to share a single

/// subscription to the underlying signal.

\- (RACMulticastConnection<ValueType> *)publish RAC_WARN_UNUSED_RESULT;



/// Creates and returns a multicast connection that pushes values into the given

/// subject. This allows you to share a single subscription to the underlying

/// signal.

\- (RACMulticastConnection<ValueType> *)multicast:(RACSubject<ValueType> *)subject RAC_WARN_UNUSED_RESULT;



/// Multicasts the signal to a RACReplaySubject of unlimited capacity, and

/// immediately connects to the resulting RACMulticastConnection.

///

/// Returns the connected, multicasted signal.

\- (RACSignal<ValueType> *)replay;



/// Multicasts the signal to a RACReplaySubject of capacity 1, and immediately

/// connects to the resulting RACMulticastConnection.

///

/// Returns the connected, multicasted signal.

\- (RACSignal<ValueType> *)replayLast;



/// Multicasts the signal to a RACReplaySubject of unlimited capacity, and

/// lazily connects to the resulting RACMulticastConnection.

///

/// This means the returned signal will subscribe to the multicasted signal only

/// when the former receives its first subscription.

///

/// Returns the lazily connected, multicasted signal.

\- (RACSignal<ValueType> *)replayLazily;



/// Sends an error after `interval` seconds if the source doesn't complete

/// before then.

///

/// The error will be in the RACSignalErrorDomain and have a code of

/// RACSignalErrorTimedOut.

///

/// interval - The number of seconds after which the signal should error out.

/// scheduler - The scheduler upon which any timeout error should be sent. This

///       must not be nil or +[RACScheduler immediateScheduler].

///

/// Returns a signal that passes through the receiver's events, until the stream

/// finishes or times out, at which point an error will be sent on `scheduler`.

\- (RACSignal<ValueType> *)timeout:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler RAC_WARN_UNUSED_RESULT;



/// Creates and returns a signal that delivers its events on the given scheduler.

/// Any side effects of the receiver will still be performed on the original

/// thread.

///

/// This is ideal when the signal already performs its work on the desired

/// thread, but you want to handle its events elsewhere.

///

/// This corresponds to the `ObserveOn` method in Rx.

\- (RACSignal<ValueType> *)deliverOn:(RACScheduler *)scheduler RAC_WARN_UNUSED_RESULT;



/// Creates and returns a signal that executes its side effects and delivers its

/// events on the given scheduler.

///

/// Use of this operator should be avoided whenever possible, because the

/// receiver's side effects may not be safe to run on another thread. If you just

/// want to receive the signal's events on `scheduler`, use -deliverOn: instead.

\- (RACSignal<ValueType> *)subscribeOn:(RACScheduler *)scheduler RAC_WARN_UNUSED_RESULT;



/// Creates and returns a signal that delivers its events on the main thread.

/// If events are already being sent on the main thread, they may be passed on

/// without delay. An event will instead be queued for later delivery on the main

/// thread if sent on another thread, or if a previous event is already being

/// processed, or has been queued.

///

/// Any side effects of the receiver will still be performed on the original

/// thread.

///

/// This can be used when a signal will cause UI updates, to avoid potential

/// flicker caused by delayed delivery of events, such as the first event from

/// a RACObserve at view instantiation.

\- (RACSignal<ValueType> *)deliverOnMainThread RAC_WARN_UNUSED_RESULT;



/// Groups each received object into a group, as determined by calling `keyBlock`

/// with that object. The object sent is transformed by calling `transformBlock`

/// with the object. If `transformBlock` is nil, it sends the original object.

///

/// The returned signal is a signal of RACGroupedSignal.

\- (RACSignal<RACGroupedSignal *> *)groupBy:(**id**<NSCopying> **_Nullable** (^)(**id** **_Nullable** object))keyBlock transform:(**nullable** **id** **_Nullable** (^)(**id** **_Nullable** object))transformBlock RAC_WARN_UNUSED_RESULT;



/// Calls -[RACSignal groupBy:keyBlock transform:nil].

\- (RACSignal<RACGroupedSignal *> *)groupBy:(**id**<NSCopying> **_Nullable** (^)(**id** **_Nullable** object))keyBlock RAC_WARN_UNUSED_RESULT;



/// Sends an [NSNumber numberWithBool:YES] if the receiving signal sends any

/// objects.

\- (RACSignal<NSNumber *> *)any RAC_WARN_UNUSED_RESULT;



/// Sends an [NSNumber numberWithBool:YES] if the receiving signal sends any

/// objects that pass `predicateBlock`.

///

/// predicateBlock - cannot be nil.

\- (RACSignal<NSNumber *> *)any:(**BOOL** (^)(**id** **_Nullable** object))predicateBlock RAC_WARN_UNUSED_RESULT;



/// Sends an [NSNumber numberWithBool:YES] if all the objects the receiving 

/// signal sends pass `predicateBlock`.

///

/// predicateBlock - cannot be nil.

\- (RACSignal<NSNumber *> *)all:(**BOOL** (^)(**id** **_Nullable** object))predicateBlock RAC_WARN_UNUSED_RESULT;



/// Resubscribes to the receiving signal if an error occurs, up until it has

/// retried the given number of times.

///

/// retryCount - if 0, it keeps retrying until it completes.

\- (RACSignal<ValueType> *)retry:(NSInteger)retryCount RAC_WARN_UNUSED_RESULT;



/// Resubscribes to the receiving signal if an error occurs.

\- (RACSignal<ValueType> *)retry RAC_WARN_UNUSED_RESULT;



/// Sends the latest value from the receiver only when `sampler` sends a value.

/// The returned signal could repeat values if `sampler` fires more often than

/// the receiver. Values from `sampler` are ignored before the receiver sends

/// its first value.

///

/// sampler - The signal that controls when the latest value from the receiver

///      is sent. Cannot be nil.

\- (RACSignal<ValueType> *)sample:(RACSignal *)sampler RAC_WARN_UNUSED_RESULT;



/// Ignores all `next`s from the receiver.

///

/// Returns a signal which only passes through `error` or `completed` events from

/// the receiver.

\- (RACSignal *)ignoreValues RAC_WARN_UNUSED_RESULT;



/// Converts each of the receiver's events into a RACEvent object.

///

/// Returns a signal which sends the receiver's events as RACEvents, and

/// completes after the receiver sends `completed` or `error`.

\- (RACSignal<RACEvent<ValueType> *> *)materialize RAC_WARN_UNUSED_RESULT;



/// Converts each RACEvent in the receiver back into "real" RACSignal events.

///

/// Returns a signal which sends `next` for each value RACEvent, `error` for each

/// error RACEvent, and `completed` for each completed RACEvent.

\- (RACSignal *)dematerialize RAC_WARN_UNUSED_RESULT;



/// Inverts each NSNumber-wrapped BOOL sent by the receiver. It will assert if

/// the receiver sends anything other than NSNumbers.

///

/// Returns a signal of inverted NSNumber-wrapped BOOLs.

\- (RACSignal<NSNumber *> *)**not** RAC_WARN_UNUSED_RESULT;



/// Performs a boolean AND on all of the RACTuple of NSNumbers in sent by the receiver.

///

/// Asserts if the receiver sends anything other than a RACTuple of one or more NSNumbers.

///

/// Returns a signal that applies AND to each NSNumber in the tuple.

\- (RACSignal<NSNumber *> *)**and** RAC_WARN_UNUSED_RESULT;



/// Performs a boolean OR on all of the RACTuple of NSNumbers in sent by the receiver.

///

/// Asserts if the receiver sends anything other than a RACTuple of one or more NSNumbers.

/// 

/// Returns a signal that applies OR to each NSNumber in the tuple.

\- (RACSignal<NSNumber *> *)**or** RAC_WARN_UNUSED_RESULT;



/// Sends the result of calling the block with arguments as packed in each RACTuple

/// sent by the receiver.

///

/// The receiver must send tuple values, where the first element of the tuple is

/// a block, taking a number of parameters equal to the count of the remaining

/// elements of the tuple, and returning an object. Each block must take at least

/// one argument, so each tuple must contain at least 2 elements.

///

/// Example:

///

///  RACSignal *adder = [RACSignal return:^(NSNumber *a, NSNumber *b) {

///    return @(a.intValue + b.intValue);

///  }];

///  RACSignal *sums = [[RACSignal

///    combineLatest:@[ adder, as, bs ]]

///    reduceApply];

///

/// Returns a signal of the result of applying the first element of each tuple

/// to the remaining elements.

\- (RACSignal *)reduceApply RAC_WARN_UNUSED_RESULT;



**@end**



NS_ASSUME_NONNULL_END