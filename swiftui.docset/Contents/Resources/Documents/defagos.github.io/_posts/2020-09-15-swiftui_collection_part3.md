---
layout: post
title: Building a Collection For SwiftUI (Part 3) - Fixes and Focus Management
---

In [part 2 of this article series](/swiftui_collection_part2) we implemented a first working collection view in SwiftUI, powered by `UICollectionView`. This implementation is promising but still has a few issues and shortcomings that we will address in this final article.

_This article is part 3 in the [Building a Collection For SwiftUI](/swiftui_collection_intro) series_.

* TOC
{:toc}

## Fixing Cell Frames

When running our shelf example, cells initially on screen have correct frames, while cells emerging from screen edges do not:

![SwiftUI CollectionView](/images/first_swiftui_collection.jpg)

When inspected in the view debugger, `UICollectionViewCell` and `UIHostingController` view frames are fine, so the problem must be related to how SwiftUI assigns a frame to views contained in a `UIHostingController`. In fact, closer inspection of the applied frames reveals that the reduction in size is due to safe area insets being somehow applied.

A [Stack Overflow thread](https://stackoverflow.com/questions/61552497/uitableviewheaderfooterview-with-swiftui-content-getting-automatic-safe-area-ins) proposes a workaround for this issue until `UIHostingController` itself provides an official API to disable this behavior. This workaround applies swizzling to all hosting view instances indiscriminately, disabling safe area inset support entirely for all hosted SwiftUI views. 

This approach is too greedy and affects hosting controllers for which this behavior is actually legitimate and desired, for example a `UIHostingController` hosting the SwiftUI root of a `UIApplication`. A more surgical approach than method swizzling is to use [dynamic subclassing](https://funwithobjc.tumblr.com/post/1482787069/dynamic-subclassing), the runtime wizardry applied by key-value observing. 

To briefly sketch the theory behind dynamic subclassing, consider you have an object of some class whose behavior you want to tweak:

- First register a new subclass of this class at runtime.
- Then add methods to the subclass (possibly calling a parent implementation if this makes sense).
- Finally change the class of the object to the subclass.

In our case, we can provide the missing opt-in behavior we need as an extension on `UIHostingController`, applied by dynamically changing the hosting view class to a subclass ignoring safe area insets:

```swift
extension UIHostingController {
    convenience public init(rootView: Content, ignoreSafeArea: Bool) {
        self.init(rootView: rootView)
        
        if ignoreSafeArea {
            disableSafeArea()
        }
    }
    
    func disableSafeArea() {
        guard let viewClass = object_getClass(view) else { return }
        
        let viewSubclassName = String(cString: class_getName(viewClass)).appending("_IgnoreSafeArea")
        if let viewSubclass = NSClassFromString(viewSubclassName) {
            object_setClass(view, viewSubclass)
        }
        else {
            guard let viewClassNameUtf8 = (viewSubclassName as NSString).utf8String else { return }
            guard let viewSubclass = objc_allocateClassPair(viewClass, viewClassNameUtf8, 0) else { return }
            
            if let method = class_getInstanceMethod(UIView.self, #selector(getter: UIView.safeAreaInsets)) {
                let safeAreaInsets: @convention(block) (AnyObject) -> UIEdgeInsets = { _ in
                    return .zero
                }
                class_addMethod(viewSubclass, #selector(getter: UIView.safeAreaInsets), imp_implementationWithBlock(safeAreaInsets), method_getTypeEncoding(method))
            }
            
            objc_registerClassPair(viewSubclass)
            object_setClass(view, viewSubclass)
        }
    }
}
```

This way we only apply a change of behavior to those `UIHostingController` instances for which safe area insets must not be taken into account, for example in our `HostCell` implementation:

```swift
hostController = UIHostingController(rootView: view, ignoreSafeArea: true)
```

With this simple trick cell frames are now correct:

![Fixed frame](/images/collection_fixed_cell_size.jpg)

## Supporting Focus on tvOS

In the shelf example discussed at the end of [part 2 of this article series](/swiftui_collection_part2), using the new tvOS 14 `CardButtonStyle`, the usual focused appearance is not applied when running the application. This is actually expected behavior, as the cell itself is focusable and the button style [is not meant to receive focus in such cases](https://developer.apple.com/videos/play/wwdc2020/10042).

In fact, SwiftUI buttons wrapped in focusable cells cannot be triggered at all. As a good SwiftUI citizen, our `CollectionView` must not prevent buttons from being used in cells, therefore host cells must not be focusable themselves. This also means we will not respond to collection cell standard selection delegate either, but rather let buttons handle actions.

One way to disable focus for a cell is by implementing `canBecomeFocused` on the cell class itself and return `false`. This seems to work well in general, but this strategy breaks when data source changes are applied with animations. In such cases the focus often spins out of control on tvOS. We therefore need a better approach.

### UICollectionView and Animated Reloads
{:.no_toc}

To understand this buggy behavior we need to figure the expected behavior of a `UICollectionView` when an animated reload occurs. To find the answer I simply implemented a basic collection in UIKit, with a simple data source and focusable `UICollectionViewCell`s:
- On tvOS I could observe that the focused item is followed when data source changes are animated. The focus never spins out of control during reloads. There is still a minor issue with the focused appearance being lost after the animation ends (likely a `UICollectionView` bug), but it suffices to swipe the remote again to have a nearby item focused. 
- On iOS the content offset is simply preserved, as there is no currently focus concept.

This user experience sets our goal for our SwiftUI `CollectionView` focus behavior.

### Disabling and Enabling Focus
{:.no_toc}

Disabling focus on cell classes directly does not work, but a similar result can be achieved thanks to `UICollectionViewDelegate`, which provides a dedicated delegate method to decide at any time whether a cell must be focusable or not. We therefore make our `Coordinator` conform to this protocol and introduce an internal flag to enable or disable focus for all cells when we want:

```swift
public struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    public class Coordinator: NSObject, UICollectionViewDelegate {
        // ...
        
        fileprivate var isFocusable: Bool = false
        
        public func collectionView(_ collectionView: UICollectionView, canFocusItemAt indexPath: IndexPath) -> Bool {
            return isFocusable
        }
    }
}
```

We can now enable `UICollectionView` cell focus during reloads, letting the collection view correctly follow the currently focused item. The rest of the time cells must not be focusable. We strive to enable focus for as little time as possible, and forcing a focus update before restting the flag to its nominal value is sufficient to achieve proper behavior:

```swift
public struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    private func reloadData(in collectionView: UICollectionView, context: Context, animated: Bool = false) {
        let coordinator = context.coordinator
        coordinator.sectionLayoutProvider = self.sectionLayoutProvider
        
        guard let dataSource = coordinator.dataSource else { return }
        
        let rowsHash = rows.hashValue
        if coordinator.rowsHash != rowsHash {
            dataSource.apply(snapshot(), animatingDifferences: animated) {
                coordinator.isFocusable = true
                collectionView.setNeedsFocusUpdate()
                collectionView.updateFocusIfNeeded()
                coordinator.isFocusable = false
            }
            coordinator.rowsHash = rowsHash
        }
    }
}
```

Note that this requires the `UICollectionView` to be provided as additional parameter to our reload method.

With these changes the focus now appears as expected and the focused item is followed when the data source is updated:

![Fixed focus](/images/collection_fixed_focus.jpg)

### Remark
{:.no_toc}

During my investigations I worked with focusable cells for a while until I realized this was a bad idea. Here are a few more findings if you are interested. 

If we keep cells focusable `Button`s are basically useless, as said above. Moreover, focused appearance must be implemented manually by tweaking view properties. Scaling is easy but tilting is another matter, and the native tvOS look & feel is not easy to reproduce accurately. 

Finally, the pressed appearance (obtained for free with `CardButtonStyle`) requires catching cell interactions and transferring them up the SwiftUI view hierarchy, for example through the `@Environment` using a custom `EnvironmentKey`. Again this requires the appearance (scaling) to be adjusted manually.

For all these reasons I recommend avoiding focusable cells if you intend to wrap SwiftUI views within them.

## Supporting Supplementary Views

Supporting supplementary views is very similar to supporting cells. We simply introduce a dedicated view builder and a host view. As for cells, type inference requires the addition of a new `SupplementaryView `type parameter to the generic `CollectionView` type:

```swift
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View, SupplementaryView: View>: UIViewRepresentable {
    private class HostSupplementaryView: UICollectionReusableView {
        private var hostController: UIHostingController<SupplementaryView>?
        
        override func prepareForReuse() {
            if let hostView = hostController?.view {
                hostView.removeFromSuperview()
            }
            hostController = nil
        }
        
        var hostedSupplementaryView: SupplementaryView? {
            willSet {
                guard let view = newValue else { return }
                hostController = UIHostingController(rootView: view, ignoreSafeArea: true)
                if let hostView = hostController?.view {
                    hostView.frame = self.bounds
                    hostView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
                    addSubview(hostView)
                }
            }
        }
    }
    
    // ...
    
    let supplementaryView: (String, IndexPath) -> SupplementaryView
    
    init(rows: [CollectionRow<Section, Item>],
         sectionLayoutProvider: @escaping (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection,
         @ViewBuilder cell: @escaping (IndexPath, Item) -> Cell,
         @ViewBuilder supplementaryView: @escaping (String, IndexPath) -> SupplementaryView) {
        self.rows = rows
        self.sectionLayoutProvider = sectionLayoutProvider
        self.cell = cell
        self.supplementaryView = supplementaryView
    }
}
```

Supplementary views are registered for a single reuse identifier (for the same reason a single identifier is required for cells) but specific kinds, e.g. header, footer or custom. We store known kinds in our coordinator as they are registered so that each required registration is made at most once:
   
```swift
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View, SupplementaryView: View>: UIViewRepresentable {
    // ...
    
    class Coordinator: NSObject, UICollectionViewDelegate {
        // ...
        fileprivate var registeredSupplementaryViewKinds: [String] = []
    }

    func makeUIView(context: Context) -> UICollectionView {
        let cellIdentifier = "hostCell"
        let supplementaryViewIdentifier = "hostSupplementaryView"
        
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout(context: context))
        collectionView.register(HostCell.self, forCellWithReuseIdentifier: cellIdentifier)
        
        let dataSource = Coordinator.DataSource(collectionView: collectionView) { collectionView, indexPath, item in
            let hostCell = collectionView.dequeueReusableCell(withReuseIdentifier: cellIdentifier, for: indexPath) as? HostCell
            hostCell?.hostedCell = cell(indexPath, item)
            return hostCell
        }
        context.coordinator.dataSource = dataSource
        
        dataSource.supplementaryViewProvider = { collectionView, kind, indexPath in
            let coordinator = context.coordinator
            if !coordinator.registeredSupplementaryViewKinds.contains(kind) {
                collectionView.register(HostSupplementaryView.self, forSupplementaryViewOfKind: kind, withReuseIdentifier: supplementaryViewIdentifier)
                coordinator.registeredSupplementaryViewKinds.append(kind)
            }
            
            guard let view = collectionView.dequeueReusableSupplementaryView(ofKind: kind, withReuseIdentifier: supplementaryViewIdentifier, for: indexPath) as? HostSupplementaryView else { return nil }
            view.hostedSupplementaryView = supplementaryView(kind, indexPath)
            return view
        }
        
        reloadData(in:collectionView, context: context)
        return collectionView
    }
    
    // ...
}
```

Supplementary views must be added to your layout and defined within the corresponding view builder. Refer to the source code (link at the end of this article) for an example of use.

## Additional Considerations

Before we wrap things up, I just wanted to mention a few important behaviors related to this implementation:

- Since cells are reused any `@State` they store is temporary. Persistent state should be stored in your model instead.
- Diffable data sources need each item to have a unique identifier, no matter the row it appears in, otherwise errors will be reported. This is because items can be moved between sections in general. Several sections can display the same item, but in order to do so you need to make the item truly unique for each section, for example by wrapping it into a `struct` and associating it with its section.
- Changes are detected based on hashes. If a displayed item changes internally (e.g. some title changes) but its hash does not change, no update will be triggered.

Further improvements could be made and are left as exercise for the reader:

- Make supplementary view support optional.
- Make tabs optionally scroll with the content on tvOS (Hint: `tabBarObservedScrollView`).
- Use decoration views to add focus guides for navigation between rows of different length on tvOS.
- Port this collection to macOS and `NSCollectionView`. I read somewhere that nested stacks and scroll view performance was poor on macOS as well, probably because macOS has focus support for keyboard navigation. It is therefore likely that the approach discussed for tvOS can be applied to macOS in a similar way.

## Source Code

The [code for the collection view](https://github.com/defagos/SwiftUICollection) (< 200 LOC) and an example of use is available on GitHub. I even added a SPM manifest so that you can quickly test the collection in a project of your own. This does not mean the code is intended to be used as a library yet (it should be applied to a broader set of cases first), but feel free to use it any way you want.

## Wrapping Up

This article is the last one in this series. I hope you enjoyed the ride and learned a few things! 

You can also return to the [introduction](/swiftui_collection_intro).
