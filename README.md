# Siesta

Swift-based iOS REST Client Framework

## The Problem

Want your app to talk to an API? Welcome to your state nightmare!

You need to display response data whenever it arrives, unless the requesting ViewController is no longer visible, unless some other currently visible ViewController happens to want the same data. You should show a loading indicator, display user-friendly errors (but no modal alert dogpiles!), give users a retry mechanism, and hide all that when a subsequent request succeeds. Be sure to avoid redundant requests. Oh, and remember not to retain your ViewController by accident in your callback closures.

What could possibly go wrong?

## The Solution

Siesta ends this headache by providing an observable model of a RESTful resource’s state. The model answers three basic questions:

* What is the latest data for this resource, if any?
* Did the latest request result in an error?
* Is there a request in progress?

…then provides notifications whenever the answers to these questions change.

Siesta handles all the transitions and corner cases to deliver these answers wrapped up with a pretty bow on top, letting you focus on your UI.

## Features

* Decouples UI state from network request state
* Observer model eliminates complex, error-prone state tracking logic
* Coordinates requests and data sharing across ViewControllers
* Eliminates redundant network requests
* Provides transparent Etag / If-Modified-Since handling
* Painless built-in response parsing for JSON, XML, plain text
* Customizable response transformation
* Unified reporting for connection errors, server errors, and client-side parsing errors
* Prebaked UI components for error overlay and progress bar

Coming soon…er or later:

* Intelligent progress reporting that accounts for request, latency, and response
* Customizable data caching

## Usage

Create a `Service` singleton for each API your app uses:

```swift
import Siesta

class MyAPI: Service {
    static let instance = MyAPI(base: "https://api.example.com")
}
```

Retrieve `Resource` objects from the service:

```swift
MyAPI.instance.resource("/profile")
MyAPI.instance.resource("/items").child("123").child("related")
MyAPI.instance.resource("/items/123/related") // same as previous
```

Within the context of a `Service`, you will keep getting the same `Resource` instance for the same URL, no matter how you navigate to it.

Add convenience accessors for commonly used resources:

```swift
class MyAPI: Service {
    static let instance = MyAPI(base: "https://api.example.com")

    var profile: Resource { return resource("profile") }
}
```

Resources start out empty — no data, no error, not loading. To trigger a GET request:

```swift
MyAPI.instance.profile.loadIfNeeded()
```

Don’t worry about calling `loadIfNeeded()` too often. Call it in your `viewWillAppear()`! Call it in response to touch events! Call it 50 times a second! It automatically suppresses redundant requests. (Data expiration time is configurable on a per-service and per-resource level.)

UI components can observe changes to a resource, either by implementing the `ResourceObserver` protocol:

```swift
MyAPI.instance.profile.addObserver(self)
```

…or by providing an observer closure:

```swift
MyAPI.instance.profile.addObserver(self) {
    resource, event in
    …
}
```

Observers receive a notification when a resource starts loading, receives new data, or receives an error. Each observer is also pinged immediately when it first starts observing, even if the resource has not changed. This lets you put all your UI-populating code in one place.

The simplest way to implement your observer is to ignore the triggering event, and take an idempotent “update everything” approach:

```swift
func resourceChanged(resource: Resource, event: ResourceEvent) {
    // The convenience .json accessor returns empty dict if no data,
    // so the same code can both populate and clear fields.
    let json = JSON(resource.json)
    nameLabel.text = json["name"].string
    favoriteColorLabel.text = json["favoriteColor"].string

    if resource.loading {
        activityIndicator.startAnimating()
    } else {
        activityIndicator.stopAnimating()
    }

    errorLabel.text = resource.latestError?.userMessage
}
```

Note the pleasantly reactive flavor this code takes on — without the overhead of adopting full-on Reactive programming with captial R.

If updating the whole UI is an expensive operation (but it rarely is; benchmark first!), you can use the `event` parameter and the metadata in `latestData` and `latestError` to fine-tune your UI updates.

Note that a resource might have failed on the last request, have older valid data, _and_ have a new request in progress. Siesta does not dictate which of these take precedence in your UI. It just tells you the current state of affairs, and leaves it to you to determine how to display it. Want to always show the latest data, even if there was a more recent error? No problem. Only show a loading indicator if no data is present? You can do that.

Putting it all together:

```swift
import Siesta
import SwiftyJSON

class ProfileViewController: UIViewController, ResourceObserver {
    @IBOutlet weak var nameLabel, favoriteColorLabel: UILabel!
    @IBOutlet weak var activityIndicator: UIActivityIndicatorView!
    @IBOutlet weak var errorLabel: UILabel!

    override func viewDidLoad() {
        super.viewDidLoad()

        MyAPI.instance.profile.addObserver(self)
    }

    override func viewWillAppear(animated: Bool) {
        MyAPI.instance.profile.loadIfNeeded()
    }

    func resourceChanged(resource: Siesta.Resource, event: Siesta.ResourceEvent) {
        let json = JSON(resource.json)
        nameLabel.text = json["name"].string
        favoriteColorLabel.text = json["favoriteColor"].string

        if resource.loading {
            activityIndicator.startAnimating()
        } else {
            activityIndicator.stopAnimating()
        }

        errorLabel.text = resource.latestError?.userMessage
    }
}
```

## UI Components

The code above is already easy — but the business of showing the activity indicator and error message can get repetitive. Siesta provides a status overlay view that takes care of that for you.

The overlay is designed to cover your entire content view when there is an error, by you can position it as you like. It comes with a tidy standard layout:

<img alt="Standard error overlay view" src="Docs/images/standard-error-overlay@2x.png" width=320 height=136>

…and you can also provide your own custom .xib.

Using the standard overlay, the example above becomes:

```swift
class ProfileViewController: UIViewController, ResourceObserver {
    @IBOutlet weak var nameLabel, favoriteColorLabel: UILabel!
    var statusOverlay: ResourceStatusOverlay!

    override func viewDidLoad() {
        super.viewDidLoad()

        statusOverlay = ResourceStatusOverlay().embedIn(self)

        MyAPI.instance.profile
            .addObserver(self)
            .addObserver(statusOverlay)
    }

    override func viewDidLayoutSubviews() {
        statusOverlay.positionToCoverParent()
    }
    
    override func viewWillAppear(animated: Bool) {
        MyAPI.instance.profile.loadIfNeeded()
    }

    func resourceChanged(resource: Siesta.Resource, event: Siesta.ResourceEvent) {
        let json = JSON(resource.json)
        nameLabel.text = json["name"].string
        favoriteColorLabel.text = json["favoriteColor"].string
    }
}
```