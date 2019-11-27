该文章分析的rn启动流程是基于0.60版本。主要分享android端RNAPP启动到JSBundle执行的过程。
1.Java层
开发者通过react-native init命令生成RN工程。此时我们可以看见android项目里有两个文件：MainApplication.java和MainActivity.java文件。
- MainApplication实现了ReactApplication接口。主要作用是保存rn初始化的一些配置信息。
- MainActivity继承了ReactActivity。如果项目做特殊的处理，那么该Activity是唯一的。RN的权限，后退键，Activity的生命周期都在该Activity处理。这些处理都是交由ReactActivityDelegate完成。
当APP启动的时候，打开页面触发的第一个方法是Activity#onCreate。由于实现在ReactActivityDelegate。我们从ReactActivityDelegate#onCreate看起。

```
 protected void onCreate(Bundle savedInstanceState) {
    String mainComponentName = getMainComponentName();
    if (mainComponentName != null) {
      loadApp(mainComponentName);//加载APP
    }
    mDoubleTapReloadRecognizer = new DoubleTapReloadRecognizer();
  }
```


```
 protected void loadApp(String appKey) {
    if (mReactRootView != null) {
      throw new IllegalStateException("Cannot loadApp while app is already running.");
    }
    mReactRootView = createRootView();//创建根视图
    mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      appKey,
      getLaunchOptions());
    getPlainActivity().setContentView(mReactRootView);
  }
```
ReactRootView可以理解为RN的根视图。所有的标签最终渲染在该视图上。该View继承了FrameLayout，重写了onMeasure、onLayout、onInterceptTouchEvent、onTouchEvent、dispatchKeyEvent、onFocusChanged、requestChildFocus、requestDisallowInterceptTouchEvent、onLayout...
主要对于触摸、焦点、按键、view的事件处理。
注意onLayout方法的实现：

```
@Override
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    if (mUseSurface) {
      super.onLayout(changed, left, top, right, bottom);
    }
    // No-op since UIManagerModule handles actually laying out children.
  }
```
如果mUseSurface为false的话，onLayout将会是空实现。刚好我们的根视图的mUseSurface就是false。由于RN使用了yoga布局引擎，所有子组件的位置都是在js层决定好了。所以onLayout对于子组件的布局没有任何作用。一种变相的性能优化。但是开发者如果使用到自定义原生组件的话，onLayout将会失效，自定义组件的UI效果可能出现问题。自定义原生组件的刷新方法也会失效。解决办法就是重写自定义原生组件的requestLayout方法。
接着再看ReactRootView#startReactApplication() ==> ReactInstanceManager#createReactContextInBackground() ==> recreateReactContextInBackgroundInner() ==> recreateReactContextInBackgroundFromBundleLoader() ==> recreateReactContextInBackground() ==> runCreateReactContextOnNewThread() ==> createReactContext() ==> setupReactContext()  ==> attachRootViewToInstance() ==> ReactRootView#runApplication()
ReactInstanceManager#createReactContext() ==> CatalystInstance.runJSBundle() ==> JSBundleLoader.loadScript() ==> CatalysInstanceImpl#loadScriptFromFile ==> jniLoadScriptFromFile();

其中runApplication的实现

```
/**
   * Calls into JS to start the React application. Can be called multiple times with the
   * same rootTag, which will re-render the application from the root.
   */
  @Override
  public void runApplication() {
      Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "ReactRootView.runApplication");
      try {
        if (mReactInstanceManager == null || !mIsAttachedToInstance) {
          return;
        }

        ReactContext reactContext = mReactInstanceManager.getCurrentReactContext();
        if (reactContext == null) {
          return;
        }

        CatalystInstance catalystInstance = reactContext.getCatalystInstance();
        String jsAppModuleName = getJSModuleName();

        if (mUseSurface) {
          // TODO call surface's runApplication
        } else {

          boolean isFabric = getUIManagerType() == FABRIC;
          // Fabric requires to call updateRootLayoutSpecs before starting JS Application,
          // this ensures the root will hace the correct pointScaleFactor.
          if (mWasMeasured || isFabric) {
            updateRootLayoutSpecs(mWidthMeasureSpec, mHeightMeasureSpec);
          }

          WritableNativeMap appParams = new WritableNativeMap();
          appParams.putDouble("rootTag", getRootViewTag());
          @Nullable Bundle appProperties = getAppProperties();
          if (appProperties != null) {
            appParams.putMap("initialProps", Arguments.fromBundle(appProperties));
          }
          if (isFabric) {
            appParams.putBoolean("fabric", true);
          }

          mShouldLogContentAppeared = true;
            //动态代理映射到JS文件
          catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
        }
      } finally {
        Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      }
  }

```
CatalystInstanceImpl#getJSModule

```
public synchronized <T extends JavaScriptModule> T getJavaScriptModule(
      CatalystInstance instance,
      Class<T> moduleInterface) {
    JavaScriptModule module = mModuleInstances.get(moduleInterface);
    if (module != null) {
      return (T) module;
    }

    JavaScriptModule interfaceProxy = (JavaScriptModule) Proxy.newProxyInstance(
        moduleInterface.getClassLoader(),
        new Class[]{moduleInterface},
        new JavaScriptModuleInvocationHandler(instance, moduleInterface));
    mModuleInstances.put(moduleInterface, interfaceProxy);
    return (T) interfaceProxy;
  }

```
动态代理，最终执行了mCatalystInstance.callFunction(getJSModuleName(), method.getName(), jsArgs) ==> callFunction()  ==> jniCallJSFunction()

