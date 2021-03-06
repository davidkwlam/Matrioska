# Matrioska

[![Language: Swift](https://img.shields.io/badge/lang-Swift-yellow.svg?style=flat)](https://developer.apple.com/swift/)
[![Build Status](https://travis-ci.org/runtastic/Matrioska.svg?branch=master)](https://travis-ci.org/runtastic/Matrioska)
[![Version](https://img.shields.io/cocoapods/v/Matrioska.svg?style=flat)](http://cocoapods.org/pods/Matrioska)
[![DocCov](https://img.shields.io/cocoapods/metrics/doc-percent/Matrioska.svg)](http://cocoadocs.org/docsets/Matrioska)
[![codecov](https://codecov.io/gh/runtastic/Matrioska/branch/master/graph/badge.svg)](https://codecov.io/gh/runtastic/Matrioska)
[![License](https://img.shields.io/cocoapods/l/Matrioska.svg?style=flat)](http://cocoapods.org/pods/Matrioska)
[![Platform](https://img.shields.io/cocoapods/p/Matrioska.svg?style=flat)](http://cocoapods.org/pods/Matrioska)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)

> Matrioska lets you create your layout and define the content of your app in a simple way.  

- [Installation](#installation)
- [Usage](#usage)
  - [Standard Components](#standard-components)
  - [Meta](#meta)
    - [ComponentMeta](#componentmeta)
    - [ExpressibleByComponentMeta](#expressiblebycomponentmeta)
  - [Creating Components](#creating-components)
  - [Layout](#layout)
  - [Load Components from JSON](#load-components-from-json)
- [Roadmap](#roadmap)
- [License](#license)

> NOTE: Matrioska is under active development, until `1.0.0` APIs might and will change a lot. The project is work in progress, see [Roadmap](#roadmap) or open issues.

The vision of Matrioska is to let you build and prototype your app easily, reusing views and layouts as well as dynamically define the content of your app.
With Matrioska you can go as far as specifing the content and layout of your views from an external source (e.g. JSON).
With this power you can easily change the structure of your app, do A/B testing, staged rollout or prototype.

To build your UI you can use nested `Component`s. A `Component` can be 3 different things:

- **View**: Any `UIViewController` that can use AutoLayout to specify its `intrinsicContentSize`
- **Cluster**: Views with children (other `Component`s). A cluster is responsible for laying out its children’s views. Since a cluster is itself a view it can also contain other clusters.
- **Wrapper**: A View with only one child (a `Component`). You can see it as a special cluster or as a special view. It’s responsible for displaying its child’s view.

The goal is to provide a tiny but powerful foundation to build your app on top of.
Matrioska will contain a limited set of standard components and we will consider to add more on a case by case basis.  
It’s really easy to extend Matrioska to add new components that fits your needs.

## Installation

Using [CocoaPods](http://cocoapods.org/):

```ruby
use_frameworks!
pod ‘Matrioska’
```

Using [Carthage](https://github.com/Carthage/Carthage):

```
github “runtastic/Matrioska”
```

## Usage

### Standard Components

Matrioska defines some standard `Component`s that can be used to create your layout:

| id | usage | config |
|---|----|-----|
| tabbar |  `ClusterLayout.tabBar(children, meta)` | `TabBarConfig` and `TabConfig` (children) |
| stack | `ClusterLayout.stack(children, meta)` | `StackConfig` |  

See the documentation for more informations.

### Meta

Every `Component` may handle additional metadata. The `Component`’s meta is optional and the `Component` is responsible for handling it correctly. Metadata can be anything from configuration or additional information, for example a view controller title.

#### ComponentMeta

Every meta has to conform to `ComponentMeta`, a simple protocol that provides a keyed (String) subscript.  
`ComponentMeta` provides a default implementation of a subscript that uses reflection (`Swift.Mirror`) to mirror the object and use its property's names and values. Objects that conform to this protocol can eventually override this behavior.  
`ZipMeta`, for example, is a simple meta wrapper that aggregates multiple metas together; see its documentation and implementation for more info.
`Dictionary` also conforms to `ComponentMeta`, this is a convenient way to provide meta but is especially useful to materialize a `ComponentMeta` coming from a json/dictionary.

#### ExpressibleByComponentMeta

When creating a new `Component` you should document which kind of meta it expects. A good way to do this is to also create an object that represents the `Component`’s meta (e.g. see `StackConfig`) and make it conform to `ComponentMeta`.  
`ExpressibleByComponentMeta`, however, provides some convenience methods that lets you load your components from a json or materialize a meta from a dictionary; that is, it lets you express your meta configuration by any `ComponentMeta` object.  
Other than `ComponentMeta`’s requirements you also need to provide a ` init?(meta: ComponentMeta)`, then you can materialize any compatible meta into your own `ExpressibleByComponentMeta`.  

Example:

```swift
public struct MyConfig: ExpressibleByComponentMeta {
    public let title: String

    public init?(meta: ComponentMeta) {
        guard let title = meta["title"] as? String else {
            return nil
        }
        self.title = title
    }
}
```

After defining `MyConfig` we can materialize it from other `ComponentMeta`s if possible:

```swift
MyConfig.materialize([“title”: “foo”]) // MyConfig(title: "foo")
MyConfig.materialize([“foo”: “foo”]) // nil
MyConfig.materialize(nil) // nil
MyConfig.materialize(anotherMyConfigInstance) // anotherMyConfigInstance
```

### Creating Components

Create custom components:

```swift
// Create a cluster by extending an existing implementation
extension UITabBarController {
    convenience init(children: [Component], meta: Any?) {
        self.init(nibName: nil, bundle: nil)
        self.viewControllers = children.flatMap { $0.viewController() }
        // handle meta
    }
}

// Any UIViewController can be used as a View
// we can define a convenience init or just use an inline closure to build the ViewController
class MyViewController: UIViewController {
    init(meta: Any?) {
        super.init(nibName: nil, bundle: nil)
        guard let meta = meta as? [String: Any] else { return }
        self.title = meta["title"] as? String
    }
}
```

Then create models that can be easily used to create the entire tree of views:

```swift
let component = Component.cluster(builder: UITabBarController.init, children: [
    Component.view(builder: MyViewController.init, meta: ["title": "tab1"]),
    Component.view(builder: { _ in UIViewController() }, meta: nil),
    ], meta: nil)

window.rootViewController = component.viewController()
```

### Layout

Views are responsible for defining their `intrinsicContentSize` using AutoLayout, clusters can decide whether to respect their dimensions or not, both vertical and horizontal or also only one of the two.
To make sure that a `Component`’s `UIViewController`has a valid `intrinsicContentSize` you need to add appropriate constraints to the view. [To know more about this read the documentation about “Views with Intrinsic Content Size”](https://developer.apple.com/library/content/documentation/UserExperience/Conceptual/AutolayoutPG/ViewswithIntrinsicContentSize.html).

### Load Components from JSON

`Components` can also be loaded from JSON. For this, you are responsible for registering factories (`Component` builders) that will be used when parsing the JSON structure. In order to register factories, usage of `JSONFactory` is needed:

```
let jsonFactory = JSONFactory()

jsonFactory.register(with: "tab_bar", factoryBuilder: { (children, meta) -> Component in
    ClusterLayout.tabBar(children: children, meta: meta)
})

jsonFactory.register(with: "navigation", factoryBuilder: { (child, meta) -> Component in
    Component.wrapper(builder: { _ in UINavigationController() }, child: child, meta: meta)
})

jsonFactory.register(with: "table_view", factoryBuilder: { (meta) -> Component in
    Component.view(builder: { _ in UITableViewController() }, meta: meta)
})
```

Whenever you register a new factory you should provide the `type` key that will match the JSON. Check the [provided JSON schema](/Documentation/JSON\ schema\ guide.md) for more details on that.

You can register different factories for `View`, `Wrapper` and  `Cluster` `Component` types using the `JSONFactory`. After registration, you can use the factory to get the component out of a JSON:

```
let component = try jsonFactory.component(from: json)
```

Besides providing `type` on the JSON, `Component`s should also match the JSON schema that the library provides by default whenever using the built-in components (TabBar or Stack) meta configuration.

Check the [JSON schem guide](/Documentation/JSON\ schema\ guide.md) for more information.

## Roadmap

- Rulesets to define the visibility of a Component [#4](https://github.com/runtastic/Matrioska/issues/4)
- Deep Linking [#5](https://github.com/runtastic/Matrioska/issues/5)

## License

Matrioska is released under the MIT License.  
At Runtastic we don't keep an internal mirror of this repo and all development on Matrioska is done in the open.
