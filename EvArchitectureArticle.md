# The EV Architecture

[image:F492DBDF-8034-4F01-8CAE-D3488DF41DE1-10131-000024D3482D7162/1*4Iuo18Fd6leQQkAaZ27Yrg.png]


## Foreword

This is my first technical article.

It was also written in a language other than English and translated with the assistance of **ChatGPT**, so please do not judge it too harshly.


## Background

A few months ago, while searching for answers to yet another painful question about a problem related to **MVVM**, I came across an interesting thread on the Apple forum:  [Stop using MVVM for SwiftUI | Apple Developer Forums](https://developer.apple.com/forums/thread/699003) 

It is an inspiring story of a person who asked the question:
*“Why do we need **MVVM** if **SwiftUI** is already an architecture in itself?”*

It’s an entertaining read, so I recommend it to everyone.

I’ll add one of the interesting images from that article:

[image:47EBE7BF-D41B-4447-BE99-2A25E20082E6-10131-0000258362DD6CF2/1*PIiZ-tSToCuyYJrtLpo1Mw.jpeg]

However, even there, during the discussions, everything still comes down to the fact that if a project is more than just a basic template, then it cannot be done without **Classes** (I’m talking about **ObservableObject**, for example). 
They exist and are used outside of the **MVVM** paradigm(in the above example), but still.

For me personally, **Classes** were a real headache. The whole reference system and working with **ARC** did not represent anything good for the average developer.
I was happy to hear that **SwiftUI** constantly mentioned the fact that it is built entirely on **Structures**.

But then came the disappointment when I realized:
*Without **Classes**, we can’t do it. Without them, and without all the accompanying **problems***.

Working with **MVVM** is an eternal struggle with itself. And the bigger the project, the bigger the problems. And, of course, in terms of data flow within the application.

Hence, the idea of creating an architecture based solely on **Structures** and without all the extra layers like **ViewModel**.


## Why EV?
Like most architectures, the name of this one is just a review of the basic underlying tools. Namely, **Enum** and **View**. The entire project, in essence, is built on this basic **Enum** and **View**. At the same time, the **View** itself can be a **Model** object.

Thus, the entire project consists of Views.

Thanks to this, it is possible to refer to any object as an object with data at any given moment.


## Approach to Architecture
This article is accompanied by code snippets from the basic, simplest example. The full code can be found at this  [link](https://github.com/riley-usagi/RandomColorEV) .

The main feature of this architecture is its unique approach to working with data flow within the application.

In this case, **EV** proposes a simple approach in the format: **Event** -> **Actions**.

Any **Action** performed by the user or internal/external **Events** triggers the activation of the corresponding **Event**, which in turn triggers the activation of all the **Actions** on the screens/views that are associated with that **Event**.

All of this is achieved using tools such as **CurrentValueSubject** from **Combine** plus **DynamicProperty**.


## Diagram

[image:86D7C95A-9D11-47CC-B5C9-904E77FF8174-10131-000025FA7CEB46AC/1*lTqANJltzRnUDZWUyJ6XsA.png]


## Example

[image:6529DE3F-C733-486A-928F-39B02AA7A8B0-10131-000026005C5ECEA5/1*aKDA3fLKGrPf7wHKAVh1_A.gif]

The entire purpose of the application is to change the background color of the left and right screens simultaneously when the button in the middle screen is pressed. The application is intentionally designed to show how a single Event can trigger multiple Actions on different screens.

Let’s take a closer look at the app’s code:

```swift
@main struct RandomColorEVApp: App {
  
  enum TabItem: Equatable {
    case left, center, right
  }
  
  @State var selectedTab: TabItem = .center
  
  var body: some Scene {
    WindowGroup {
      TabView(selection: $selectedTab) {
        LeftScreen()
          .tag(TabItem.left)
        
        CenterScreen()
          .tag(TabItem.center)
        
        RightScreen()
          .tag(TabItem.right)
      }
      .ignoresSafeArea()
      .tabViewStyle(PageTabViewStyle())
    }
  }
}
```

The app uses a `TabView` to display three screens: left, center, and right.

Let’s take a closer look at one of the screens:

```swift
struct CenterScreen: View {
  var body: some View {
    VStack {
      Text("Center")
      
      Button {
        eventSubject.send(.changeBothSidesColors)
      } label: {
        Text("Change colors")
      }
    }
  }
}
```

This code defines the design of the screen and contains a button that sends an `EventEnum` value to `eventSubject`.

```swift
let eventSubject: CurrentValueSubject<EventEnum, Never> = .init(.initial)
```

In this architecture, `EventEnum` is a central part of the system and contains all possible events and actions that can be performed by the app:

```swift
enum EventEnum: Equatable {
  case initial
  case changeBothSidesColors
  
  static func actionsByEvent(_ event: Self) -> [any InnerViewAction] {
    switch event {
    case .changeBothSidesColors:
      return [LeftScreen.InnerAction.changeLeftColor, RightScreen.InnerAction.changeRightColor]
    default:
      return []
    }
  }
}
```

Each case in the `EventEnum` enumeration represents a possible event in the app, and the static method `actionsByEvent` returns a list of actions associated with a particular event.

Note the `[any InnerViewAction]`. The `InnerViewAction` protocol defines all actions that can be performed by a view in the app.

```swift
protocol InnerViewAction: Equatable {
  static var `default`: Self { get }
}
```

Here’s an example of how this works in `LeftScreen` (and `RightScreen` is similar):

```swift
struct LeftScreen: View {
  
  @Action(InnerAction.initial) var action
  
  @State var color: Color? = nil
  
  var body: some View {
    ZStack {
      color ?? .white
      Text("Left")
    }

    .onReceive($action) { newAction in
      switch newAction {
        
      case .changeLeftColor:
        color = randomColor()
        
      default:
        break
      }
    }
  }
  
  enum InnerAction: InnerViewAction {
    static var `default`: LeftScreen.InnerAction { .initial }

    case initial
    case changeLeftColor
  }
}
```

The `.onReceive` modifier is triggered only in the views that contain a specific action associated with a specific event. This is achieved through the use of a special `DynamicProperty` called `@Action`, which is implemented as follows:

```swift
@propertyWrapper struct Action<T: InnerViewAction> {
  
  private let currentValue: CurrentValueSubject<T, Never>
  
  init(_ receivedValue: T) {
    
    self.currentValue = CurrentValueSubject(receivedValue)
    
    _ = eventSubject
    
      .flatMap { receivedEvent -> AnyPublisher<any InnerViewAction, Never> in
        return EventEnum.actionsByEvent(receivedEvent).publisher.eraseToAnyPublisher()
      }
    
      .compactMap { receivedAction in
        return receivedAction as? T
      }
    
      .subscribe(on: DispatchQueue.main)
    
      .assign(to: \.wrappedValue, on: self)
  }
  
  public var wrappedValue: T {
    get { currentValue.value }
    
    nonmutating set {
      currentValue.value = newValue
      currentValue.value = T.default
    }
  }
  
  public var projectedValue: AnyPublisher<T, Never> {
    get {
      currentValue
        .compactMap({ $0 })
        .drop { $0 == T.default }
        .eraseToAnyPublisher()
    }
  }
}
```

* In the constructor, we subscribe to all events from the global `eventSubject`.
* Using `.flatMap`, we obtain an array of actions associated with a specific event in the form of a publisher.
* Using `.compactMap`, we filter only those objects that match our `T` type, which comes to this structure from the view, for example, `@Action(InnerAction.initial) var action`, or in a more complete form of writing, `@Action(LeftScreen.InnerAction.initial) var action`.
* In `DispatchQueue.main`, we store the action object in the `wrappedValue` of our `@Action`.

This is how @Action in any view receives only those objects that are relevant to it, but at the same time, it is triggered by the same event that was sent.

This is the basic principle of the EV architecture.

---

# Extra

## Models — Data Source

In most cases, **Models** are essential in EV architecture. While in the standard **MVVM** architecture, it is mandatory to have an additional layer in the form of **ViewModel**, whether needed or not, in the case of **EV**, I use **Models** to the maximum, following the principles outlined in the article *“Stop using MVVM for SwiftUI | Apple Developer Forums.”*

[image:DEAFB669-3BE4-4E82-9E18-FF9EE8A0F9AA-10131-000026C144AF4DE4/1*NwF9TmlpiAAIkvt-O_yOsA.png]

It is unnecessary to create a separate layer when data can be accessed directly from the Model.

## Model is a View

In my research, I came to the conclusion that the **Model**, as such, is a single layer that can be built upon by converting it into a **View**. This way, the data object can be accessed both as a minimalistic and an extended object, depending on the need.

Example:

```swift
struct ColorModel: View, Identifiable {
  
  var id: UUID      = UUID()
  var color: Color  = .clear
  
  @State private var modelViewType: ModelViewType = .initial
  
  var body: some View {
    switch modelViewType {
    case .initial:
      color
    case .withLabel:
      ZStack {
        Text(String(describing: color))
        color
      }
    }
  }
  
  enum ModelViewType: Equatable {
    case initial
    case withLabel
  }
}
```


## State Machine

In **EV** architecture, **Views** tend to grow rapidly in complexity, and keeping track of the View’s state becomes increasingly difficult. To address this issue, I developed a basic template for each **View**, which separates the logic and reduces the amount of code for each state:

```swift
import SwiftUI

struct InitialScreen: View {
  
  
  // MARK: - Parameters
  
  /// The current View Action
  @Action(InnerAction.initial) var action
  
  /// The current View state
  @State var viewState: ViewStateEnum = .initial
  
  
  // MARK: - Body
  
  var body: some View {
    content
      .onReceive($action) { newAction in
        switch newAction {
          
        case let .changeStateTo(newState):
          viewState = newState
          
        default:
          break
        }
      }
  }
}


// MARK: - Content

extension InitialScreen {
  
  private var content: some View {
    switch viewState {
    case .initial:
      return AnyView(initial)
    case .isLoading:
      return AnyView(isLoading)
    case .loaded:
      return AnyView(loaded)
    case let .failed(error):
      return AnyView(Text(String(describing: error)))
    }
  }
}


// MARK: - Initial

extension InitialScreen {
  var initial: some View {
    Text("")
      .task { viewState = .isLoading }
  }
}


// MARK: - Loading

extension InitialScreen {
  var isLoading: some View {
    ProgressView()
      .task { viewState = .loaded }
  }
}


// MARK: - Loaded

extension InitialScreen {
  var loaded: some View {
    Text("Loaded")
  }
}


// MARK: - Actions

extension InitialScreen {
  enum InnerAction: InnerViewAction {
    case initial
    case changeStateTo(ViewStateEnum)
  }
}
```

Each extension can easily be stored in a separate file.

The **View** state is always in one of four states: *initial*, *isLoading*, *loaded*, and *failed*.
With the help of the **Action** .changeStateTo, the entire **View** can be redrawn quickly, and actions can be performed in the background.

Inside the asynchronous modifier .task, in the initial state, all possible logical checks are performed.

In this example, in the.isLoading state, data is asynchronously loaded from a remote server, local storage, or a database.

Once all the necessary information is loaded, the status can be changed to.loaded, and the user can be sure that all the necessary data is displayed on the screen.

If an error occurs during the checks in the.initialstate or during data loading in the.isLoadingstate, the status is changed to.failed, and the user sees an error message.


## Conclusion

That’s all for now.

You can find more information on these and other mechanics that help solve non-standard tasks with standard tools at the links below.


## Links

 [RandomColorEV](https://github.com/riley-usagi/RandomColorEV)  — Level: Easy

 [AncientCowboy](https://github.com/riley-usagi/AncientCowboy)  — Level: Advanced

 [Stop using MVVM for SwiftUI | Apple Developer Forums](https://developer.apple.com/forums/thread/699003) 