```
public void callFunction(PendingJSCall function) {
    if (mDestroyed) {
      final String call = function.toString();
      FLog.w(ReactConstants.TAG, "Calling JS function after bridge has been destroyed: " + call);
      return;
    }
    if (!mAcceptCalls) {
      // Most of the time the instance is initialized and we don't need to acquire the lock
      //因为JSLoad在另一个线程，可能存在jsload未完成，所有callFunction都要保存好，供后面使用
      synchronized (mJSCallsPendingInitLock) {
        if (!mAcceptCalls) {
          mJSCallsPendingInit.add(function);
          return;
        }
      }
    }
    function.call(this);
  }
```

2.·C++·层
从上面分析发现进入C++有两个重要的方法：jniLoadScriptFromFile、jniCallJSFunction。

其中jniLoadScriptFromFile最终将jsbundle转成字符串交给JSCRuntime#evaluateJavaScript运行js文件。


```
void JSIExecutor::loadApplicationScript(
    std::unique_ptr<const JSBigString> script,
    std::string sourceURL) {
  SystraceSection s("JSIExecutor::loadApplicationScript");

  // TODO: check for and use precompiled HBC

  runtime_->global().setProperty(
      *runtime_,
      "nativeModuleProxy",
      Object::createFromHostObject(
          *runtime_, std::make_shared<NativeModuleProxy>(*this)));

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

  runtime_->global().setProperty(
      *runtime_,
      "nativeCallSyncHook",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "nativeCallSyncHook"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) { return nativeCallSyncHook(args, count); }));

#if DEBUG
  runtime_->global().setProperty(
      *runtime_,
      "globalEvalWithSourceUrl",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "globalEvalWithSourceUrl"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) { return globalEvalWithSourceUrl(args, count); }));
#endif

  if (runtimeInstaller_) {
    runtimeInstaller_(*runtime_);
  }

  bool hasLogger(ReactMarker::logTaggedMarker);
  std::string scriptName = simpleBasename(sourceURL);
  if (hasLogger) {
    ReactMarker::logTaggedMarker(
        ReactMarker::RUN_JS_BUNDLE_START, scriptName.c_str());
  }
  //运行js文件
  runtime_->evaluateJavaScript(
      std::make_unique<BigStringBuffer>(std::move(script)), sourceURL);
  flush();
  if (hasLogger) {
    ReactMarker::logMarker(ReactMarker::CREATE_REACT_CONTEXT_STOP);
    ReactMarker::logTaggedMarker(
        ReactMarker::RUN_JS_BUNDLE_STOP, scriptName.c_str());
  }
}
```
runtime_->global().setProperty的作用是给js环境的global设置属性值。开发者也可以通过这种方式设置自己的初始值。


jniCallJSFunction ==> Instance#callJSFunction ==> NativeToJSBridge#callFunction ==>JSExecutor#callFunction
前面java层我们看见AppRegistry.runApplication。相当于调用了js层的AppRegistry#runApplication

3.JavaScript层

```
 /**
   * Loads the JavaScript bundle and runs the app.
   *
   * See http://facebook.github.io/react-native/docs/appregistry.html#runapplication
   */
  runApplication(appKey: string, appParameters: any): void {
    const msg =
      'Running application "' +
      appKey +
      '" with appParams: ' +
      JSON.stringify(appParameters) +
      '. ' +
      '__DEV__ === ' +
      String(__DEV__) +
      ', development-level warning are ' +
      (__DEV__ ? 'ON' : 'OFF') +
      ', performance optimizations are ' +
      (__DEV__ ? 'OFF' : 'ON');
    infoLog(msg);
    BugReporting.addSource(
      'AppRegistry.runApplication' + runCount++,
      () => msg,
    );
    invariant(
      runnables[appKey] && runnables[appKey].run,
      'Application ' +
        appKey +
        ' has not been registered.\n\n' +
        "Hint: This error often happens when you're running the packager " +
        '(local dev server) from a wrong folder. For example you have ' +
        'multiple apps and the packager is still running for the app you ' +
        'were working on before.\nIf this is the case, simply kill the old ' +
        'packager instance (e.g. close the packager terminal window) ' +
        'and start the packager in the correct app folder (e.g. cd into app ' +
        "folder and run 'npm start').\n\n" +
        'This error can also happen due to a require() error during ' +
        'initialization or failure to call AppRegistry.registerComponent.\n\n',
    );

    SceneTracker.setActiveScene({name: appKey});
    //appKey对应MainActivity#getMainComponentName ，appParameters初始化的一些数据
    runnables[appKey].run(appParameters);
  },

```
在js层的index文件我们调用了AppRegistry.registerComponent(appName, () => App);

```
  /**
   * Registers an app's root component.
   *
   * See http://facebook.github.io/react-native/docs/appregistry.html#registercomponent
   */
  registerComponent(
    appKey: string,
    componentProvider: ComponentProvider,
    section?: boolean,
  ): string {
    let scopedPerformanceLogger = createPerformanceLogger();
    runnables[appKey] = {
      componentProvider,
      run: appParameters => {
        renderApplication(
          componentProviderInstrumentationHook(
            componentProvider,
            scopedPerformanceLogger,
          ),
          appParameters.initialProps,
          appParameters.rootTag,
          wrapperComponentProvider && wrapperComponentProvider(appParameters),
          appParameters.fabric,
          showFabricIndicator,
          scopedPerformanceLogger,
        );
      },
    };
    if (section) {
      sections[appKey] = runnables[appKey];
    }
    return appKey;
  },
```











