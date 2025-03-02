---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
#
# File name format: yyyy-mm-dd-tilte.md
title: Flutter Provider 使用介绍
author: FriesI23
date: 2024-05-10 15:15:00 +0800
category: flutter
tags:
  - flutter
  - provider
  - mvvm
---

作为 `flutter` 官方推荐的状态管理工具 (详见[这里][flutter-rcmd]),
`Provider` 相比于一些状态管理框架 `BloC` 更加轻量, 可以在 app 开发中提供更高的灵活性.
下面将先简单介绍一下 `Provider`, 然后将给出一些简单的使用示例.

## 1. Provider

> - [Provider][provider-provider]

`Provider` 作为包中基础的一个 `Widget`, 主要作用为: 向该 `Widget` 树上的所有子孙暴露一个公共的值.

想象一下一个 `Widget` 树中, 有多个 `Widget` 需要共享获取一个变量, 并能够进行更新
(注意, 这里不涉及监听, 监听需要使用 `ListenableProvider` 以及其继承 `Widget`, 比如 `ChangeNotifierProvider`).
此时有几个基础的解决方案:

1. 使用全局变量或者单例, 但是不管哪一种都存在管理数据生命周期的问题.
   过多的全局变量或者单例会是使得代码复杂度快速增长, 最后不得不手动创建一个管理生命周期的模块.
   最后很可能是重复造轮子, 实现了一遍 `bloC` (一点私货, 我个人很不喜欢重复造轮子).
2. 通过 `Widget` 构造参数将数据一路传下去. 很明显, 这一方面会导致 `Widget` 参数快速增长且包含了一堆自身不需要的参数;
   同时随着代码复杂度增加, 增加参数会变得越来越**重**, 可能增加一个参数需要修改十几个 `Widget` 的构造函数,
   但仅仅是为了偷穿参数.
3. 自己实现一个 `InheritedWidget`, 然后在 `Widget` 中使用 `<Your InheritedWidget>.of(context)` 获取.
   这个相比上面两种方法已具备一定可行性, 但是 `Provider Package` 本身就是针对 `InheritedWidget` 的封装;
   So, 不要重复造轮子!
   > A wrapper around InheritedWidget to make them easier to use and more reusable.

有了 `Provider`, 我们便可以写出以下代码:

```dart
/// 这里先创建一个简单对象, 假设该对象有很多 Widget 需要使用其中的值.
class InfoModel {
    final String name;
    final String addr;
    int age;

    YourModel(this.name, this.addr, this.age);
}
```

```dart
/// 这里 Provider Widget 对数据进行初始化
Provider<InfoModel>(
    create: (context) => InfoModel("John", "Earth", 10),
    child: // 这里传入子 Widget
),
```

```dart
/// 使用以下两种方法都可以获取数据, 两者是等级的, 事实上第一行代码就是对第二行代码的包装
final info = context.read<InfoModel>();
final info = Provider.of<InfoModel>(context, listen: false);
info.age += 1; //将 age + 1
```

### 1.1. 完整示例

