---
title: "Advanced text validation in UIKit with Combine."
date: 2023-07-19T18:11:49+02:00
author: "Vebjørn Daniloff"
description: "This article shows you how you can validate text with Combine in
UIKit." 
draft: true
tags: ["UIKit", "Combine", "Advanced", "UITextField"]
categories: ["UIKit", "Combine"]
series: ["messenger-clone"]
lightgallery: true
---

<!--more-->

## Introduction/Problem

Hey, and welcome back to the channel. In this video I want to show you how you can create a powerful text validation in UIKit with Combine. We are going to turn the nonsense ViewModel you see in most validation with Combine tutorials and turn it to something amazing.

### Creating the ViewModel
To show you why the approach most tutorials take on this subject is a bad approach -- I think we should actually program out their solution. Then I'll explain to you to the limitations I met with their solution in the Messenger clone application and then we turn the bad code into something amazing.

### CustomTextField
But before we start, we're going to use the CustomTextField from the previous video to perform our validation. If you didn't watch that one, it's just a custom class for a textField that can create different instances of itself (name, email, password). So I'm going to use the repository from the last video as a starting point for this one and I'll leave a link to it in the description below.

## NameViewModel
Okay, cool. Now lets get back to the ViewModel I promised to create. So let's say that we wanted to validate that the user typed in a valid name in our custom textField. 

```swift
final class NameViewModel {

}
```

Now since we have a NameViewModel, let's rename the TestViewController to a NameViewController and pretend that it is our screen for a name validaiton.

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

*Explain publisher and subsciber by your channel.*

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
// This is now declarative programming, rather than imperative programming.
// We are telling our code what we desire it to do,
// rather than the exact steps it should take to get there.
// Which is exactly why I love using Combine.
```

Now the code won't compile because the two last uses string methods that doesn't exist, so let's create them:

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

To combine all the states and translate them into the current state, lets create a new publisher that will start the validation

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

What is nice about this approach is that we're receiving the current state of all our validations concurrently and we can use the tuple output that we get in return from them to create a error hierarchy.  

Now you may ask, why a function? Often we don't want to give error feedback to the user before a button is tapped. So a method that we can trigger when it fits us is more suitable.

All we have to do now is bind the textField to the name publisher property in our code. Let's extend the functionality of the textField so it can return us a publisher with the current text input:

```swift
// UITextField + Extensions
extension UITextField {
    func textPublisher() -> AnyPublisher<String, Never> {
        NotificationCenter.default
            .publisher(for: UITextField.textDidChangeNotification, object:  self)
            .compactMap { ($0.object as? UITextField)?.text }
            .eraseToAnyPublisher()
    }
}
```

And bind the textField to the firstName publisher in the nameViewModel:

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

Now we only need to start the validation and subscribe to it's updates.
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

Now XCode will yell at us since we haven't implemented this method in our CustomTextField.

## CustomTextField
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

Now our code won't compile because we haven't implemented the errorLabel and we don't have an container that can take care of showing and hiding our errorLabel. So let's implement the errorLabel:

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
So how do you show and hide views in UIKit? The best way is by using a UIStackView. Why? A stackView is much more flexible because it will offer you automatic constraints when you hide and show the views. This also applies if the process of hiding/showing views are done with an animation - which makes the stackView perfect for the job.

```swift
private lazy var expandingVstack: UIStackView = {
    let stack = UIStackView(frame: .zero)
    stack.axis = .vertical
    stack.spacing = 10
    stack.translatesAutoresizingMaskIntoConstraints = false 
    return stack
}()
```

We need to show some sort of feedback when the validation state returns an error. 

### Creating a UILabel for the feedback
We make a simple label in the CustomTextField class that should show a feedback conditionally by the validation state (we get to the state later).

```swift
private lazy var errorLabel: UILabel = {
    let label = UILabel(frame: .zero)
    label.numberOfLines = 0 // make it multiline
    label.textAlignment = .left
    label.textColor = .red
    label.font = .preferredFont(forTextStyle: .footnote)
    label.isHidden = true // hiding it
    label.translatesAutoresizingMaskIntoConstraints = false
    return label
}()
```

### Reconfigure the setup method
Now we should reconfigure our setup method by adding the old text input UI (textFieldBackgroundView with textField) and the errorLabel in the stackView. Since the errorLabel should be constrained relative to the old text input UI.

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

Now our stackView will act as the container for all the subviews in our CustomTextField class, which means that it should determine the height. Why? When it shows the validation label it will automatically update its constraints and grow in height. So let's change the heightAnchor at the bottom of the setup code:

```swift
heightAnchor.constraint(equalTo: expandingVstack.heightAnchor).isActive = true
```

Nice. Now run the code and see the errorLabel conditionally return us an error when the name input is invalid.

## Why is this a bad approach?

### Duplication
The first problem I faced by following most of the tutorials/articles online was that by using a ViewModel I had to duplicate the validation process for each screen. Why? Each validation was now in the scope of a single ViewModel, and that ViewModel was tightly coupled with the ViewController for that screen -- NameViewModel with NameViewController, EmailViewModel with EmailViewController etc.

### Forcing a ViewModel
The second problem was that the ViewModel felt more forced; the screens
were fairly simple, and I didn't need another reference type for the validation of the text input. So I thought there must be a better way. And there is!

## Start with a protocol
Whenever you're developing classes for behavioral purposes and you've created some behavior that's hard to replicate, think protocols. What do I mean by behavioral purposes, think of what we are actually trying to achieve with our NameViewModel class. We're not trying to create a layer between us and a specific service class, where we can transform the models into actual data for our view, no, we have just created a validation behavior. The same goes for if we created a class for drawing behavior. We will again restrict ourselves to the scope of our class and the limitations of classes. 

### Protocol-oriented programming
In WWDC 2015 apple introduced Protocol-oriented programming, with the purpose of tackling problems like these. If you haven't seen that talk, I highly recommend that you do -- I'll leave a link to the video. I don't see this approach too often in applications, but when it works it's absolutely beautiful. 

### Validatable
Like Apple says, don't start with a class, start with a protocol. So, let's define a blueprint of the expected behavioral for all validations. We want them to execute a function on a text publisher and use filters to return a state that we can deal with. Since we're refactoring the NameViewModel, you think of it as an abstraction of the behavior of our startValidation function.

```swift
protocol Validatable {
    func validate(publisher: AnyPublisher<String, Never>) -> AnyPublisher<ValidationState, Never>
}
```

XCode will now yell at you, since we haven't defined the validation state. Let's transform the enum we had for the state of our textField into one that fits all our textField cases.

```swift
enum ValidationState {
    case idle
    case error(ErrorState)
    case valid

