---
layout: post
title: Building a Collection For SwiftUI - Introduction
---

`UICollectionView` has been an essential tool for Apple app developers since its introduction in iOS 6. Over the years it received a range of improvements and was ported to tvOS so that you can build layouts with the same formalism on both platforms. In iOS and tvOS 13 `UICollectionView` received the most significant enhancements ever made since its introduction, namely diffable data sources and compositional layouts.

Last year Apple introduced SwiftUI, its declarative UI framework. Now in its second iteration with iOS and tvOS 14, SwiftUI is obviously still lagging behind in terms of functionality. While SwiftUI provides lists, scrollable views and grids as of version 14, it still does not provide a collection which could compete with `UICollectionView` in terms of feature set or versatility. This does not mean that grid and table layouts usually achieved with `UICollectionView` are not possible with SwiftUI. The smaller components already available can be easily combined to create complex scrollable layouts, so that what can be achieved in UIKit can be more or less achieved in SwiftUI as well.

When evaluating SwiftUI as a viable option for porting our [iOS apps](https://github.com/SRGSSR/playsrg-apple) to tvOS, one of the essential requirements to satisfy was to ensure grid and table layouts, used throughout our implementation, would be easy to create in SwiftUI, with satisfying user experience. I initially and naively thought so, but I did not quite imagine how wrong I was.

I thought sharing my experience could probably be helpful to other developers who might be considering SwiftUI for their own apps as well. Since the material to be covered is larger than initially expected, I opted out for a series of articles requiring some prior SwiftUI and UIKit knowledge, but which I tried to keep accessible for newcomers as well:

- [Part 1: Mimicking Collections in SwiftUI](/swiftui_collection_part1): A mild introduction into the world of collection aleternatives in SwiftUI (and associated delusions).
- [Part 2: SwiftUI Collection Implementation](/swiftui_collection_part2): Where we build a first working SwiftUI collection from the ground up.
- [Part 3: Fixes and Focus Management](/swiftui_collection_part3): Where we solve issues with the implementation and implement focus management for tvOS.

The [source code](https://github.com/defagos/SwiftUICollection) for this series is available if you want to have a look at the implementation while reading the articles.