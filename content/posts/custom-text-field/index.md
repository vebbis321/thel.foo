---
title: "How I handle advanced text input classes in UIKit"
date: 2023-07-15T17:28:41+02:00
author: "Vebjørn Daniloff"
description: "This article shows you how you can handle advanced classes with text input in
UIKit"
draft: false
tags: ["UIKit", "Advanced", "UITextField"]
categories: ["UIKit"]
series: ["messenger-clone"]
lightgallery: true
---

<!--more-->

## Introduction/Problem

In this blog post I want to show you how I deal with advanced textFields that are similar. What do I mean by that?

Let's say your client wanted you to make textFields for various inputs in an authentication flow -- A textField for the name input, the date input, the password input, etc. What is usually the case is that these textFields will now follow similar rules for their layout: backgroundColor, font, borderColor etc. like in the example below.

![Email and password](email-and-password-text-field.jpeg)
![Name](name-text-field.jpeg)


Also! I'm referring to all these components as a "textField", but they are either a UIStackView or UIView that acts as a container for the UITextField. They may also hold other components like a UIButton for clearing, maybe one of them has another UIView for the backgroundColor and so on - You get the idea, it would be hard to create an advanced textField with only the UITextField class itself.


Okay, so how do I deal with this problem? I would say that there are three approaches in my mind that you can take; one bad, one that is kind of bad and one that is great.

### Subclassing

The first and the worst (sick rhymes) approach would be to focus on subclassing. What do I mean by that?

Let's say you created a textField for the email input.  

```swift
class EmailTextField: UIView {
    // this would be the setup code; layout, constraints etc.
    func setup() {}
    // a specific method related to this class
    func doSomEmailMethod() {}
}
```

Now we can create a textField that represents a password input with the same layout by subclassing, but it also needs independent methods, and since it is a PasswordTextField it doesn't need the doSomEmailMethod. To cancel the doSomEmailMethod we just override it.

```swift
class PasswordTextField: EmailTextField {
    override func doSomeEmailMethod() {
        
    }
    
    func doSomePasswordMethod() {}
}
```

Why is this bad? The problem now is that every change in the EmailTextField will affect the PasswordTextField. This approach also gets incrementally worse. Why? What if we created a textField for the name input (NameTextField) and used the PasswordTextField as the superclass? Now the NameTextField is affected by the changes in PasswordTextField which is affected by the changes in the EmailTextField. This leads to no clear separation and is considered bad practice.

### Write them from scratch
If you wrote all the classes from scratch you will achieve clear separation between the classes and they will not depend on each other. So why is this bad? The reason is that we now have a lot of unnecessary code duplication. We know that all of these classes have similar setup, so there must be a better way.

### Use custom configuration for each textField
This is the best approach, we can create a dynamic configuration that will set up each of these textFields differently when we create an instance of them, but they will still use the same class. I use this in the Messenger clone application and this is common in production code.

## SwiftUI preview 

It would seem foolish in this case to build the project in the simulator for each little change that we would make in our class, so let's configure the SwiftUI preview to work with our UIKit code.

Since we want to view multiple instances of the same textField, a preview for a UIViewController that displays all our textFields would improve our workflow.

```swift
import SwiftUI

// 1) Extend the functionality of viewcontrollers so that they can be used in a SwiftUI Preview
extension UIViewController {
    // 2) We create a struct that conforms to UIViewControllerRepresentable.
    // Why? with this protocol our struct can act as a SwiftUI view wrapper for viewController.
    // Which means that we can pass in a viewController to this struct and it will wrap it into a SwiftUI view
    private struct Preview: UIViewControllerRepresentable {

        let viewController: UIViewController
        
        func makeUIViewController(context: Context) -> UIViewController {
            return viewController
        }
        
        func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
    }
    
    // this method now expects a SwiftUI view
    func showPreview() -> some View {
    
        Preview(viewController: self)
    }
}

```

## Creating our CustomTextField

