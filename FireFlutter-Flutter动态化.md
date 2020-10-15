
[[_TOC_]]

# 项目介绍

FireFlutter是一套Flutter动态化的解决方案。应该是目前已知方案动态化最彻底，最理想的一种方案。FireFlutter使用原汁原味的Flutter开发，开发者在Flutter环境下完成业务开发及调试。FireFlutter提供的编译工具将项目一键转换为动态化的文件包，运行在FireFlutter之上。

# 项目特性

* 原生Flutter开发
* 对原Flutter工程没有入侵
* 支持所有三方Dart库
* 支持所有三方Dart插件(完美支持消息通道，FlutterBoost等都支持)
* 支持模块化开发，业务模块可以单独拆分下发(业务代码需要声明懒加载)

# 动态化原理

目前已和Flutter动态化的方案有以下几种


1. 下发Flutter编译产物

下发新的Flutter编译产物，替换App安装目录下的编译产物，从而实现动态化。这种方案仅能在Android端可行，Ios端由于审核及沙盒的原因不可行。


2. 使用JIT虚拟机

原生项目需要嵌入JIT虚拟机，下发编译好的dill。这种方案会增加原生包的大小，另外也会面临审核问题。

3. 重建Widget树

目前大多动态化方案都集中在重建Widget树这一块:

* 基于模板，下发模板配置文件，再由Flutter解码为对应的Widget树，由Flutter去渲染。实现简单，但动态化能力弱。
* 动态生成Widget树,比如MXFlutter，通过JS动态生成Widget树，交给Flutter去渲染。实现成本较高，但动态化能力强。


评估这些方案，都不能完全做到彻底的动态化。

## Flutter中的四棵树

Flutter框架里存在四棵树，这是我们能实现动态化的关键。动态化思路就是用其它方式重构Flutter中的某一棵树，再交给Flutter去渲染。

