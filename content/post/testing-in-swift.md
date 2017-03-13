---
authors:
- ybai
- dancarter
categories:
- Swift
- Swift 3
- Protocol
- Testing
date: 2017-02-28T10:55:42-05:00
draft: true
short: |
  Testing against frameworks/libraries is tricky in Swift because we can't just spy on dependencies and fake out the response. Here is how we test neatly in Swift 3.
title: Testing in Swift with dependencies out of control
---
## Background
It is common for pivots to find their phones cluttered with work photos that never get deleted. To address this problem, we built an iOS app in Swift 3 to help pivots quickly find and take action on their work photos. By the nature of the app, we found ourselves working with the native [iOS Photos library](https://developer.apple.com/reference/photos), integrating with Google Drive, and working with various views for sharing photos. The ability to easily test our code while working around various integrations was crucial for our project.

{{< responsive-figure src="/images/testing-in-swift/icon.png" class="right" >}}

It was quite a journey for us to find maintainable testing strategies for our code that dealt with these external dependencies. Looking back, there are three main points we would love to be able to share with ourselves three months ago:

* Inject all dependencies with protocols and use fakes in tests
* If something is hard to test, don't spend too much time on it, abstract it out instead
* If you are digging too deeply into the dependent library's source code, then you are doing it wrong

To illustrate the testing strategies we landed upon, we will use [UIViewController](https://developer.apple.com/reference/uikit/uiviewcontroller) as an example of a native library dependency, and Google services([Signin](https://developers.google.com/identity/sign-in/ios/start-integrating) and [Drive](https://developers.google.com/drive/v3/web/quickstart/ios?ver=swift)) as examples of external dependencies.

---
## Unit Tests

The PhotosViewController is the rootViewController and our main target for unit testing in our app. It relies on over a dozen other views and services to complete tasks and isolate responsibilities.

### **Dependency Injection**

!!!!!!!!!!!!!-------- REWRITE --------!!!!!!!!!!!!!
With [dependency injection](https://en.wikipedia.org/wiki/Inversion_of_control), the depending modules are initiated outside of our testing subject. Our testing subject can use the injected modules according to methods defined through abstraction ([protocol](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html) in Swift, [interface](https://docs.oracle.com/javase/tutorial/java/concepts/interface.html) in Java). This is essential for unit testing because we can initiate our testing subject with fake modules, which also conform the same abstraction, in tests.
!!!!!!!!!!!!!-------- REWRITE --------!!!!!!!!!!!!!


!!!!!!!!!!!!!-------- MOVE TO UI TESTS --------!!!!!!!!!!!!!
Here is the example of using [Swinject](https://github.com/Swinject/Swinject) to inject dependencies in our iOS app:

```Swift
import Swinject

class ContainerFactory {

    let container = Container() { c in
        c.register(PhotosViewController.self) { r in
            return PhotosViewController(withGoogleService: r.resolve(GoogleServiceProtocol.self)!)
        }
        c.register(GoogleServiceProtocol.self) { r in
            return GoogleService(withSignIn: GIDSignIn.sharedInstance(), withDrive: GTLRDriveService())
        }
        // And more other dependencies
    }
}
```

We registered each abstracted type with actual implementation instance, so that we can resolve it whenever needed. And when we initiate our PhotosViewController,  we inject a GoogleService instance that conforms GoogleServiceProtocol.

```Swift
class AppDelegate: UIResponder, UIApplicationDelegate {

    let containerFactory = ContainerFactory()
    var window: UIWindow?

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        let appContainer = containerFactory.container

        self.window = UIWindow(frame: UIScreen.main.bounds)
        self.window!.rootViewController = appContainer.resolve(PhotosViewController.self)!

        return true
    }
}
```
!!!!!!!!!!!!!-------- MOVE TO UI TESTS --------!!!!!!!!!!!!!

### **Faking with Protocol**

You might have noticed that we are using protocols when we declare dependencies and instantiate our PhotosViewController with implementations. Here is the use case:

**Goal**: Upload photos to Google Drive.

**Tasks**: Sign in with Google and cooperate with the Drive API.

**Solution**: We defined the following GoogleServiceProtocol with a function to upload photos with given images, and extracted all the Google related operation in GoogleService.

**Test**: We created FakeGoogleService class to conform GoogleServiceProtocol and initialize our testing target with the fake instances.

**Celebrate!** Our main controller only need to call uploadPhotos and doesn't need to worry about checking authorization status, how to assemble requests to actually upload photos, or knowing when the requests are finished.

```Swift
protocol GoogleServiceProtocol {
    func uploadPhotos(forImages: [Data]?, completion: (() -> ())?)
}
```

**Now, how to verify in tests?**

#### 1. *Synchronous calls*

We just need to verify our photosViewController has called uploadPhotos because we trust the service, so we can simply add a boolean property in our fake service and update it when the function is called, or one step further to store the parameters and verify them in the tests.

```Swift
class FakeGoogleService: GoogleServiceProtocol {

    var uploadPhotosCalled = false
    // var photosToUpload : [Data]?

    func uploadPhotos(forImages images: [Data]?, completion: (() -> ())?) {
        self.uploadPhotosCalled = true
        // self.photosToUpload = images
        completion?()
    }
}
```

#### 2. *Asynchronous calls*

What if we want to verify some behavior before the async call finishes? For example an "Uploading..." indicator that goes away after finishes.

Here is what we do: *store the callback method* -> *verified the indicator is presented* -> *call the stored callback* -> *verify the indicator is gone*.

```Swift
class FakeGoogleService: GoogleServiceProtocol {

    var completion: (() -> ())?
    var shouldCallCompletion = true

    func uploadPhotos(forImages images: [Data]?, completion: (() -> ())?) {
        self.completion = completion
        if shouldCallCompletion {
            completion?()  
        }
    }
}
```

### **Signature wrapping with extension**
#### *Case 1 - Check if a overlay view is presented*
We have a overlay view to present on top of our photosViewController when we are waiting for GoogleService to finish uploading.

Originally it's done by calling present method on the testing target itself:

```Swift
//PhotosViewController.Swift
self.present(activityOverlayViewController as UIViewController, animated: false) {
    googleService.uploadPhotos(forImages: images) {
        activityOverlayViewController.dismiss(animated: false, completion: maybeDoSomethingElse)
    }
}
```

We struggled for a long time to test it properly but we failed for the following reasons:

1. Dismissing the overlay doesn't work well for standalone view controller tests.
2. It's tedious and hard to mimic the async callback chain.

After a Ping Pong break, we came up with the idea to let the overlay view present itself:

```Swift
protocol UIViewControllerProtocol {
    func presentOn(_ view: UIViewController, withMessage: String, animated: Bool, completion: (() -> ())?)
    func dismiss(animated: Bool, completion: (() -> Void)?)
}

extension UIViewController: UIViewControllerProtocol {
    func presentOn(_ view: UIViewController, withMessage message: String, animated: Bool, completion: (() -> ())?) {
        view.present(self as UIViewController, animated: animated, completion: {
            completion?()
        })
    }
}
```

By extending UIViewController to conform UIViewControllerProtocol with the new signature, we avoid spinning up a real view on top of our testing view, and are able to write more readable tests.

```Swift
class FakeActivityOverlayViewController: UIViewControllerProtocol {

    var isPresented = false

    func presentOn(_ view: UIViewController, withMessage message: String, animated: Bool, completion: (() -> ())?) {
        isPresented = true
        completion?()
    }

    func dismiss(animated: Bool, completion: (() -> ())?) {
        isPresented = false
        completion?()
    }
}

```
#### *Case 2 - when existing signature is hard to fake out*
Sometimes it's hard to fake out the response due to strong constraints of the original method signature. For example:

```Swift
func execute(query: GTLRQueryProtocol, completionHandler handler: ((GTLRServiceTicket, Any, Error) -> Void)) -> GTLRServiceTicket
```

The function above requires a GTLRServiceTicket instance as return type, while this class doesn't have a default simple initializer. After digging into the GTLRDriveService source code for a while, we realized that we have known too much about the depending library.

To encounter that, we loosen the constraint of the signature in protocol, and extended the real service to implement the new signature and call the real function internally.

```Swift
protocol GTLRDriveServiceProtocol {
    func execute(query: GTLRQueryProtocol, completionHandler handler: ((GTLRServiceTicket?, Any?, Error?) -> Void)?) -> GTLRServiceTicket?
}

extension GTLRDriveService: GTLRDriveServiceProtocol {
    func execute(query: GTLRQueryProtocol, completionHandler handler: ((GTLRServiceTicket?, Any?, Error?) -> Void)?) -> GTLRServiceTicket? {
        return self.executeQuery(query) { (ticket, any, error) in
            handler?(ticket, any, error)
        }
    }
}
```

---
## UI Tests
To test our app can really load images from iPhone's camera roll and respond to user interaction correctly, we need to setup the device to have some and the same images with desired metadata to test against every time.

### Responding to system dialogs

The test setup doesn't sound hard right? But iOS will prompt user to allow access to photos and confirm the deletion of the photos. Here is the cheatsheet to solve that problem:

```Swift
class cameraRollSetupUITests: XCTestCase {

    var app:XCUIApplication!

    override func setUp() {
        super.setUp()
        continueAfterFailure = false

        app = XCUIApplication()
        app.launch()

        // Have this ready before system dialogs pop up
        addUIInterruptionMonitor(withDescription: "alert handler") { alert -> Bool in
            if (alert.buttons["OK"].exists) {
                alert.buttons["OK"].tap()
                // Prepare for the next system dialog
                RunLoop.current.run(until: Date(timeInterval: 1, since: Date()))
            }
            else if (alert.buttons["Delete"].exists) {
                alert.buttons["Delete"].tap()
            }
            else {
                XCTFail("We don't know what's going on!?")
            }

            return true
        }
    }

    func testDoesNothingJustWaitForSetupToFinish() {
        // System dialogs are in different thread, so give them some time to sync up
        RunLoop.current.run(until: Date(timeInterval: 2, since: Date()))
        // Activate the app to dismiss dialogs
        app.tap()
    }

}
```

### A separate app to setup device

Deleting and inserting images to camera roll is using iOS Photos library, so it has to be in production code instead of tests. To avoid putting testing purpose code in our add and maybe interrupting the normal flow, we decided to have a separate app to do all the setup and fill camera roll with this cute cat picture.

{{< responsive-figure src="/images/testing-in-swift/cat.jpg" class="center" >}}
```Swift
class CameraRollSetupViewController: UIViewController {

    override func viewDidAppear(_ animated: Bool) {

        super.viewDidAppear(animated)

        PHPhotoLibrary.requestAuthorization { (authorizationStatus) in
            self.cleanUpCameraRoll {
                self.addPhotosToCameraRoll()
            }
        }
    }

    private func cleanUpCameraRoll(_ completion: @escaping () -> ()) {
        PHPhotoLibrary.shared().performChanges({
            let assets = PHAsset.fetchAssets(with: nil)
            PHAssetChangeRequest.deleteAssets(assets)
        }, completionHandler: { success, error in
            completion()
        })
    }

    private func addPhotosToCameraRoll() {
        try? PHPhotoLibrary.shared().performChangesAndWait {
            self.generateAssetCreationRequest(atLocation: self.nonPivotalLocation, onDate: self.date2)

            for _ in (1...8) {
                self.generateAssetCreationRequest(atLocation: self.pivotalLocation, onDate: self.date1)
            }
        }
    }
}
```

Here we are, all set for UI tests. Let's go grab a beer and play more Ping Pong!