```swift
final class CustomTextField: UIView {
    
    // MARK: - Components
    // we may refer to this class, outside of the CustomTextField class
    lazy var textField: UITextField = {
        let textField = UITextField(frame: .zero)
        textField.textColor = .label
        textField.translatesAutoresizingMaskIntoConstraints = false
        return textField
    }()
    
    private lazy var textFieldBackgroundView: UIView = {
        let view = UIView(frame: .zero)
        view.backgroundColor = .black.withAlphaComponent(0.125)
        view.layer.cornerRadius = 10
        view.layer.masksToBounds = true
        view.translatesAutoresizingMaskIntoConstraints = false
        return view
    }()
    
    // MARK: - LifeCycle
    override init(frame: CGRect = .zero) {
        super.init(frame: frame)
        setup()
    } 

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    // MARK: - setup
    private func setup() {
        
        addSubview(textFieldBackgroundView)
        textFieldBackgroundView.addSubview(textField)
        
        // use the intrinsic height of the UITextField to configure top and bottom for the textFieldBackgroundView
        textFieldBackgroundView.leftAnchor.constraint(equalTo: leftAnchor).isActive = true
        textFieldBackgroundView.rightAnchor.constraint(equalTo: rightAnchor).isActive = true
        textFieldBackgroundView.topAnchor.constraint(equalTo: textField.topAnchor, constant: -9).isActive = true
        textFieldBackgroundView.bottomAnchor.constraint(equalTo: textField.bottomAnchor, constant: 9).isActive = true
        
        textField.leftAnchor.constraint(equalTo: textFieldBackgroundView.leftAnchor, constant: 6).isActive = true
        textField.rightAnchor.constraint(equalTo: textFieldBackgroundView.rightAnchor, constant: -6).isActive = true
        
        translatesAutoresizingMaskIntoConstraints = false
        heightAnchor.constraint(equalTo: textFieldBackgroundView.heightAnchor).isActive = true
    }
}
```
*If you are curious on how I structure my Swift files; [How I structure my folders and Swift files.](https://google.com)*

Now we can preview our textFields in a viewController.

```swift
class TestViewController: UIViewController {
    
    private lazy var vStack: UIStackView = {
        let stack = UIStackView(frame: .zero)
        stack.axis = .vertical
        stack.spacing = 10
        stack.translatesAutoresizingMaskIntoConstraints = false
        return stack
    }()
    
    lazy var nameTextField = CustomTextField()
    lazy var emailTextField = CustomTextField()
    lazy var passwordTextField = CustomTextField()
                                            
                                               
    override func viewDidLoad() {
        super.viewDidLoad()
        
        view.backgroundColor = .white
        
        vStack.addArrangedSubview(nameTextField)
        vStack.addArrangedSubview(emailTextField)
        vStack.addArrangedSubview(passwordTextField)
        
        view.addSubview(vStack)
        
        vStack.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 50).isActive = true
        vStack.leftAnchor.constraint(equalTo: view.leftAnchor, constant: 10).isActive = true
        vStack.rightAnchor.constraint(equalTo: view.rightAnchor, constant: -10).isActive = true
        
    }
}
 
struct TestViewController_Previews: PreviewProvider {
    static var previews: some View {
        TestViewController().showPreview()
            .previewDevice("iPhone 14 Pro")
    }
}

```

If your run the code, you realize that all our instances of CustomTextField produce the same textField.
So how can we make our instances dynamic? One option is to create variables that resembles the properties of our textField:

```swift
lazy var textField: UITextField = {
    let textField = UITextField(frame: .zero)
    textField.textColor = .label
    textField.isSecureTextEntry = isSecure // here
    textField.placeholder = placeholder // and here
    textField.translatesAutoresizingMaskIntoConstraints = false
    return textField
}()

var isSecure: Bool
var placeholder: String

init(isSecure: Bool, placeholder: String) {
    self.isSecure = isSecure
    self.placeholder = placeholder
    super.init(frame: .zero)
}
```

This seems like a good solution, but as our textField grows in complexity we end up with to many variables and we also don't have an approach for handling individual methods. In other words; we need a light object with all our properties, but the logic of our properties/methods should be determined by some input case. Whether you realize it or not, we just described a struct that is configured by the input of an enum.

### Creating the custom conifguration

We create the enum that we pass to our object:
```swift
extension CustomTextField {
    enum Types: String {
        case name
        case email
        case password
        
        // just to simplify the placeholder
        func defaultPlaceholder() -> String {
            return "Enter your \(self.rawValue)..."
        }
    }
}
```

And then our object.

```swift
extension CustomTextField {
    struct ViewModel {
        
        var type: Types
        var placeholder: String?

        init(type: Types, placeholder: String? = nil) {
            self.type = type
            // custom placeholder or "" placeholder sticks, but no value return our default implementation
            // ternary operator, basically an if else statement in one line
            self.placeholder = placeholder == nil ? type.defaultPlaceholder() : placeholder
        }
               
        var isSecure: Bool {
            type == .password ? true : false 
        }
        
         var keyboardType: UIKeyboardType? {
            switch type {
            case .name, .password:
                return .default
            case .email:
                return .emailAddress
            }
        }
        
        var autoCap: UITextAutocapitalizationType {
            type == .name ? .words : .none
        }
    }
}
```
Now this is logical, scalable and you'll find this approach in production code (which I will show you later).
Let's implement this in our textField.

```swift
private var viewModel: ViewModel
    
init(viewModel: ViewModel) {
    self.viewModel = viewModel
    super.init(frame: .zero)
    setup()
 }

required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
}

private func setup() {
    textField.isSecureTextEntry = viewModel.isSecure
    textField.placeholder = viewModel.placeholder
    textField.keyboardType = viewModel.keyboardType
    textField.autocapitalizationType = viewModel.autoCap   
    // ...
}
```

And the init of the textFields:

```swift
lazy var nameTextField = CustomTextField(viewModel: .init(type: .name, placeholder: "Custom placeholder"))
lazy var emailTextField = CustomTextField(viewModel: .init(type: .email))
lazy var passwordTextField = CustomTextField(viewModel: .init(type: .password))
```

Nice! Now we have three different textFields instances, but they are created from the same class. Now you might ask: "But, what about states? How can I show a border if one of the textFields is active?". If you think about it states are a list of possibilities, just like an enum;

```swift
extension CustomTextField {
    enum FocusState {
        case active
        case inActive
        
        var borderColor: CGColor? {
            self == .inActive ? .none : UIColor.black.withAlphaComponent(0.6).cgColor
        }
        
        var borderWidth: CGFloat {
            self == .inActive ? 0 : 1
        }
    }
}
```

And in the CustomTextField:
```swift
private var focusState: FocusState = .inActive
```

### The Delegate

Okay so where do we set the focusState? The state has to be set when the textField editing did begin  (textFieldDidBeginEditing), and it has to be disabled when editing has ended (textFieldDidEndEditing). These two methods are owned by the UITextField, but they can be used by any class of our choice. To tell the UITextField that our CustomTextField can use these methods we have to assign our CustomTextField as the UITextField's delegate.

```swift
textField.delegate = self // self is CustomTextField
```

Now XCode will yell at you because our CustomTextField isn't capable of handling the delegate methods of a UITextField. So we need it to conform to the UITextFieldDelegate.

```swift
extension CustomTextField: UITextFieldDelegate {
    
}
```

Now handle the focus with the two methods mentioned.

```swift
extension CustomTextField: UITextFieldDelegate {
    func textFieldDidBeginEditing(_ textField: UITextField) {
        focusState = .active
    }
    
    func textFieldDidEndEditing(_ textField: UITextField) {
        focusState = .inActive
    }
}
```

Nice. Now the UITextField will notify our CustomTextField class every time the textField is in focus and when editing stopped.

### Handling state 

But now nothing happens when the state changes. We need some method that can respond in relation to changes in the focusState. So how can we do that?. If we attach a didSet property observer to the focusState it will execute code whenever the property changes, which is exactly what we want, because we want to trigger a method for handling new changes when our state changes.

```swift
private var focusState: FocusState = .inActive {
    // The didSet will execute code when a property has just been set.
    didSet {
        focusStateChanged()
    }
}

// ...

// MARK: - Private Methods
private func focusStateChanged() { 
    textFieldBackgroundView.layer.borderColor = focusState.borderColor
    textFieldBackgroundView.layer.borderWidth = focusState.borderWidth
}
```

Now run the code and see our textField responding to the focusState.

-Finn.no

