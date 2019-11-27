##  1.原生层自定义global属性
在分析react-native源码的时候我们发现，js向native发送消息都是通过global这个变量。react-native在初始化的时候提前向global挂载方法和刷新。js调用global上面的方法就可以将参数传入native层。例如react-native通过requireNativeComponent()这种方式向原生发送创建view的请求。
阅读源码我们可以看见向global注入方法是通过runtime_->global().setProperty来实现的。所以实现global注入属性的核心是拿到jsi::runtime。

```
 /**
   * Get the C pointer (as a long) to the JavaScriptCore context associated with this instance. Use
   * the following pattern to ensure that the JS context is not cleared while you are using it:
   * JavaScriptContextHolder jsContext = reactContext.getJavaScriptContextHolder()
   * synchronized(jsContext) { nativeThingNeedingJsContext(jsContext.get()); }
   */
  public JavaScriptContextHolder getJavaScriptContextHolder() {
    return mCatalystInstance.getJavaScriptContextHolder();
  }
```
在ReactContext中，我们可以看到这样一个方法。官方给出了注释，我们可以通过该方法拿到jsi::runtime。


```
 @Override
  protected void onPause() {
    super.onPause();
    getReactInstanceManager().removeReactInstanceEventListener(this);
  }

  @Override
  protected void onResume() {
    super.onResume();
    getReactInstanceManager().addReactInstanceEventListener(this);
  }

  @Override
  public void onReactContextInitialized(ReactContext context) {
    install(context.getJavaScriptContextHolder().get());
  }

  public native void install(long jsContextNativePointer);
```
在java层拿到jsi::runtime的引用通过jni传入c++层。


```
#include "TestBinding.h"

#if ANDROID
extern "C"
{
JNIEXPORT void JNICALL
Java_com_example_MainActivity_install(JNIEnv *env, jobject thiz, jlong runtimePtr) {
    auto testModuleName = "nativeTest";
    auto &runtime = *((jsi::Runtime *) runtimePtr);
    auto testBinding = std::make_shared<TestBinding>();
    auto object = jsi::Object::createFromHostObject(runtime, testBinding);
    runtime.global().setProperty(runtime, testModuleName, std::move(object));
}

TestBinding::TestBinding() {}

jsi::Value TestBinding::get(
        jsi::Runtime &runtime,
        const jsi::PropNameID &name) {
    auto methodName = name.utf8(runtime);
    if (methodName == "runTest") {
        return jsi::Function::createFromHostFunction(runtime, name, 0, [](
                jsi::Runtime &runtime,
                const jsi::Value &thisValue,
                const jsi::Value *arguments,
                size_t count) -> jsi::Value {
            return 100;
        });
    }
    return jsi::Value::undefined();
}
}
#endif
```
通过runtime.global().setProperty注入方法。这样我们开发的时候可以通过global调用C++ 的方法。作者测试过同样的时间复杂度的代码,c++的效率远高于js的效率。

## 2.调用c++方法的另一种方式
我们可以通过上面那种global的方式来调用c++ 方法来提高程序的质量。但是同时react-native官方也提供了一种桥接c++ 的方式。虽然官方文档还未写上，但是源码里面已经有例子了。有兴趣的同学可以在源码里面搜索下cxxModule。下面介绍下cxxModule的接入方式。

```
public final class HelloCxxPackage implements ReactPackage {
  @Override
  public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    return Arrays.<NativeModule>asList(
        //类似于java module的注入方式
        CxxModuleWrapper.makeDso("rnpackage-hellocxx", "createHelloCxxModule")
    );
  }
  @Override
  public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.emptyList();
  }
}

```


```
//指定module名
LOCAL_MODULE := rnpackage-hellocxx

LOCAL_SRC_FILES := \
  $(RN_DIR)/ReactCommon/jsi/jsi/jsi.cpp \
  HelloCxxModule.cpp \
  md5.cpp \
  TestBinding.cpp \

```


```
#include "cxxreact/JsArgumentHelpers.h"

#include "cxxreact/Instance.h"
#include "md5.h"

HelloCxxModule::HelloCxxModule() {}

using namespace folly;

std::string HelloCxxModule::getName() {
    return "HelloCxxModule";
}

auto HelloCxxModule::getConstants() -> std::map<std::string, folly::dynamic> {
    return {
            {"one",    1},
            {"two",    2},
            {"animal", "fox"},
    };
}

auto HelloCxxModule::getMethods() -> std::vector<Method> {
    return {
            Method("foo", [](folly::dynamic args, Callback cb) { cb({"foo"}); }),
            Method("bar",
                   [this]() {
                       if (auto reactInstance = getInstance().lock()) {
                           reactInstance->callJSFunction(
                                   "RCTDeviceEventEmitter", "emit",
                                   folly::dynamic::array(
                                           "appStateDidChange",
                                           folly::dynamic::object("app_state",
                                                                  "active")));
                       }
                   }),
            Method("md5", [](folly::dynamic args, Callback cb) {
                std::string a = facebook::xplat::jsArgAsString(args, 0);
                MD5 s(a);
                cb({s.toStr()});
            }),
            Method("testSpeed", [](folly::dynamic args, Callback cb) {
                int i = 0,j = 0,k=0,ret = 0;
                for(;i<10000;i++){
                    for(;j<10000;j++){
                        for (; k < 10000; k++) {
                            ret+=k;
                        }
                    }
                }
                cb({ret});
            })
    };
}


extern "C" HelloCxxModule *createHelloCxxModule() {
    return new HelloCxxModule();
}

```

有兴趣的同学可以点击[这里](https://github.com/mingmingjiu/react-native-message-bridge-demo)查看我放在github上面的例子。