[点击运行](https://dartpad.dev/?id=e32f9e3da45e0ec1ede6006ec4859288&run=true)

{% include github_gist.html id="e32f9e3da45e0ec1ede6006ec4859288" %}

### 1.2. 需要注意

`Provider` 生效的范围, 也就是 `context` 的位置. 只有 `Provider` 下面的 `context` 才能获取导数据.
且 `Navigator` 导航到新页面后, 需要使用 `Provider.value` 将对象传递过去.

```dart
/// e.g.1 在 Widget Tree 中
Widget1(
    data: context.read<InfoModel>();    // throw ProviderNotFoundException
    child: Provider<InfoModel>(
        create: (context) => InfoModel("John", "Earth", 10),
        child: Widget2(
            data: context.read<InfoModel>();    // ok
        ),
    ),
);

/// e.g.2 在导航到新界面中, 假设方法是一个StatefulWidget中定一个callback
void _onPressed() async {
    final info = context.read<InfoModel>(); // 注意context, 必须在导航前获取
    final result = Navigator.of(context).push(
        MaterialPageRoute(
            // builder内部已经切换到新的 Widget Tree, 这里使用 `context.read<InfoModel>()` 将会报错
            builder: (context) => Provider.value(
                // 没有 Provider 内部使用时会报错(ProviderNotFoundException)
                value: info,
                child: const NewPage(),
            ),
        ),
    );
}
```

## 2. ChangeNotifierProvider

> - [ChangeNotifierProvider][provider-changenotifierprovider]
> - [ChangeNotifier][flutter-changenotifier]

相对于 [`Provider`](#1-provider), `ChangeNotifierProvider` 提供了监听与通知的功能,
`Widget` 可以监听数据行为并进行刷新. 这种行为使得 `ChangeNotifierProvider`
很适合作为 [MVVM][mvvm] 模式中 `ViewModel` 部分.

首先, 我们和 `Provider` 的示例一样创建一个对象, 但是不同在于这次我们引入一个新类 `ChangeNotifier`.

```dart
// or: class InfoModel extends ChangeNotifier {
class InfoModel with ChangeNotifier {
  final String name;
  final String addr;
  int _age;

  YourModel(this.name, this.addr, int age): _age = age;

  int get age => _age;

  set age(int newAge) {
    _age = newAge;
    // 炸裂我们通知所有监听组件变化行为
    notifyListeners();
  }
}
```

```dart
/// 使用 ChangeNotifierProvider Widget 对数据进行初始化
ChangeNotifierProvider<InfoModel>(
    create: (context) => InfoModel("John", "Earth", 10),
    child: // 这里传入子 Widget
),
```

对于如何使用这个 `Provider`, 最简单的方式就是使用 `context.read<InfoModel>()` 获取数据,
`context.watch<InfoModel>()` 对 `context` 对应的 `Widget` 进行绑定, 或者使用
`context.select<InfoModel，int>(callback)` 对单独的数据进行监听. 不过一般情况下针对后两者,
更推荐使用 `Consumer` 与 `Selector` 这两个 `Widget`, 这会在后面介绍, 这里先列举最简单的使用方法:

```dart
Widget build(BuildContext context) {
  // 只要 InfoModel 内调用了 notifyListeners, 该 build 对应的 Widget 就会被重建.
  final vm = context.watch<InfoModel>();
  return TextButton(onPressed: () => context.read<InfoModel>.age += 1, child: Text(vm.toString()));
}

Widget build(BuildContext context) {
  // 只有 select 中的值发生变化, 该 build 对应的 Widget 才会被重建.
  final age = context.select<InfoModel>((vm) => vm.age);
  return TextButton(onPressed: () => context.read<InfoModel>.age += 1, child: Text(age));
}
```

### 2.1. 完整示例

[点击运行](https://dartpad.dev/?id=80b552313b127a54563a1658f9c88ee1&run=true)

{% include github_gist.html id="80b552313b127a54563a1658f9c88ee1" %}

### 2.2. 需要注意

`context.watch<T>()`, `context.select<T,R>(cb)`, `Provider.of<T>(context)` 都只能在 `build` 中使用;
如果需要在构建树外或只获取数据结构, 永远使用 `context.read<T>()`, 这些获取函数都是 `O(1)` 的, 不用担心性能问题.

获取和绑定 `Provider` 的时候请务必注意 `context` 的范围, 只有在清楚自己在干什么的时候使用变量引用 `Provider`,
否则请直接使用 `context.read<T>()` 进行获取. 这里留一个问题, 你能看出这段代码片段可能会导致的问题么:

```dart
// 假设方法在一个 StatefulWidget 的 State 中
void _onPressed() async {
  if (!mounted) return;
  final Model vm = context.read<Model>();
  final bool result = await openDialog();
  if (!mounted || !result) return;
  vm.confirmed = result;
}
```

## 3. FutureProvider / StreamProvider

顾名思义, 这两个就是 [`Provider`](#1-provider) 的 `Future` 和 `Stream` 版本, 使用方法也大差不差,
因此这里就以 `FutureProvider` 为例:

```dart
class InfoModel {
  late final String name;
  late final String addr;
  late int age;
  late final Future<bool> _init;

  InfoModel() {
    Future<bool> init() async {
      name = "John";
      addr = "Earth";
      age = 10;
      return true;
    }

    _init = init();
  }

  Future<bool> get init => _init;
}
```

```dart
FutureProvider<InfoModel?>(
  create: (context) async {
    final info = InfoModel();
    await info.init;
    return info;
  },
  initialData: null,
  child: // 这里传入子 Widget
),
```

```dart
final info = context.read<InfoModel?>();
```

### 3.1. 完整代码

[点击运行](https://dartpad.dev/?id=895c6fb58ce375548e95bbeed6fe2f3e&run=true)

{% include github_gist.html id="895c6fb58ce375548e95bbeed6fe2f3e" %}

### 3.2. 需要注意

`FutureProvider` / `SteamProvider` 和 `Provider` 一样只会构建一次数据, 除非这些 `Provider` 被重新构建.

## 4. ProxyProvider / ChangeNotifierProxyProvider

上面介绍的各类 `Provider` 都可以用于创建一个数据结构, 但是如果我们的应用中存在多个数据结构且存在依赖关系的时候,
这两个 `Proxy` 就可以发挥他们的作用.

`Proxy` 将一个多个多个 `Provider` 传递到一个的 `Provider/ChangeNotifierProvider` 中, 使其聚合为一个新的对象.
假设现在有一个玩家的对象 `PlayerInfo`, 但是玩家的不同属性需要从不同地方获取
(可能是一个 id 到 name 的对应, 也可能是一个账户信息 `AcccountInfo`). 完整代码如下:

[点击运行](https://dartpad.dev/?id=0f9a3cfb4f4364e86e6d8ff7cbd6f301&run=true)

{% include github_gist.html id="0f9a3cfb4f4364e86e6d8ff7cbd6f301" %}

## 5. Consumer / Selector

前文介绍了 `context.read / context.watch / context.select`, 这一节将介绍 后两者的代替品;
他们相较于前者主要有以下优点:

1. 使用更加方便.
2. 更符合 `Flutter` 设计思想 (万物皆 Widget).

下面会给出相互等价的代码, 可以观察他们的区别:

```dart
/// [context.watch)]
Widget buld(BuildContext context) {
  return ListTile(
    title: Builder(
      builder: (context) => Text(context.watch<Model>().title),
    ),
    subTitle: "subtitle",
  );
}
/// [Consumer]
Widget buld(BuildContext context) {
  return ListTile(
    title: Consumer<Model>(
      builder: (_, model, __) => Text(model.title),
    ),
    subTitle: "subtitle",
  );
}
```

```dart
/// [context.select]
Widget buld(BuildContext context) {
  return ListTile(
    title: Text(context.select<Model>((m) => m.title)),
    subTitle: Text(context.select<Model>((m) => m.subtitle)),
    leading: context.read<Model>.buildLeading(),
  );
}

/// [Selector]
Widget buld(BuildContext context) {
  return Selector<Model, (String, String)>(
    selector: (context, m) => (m.title, m.subtitle),
    builder: (context, value, child) => ListTile(
      title: value.$1,
      subtitle: value.$2,
      leading: context.read<Model>.buildLeading(),
    ),
  );
}
```

具体使用那种方式看使用的位置和个人喜好, 不过我个人倾向于使用 `Selector` 与 `Consumer`,
这种方式可以更直观体现出 `Widget` 刷新间的层级关系. 而 `context.watch` 与 `context.select`
则更适用于框架代码或者一些很小的 `Widget`.

必须再次强调, 两种代码之间并无区别, 需要根据代码整体风格进行选取或混用.

### 5.1. 关于 `Consumer2` ,`Selector2` 等类似结尾含有 `2/3/4/5/6` 的方法

带有 `ConsumerX` 的 `Widget` 可以同时监听多个 `ChangeNotifier` 的变更, 只要其中一个通知后便会重构子树.

带有 `SelectorX` 的 `Widget` 和 `ConsumerX` 类似, 只不过可以选择向子 `Widget` 暴露一个值.
`Dart3` 之前由于缺乏对 `Tuple` 类型的支持, 必须引入 `Tuple` 的第三方 package,
但如果使用 `Dart3` 以及以后的版本, 语言内部已经原生实现了对 `Tuple` 的支持, 代码如下:

```dart
/// Compatible with dart2, bad for new code after dart3
import 'package:tuple/tuple.dart';
Selector<Model, Tuple2<int, String>>(
    selector: (context, m) => Tuple2(m.level, m.name),
    builder: (context, value, child) => Text("${value.item1}, ${value.item2}");
)

/// Good for dart3, But not compatible with dart2,,
Selector<Model, ({String name, int level})>(
    selector: (context, m) => (name: m.name, level: m.level),
    builder: (context, value, child) => Text("${value.name}, ${value.level}"),
);
// or
Selector<Model, (String name, int level)>(
    selector: (context, m) => (m.name, m.level),
    builder: (context, value, child) => Text("${value.$1}, ${value.$2}"),
);
```

## 6. 总结

以上便是个人在学习并使用 `Provider` 时的一些总结. 本文只是简单介绍 `Provider` 的一些优势和使用姿势.
后续有机会会另起一篇博客粗浅讲解一下 `Provider` 的源码, 以及如何通过 `InheritedNotifier`
自定义一个 `Provider`.

## a. 关于各种 `Provider` 中的 `lazy` 参数

`Provider` 默认是 "懒加载" 的, 如果我们需要立刻执行 `create` 或者 `update` 方法, 则必须将 `lazy` 置为 `false`.
一般情况下按需加载能够起到优化性能的目的, 但是如果有些对象必须尽快完成初始化(比如 `db`, 或者读取本地配置),
则将其行为改变为立刻加载是有必要的, 否则可能会存在一些意想不到的情况 (e.g. 漏写本地日志, 关键配置没有及时读取等).

## b. 各种 `Provider` 的 `.value` 构造函数

`Provider` 官方建议, 如果是在 `Provider` 内部初始化对象, 则使用 `Provider()`, 而如果对象已经在外部初始化完毕,
则使用 `Provider.value`.

```dart
// bad
final model = Model();
Provider(create: (context) => model);
Provider.value(value: Model());
// good
final model = Model();
Provider.value(value: model);
Provider(create: (context) => Model());
```

`Provider.value` 在界面跳转时很有用, 因为新的界面是一个新的树, 而新树中不包含当前界面中已经初始化完毕的数据对象.
而我们可以通过以下代码完成不同界面之间数据对象的传递.

```dart
// .... in StatefulWidget
void _onPress() {
  if (!mounted) return;
  final model = context.read<Model>();
  final result = Navigator.of(context).push(
    MaterialPageRoute(
      builder: (context) => MultiProvider(
        providers: [
          ChangeNotifierProvider.value(value: model),
          // 注意: 这样获取是不行的, 因为context是 builder(新界面)的,
          // 一定要在这里获取的话只能使用 this.context,
          // 总之需要使用当前界面而不是跳转后界面的 context 来获取.
          // ChangeNotifierProvider.value(value: context.read<Model>()),
        ],
        child: ChildPage(),
      ),
    ),
  )
}
```

需要要注意的是: 最好在数据结构内部使用 mounted 检查, 否则过深的传递数据对象可能会导致子界面持有一个已经失效的 `Provider`.
最好**只**在一些简单的二级页面(比如一些依赖当前界面的 dialog 或者子节面)中使用这种传递方式.

```dart
class Model with ChangeNotifier {
    _mounted = true;

    bool get mounted => _mounted;

    @override
    void dispose() {
        _mounted = false;
        super.dispose();
    }
}

// .... in StatefulWidget
void onAction() {
    if (!mounted) return;
    final vm = context.read<Model>();
    if (!vm.mounted) return;
    // do something here
}
```

[flutter-rcmd]: https://docs.flutter.dev/data-and-backend/state-mgmt/simple#accessing-the-state
[mvvm]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
[provider-provider]: https://pub.dev/documentation/provider/latest/provider/Provider-class.html
[provider-changenotifierprovider]: https://pub.dev/documentation/provider/latest/provider/ChangeNotifierProvider-class.html
[flutter-changenotifier]: https://api.flutter.dev/flutter/foundation/ChangeNotifier-class.html
