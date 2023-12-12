---
title: "Advanced text validation in UIKit with Combine."
date: 2023-08-24T18:11:49+02:00
author: "Vebjørn Daniloff"
description: "This article shows you how you can validate text with Combine in UIKit."
draft: false
tags: ["UIKit", "Combine", "Advanced", "UITextField"]
categories: ["UIKit", "Combine"]
series: ["messenger-clone"]
lightgallery: true
---

<!--more-->

## Introduction/Problem

In this article I want to show you how you can create a powerful text validation in UIKit with Combine. We're going to start with the nonsense view model you see in most validation with Combine tutorials and turn it to something amazing.

### Creating the view model

To show you why the approach most tutorials take on this subject is a bad approach -- I think we should actually program out their solution. Then I'll explain to you to the limitations I met with their solution in the [Messenger Clone application](https://github.com/vebbis321/MessengerClone) and then we turn the bad code into something amazing.

### CustomTextField

Before we start, we're going to use the `CustomTextField` from the previous article to perform our validation. If you didn't read that one, it's just a custom class for a text field that can create different instances of itself (name, email, password). So I'm going to use the completed [repository](https://github.com/vebbis321/CustomTextField) from the last article, as a starting point for this one.

## NameViewModel

Now, let's go back to the view model I promised to create. So, say that we wanted to validate that the user typed in a valid name in our custom text field.

```swift
final class NameViewModel {

}
```

Since we have a `NameViewModel`, lets rename the `TestViewController` to a `NameViewController` and pretend that it is our screen for a name validation.

```swift
class NameViewController: UIViewController {
    // ...
}
```

The `NameViewModel` will now have one publisher for the text that will be bound to the text field and another publisher that will represent the state of our text field.

```swift
final class NameViewModel {

    // first we declare a state representing the validation of our nameTextField
    enum NameState {
        case idle
        case error
        case success
    }

    @Published var firstName = "" // we will bind the text of our textField to this publisher
    @Published var state: NameState = .idle // we will subscribe to this state for updates on our textField
}
```

I think just leaving the error case in the enum without a feedback is foolish, so let's upgrade it:

```swift
enum NameState: Equatable {
    case idle
    case error(ErrorState)
    case success

    enum ErrorState {
        case empty
        case toShort
        case numbers
        case specialChars

        var description: String {
            switch self {
            case .empty:
                return "Field is empty."
            case .toShort:
                return "Name is to short"
            case .numbers:
                return "Name can't contain numbers."
            case .specialChars:
                return "Name can't contain special characters."
            }
        }
    }
}

```

Now we need to translate the text input into the state of our validation. We can do that by creating new publishers that uses our name publisher as a starting point, then they will perform some validation on the output and end the publisher with a boolean indicating whether or not the filter was successful.

```swift
var isEmtpy: AnyPublisher<Bool, Never> {
    $firstName
        .map { $0.isEmpty }
        .eraseToAnyPublisher()
        // this will "erase" the type or hide it
        // so it can capture what's actually important
        // which is the boolean
        // and return it wrapped in an AnyPublisher
}

var isToShort: AnyPublisher<Bool, Never> {
    $firstName
        .map { !($0.count >= 2) }
        .eraseToAnyPublisher()
}

var hasNumbers: AnyPublisher<Bool, Never> {
    $firstName
        .map { $0.hasNumbers() }
        .eraseToAnyPublisher()
}

var hasSpecialChars: AnyPublisher<Bool, Never> {
    $firstName
        .map { $0.hasSpecialCharacters() }
        .eraseToAnyPublisher()
}
// Observe how we are telling our code what we desire it to do,
// rather than the exact steps it should take to get there.
```

Now the code won't compile because `hasNumbers` and `hasSpecialChars` uses string methods that doesn't exist, so let's create them:

```swift
// String + Extensions
extension String {
    func hasNumbers() -> Bool {
        return stringFulfillsRegex(regex: ".*[0-9].*")
        // .* means "any character, any number of repetitions."
        // We need it to match the whole string, otherwise it will just return false,
        // ...even though it should return true
    }

    func hasSpecialCharacters() -> Bool {
        return stringFulfillsRegex(regex: ".*[^A-Za-z0-9].*") // ^ means not
    }

    private func stringFulfillsRegex(regex: String) -> Bool {
        let textTest = NSPredicate(format: "SELF MATCHES %@", regex)
        guard textTest.evaluate(with: self) else {
            return false
        }
        return true
    }
}
```

To combine all the states and translate them into the current state, let's create a new publisher that will start the validation.

```swift
 func startValidation() {
        guard state == .idle else { return }

         Publishers.CombineLatest4(
            isEmtpy,
            isToShort,
            hasNumbers,
            hasSpecialChars
        ).map {
            if $0.0 { return .error(.empty)  }
            if $0.1 { return .error(.toShort) }
            if $0.2 { return .error(.numbers) }
            if $0.3 { return .error(.specialChars) }
            return .success
        }
        .assign(to: &$state)
    }
```

What is nice about this approach is that we're receiving the current state of all our validations concurrently -- and we can use the tuple output that we get in return from them to create a error hierarchy.

Now you may ask, why a function? Often, we don't want to give error feedback to the user before a button is tapped. So, a method that we can trigger when it fits us is more suitable.

All we have to do now is bind the text field to the name publisher property in our code. So, let's extend the functionality of the text field so it can return us a publisher with the current text input:

```swift
// UITextField + Extensions
extension UITextField {

    // now lets create a publisher based on the notification that we observe...
    func textPublisher() -> AnyPublisher<String, Never> {
        NotificationCenter.default
            .publisher(for: UITextField.textDidChangeNotification, object:  self) // ...which is the textDidChangeNotification
            .compactMap { ($0.object as? UITextField)?.text } // we have our object with the text property
            .eraseToAnyPublisher()

    }
}
```

And bind the text field to the `firstName` publisher in our instance of the `NameViewModel`:

```swift
// NameViewController

// MARK: - Properties
private let nameViewModel = NameViewModel()
private var subscriptions = Set<AnyCancellable>()

// ...

// MARK: - bind
private func bind() {
    nameTextField.textField
        .textPublisher()
        .assign(to: \.firstName, on: nameViewModel)
        .store(in: &subscriptions)
}
```

Now we only need to start the validation and subscribe to its updates.

```swift
// MARK: - LifeCycle
override func viewDidLoad() {
    super.viewDidLoad()

    setup()
    bind()

    nameViewModel.startValidation()
    nameViewModel.$state
        .sink { [weak self] state in
            self?.nameTextField.validationStateChanged(state: state)
        }.store(in: &subscriptions)
}
```

Xcode will yell at us since we haven't implemented this method in our `CustomTextField`.

```swift
// MARK: - Methods
func validationStateChanged(state: NameViewModel.NameState) {
    switch state {
    case .idle:
        break
    case .error(let errorState):
        errorLabel.text = errorState.description
        errorLabel.isHidden = false
    case .success:
        errorLabel.text = nil
        errorLabel.isHidden = true
    }
 }
```

Nice! But our code still won't compile because we haven't implemented the `errorLabel` -- and we don't have a container that can show and hide our `errorLabel`. So let's implement the `errorLabel` first:

```swift
private lazy var errorLabel: UILabel = {
    let label = UILabel(frame: .zero)
    label.numberOfLines = 0
    label.textAlignment = .left
    label.textColor = .red
    label.font = .preferredFont(forTextStyle: .footnote)
    label.isHidden = true
    label.translatesAutoresizingMaskIntoConstraints = false
    return label
}()
```

### Hiding and showing the error label

So how do you show and hide views in UIKit? The best way is by using a `UIStackView`. Why? A `UIStackView` is much more flexible because of its automatic constraints when you hide and show views.

So, let's create a expanding vertical stack that will hold all our components and show/hide the `errorLabel`.

```swift
private lazy var expandingVstack: UIStackView = {
    let stack = UIStackView(frame: .zero)
    stack.axis = .vertical
    stack.spacing = 10
    stack.translatesAutoresizingMaskIntoConstraints = false
    return stack
}()
```

### Reconfigure the setup method

Now we should reconfigure our setup method by adding the old UI of the text input (`textFieldBackgroundView` with `textField`) and the `errorLabel` into the `expandingVstack`.

```swift
private func setup() {
    textField.placeholder = viewModel.placeholder
    textField.isSecureTextEntry = viewModel.isSecure
    textField.keyboardType = viewModel.keyboardType
    textField.autocapitalizationType = viewModel.autoCap

    textFieldBackgroundView.addSubview(textField)
    addSubview(expandingVstack)
    expandingVstack.addArrangedSubview(textFieldBackgroundView) // old text input UI
    expandingVstack.addArrangedSubview(errorLabel) // is hidden

    textFieldBackgroundView.widthAnchor.constraint(equalTo: widthAnchor).isActive = true // isn't required, but I like to keep it
    textFieldBackgroundView.topAnchor.constraint(equalTo: textField.topAnchor, constant: -9).isActive = true
    textFieldBackgroundView.bottomAnchor.constraint(equalTo: textField.bottomAnchor, constant: 9).isActive = true

    textField.leftAnchor.constraint(equalTo: textFieldBackgroundView.leftAnchor, constant: 6).isActive = true
    textField.rightAnchor.constraint(equalTo: textFieldBackgroundView.rightAnchor, constant: -6).isActive = true

    errorLabel.widthAnchor.constraint(equalTo: widthAnchor).isActive = true

    translatesAutoresizingMaskIntoConstraints = false
    // and the height ...
}
```

### Updating the height of the CustomTextField

Now our `expandingVstack` will act as the container for all the subviews in our `CustomTextField` class, which means that it should determine the height. Why? When it shows the validation label it will automatically update its constraints and grow in height. So let's change the `heightAnchor` at the bottom of the setup code:

```swift
heightAnchor.constraint(equalTo: expandingVstack.heightAnchor).isActive = true
```

Nice. Now run the code and see the `errorLabel` conditionally return us an error when the name input is invalid.

## So why is this a bad approach?

### Duplication

The first problem I faced by using a view model is that I had to duplicate the validation process for each screen. Why? Each validation was now in the scope of a single view model, and that view model was tightly coupled with the view controller for that screen -- `NameViewModel` with `NameViewController`, `EmailViewModel` with `EmailViewController` etc.

### Forcing a view model

The second problem was that the view model felt more forced, what do I mean by that:

- In production code I found the validation code in the class for the custom text fields. Which makes a lot more sense. Why? If it's outside our text field we have to implement it every single time we use the text field.
- The screens were fairly simple, and I didn't need another reference type for the validation of the text input. So I thought there must be a better way. And there is!

## Start with a protocol

Whenever you're developing classes for behavioral purposes, and you've created some behavior that's hard to replicate, think protocols. What do I mean by behavioral purposes -- think of what we are actually trying to achieve with our `NameViewModel` class. We're not trying to create a layer between us and a specific service class where we can transform the models into actual data for our view. No, we have only built a validation behavior. The same applies if we created a class for drawing behavior. We will again restrict ourselves to the scope of our class and the limitations of classes.

### Protocol-oriented programming

In [WWDC 2015 apple introduced Protocol-oriented programming](https://www.youtube.com/watch?v=p3zo4ptMBiQ), with the purpose of tackling problems like these. If you haven't seen that talk, I highly recommend that you do. I don't see this approach too often in applications, but when it works it's absolutely beautiful.

### Validatable

Like Apple says, don't start with a class, start with a protocol. So, let's define a blueprint of the expected behavioral for all validations. We want them to execute a function on a text publisher and use filters to return a state we can deal with. Since we're refactoring the `NameViewModel`, you can think of it as an abstraction of the behavior of our `startValidation` function.

```swift
protocol Validatable {
    func validate(publisher: AnyPublisher<String, Never>) -> AnyPublisher<ValidationState, Never>
}
```

Xcode will now yell at you, since we haven't defined the validation state. Let's transform the enum we had for the state of our text field into one that fits all our text field cases.

```swift
enum ValidationState: Equatable {
    case idle
    case error(ErrorState)
    case valid

    enum ErrorState: Equatable {
        case empty
        case invalidEmail
        case toShortPassword
        case passwordNeedsNum
        case passwordNeedsLetters
        case nameCantContainNumbers
        case nameCantContainSpecialChars
        case toShortName
        case custom(String) // if default descriptions doesn't fit

        var description: String {
            switch self {
            case .empty:
                return "Field is empty."
            case .invalidEmail:
                return "Invalid email."
            case .toShortPassword:
                return "Your password is to short."
            case .passwordNeedsNum:
                return "Your password doesn't contain any numbers."
            case .passwordNeedsLetters:
                return "Your password doesn't contain any letters."
            case .nameCantContainNumbers:
                return "Name can't contain numbers."
            case nameCantContainSpecialChars:
                return "Name can't contain special characters."
            case .toShortName:
                return "Your name can't be less than two characters."
            case .custom(let text):
                return text
            }
        }
    }
}
```

Nice. But whatever that implements this protocol needs help, because in the `startValidation` function we get help from our filters to determine the state. We can add default implementations to our protocol by extending it.

```swift
extension Validatable {
    // this is exactly the same as we had earlier,
    // but now we aren't restricted to $firstName publisher.

    func isEmtpy(publisher: AnyPublisher<String, Never>) -> AnyPublisher<Bool, Never> {
        publisher
            .map { $0.isEmpty }
            .eraseToAnyPublisher()
    }

    // remember to upgrade this one
    func isToShort(publisher: AnyPublisher<String, Never>, count: Int) -> AnyPublisher<Bool, Never> {
        publisher
            .map { !($0.count >= count) }
            .eraseToAnyPublisher()
    }

    func hasNumbers(publisher: AnyPublisher<String, Never>) -> AnyPublisher<Bool, Never> {
         publisher
            .map { $0.hasNumbers() }
            .eraseToAnyPublisher()
    }

    func hasSpecialChars(publisher: AnyPublisher<String, Never>) -> AnyPublisher<Bool, Never> {
        publisher
            .map { $0.hasSpecialCharacters() }
            .eraseToAnyPublisher()
    }
}
```

If you think about it, what we really want are value types, not reference types; we want to make plain copies of the validation behavior and then customize it.

```swift
struct NameValidation: Validatable {
    func validate(
        publisher: AnyPublisher<String, Never>
    ) -> AnyPublisher<ValidationState, Never> {

        Publishers.CombineLatest4(
            isEmpty(with: publisher),
            isToShort(with: publisher, count: 2),
            hasNumbers(with: publisher),
            hasSpecialChars(with: publisher)
        )
        .removeDuplicates(by: { prev, curr in
            prev == curr
        })
        .map { isEmpty, toShort, hasNumbers, hasSpecialChars in
            if isEmpty { return .error(.empty) }
            if toShort { return .error(.toShortName) }
            if hasNumbers { return .error(.nameCantContainNumbers) }
            if hasSpecialChars { return .error(.nameCantContainSpecialChars) }
            return .valid
        }
        .eraseToAnyPublisher()
    }
}
```

Now we can use this struct for the name validation and we can get rid of the bloated `NameViewModel`. But I promised that it can be dynamic and that we can replicate the behavior. So let's create a `EmailValidation` and `PasswordValidation` the same way.

```swift
struct EmailValidation: Validatable {
    func validate(
        publisher: AnyPublisher<String, Never>
    ) -> AnyPublisher<ValidationState, Never>{

        Publishers.CombineLatest(
            isEmpty(with: publisher),
            isEmail(with: publisher)
        )
        .removeDuplicates(by: { prev, curr in
            prev == curr
        })
        .map { isEmpty, isEmail in
            if isEmpty { return .error(.empty) }
            if !isEmail { return .error(.invalidEmail) }
            return .valid
        }
        .eraseToAnyPublisher()
    }
}


struct PasswordValidator: Validatable {
    func validate(
        publisher: AnyPublisher<String, Never>
    ) -> AnyPublisher<ValidationState, Never> {

        Publishers.CombineLatest4(
            isEmpty(with: publisher),
            isToShort(with: publisher, count: 6),
            hasNumbers(with: publisher),
            hasLetters(with: publisher)
        )
        .removeDuplicates(by: { prev, curr in
            prev == curr
        })
        .map { isEmpty, toShort, hasNumbers, hasLetters in
            if isEmpty { return .error(.empty) }
            if toShort { return .error(.toShortPassword) }
            if !hasNumbers { return .error(.passwordNeedsNum) }
            if !hasLetters { return .error(.passwordNeedsLetters) }
            return .valid
        }
        .eraseToAnyPublisher()
    }
}
```

Now Xcode will yell at us, since we miss the `isEmail` and `hasLetters` publisher.

```swift
// ... Validatable
func isEmail(with publisher: AnyPublisher<String, Never>) -> AnyPublisher<Bool, Never> {
   publisher
        .map { $0.isValidEmail() }
        .eraseToAnyPublisher()
}

func hasLetters(with publisher: AnyPublisher<String, Never>) -> AnyPublisher<Bool, Never> {
    publisher
        .map { $0.contains(where: { $0.isLetter }) }
        .eraseToAnyPublisher()
}
```

And implement the email regex:

```swift
func isValidEmail() -> Bool {
    return stringFulfillsRegex(regex: "[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}")
}
```

To make use of our validations, let's create a protocol, so that if a class conforms to the protocol, it now has the capability to use the methods of our custom `Validatable` objects.

```swift
protocol Validator {
    func validateText(
        validationType: ValidatorType,
        publisher: AnyPublisher<String, Never>
    ) -> AnyPublisher<ValidationState, Never>
}
```

We need an enum to determine the `Validatable` object of our choice.

```swift
enum ValidatorType {
    case email
    case password
    case name
}
```

### ValidationFactory

To handle it dynamically, we can create a factory and use our enum as the input type.

```swift
enum ValidatorFactory {
    static func validateForType(type: ValidatorType) -> Validatable {
        switch type {
        case .email:
            return EmailValidator()
        case .password:
            return PasswordValidator()
        case .name:
            return NameValidator()
        }
    }
}
```

To finish our publisher, we can create a default implementation of the `validateText` function so we don't have to implement it each time we conform to the `Validator` protocol.

```swift
extension Validator {
    func validateText(
        validationType: ValidatorType,
        publisher: AnyPublisher<String, Never>
    ) -> AnyPublisher<ValidationState, Never> {
        let validator = ValidatorFactory.validateForType(type: validationType)
        return validator.validate(publisher: publisher)
    }
}
```

Nice. Now every class that conforms to this protocol can perform any validation of our choice.

Let's refactor our `NameViewController`.

Remove the `NameViewModel` and clean up `viewDidLoad`.

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    setup()
    startValidation()
}
```

Let's try out our implementation:

```swift
// MARK: - Validator
extension NameViewController: Validator {
    private func startValidation() {
        validateText(
            validationType: .name,
            publisher: nameTextField.textField.textPublisher()
        )
        .sink { [weak self] state in
            self?.nameTextField.validationStateChanged(state: state)
        }.store(in: &subscriptions)

        // If text is empty.
        // Won't get notified until the text actually changes, so we toggle the method manually to
        // ...notify our publisher.
         NotificationCenter.default.post(
            name:UITextField.textDidChangeNotification, object: nameTextField.textField)
    }
}
```

Awesome! We now have the same validation behavior, but we can choose from multiple validations, and we have removed the unnecessary `NameViewModel` class. However I'm still not satisfied with our solution. Because now, we have to implement the validation every time we use the `CustomTextField`.

So, let me show you two amazing things that we can do to make our code beautiful.

## RawValue

First, move the conformance to the `Validator` protocol into the `CustomTextField` class instead. Why? As I mentioned earlier, we don't want to implement validation behavior every time we use a `CustomTextField`.

```swift
// MARK: - Validator
extension CustomTextField: Validator {
    func startValidation() {
        validateText(validationType: .name, publisher: textField.textPublisher())
            .sink { [weak self] state in
                self?.validationStateChanged(state: state)
            }.store(in: &subscriptions)

        NotificationCenter.default.post(
            name:UITextField.textDidChangeNotification, object: textField)
    }
}
```

And remove all the validation code in `viewDidLoad`.

Now since we know that the enum `CustomTextFieldType` has the same cases as our `ValidationType` enum, and it is big chance that it stays that way, we can actually transform one enum into the other. Then you might ask: "Why can't we use one for both?". Even though we duplicate the naming cases, I still think we have better code with two enums -- since one should belong to the `CustomTextField` class and one should be for the `ValidationType`. If we didn't use two enums the naming would be `CustomTextFieldValidationType`, which doesn't make sense. So let me show you how we we can do with the `rawValue`.

Assign a string `rawValue` to the `ValidationType`.

```swift
enum ValidatorType: String {
    case email
    case password
    case name
}
```

Now remove the hard coded `.name` in the `validaitonType` of `startValidation()` and replace it with this magic:

```swift
guard let validationType = ValidatorType(rawValue: viewModel.type.rawValue) else { return}

validateText(
    validationType: validationType,
    publisher: textField.textPublisher()
)
//...
```

Now, all of our text fields chooses their validations dynamically without we having to lift a finger. But before we jump to the final magic, we need to make some changes. We don't have a way of telling the parent class that the validation state of our text field has changed. So, let's move the validation state we had in the `NameViewModel` into our text field.

```swift
@Published var validationState: ValidationState = .idle
```

And assign our validation publisher to it. We change the `.sink` subscriber to a `.assign` subscriber, and we should perform a check to ensure that the state of the validation is .idle (we don't want to start the validation multiple times on the same text field).

```swift
 func startValidation() {
        guard validationState == .idle, let validationType = ValidatorType(rawValue: viewModel.type.rawValue) else { return}

        validateText(validationType: validationType, publisher: textField.textPublisher())
            .assign(to: &$validationState)

        NotificationCenter.default.post(
            name:UITextField.textDidChangeNotification, object: textField)
    }
```

Now we don't observe the state changes, so let's use our newly created publisher and listen to its outputs.

```swift
// MARK: - listen
private func listen() {
    $validationState
        .receive(on: DispatchQueue.main) // isn't required
        .sink { [weak self] state in
            self?.validationStateChanged(state: state)
        }.store(in: &subscriptions)
}
```

And start it in the init.

```swift
init(frame: CGRect = .zero, viewModel: ViewModel) {
    self.viewModel = viewModel
    super.init(frame: frame)
    setup()
    listen()
}
```

Now everything is nearly perfect, but we don't have any control on the pipeline of our text publisher. The validation should wait 0.2 seconds so it doesn't send different validation states while the user is typing, and remove the duplicates if the user types really fast back and forth. You could solve this by creating a computed property in our text field class:

```swift
private var customTextPublisher: AnyPublisher<String, Never> {
    textField.textPublisher()
        .removeDuplicates()
        .debounce(for: 0.2, scheduler: RunLoop.main)
        .eraseToAnyPublisher()
}
```

And update `startValidation`:

```swift
validateText(validationType: validationType, publisher: customTextPublisher)
```

Now this works, but here comes final magic.

## Combine operator

If you think about it what a validation should be is a combine operator. Why? We already have a text publisher, so why not perform the validation on the text publisher with a validation operator.

```swift
extension Publisher where Self.Output == String, Failure == Never {
    func validateText(validationType: ValidatorType) -> AnyPublisher<ValidationState, Never> {
        let validator = ValidatorFactory.validateForType(type: validationType)
        return validator.validate(publisher: self.eraseToAnyPublisher())
    }
}
```

Copy the part where we handle the pipeline in the `customTextPublisher` and delete the property. Move it into the `startValidation` function and replace the `validateText` call.

```swift
guard validationState == .idle, let validationType = ValidatorType(rawValue: viewModel.type.rawValue) else { return }

textField.textPublisher()
    .removeDuplicates()
    .debounce(for: 0.2, scheduler: RunLoop.main)
    .eraseToAnyPublisher()
```

Now remove `.eraseToAnyPublisher` and add the operator we just created.

```swift
textField.textPublisher()
        .removeDuplicates()
        .debounce(for: 0.2, scheduler: RunLoop.main)
        .validateText(validationType: validationType)
        .assign(to: &$validationState)
```

Run the code now, and see all our text fields being validated dynamically:

![Simulator screen](images/validation.gif)

Beautiful! To recap what the little beautiful code snippet above the simulator does:

- Controls the flow of our text before it's validated.
- Validates all our text fields dynamically without us lifting a finger.
- Validates our text with a publisher.
- Assigns the state of our validation to a state that reacts to the current state of our text field.

{{< admonition type=tip title="Validation operator" open=true >}}
We can now also use the validation operator on any publisher that publishes a string value.
{{< /admonition >}}

How cool is that! I hope you enjoyed this article as much as I did, and thank you for reading.

**Useful Links:**

- [If you prefer video](https://www.youtube.com/watch?v=xuviaFBOanE)
- [Code repo](https://github.com/vebbis321/Validation)
