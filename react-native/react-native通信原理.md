## 1.js调取native方法
根据通常RN调用原生方法的步骤如下：
1. 自定义原生模块
2. 支持自定义模块
3. NativeModules.xxxModule.xxxFunction()调用方法;
我们先从NativeModules看起

```
let NativeModules: {[moduleName: string]: Object} = {};
if (global.nativeModuleProxy) {
  NativeModules = global.nativeModuleProxy;
} else if (!global.nativeExtensions) {
    //debug发现走这条分支
    //c++里对__fbBatchedBridgeConfig赋值
  const bridgeConfig = global.__fbBatchedBridgeConfig;
  invariant(
    bridgeConfig,
    '__fbBatchedBridgeConfig is not set, cannot invoke native modules',
  );

  const defineLazyObjectProperty = require('../Utilities/defineLazyObjectProperty');
  (bridgeConfig.remoteModuleConfig || []).forEach(
    (config: ModuleConfig, moduleID: number) => {
      // Initially this config will only contain the module name when running in JSC. The actual
      // configuration of the module will be lazily loaded.
      //根据配置文件生成module
      const info = genModule(config, moduleID);
      if (!info) {
        return;
      }

      if (info.module) {
        NativeModules[info.name] = info.module;
      }
      // If there's no module config, define a lazy getter
      else {
        defineLazyObjectProperty(NativeModules, info.name, {
          get: () => loadModule(info.name, moduleID),
        });
      }
    },
  );
}
```
通过阅读genMethod方法得知最终调用BatchedBridge.enqueueNativeCall。

```

  enqueueNativeCall(
    moduleID: number,
    methodID: number,
    params: any[],
    onFail: ?Function,
    onSucc: ?Function,
  ) {
    //保存回调标识
    this.processCallbacks(moduleID, methodID, params, onFail, onSucc);

    //将moduleID和methodID压入队列
    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);

    if (__DEV__) {
      // Validate that parameters passed over the bridge are
      // folly-convertible.  As a special case, if a prop value is a
      // function it is permitted here, and special-cased in the
      // conversion.
      const isValidArgument = val => {
        const t = typeof val;
        if (
          t === 'undefined' ||
          t === 'null' ||
          t === 'boolean' ||
          t === 'string'
        ) {
          return true;
        }
        if (t === 'number') {
          return isFinite(val);
        }
        if (t === 'function' || t !== 'object') {
          return false;
        }
        if (Array.isArray(val)) {
          return val.every(isValidArgument);
        }
        for (const k in val) {
          if (typeof val[k] !== 'function' && !isValidArgument(val[k])) {
            return false;
          }
        }
        return true;
      };

      // Replacement allows normally non-JSON-convertible values to be
      // seen.  There is ambiguity with string values, but in context,
      // it should at least be a strong hint.
      const replacer = (key, val) => {
        const t = typeof val;
        if (t === 'function') {
          return '<<Function ' + val.name + '>>';
        } else if (t === 'number' && !isFinite(val)) {
          return '<<' + val.toString() + '>>';
        } else {
          return val;
        }
      };

      // Note that JSON.stringify
      invariant(
        isValidArgument(params),
        '%s is not usable as a native method argument',
        JSON.stringify(params, replacer),
      );

      // The params object should not be mutated after being queued
      deepFreezeAndThrowOnMutationInDev((params: any));
    }
    //保存请求参数
    this._queue[PARAMS].push(params);

    const now = Date.now();
    //js native交互频率，最快5ms
    if (
      global.nativeFlushQueueImmediate &&
      now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS
    ) {
      const queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      //js初始化的时候在C++层设置的global function
      global.nativeFlushQueueImmediate(queue);
    }
    Systrace.counterEvent('pending_js_to_native_queue', this._queue[0].length);
    if (__DEV__ && this.__spy && isFinite(moduleID)) {
      this.__spy({
        type: TO_NATIVE,
        module: this._remoteModuleTable[moduleID],
        method: this._remoteMethodTable[moduleID][methodID],
        args: params,
      });
    } else if (this.__spy) {
      this.__spy({
        type: TO_NATIVE,
        module: moduleID + '',
        method: methodID,
        args: params,
      });
    }
  }

```


```
//C++层，接收js层发送的消息
runtime_->global().setProperty(
      *runtime_,
      "nativeFlushQueueImmediate",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "nativeFlushQueueImmediate"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) {
            if (count != 1) {
              throw std::invalid_argument(
                  "nativeFlushQueueImmediate arg count must be 1");
            }
            callNativeModules(args[0], false);
            return Value::undefined();
          }));
```


```
 void callNativeModules(
      __unused JSExecutor& executor, folly::dynamic&& calls, bool isEndOfBatch) override {

    CHECK(m_registry || calls.empty()) <<
      "native module calls cannot be completed with no native modules";
    m_batchHadNativeModuleCalls = m_batchHadNativeModuleCalls || !calls.empty();

    // An exception anywhere in here stops processing of the batch.  This
    // was the behavior of the Android bridge, and since exception handling
    // terminates the whole bridge, there's not much point in continuing.
    for (auto& call : parseMethodCalls(std::move(calls))) {
    //继续调用
      m_registry->callNativeMethod(call.moduleId, call.methodId, std::move(call.arguments), call.callId);
    }
    if (isEndOfBatch) {
      // onBatchComplete will be called on the native (module) queue, but
      // decrementPendingJSCalls will be called sync. Be aware that the bridge may still
      // be processing native calls when the bridge idle signaler fires.
      if (m_batchHadNativeModuleCalls) {
        m_callback->onBatchComplete();
        m_batchHadNativeModuleCalls = false;
      }
      m_callback->decrementPendingJSCalls();
    }
  }

```

