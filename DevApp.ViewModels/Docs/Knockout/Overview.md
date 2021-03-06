## Overview

DotNetify makes it super easy to connect your [Knockout](http://knockoutjs.com) app to a cross-platform .NET back-end. It frees you up from having to write AJAX requests or Web APIs, and your app gets real-time two-way communication through WebSockets with [SignalR](https://docs.microsoft.com/en-us/aspnet/core/signalr/?view=aspnetcore-2.1) for free.

Whether in a brand-new ASP.NET Core project that runs on Windows, Linux, and Mac, or in an existing ASP.NET MVC codebase, you can use it right away with only a tiny footprint and get your app running in minutes! Need proof? Try this:

[inset]

#### Hello World

In the most basic form, you can just use dotNetify to quickly set up a two-way data binding between your Knockout-annotated HTML elements and a back-end C# class. Simply enclose it with an HTML element with __data-vm__ attribute that's set to the name of that C# class.

```html
<div data-vm="HelloWorld">
  <div data-bind="text: Greetings" />
</div>
```
```csharp
public class HelloWorld : BaseVM
{
   public string Greetings => "Hello World!";
}
```

Write a C# view model class that inherits from __BaseVM__ in your ASP.NET project, and add public properties for all the data your component will need. When the connection is established, the class instance will be serialized to JSON and sent to the browser. 

On the browser side, dotNetify will generate a Knockout view model from the JSON, and apply the binding to the HTML elements within the __data-vm__ scope.


#### Real-Time Push

With very little effort, you can make your app gets real-time data update from the back-end:

```html
<div data-vm="HelloWorld">
  <p data-bind="text: Greetings" />
  <p data-bind="text: ServerTime" />
</div>
```
```csharp
public class HelloWorld : BaseVM
{
   private Timer _timer;
   public string Greetings => "Hello World!";
   public DateTime ServerTime => DateTime.Now;

   public HelloWorld()
   {
      _timer = new Timer(state =>
      {
         Changed(nameof(ServerTime));
         PushUpdates();
      }, null, 0, 1000); // every 1000 ms.
   }
   public override void Dispose() => _timer.Dispose();
} 
```
[inset]
We added two new back-end APIs, __Changed__ that accepts the name of the property we want to update, and __PushUpdates__ to do the actual push of all changed properties to the front-end.

#### Server Update

At some point in your app, you probably want to send data back to the server to be persisted. Let's add to this example something to submit:

```html
<div data-vm="HelloWorld">
  <div data-bind="text: Greetings"></div>
  <input type="text" data-bind="value: firstName" />
  <input type="text" data-bind="value: lastName" />
  <button data-bind="vmCommand: submit">Submit</button>
</div>
<script>
  window.HelloWorld = {
    firstName: ko.observable(),
    lastName: ko.observable(),
    submit: function() { this.Submit({FirstName: this.firstName(), LastName: this.lastName()}) }
  }
</script>
```
```csharp
public class HelloWorld : BaseVM
{
   public class Person
   {
      public string FirstName { get; set; }
      public string LastName { get; set; }
   }

   public string Greetings { get; set; } = "Hello World!";
   public Action<Person> Submit => person =>
   {
      Greetings = $"Hello {person.FirstName} {person.LastName}!";
      Changed(nameof(Greetings));
   };
}
```
[inset]
We introduce a client-side view model (or code-behind) for our HTML in order to dome some client-side processing, which is in this case, to store user inputs to local Knockout observable objects.  

The submit button uses __vmCommand__ binding, which is a dotNetify's binding that wraps the Knockout _click_ binding.  When the user clicks it, it will execute the _submit_ function and pass the local observable values to the back-end instance's _Submit_ property.

On the back-end, we set up the Submit property to be of an Action delegate type since we don't expect it to send any value to the front-end. Setting _Submit_ observable on the front-end will cause that action to be invoked. After the action modifies the Greetings property, it marks it as changed so that the new text will get sent to the front-end.

Notice that we don't need to use __PushUpdates__ to get the new greetings sent. That's because dotNetify employs something similar to the request-response cycle, but in this case it's _action-reaction cycle_: the front-end initiates an action that mutates the state, the back-end processes the action and then sends back to the front-end any other states that mutate as the reaction to that action.


#### Object Lifetime

Those back-end classes as models, but in dotNetify's scheme of things, they are considered view models. Whereas a model represents your app's business domain, a view model is an abstraction of a view, which embodies data and behavior necessary for the view to fulfill its use case.

View-models gets their model data from the back-end service layer and shape them to meet the needs of the views. This enforces separation of concerns, where you can keep the views dealing only with UI presentation and leave all data-driven operations to the view models. It's great for testing too, because you can test your use case independent of any UI concerns.

View model objects stay alive on the back-end until the browser page is closed, reloaded, navigated away, or the session times out. 

#### API Essentials

##### Client APIs
To get started, you only need __data_vm__ attribute to define the scope and the C# view model name to connect to.  Use both standard Knockout data bindings to bind your HTML elements to C# class properties, and these custom data bindings:

- __vmCommand__: similar to Knockout _click_ binding, but bind to view model action, and can pass an argument. For example: `data-bind="vmCommand: {Submit: arg}"`.
- __vmOn__: bind to a client-side view model's function, which gets called on value update.  For example: `data-bind="vmOn: {ServerTime: doSomething}`.
- __vmOnce__: similar to _vmOn_, but only gets executed once.
- __vmItemKey__: used with _foreach_ binding to identify the key property name of the collection.  For example: `data-bind: foreach: Employee; vmItemKey: 'Id'`.

Use __data_master_vm__ attribute to nest multiple view models within a single master/parent view model.  A parent view model doesn't itself bind to HTML markups; its purpose is to manage the view models under its scope, like initializing them before being bound to their views, or facilitating event-based communication.  See _CompositeView_ for an example.

##### Server APIs
On the back-end, inherit from __BaseVM__ and use:
- __Changed__(_propName_)
- __PushUpdates__(): for real-time push.

When __data_master_vm__ is used, you can override these methods to hook to various lifecycles:
- __GetSubVM__(vmType): give the parent the opportunity to create its own instance of a child view model.
- __OnSubVMCreated__(vmObject): pass a newly created child view model instance to the parent.
- __OnSubVMDestroyed__(vmObject): pass the child view model instance that will soon be disposed to the parent.

Not essential, but these property accessor/mutator helper __Get/Set__ can make your code more concise:

```csharp
public string Greetings
{
   get => Get<string>();
   set => Set(value);
}
// Above property is equivalent to this:
private string _greetings;
public string Greetings
{
   get { return _greetings; }
   set { _greetings = value; Changed(nameof(Greetings)); }
}
```