![](https://github.com/mh0717/myimages/blob/master/Flutter4Tree.jpg?raw=true)

* Widget树，是整个UI的一个快照，是轻量的，不可变的，生命周期短。目前大多数方案都是重建这棵树，实现动态化。
* Element树，在Flutter框架当充当管理中中间沟通的作用，如果是Render结点，则起沟通Widget和RenderObject的角色，如果是容器类结点，主要负责管理容器类Widget的重构工作(Build)。所以Element树不太容易替换重构。
* RenderObject树，整个UI的布局及绘制就由这棵树来完成，面RenderObject的类型不是特别多，可以通过重建这棵树来实现动态化(这种方案尝试过，也出来一定的成果，有简单的demo，但最终因为RenderObject有些接口比较复杂，依赖太多而放弃)。
* Layer树，RenderObject树最终才会生成Layer树，发送给Flutter引擎去渲染。重建Layer树，完全可以做到动态化

以上四棵树是一个依次递减的过程，Widget树可能比较深，但是流转到时Layer树时深度会减很多。

## Flutter Web的动态化

官方Flutter Web依次尝试过Widget树，RenderObject树及Layer树的替代，最终还是采用提供Layer树的H5等效替代物，完成了Flutter的Web化。

具体来说就是重新用H5技术实现了dart:ui库，通过dart2js将Flutter工程编译为js，交给浏览器去运行。

![](https://github.com/mh0717/myimages/blob/master/flutterweb.jpg?raw=true)

## 不成功的尝试

这是一版基于重建RenderObject树的尝试，由于RenderObject有些比较复杂，依赖较多，不太好桥接，做出简单Demo，从原理上证明可行便没再继续。

![](https://github.com/mh0717/myimages/blob/master/fluttertree3.jpg?raw=true)

# FireFlutter动态化原理

我们在探索动态化的过程中，也是参照Flutter Web的实现原理，由于已经有基于Widget树的解决方案，便着重探索了基于替换RenderObject树及Layer树的方案，最终还是和Flutter Web一样，选择了重建Layer树这套方案，原因与Flutter Web一致。

实现过程与Flutter Web类似:

* 重新实现了drt:ui即fire_flutter_ui库，这样Flutter的渲染以及与外界交互全部集中到新的fire_flutter_ui库。
* 业务代码，三方代码，Flutter框架以及fire_flutter_ui库通过dart2js编译为js文件。
* js文件通过JSC去运行，当然该JSC是经过定制的。
* JSC运行时，会通过FFI向JSC注入所有原生dart:ui的功能，这样js对fire_flutter_ui库的调用实时转接到原生dart:ui库的调用，动态化与Flutter原生的桥梁便架起来了。
* dart:io库的实现，dart2js无法提供dart:io的实现，所以物们进一步桥接了dart:io库，原理与桥接dart:ui库一样。

![](https://github.com/mh0717/myimages/blob/master/fireflutter.jpg?raw=true)

## 桥接的实现

桥接，即将dart:ui提供的原生功能，通过FFI封装为JSC的函数对象，注入到JSC运行时环境，然后在fire_flutter_ui库中通过dart:js调用该方法，完整还原整个dart:ui功能。

我们以Canvas类举例

通过FFI向JSC注入Canvas功能，这些代码运行在Flutter原生环境下，代码如下:

```dart
void registeFuncToJSContext(Pointer<JSContext> ctx, Pointer<JSObject> globalObject, String nameStr, Pointer<NativeFunction<JSObjectCallAsFunctionCallback>> callAsFunction) {
  final nameUtf8 = Utf8.toUtf8(nameStr);
  final name = JSStringCreateWithUTF8CString(nameUtf8);
  final jsfunc = JSObjectMakeFunctionWithCallback(ctx, name, callAsFunction);  
  JSObjectSetProperty(ctx, globalObject, name, jsfunc.cast(), kJSPropertyAttributeReadOnly, nullptr);
  JSStringRelease(name);
  free(nameUtf8);
}

void prepareCanvas(Pointer<JSContext> ctx, Pointer<JSObject> globalObject) {
  registeFuncToJSContext(ctx, globalObject, 'Canvas_constructor', Pointer.fromFunction(_Canvas_constructor));
  registeFuncToJSContext(ctx, globalObject, 'Canvas_save', Pointer.fromFunction(_Canvas_save));
  registeFuncToJSContext(ctx, globalObject, 'Canvas_saveLayer', Pointer.fromFunction(_Canvas_saveLayer));
  registeFuncToJSContext(ctx, globalObject, 'Canvas_saveLayerWithoutBounds', Pointer.fromFunction(_Canvas_saveLayerWithoutBounds));
  registeFuncToJSContext(ctx, globalObject, 'Canvas_restore', Pointer.fromFunction(_Canvas_restore));
  ...
}


/// Canvas_constructor
/// record, left, top, right, bottom
Pointer<JSValue> _Canvas_constructor(Pointer<JSContext> ctx, Pointer<JSObject> function, Pointer<JSObject> thisObject, int argumentCount, Pointer<Pointer<JSValue>> arguments, Pointer<Pointer<JSValue>> exception) {
  assert((){
    print('Canvas_constructor');
    return true;
  }());
  assert(argumentCount >= 5);
  if (argumentCount < 5) return JSValueMakeUndefined(ctx);

  final recorder = JSDartObjectRef.fromJSDartObjectRef<PictureRecorder>(ctx, arguments.elementAt(0).value.cast());
  assert(recorder != null);
  if (recorder == null) return JSValueMakeUndefined(ctx);

  final left = JSValueToNumber(ctx, arguments.elementAt(1).value.cast(), exception);
  final top = JSValueToNumber(ctx, arguments.elementAt(2).value.cast(), exception);
  final right = JSValueToNumber(ctx, arguments.elementAt(3).value.cast(), exception);
  final bottom = JSValueToNumber(ctx, arguments.elementAt(4).value.cast(), exception);
  
  final canvas = Canvas(recorder, Rect.fromLTRB(left, top, right, bottom));
  final canvasRef = JSDartObjectRef.toJSDartObjectRef(ctx, canvas);

  return canvasRef.cast();
}


/// Canvas_save
Pointer<JSValue> _Canvas_save(Pointer<JSContext> ctx, Pointer<JSObject> function, Pointer<JSObject> thisObject, int argumentCount, Pointer<Pointer<JSValue>> arguments, Pointer<Pointer<JSValue>> exception) {
  assert((){
    print('_Canvas_save');
    return true;
  }());
  assert(thisObject != nullptr);
  if (thisObject == nullptr) return JSValueMakeUndefined(ctx);

  final canvas = JSDartObjectRef.fromJSDartObjectRef<Canvas>(ctx, thisObject);
  assert(canvas != null);
  if (canvas == null) return JSValueMakeUndefined(ctx);

  canvas.save();

  return JSValueMakeUndefined(ctx);
}

...
```

借助dart:js库，还原Canvas类，这段代码会通过dart2js编译为js运行，代码如下:

```dart

/// 通过JS注解，运行时调用对应的js函数
@JS()
external dynamic  Canvas_constructor(dynamic rocorder, double left, double top, double right, double bottom);

@JS('Canvas_save.call')
external void Canvas_save(dynamic thisObj);

@JS('Canvas_saveLayer.call')
external void Canvas_saveLayer(dynamic thisObj, double left, double top, double right, double bottom, dynamic paint);

@JS('Canvas_saveLayerWithoutBounds.call')
external void Canvas_saveLayerWithoutBounds(dynamic thisObj, dynamic paint);

...


/// 复原Canvas

class Canvas {

  @pragma('vm:entry-point')
  Canvas(PictureRecorder recorder, [ Rect? cullRect ]) : assert(recorder != null) {
    if (recorder.isRecording)
      throw ArgumentError('"recorder" must not already be associated with another Canvas.');
    cullRect ??= Rect.largest;
    _nativeRef = Canvas_constructor(recorder.nativeRef, cullRect.left, cullRect.top, cullRect.right, cullRect.bottom);
  }

  dynamic _nativeRef;

  
  void save() => Canvas_save(_nativeRef);

  
  void saveLayer(Rect bounds, Paint paint) {
    assert(paint != null);
    if (bounds == null) {
      Canvas_saveLayerWithoutBounds(_nativeRef, paint.nativeRef);
    } else {
      assert(_rectIsValid(bounds));
      Canvas_saveLayer(_nativeRef, bounds.left, bounds.top, bounds.right, bounds.bottom,
                 paint.nativeRef);
    }
  }
  
  void restore() => Canvas_restore(_nativeRef);

  ...

}


```

## 实时回调如何实现

dart:ui有些函数是需要实时回调的，比如Window.onBeginFrame，它是如何实现的呢?

回调函数是`Window_onBeginFrame`，它是一个js的函数对象，在Flutter原生端，通过FFI调用JSC执行该函数。

Flutter原生端:

```dart

/// 先注入
registeFuncToJSContext(ctx, globalObject, 'Window_set_onBeginFrame', Pointer.fromFunction(_Window_set_onBeginFrame));



Pointer<JSValue> _Window_set_onBeginFrame(Pointer<JSContext> ctx, Pointer<JSObject> function, Pointer<JSObject> thisObject, int argumentCount, Pointer<Pointer<JSValue>> arguments, Pointer<Pointer<JSValue>> exception) {
  assert((){
      print('Window_set_onBeginFrame');
    return true;
  }());

  assert(thisObject != nullptr);
  if (thisObject == nullptr) return JSValueMakeUndefined(ctx);

  final instanceWindow = JSDartObjectRef.fromJSDartObjectRef<Window>(ctx, thisObject);
  assert(instanceWindow != null);
  if (instanceWindow == null) return JSValueMakeUndefined(ctx);

  assert(argumentCount >= 1);

  final onBeginFrameJSObject = arguments.value;
  JSValueProtect(ctx, onBeginFrameJSObject);
  
  /// 通过FFI调用JSC执行该函数对象
  instanceWindow.onBeginFrame = (duration) {
    // print('Window_onBeginFrame');
    final exception = allocate<Pointer<JSValue>>();
    exception.value = nullptr;
    final callArguments = allocate<Pointer<JSValue>>(count: 2);
    callArguments.elementAt(0).value = JSValueMakeNumber(globalContext, duration.inMicroseconds.toDouble());
    callArguments.elementAt(1).value = nullptr;
    JSObjectCallAsFunction(globalContext, onBeginFrameJSObject.cast(), nullptr, 1, callArguments, exception);
    // print('onbe free');
    free(callArguments);
    // print('free end');
    free(exception);
    // print('free exception');
  };

  return JSValueMakeUndefined(ctx);
}
```



js端:

```dart

typedef _OnBeginFrameCallback = void Function(int duration);

@JS('Window_set_onBeginFrame.call')
external void Window_set_onBeginFrame(dynamic thisObj, _OnBeginFrameCallback onBeginFrame);




class Window with NativeRef{

  ...

  FrameCallback? get onBeginFrame => _onBeginFrame;
  FrameCallback? _onBeginFrame;
  Zone? _onBeginFrameZone;
  set onBeginFrame(FrameCallback? callback) {
    _onBeginFrame = callback;
    _onBeginFrameZone = Zone.current;

    if (!_setOnBegeinFramed) {
      _setOnBegeinFramed = true;
      print('onbe : $_nativeRef');
      Window_set_onBeginFrame(_nativeRef, allowInterop(_jsonBeginFrame));
    }
    
  }

  bool _setOnBegeinFramed = false;
  void _jsonBeginFrame(int duration) {
    print('jsonbeginframe: $duration');
    final du = Duration(microseconds: duration);
    if (_onBeginFrame != null) {
      _onBeginFrame?.call(du);
    }
  }

  ...

}

```

该回调的关键是`allowInterop`，它将一个dart函数转换为一个js函数，这点在早期摸索过程中不清楚，总是回调不了。


## 二进制数据块怎么传递

Uin8List编译为js后，对应Uin8Array。

Uin8Array通过FFI调用JSC能够拿到二进制块的原始地址及长度，再通过FFI可以将指针复原为Uint8List。


## 怎么调试排除错误

dart2js编译的js代码，可以在chrome通过map调试dart源码，但FireFlutter是运行在定制JSC环境下的，无法直接调试dart源码。通过Mac的safari可以调试JSC，异常栈通过`source_map_stack_trace`对应。

比如:

```
expect_async_test.dart.browser_test.dart.js 2636:15   dart.wrapException
expect_async_test.dart.browser_test.dart.js 14661:15  main__closure16.call$0
expect_async_test.dart.browser_test.dart.js 18237:26  Declarer_test__closure.call$1
expect_async_test.dart.browser_test.dart.js 17905:23  StackZoneSpecification_registerUnaryCallback__closure.call$0
expect_async_test.dart.browser_test.dart.js 17876:16  StackZoneSpecification._stack_zone_specification$_run$2
expect_async_test.dart.browser_test.dart.js 17899:26  StackZoneSpecification_registerUnaryCallback_closure.call$1
expect_async_test.dart.browser_test.dart.js 6115:16   _rootRunUnary
expect_async_test.dart.browser_test.dart.js 8576:39   _CustomZone.runUnary$2
expect_async_test.dart.browser_test.dart.js 7135:57   _Future__propagateToListeners_handleValueCallback.call$0
expect_async_test.dart.browser_test.dart.js 7031:147  dart._Future.static._Future__propagateToListeners
```

to 

```
dart:_internal/compiler/js_lib/js_helper.dart 1210:1          wrapException
test/frontend/expect_async_test.dart 24:5                     main.<fn>.<fn>
package:test/src/backend/declarer.dart 45:48                  Declarer.test.<fn>.<fn>
package:stack_trace/src/stack_zone_specification.dart 134:30  StackZoneSpecification.registerUnaryCallback.<fn>.<fn>
package:stack_trace/src/stack_zone_specification.dart 210:7   StackZoneSpecification._run
package:stack_trace/src/stack_zone_specification.dart 135:5   StackZoneSpecification.registerUnaryCallback.<fn>
dart:async/zone.dart 904:14                                   _rootRunUnary
dart:async/zone.dart 806:3                                    _CustomZone.runUnary
dart:async/future_impl.dart 486:13                            _Future._propagateToListeners.handleValueCallback
dart:async/future_impl.dart 567:32                            _Future._propagateToListeners
```



## 定制Flutter框架

由于要从动态包中加载资源，所以对官方Flutter框架做了定制。主要是替换了`rootBundle`，其它几处都是为了适配js环境，改动很小。而且这几处改进都是通过条件导入，不会破坏官方框架功能(用于Flutter原生时，功能无影响)

这样做的好处是定制的Flutter框架既可以用于Flutter原生，也可以用于动态化。当前工程只要很小的改动，既能编译为原生包，又能编译为动态化包。

# FireFlutter打包

1. FireFlutter先通过flutter命令打出Flutter的资源包
2. 使用定制的平台二进制(PlatformBinary)
3. 使用dart2js一键编译为main.js

## 资源包

FireFlutter使用Flutter本身打包资源包，在项目根目录运行`flutter build bundle`，将在当前目录`build`下生成资源包文件夹flutter_assets。

通过定制Flutter框架的rootBundle, 实现图片，字体等资源的动态加载。



```dart

@JS('consolidateAssetBytes')
external void jsconsolidateAssetBytes (
  String key,
  void Function(Uint8List bytes) resback,
  void Function(String error) errback,
);

final AssetBundle rootBundle = FireAssetBundle();//_initRootBundle();


class FireAssetBundle extends CachingAssetBundle {
  @override
  Future<ByteData> load(String key) async {
    final completer = Completer<ByteData>.sync();
    final resback = (Uint8List bytes) {
      print('rootBundle load');
      completer.complete(bytes.buffer.asByteData());
      print('rootBundle load end!');
    };
    final errback = (String error) {
      completer.completeError(error);
    };
    jsconsolidateAssetBytes(key, allowInterop(resback), allowInterop(errback));
    return completer.future;
  }
}

```

## 平台二进制(PlatfromBinary)

dart在编译时，其核心库(以dart:开头的)都是通过提供PlatformBinary提供。核心库会提供部分实现，与平台相关的代码，会申明为外部方法，没有实现，然后针对各个平台打相应的补丁。

目前dart提供了4份平台补丁:

* vm: 用于Native环境
* dart2js: 用于浏览器环境
* dart2js_server: 用于纯js环境
* dartdevc: 用于浏览器调试环境

补丁都通过`libraries.json`配置

```json
{
  "comment:0": "NOTE: THIS FILE IS GENERATED. DO NOT EDIT.",
  "comment:1": "Instead modify 'sdk/lib/libraries.yaml' and follow the instructions therein.",
  "none": {
    "libraries": {}
  },
  "vm": {
    "libraries": {
      "_builtin": {
        "uri": "_internal/vm/bin/builtin.dart"
      },
      "_internal": {
        "uri": "internal/internal.dart",
        "patches": [
          "_internal/vm/lib/internal_patch.dart",
          "_internal/vm/lib/class_id_fasta.dart",
          "_internal/vm/lib/print_patch.dart",
          "_internal/vm/lib/symbol_patch.dart",
          "internal/patch.dart"
        ]
      },
      ...
    }
  },
  "dart2js": {
    "libraries": {
      "async": {
        "uri": "async/async.dart",
        "patches": "_internal/js_runtime/lib/async_patch.dart"
      },
      "collection": {
        "uri": "collection/collection.dart",
        "patches": "_internal/js_runtime/lib/collection_patch.dart"
      },
      ...
    }
  },
  "dart2js_server": {
    "libraries": {
      "async": {
        "uri": "async/async.dart",
        "patches": "_internal/js_runtime/lib/async_patch.dart"
      },
      ...
    }
  },
  "dartdevc": {
    "libraries": {
      "_runtime": {
        "uri": "_internal/js_dev_runtime/private/ddc_runtime/runtime.dart"
      },
      "_debugger": {
        "uri": "_internal/js_dev_runtime/private/debugger.dart"
      },
      ...
    }
  }
}
```

由于FireFlutter在纯JS环境下运行，所能选择了dart2js_server平台补丁，dart在js环境下无法提供dart:io的功能，为dart:io提供的补丁仅仅是抛出异常。FireFlutter对dart:io提供了新的补丁，通过FFI桥接到Flutter原生的io功能上。

定制后的`libriarys.json`为:

```json

...

"dart2js_server": {
    "libraries": {
      "ui": {
        "uri": "fire_flutter_ui/lib/ui.dart"
      },
      "io": {
        "uri": "io/io.dart",
        "patches": [
          "ffpatch/io_patch.dart",
          "ffpatch/io_service_patch.dart",
          "ffpatch/file_patch.dart",
          "ffpatch/directory_patch.dart"
        ]
      },
      ...
    }
  },

...

```

然后通过`copile_platform.dart`打包为平台二进制

```
$DART $PACKAGES /dart-sdk/sdk/pkg/front_end/tool/_fasta/compile_platform.dart -v --target=dart2js_server --no-defines dart:core --enable-experiment=non-nullable libraries.json dart2js_server_outline.dill dart2js_server_platform.dill dart2js_server_outline.dill

```

编译的产物为

* dart2js_server_outline.dill
* dart2js_server_platform.dill
* dart2js_server_platform.dill.d

FireFlutter使用的产物则是dart2js_server_platform.dill

## 编译为main.js

资源包有了，定制的平台二进制也有了，可能通过dart2js一键编译工程了

```

WKPATH=$(cd "$(dirname "$0")";pwd)
DART2JS=$WKPATH/dart-sdk/bin/dart2js
JSOUT=$WKPATH/hello_boost/build/flutter_assets/main.js


$DART2JS --server-mode --platform-binaries=$WKPATH   --libraries-spec=$WKPATH/fire_flutter_ui/libraries.json -Ddart.vm.product=true -Dfire_flutter.vm.product=true -O0  -o $JSOUT $WKPATH/hello_boost/lib/main.flutter.dart 
```

--platform-binaries指定平台二进制文件的目录


编译完成后，flutter_assets文件夹下多了一个main.js文件。


## FireFlutter运行

FireFlutter提供运行动态包的api: `runFireApp(Strinb bundle)`

bundle可以是一个具体的文件夹路径，也可以是一个网址，指向前面生成的flutter_assets包

```dart
import 'package:fire_flutter/fire_flutter.dart';

void main() {
  // runFireApp('http://127.0.0.1:8888');
  runFireApp('/Users/xuewanyun/fire_flutter_ws17/hello_boost/build/flutter_assets');
}

```