    enum ErrorState {
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

Nice. But whatever that implements this protocol needs help, because in the startValidation function we get help from our filters to determine the state. We can add default implementations to our protocol by extending it. 

```swift
extension Validatable {
    // this is exactly the same as we had earlier, but now we aren't restricted to $firstName publisher.

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
            prev.0 == curr.0 && prev.1 == curr.1 && prev.2 == curr.2 && prev.3 == curr.3
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

Now we can use this struct for the name validation and we got rid of the bloated NameViewModel. But I promised that it can be dynamic and that we can replicate the behavior. So let's create a EmailValidation and PasswordValidation the same way.

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
            prev.0 == curr.0 && prev.1 == curr.1
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
            prev.0 == curr.0 && prev.1 == curr.1 && prev.2 == curr.2 && prev.3 == curr.3
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

Now XCode will yell at us since, we miss the isEmail and hasLetters publisher.

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

To make use of our validations, let's create a protocol, so that if a class conforms to the protocol, it now has the capability to use the methods of our custom Validatable objects. Like if a class conform to the UITextFieldDelegate it now have to capability to use the methods of the UITextFieldDelegate.

```swift
protocol Validator {
    func validateText(
        validationType: ValidatorType,
        publisher: AnyPublisher<String, Never>
    ) -> AnyPublisher<ValidationState, Never>
}
```

We need an enum to determine the Validatable object of our choice. 

```swift
enum ValidatorType {
    case email
    case password
    case name
}
```

### ValidationFactory
Now to determine what validaiton we are dealing with in a dynamic way we can create a factory and use our enum as the input type.

```swift
enum ValidatorFactory {
    static func validateForType(type: ValidatorType) -> CustomValidation {
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

Now to finish our publisher, we can create a default implementation of the validateText function so we don't have to implement each time we conform to the Validator protocol.

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

Amazing now every class that conforms to this protocol can perform any validation of our choice.

Let's refactor our NameViewController.

Remove the NameViewModel and clean up viewDidLoad.

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

Awesome! We now have the same validation behavior, but we can choose from multiple validations and we have got rid of the unnecessary NameViewModel class. However I'm actually still not satisfied with our solution -- Because now we have to implement the validation every time we use the CustomTextField.

So, let me show you two amazing things that we can do to make our code beautiful.

## RawValue

Now, move the conformance to the Validator protocol into the CustomTextField class instead. Why? The're is no reason to keep the validation that will be applied to the CustomTextField outside of CustomTextField, I have also seen this in production code. 

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

And remove all the validaion code in viewDidLoad.

Now Since we know that the enum case for the CustomTextFieldType is the same cases as our ValidationType, and it is big chance that it stays that way, we can actually transform one enum into the other. Then you might ask: Why don't just use one for both? Even though we duplicate the naming cases, I still think we have better code with two enums, since one should belong to the CustomTextField class and one should be for the ValidationType, if we didn't do that the naming would be CustomTextFieldValidationType which doesn't make sense. So let me show you what we can do with the rawValue.

Assign a string rawValue to the ValidationType.

```swift
enum ValidatorType: String {
    case email
    case password
    case name
}
```

Now remove the hardcoded .name in the validaitonType of startValidation() and replace it with this magic:

```swift
guard let validationType = ValidatorType(rawValue: viewModel.type.rawValue) else { return}
        
validateText(
    validationType: validationType,
    publisher: textField.textPublisher()
)
//...
```

Now all of our textFields chooses their validations dynamically without we having to lift a finger. But before we jump to the final magic we need to make some changes. We need to have a way of telling the parent class of our CustomTextField that the validation state has changed. So let's move to validation state that we had in the NameViewModel to CustomTextField.

```swift
@Published var validationState: ValidationState = .idle
```

And assign our validation publisher to it. We change the sink subcriber to a assign subscriber instead and we should do a check that the state of the validaiton is .idle.

```swift
 func startValidation() {
        guard validationState == .idle, let validationType = ValidatorType(rawValue: viewModel.type.rawValue) else { return}
        
        validateText(validationType: validationType, publisher: textField.textPublisher())
            .assign(to: &$validationState)
        
        NotificationCenter.default.post(
            name:UITextField.textDidChangeNotification, object: textField)
    }
```
Now we don't handle that the state has changed, so let's use our newly created publisher to set up a listener that observe the state changes.

```swift
// MARK: - listen
private func listen() {
    $validationState
        .receive(on: DispatchQueue.main)
        .sink { [weak self] state in
            self?.validationStateChanged(state: state)
        }.store(in: &subscriptions)
}
```

And of course, in our init:

```swift
init(frame: CGRect = .zero, viewModel: ViewModel) {
    self.viewModel = viewModel
    super.init(frame: frame)
    setup()
    listen()
}

```

Now everything is nearly perfect, but we don't have any control on our textPublisher. The validation should wait 0.2 seconds so it doesn't send different validaiton states while the user is typing and we also doesn't remove the duplicates. You could solve this by creating a seperate computed property like we did before:

```swift
private var customTextPublisher: AnyPublisher<String, Never> {
    textField.textPublisher()
        .removeDuplicates()
        .debounce(for: 0.2, scheduler: RunLoop.main)
        .eraseToAnyPublisher()
}
```

And update startValidation:

```swift
validateText(validationType: validationType, publisher: customTextPublisher)
```

Now this works, but here comes final magic.

## Combine operator

If you think about it what a validation should be is a combine operator. We have the textPublisher why not perform the validation on the textPublisher with an validation operator.

```swift
extension Publisher where Self.Output == String, Failure == Never {
    func validateText(validationType: ValidatorType) -> AnyPublisher<ValidationState, Never> {
        let validator = ValidatorFactory.validateForType(type: validationType)
        return validator.validate(publisher: self.eraseToAnyPublisher())
    }
}
```

Copy the part where we handle the pipeline in the customTextPublisher and delete the property. Move to the startValidation function and add it instead of the function.

```swift
guard validationState == .idle, let validationType = ValidatorType(rawValue: viewModel.type.rawValue) else { return }
        
textField.textPublisher()
    .removeDuplicates()
    .debounce(for: 0.2, scheduler: RunLoop.main)
    .eraseToAnyPublisher()
```

Now remove .eraseToAnyPublisher and add the operator we just created.

```swift
textField.textPublisher()
        .removeDuplicates()
        .debounce(for: 0.2, scheduler: RunLoop.main)
        .validateText(validationType: validationType)
        .assign(to: &$validationState)
```