```
void ModuleRegistry::callNativeMethod(unsigned int moduleId, unsigned int methodId, folly::dynamic&& params, int callId) {
  if (moduleId >= modules_.size()) {
    throw std::runtime_error(
      folly::to<std::string>("moduleId ", moduleId, " out of range [0..", modules_.size(), ")"));
  }
  modules_[moduleId]->invoke(methodId, std::move(params), callId);
}
```

```
void CxxNativeModule::invoke(unsigned int reactMethodId, folly::dynamic&& params, int callId) {
  if (reactMethodId >= methods_.size()) {
    throw std::invalid_argument(folly::to<std::string>("methodId ", reactMethodId,
        " out of range [0..", methods_.size(), "]"));
  }
  if (!params.isArray()) {
    throw std::invalid_argument(
      folly::to<std::string>("method parameters should be array, but are ", params.typeName()));
  }

  CxxModule::Callback first;
  CxxModule::Callback second;

  const auto& method = methods_[reactMethodId];

  if (!method.func) {
    throw std::runtime_error(folly::to<std::string>("Method ", method.name,
        " is synchronous but invoked asynchronously"));
  }

  if (params.size() < method.callbacks) {
    throw std::invalid_argument(folly::to<std::string>("Expected ", method.callbacks,
        " callbacks, but only ", params.size(), " parameters provided"));
  }

  if (method.callbacks == 1) {
    first = convertCallback(makeCallback(instance_, params[params.size() - 1]));
  } else if (method.callbacks == 2) {
    first = convertCallback(makeCallback(instance_, params[params.size() - 2]));
    second = convertCallback(makeCallback(instance_, params[params.size() - 1]));
  }

  params.resize(params.size() - method.callbacks);

  // I've got a few flawed options here.  I can let the C++ exception
  // propagate, and the registry will log/convert them to java exceptions.
  // This lets all the java and red box handling work ok, but the only info I
  // can capture about the C++ exception is the what() string, not the stack.
  // I can std::terminate() the app.  This causes the full, accurate C++
  // stack trace to be added to logcat by debuggerd.  The java state is lost,
  // but in practice, the java stack is always the same in this case since
  // the javascript stack is not visible, and the crash is unfriendly to js
  // developers, but crucial to C++ developers.  The what() value is also
  // lost.  Finally, I can catch, log the java stack, then rethrow the C++
  // exception.  In this case I get java and C++ stack data, but the C++
  // stack is as of the rethrow, not the original throw, both the C++ and
  // java stacks always look the same.
  //
  // I am going with option 2, since that seems like the most useful
  // choice.  It would be nice to be able to get what() and the C++
  // stack.  I'm told that will be possible in the future.  TODO
  // mhorowitz #7128529: convert C++ exceptions to Java

  messageQueueThread_->runOnQueue([method, params=std::move(params), first, second, callId] () {
  #ifdef WITH_FBSYSTRACE
    if (callId != -1) {
      fbsystrace_end_async_flow(TRACE_TAG_REACT_APPS, "native", callId);
    }
  #else
    (void)(callId);
  #endif
    SystraceSection s(method.name.c_str());
    try {
      method.func(std::move(params), first, second);
    } catch (const facebook::xplat::JsArgumentException& ex) {
      throw;
    } catch (std::exception& e) {
      LOG(ERROR) << "std::exception. Method call " << method.name.c_str() << " failed: " << e.what();
      std::terminate();
    } catch (std::string& error) {
      LOG(ERROR) << "std::string. Method call " << method.name.c_str() << " failed: " << error.c_str();
      std::terminate();
    } catch (...) {
      LOG(ERROR) << "Method call " << method.name.c_str() << " failed. unknown error";
      std::terminate();
    }
  });
}

```

```
void JMessageQueueThread::runOnQueue(std::function<void()>&& runnable) {
  // For C++ modules, this can be called from an arbitrary thread
  // managed by the module, via callJSCallback or callJSFunction.  So,
  // we ensure that it is registered with the JVM.
  jni::ThreadScope guard;
  //反射java层线程,java层initializeBridge将该线程传入c++。java module方法运行在该线程内。如果方法对线程有要求可以注意下
  static auto method = JavaMessageQueueThread::javaClassStatic()->
    getMethod<void(Runnable::javaobject)>("runOnQueue");
  method(m_jobj, JNativeRunnable::newObjectCxxArgs(wrapRunnable(std::move(runnable))).get());
}

```
##  2.native调用js方法
在react-native项目启动的时候，native层会调用js的AppRegistry.runApplication方法。有兴趣的同学可以查看我的另一篇文章。[react-native启动流程解析](https://github.com/mingmingjiu/xBlog/blob/master/react-native/react-native%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E8%A7%A3%E6%9E%90.md)
