---
title: A Blazor Validation Control
oneliner: A Blazor validation control to manage and monitor validation state in a form.
precis: The second article in a series looking at how to build Blazor edit forms/controls with state management, validation and form locking.  This article focuses on validation state.
date: 2021-03-11
published: 2021-03-16
---

# The Blazor ValidationFormState Control

published: 2021-03-11
Last Update: 2021-03-16

## Overview

This is the second in a series of articles describing a set of useful Blazor Edit controls that solve some of the current shortcomings in the out-of-the-box edit experience without the need to buy expensive toolkits.

This article covers how form validation works and shows how to build a relatively simple but fully featured validation system from scratch. Once the basic structure and classes are defined, it's easy to write additional validation chain methods for any new validation requirement or validator for a custom class.

![EditForm](https://shauncurtis.github.io/siteimages/Articles/Editor-Controls/ValidationFormState.png)

## Code and Examples

The repository contains a project that implements the controls for all the articles in this series.  You can find it [here](https://github.com/ShaunCurtis/Blazor.Database).

The example site is here [https://cec-blazor-database.azurewebsites.net/](https://cec-blazor-database.azurewebsites.net/).

The example form described at this end of this article can be seen at [https://cec-blazor-database.azurewebsites.net//validationeditor](https://cec-blazor-database.azurewebsites.net//validationeditor).

> The Repo is a Work In Progress for future articles so will change and develop.

## The Blazor Edit Setting

To begin lets look at the out-of-the-box form controls and how validation works.  A classic form looks something like this:

```html
<EditForm Model="@exampleModel" OnValidSubmit="@HandleValidSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <InputText id="name" @bind-Value="exampleModel.Name" />
    <ValidationMessage For="@(() => exampleModel.Name)" />

    <button type="submit">Submit</button>
</EditForm>
```

The first article describes the basic interacts of `EditForm` and `EditContext` so we'll skip that and concentrate on the validation process.

When the user clicks on the *Submit* button, `EditForm` either:
1. If a delegate is registered with `OnSubmit`, it triggers it and ignores validation.
2. If there's no `OnSubmit` delegate, it calls `EditContext.Validate`.  Depending on the result either triggers `OnValidSubmit` or `OnInvalidSubmit`.

`EditContext.Validate` checks if there's a delagate registered for `OnValidationRequested` and if so runs it synchronously.  Once complete it checks if there are any messages in the `ValidationMessageStore`.  If it's empty, the form passes validation and `OnValidSubmit` is invoked, otherwise `OnInvalidSubmit` is invoked.  

 A Validator is a form component with no emitted markup.  It's placed within `EditForm` and captures the cascaded `EditContext`.  On initialization it registers an event handler with `EditContext.OnValidationRequested` to trigger validation.  On validation, the validator does whatever it's coded to do, logs validation failure messages to the `EditContext` `ValidationMessageStore` and finally calls `EditContext.NotifyValidationStateChanged` which triggers `EditContext.OnValidationStateChanged`.

#### Validation Controls

Controls such as `ValidationMessage` and `ValidationSummary` capture the cascaded `EditContext` and register event handlers on `EditContext.OnValidationStateChanged`.  When triggered they check for any relevant messages and display them.

In the form shown above `<DataAnnotationsValidator />` adds the `DataAnnotationsValidator` control to the form.  This hooks in as described above, and uses the custom attribute annotations on the model class to validate values. 

## Validator

`Validator` is the base validator class.  It's declared abstract and uses generics.  Validators work on a chaining principle.  The base class contains all the common boilerplate code.

1. The first call is on an extension method defined for the object type to be validated.  Each object type needs it's own extension method to call it's specific validator.  This extension method returns the appropriate validator for the object type.
2. Once you have the validator instance you can chain as many validation methods as you wish together.  Each is coded to run it's validation test, log any specific messages to the validator, trigger the trip if necessary, and return the validator instance.
3. Validation finishes by calling `Validate`, which trips the passed tripwire if necessary, and log all the validation messages to the `ValidationMessageStore`.

The `Validator` Properties/Fields are:
```
public bool IsValid => !Trip;
public List<string> Messages { get; } = new List<string>();
protected bool Trip { get; set; } = false;
protected string FieldName { get; set; }
protected T Value { get; set; }
protected string DefaultMessage { get; set; } = "The value failed validation";
protected ValidationMessageStore ValidationMessageStore { get; set; }
protected object Model { get; set; }
```

The constructor populates the validator
```csharp
public Validator(T value, string fieldName, object model, ValidationMessageStore validationMessageStore, string message)
{
    this.FieldName = fieldName;
    this.Value = value;
    this.Model = model;
    this.ValidationMessageStore = validationMessageStore;
    this.DefaultMessage = string.IsNullOrWhiteSpace(message) ? this.DefaultMessage : message;
}
```
There are two `Validate` methods: a public method for external usage and a protected one for specific validators to override.
```csharp

public virtual bool Validate(ref bool tripwire, string fieldname, string message = null)
{
    if (string.IsNullOrEmpty(fieldname) || this.FieldName.Equals(fieldname))
    {
        this.Validate(message);
        if (!this.IsValid)
            tripwire = true;
    }
    else this.Trip = false;
    return this.IsValid;
}
```
```csharp
protected virtual bool Validate(string message = null)
{
    if (!this.IsValid)
    {
        message ??= this.DefaultMessage;
        // Check if we've logged specific messages.  If not add the default message
        if (this.Messages.Count == 0) Messages.Add(message);
        //set up a FieldIdentifier and add the message to the Edit Context ValidationMessageStore
        var fi = new FieldIdentifier(this.Model, this.FieldName);
        this.ValidationMessageStore.Add(fi, this.Messages);
    }
    return this.IsValid;
}

protected void LogMessage(string message)
{
    if (!string.IsNullOrWhiteSpace(message)) Messages.Add(message);
}
```

#### StringValidator

Let's look at `StringValidator` as an example implementation of a validator.  The full set of validators is in the Repo.  There are two classes:

1. `StringValidatorExtensions` is a static class declaring as an extension method to `string`.
2. `StringValidator` is a implementation of `Validator` specifically for strings.

`StringValidatorExtensions` declares a single static extension method `Validation` for `string`.  It returns a `StringValidator` instance.  Call `StringValidator` on any string to initialise a validation chain. 

```csharp
public static class StringValidatorExtensions
{
    public static StringValidator Validation(this string value, string fieldName, object model, ValidationMessageStore validationMessageStore, string message = null)
    {
        var validation = new StringValidator(value, fieldName, model, validationMessageStore, message);
        return validation;
    }
}
```
`StringValidator` inherits from `Validator` and declares the specific validation chain methods for strings.  Each runs it's test.  If validation fails it logs any provided message to the message store and trips the tripwire.  Finally it returns `this`.  For strings, we have two length methods and a RegEx method to cover most circumstances. 

```csharp
public class StringValidator : Validator<string>
{
    public StringValidator(string value, string fieldName, object model, ValidationMessageStore validationMessageStore, string message) : base(value, fieldName, model, validationMessageStore, message) { }

    /// Check of the string is longer than test
    public StringValidator LongerThan(int test, string message = null)
    {
        if (string.IsNullOrEmpty(this.Value) || !(this.Value.Length > test))
        {
            Trip = true;
            LogMessage(message);
        }
        return this;
    }

    /// Check if the string is shorter than
    public StringValidator ShorterThan(int test, string message = null)
    {
            
        if (string.IsNullOrEmpty(this.Value) || !(this.Value.Length < test))
        {
            Trip = true;
            LogMessage(message);
        }
        return this;
    }

    /// Check if the string is matches a RegEx pattern
    public StringValidator Matches(string pattern, string message = null)
    {
        if (!string.IsNullOrWhiteSpace(this.Value))
        {
            var match = Regex.Match(this.Value, pattern);
            if (match.Success && match.Value.Equals(this.Value)) return this;
        }
        this.Trip = true;
        LogMessage(message);
        return this;
    }
}
```

#### IValidation

The  `IValidation` interface looks like this.  It simply defines a `Validate` method.

```csharp
public interface IValidation
{
    public bool Validate(ValidationMessageStore validationMessageStore, string fieldname, object model = null);
}
```

### WeatherForecast

`WeatherForecast` is a typical data class. 

1. It implements `IValidation` so the control can run validation. 
1. Each field is declared as a property with default values.
2. It implements `IValidation.Validate` which calls three validations.

Each validation:

1. Calls the `Validation` extension method on the type.
2. Calls one or more validation chain methods.
3. Calls `Validate` to log any validation messages to the `ValidationMessageStore` on `EditContext` and if necessary trip the tripwire.

```csharp
public class WeatherForecast : IValidation
{
    public int ID { get; set; } = -1;
    public DateTime Date { get; set; } = DateTime.Now;
    public int TemperatureC { get; set; } = 0;
    [NotMapped] public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
    public string Summary { get; set; } = string.Empty;

    public bool Validate(ValidationMessageStore validationMessageStore, string fieldname, object model = null)
    {
        model = model ?? this;
        bool trip = false;

        this.Summary.Validation("Summary", model, validationMessageStore)
            .LongerThan(2, "Your description needs to be a little longer! 3 letters minimum")
            .Validate(ref trip, fieldname);

        this.Date.Validation("Date", model, validationMessageStore)
            .NotDefault("You must select a date")
            .LessThan(DateTime.Now.AddMonths(1), true, "Date can only be up to 1 month ahead")
            .Validate(ref trip, fieldname);

        this.TemperatureC.Validation("TemperatureC", model, validationMessageStore)
            .LessThan(70, "The temperature must be less than 70C")
            .GreaterThan(-60, "The temperature must be greater than -60C")
            .Validate(ref trip, fieldname);

        return !trip;
    }

}
```

## ValidationFormState Control

The `ValidationFormState` control replaces the basic Validator provided with Blazor.

1. It captures the cascaded `EditContext`.
2. `DoValidationOnFieldChange` controls field level validation. if true it validates a field when a user exits the field.  if false it only responds to form level validation requests through `EditContext`.
3. `ValidStateChanged` is a callback for the parent to attach an event handler if required.
4. `IsValid` is a public readonly property exposing the current validation state.  It checks if `EditContext` has any validation messages.
5. `ValidationMessageStore` is the `EditContext`'s `ValidationMessageStore`.
6. `validating` is a boolean field to ensure we don't stack validations.
7. `disposedValue` is part of the `IDisposable` implementation.

```csharp
    [CascadingParameter] public EditContext EditContext { get; set; }
    [Parameter] public bool DoValidationOnFieldChange { get; set; } = true;
    [Parameter] public EventCallback<bool> ValidStateChanged { get; set; }
    public bool IsValid => !EditContext?.GetValidationMessages().Any() ?? true;

    private ValidationMessageStore validationMessageStore;
    private bool validating = false;
    private bool disposedValue;
```

When the component initializes it gets the `ValidationMessageStore` from `EditContext`.  It checks if it's running field level validation, and if so registers `FieldChanged` with `EditContext.OnFieldChanged` event. Finally it registers `ValidationRequested` with `EditContext.OnValidationRequested`.

```csharp
    protected override Task OnInitializedAsync()
    {
        Debug.Assert(this.EditContext != null);

        if (this.EditContext != null)
        {
            // Get the Validation Message Store from the EditContext
            this.validationMessageStore = new ValidationMessageStore(this.EditContext);
            // Wires up to the EditContext OnFieldChanged event
            if (this.DoValidationOnFieldChange)
                this.EditContext.OnFieldChanged += FieldChanged;
            // Wires up to the Editcontext OnValidationRequested event
            this.EditContext.OnValidationRequested += ValidationRequested;
        }
        return Task.CompletedTask;
    }
```

The two event handlers call `Validate`, one with and one without the field name.

```csharp
private void FieldChanged(object sender, FieldChangedEventArgs e)
    => this.Validate(e.FieldIdentifier.FieldName);

private void ValidationRequested(object sender, ValidationRequestedEventArgs e)
    => this.Validate();
```
The comments within `Validate` explain what it's doing.  It casts the `Model` as an IValidator and check if it's valid.  if so it calls the `Validate` method on the interface. We've seen *model*.`Validate` in the `WesatherForecast` data class.  When it passes a `fieldname` to `Validate` it only clears any validation messages for that specific `fieldname`.

```csharp
private void Validate(string fieldname = null)
{
    // Checks to see if the Model implements IValidation
    var validator = this.EditContext.Model as IValidation;
    if (validator != null || !this.validating)
    {
        this.validating = true;
        // Check if we are doing a field level or form level validation
        // Form level - clear all validation messages
        // Field level - clear any field specific validation messages
        if (string.IsNullOrEmpty(fieldname))
            this.validationMessageStore.Clear();
        else
            validationMessageStore.Clear(new FieldIdentifier(this.EditContext.Model, fieldname));
        // Run the IValidation interface Validate method
        validator.Validate(validationMessageStore, fieldname, this.EditContext.Model);
        // Notify the EditContext that the Validation State has changed - 
        // This precipitates a OnValidationStateChanged event which the validation message controls are all plugged into
        this.EditContext.NotifyValidationStateChanged();
        // Invoke ValidationStateChanged
        this.ValidStateChanged.InvokeAsync(this.IsValid);
        this.validating = false;
    }
}
```

The rest of the code consists of utility methods and `IDisposable` implementation.

```csharp

        /// <summary>
        /// Method to clear the Validation and Edit State 
        /// </summary>
        public void Clear()
            => this.validationMessageStore.Clear();

        // IDisposable Implementation
        protected virtual void Dispose(bool disposing)
        {
            if (!disposedValue)
            {
                if (disposing)
                {
                    if (this.EditContext != null)
                    {
                        this.EditContext.OnFieldChanged -= this.FieldChanged;
                        this.EditContext.OnValidationRequested -= this.ValidationRequested;
                    }
                }
                disposedValue = true;
            }
        }

        public void Dispose()
        {
            // Do not change this code. Put cleanup code in 'Dispose(bool disposing)' method
            Dispose(disposing: true);
            GC.SuppressFinalize(this);
        }
```

## A Simple Implementation

![EditForm](https://shauncurtis.github.io/siteimages/Articles/Editor-Controls/ValidationFormState.png)

To test the component, here's a simple test page.

Change the temperature up and down and you should see the buttons change colour and Text, and enabled/disabled state.  Change the Temperature to 200 to get a validation message.

You can see this at [https://cec-blazor-database.azurewebsites.net//validationeditor](https://cec-blazor-database.azurewebsites.net//validationeditor).

```html
@using Blazor.Database.Data
@page "/validationeditor"

<EditForm Model="@Model" OnValidSubmit="@HandleValidSubmit">
    <EditFormState @ref="editFormState" EditStateChanged="this.EditStateChanged"></EditFormState>
    <ValidationFormState @ref="validationFormState"></ValidationFormState>

    <label class="form-label">ID:</label> <InputNumber class="form-control" @bind-Value="Model.ID" />
    <label class="form-label">Date:</label> <InputDate class="form-control" @bind-Value="Model.Date" /><ValidationMessage For="@(() => Model.Date)" />
    <label class="form-label">Temp C:</label> <InputNumber class="form-control" @bind-Value="Model.TemperatureC" /><ValidationMessage For="@(() => Model.TemperatureC)" />
    <label class="form-label">Summary:</label> <InputText class="form-control" @bind-Value="Model.Summary" /><ValidationMessage For="@(() => Model.Summary)" />

    <div class="mt-2">
        <div>Validation Messages:</div>
        <ValidationSummary />
    </div>

    <div class="text-right mt-2">
        <button class="btn @btnStateColour" disabled>@btnStateText</button>
        <button class="btn @btnValidColour" disabled>@btnValidText</button>
        <button class="btn btn-primary" type="submit" disabled="@_btnSubmitDisabled">Submit</button>
    </div>

</EditForm>
```
```csharp
@code {
    protected bool _isDirty = false;
    protected bool _isValid => validationFormState?.IsValid ?? true;
    protected string btnStateColour => _isDirty ? "btn-danger" : "btn-success";
    protected string btnStateText => _isDirty ? "Dirty" : "Clean";
    protected string btnValidColour => !_isValid ? "btn-danger" : "btn-success";
    protected string btnValidText => !_isValid ? "Invalid" : "Valid";
    protected bool _btnSubmitDisabled => !(_isValid && _isDirty);

    protected EditFormState editFormState { get; set; }
    protected ValidationFormState validationFormState { get; set; }

    private WeatherForecast Model = new WeatherForecast()
    {
        ID = 1,
        Date = DateTime.Now,
        TemperatureC = 22,
        Summary = "Balmy"
    };

    private void HandleValidSubmit()
        => this.editFormState.UpdateState();

    private void EditStateChanged(bool editstate)
        => this._isDirty = editstate;
}
```

## Wrap Up

Hopefully I've explained how validation works and how to build a simple, but comprehensive and extensible validation system.

The most common problem with validation is `ValidationMessage` controls not showing messages.  There are normally two reasons for this:

1. The UI hasn't updated.  Step through the code to check what's happening when.
2. The `FieldIdentifier` generated from the `For` property of `ValidationMessage` doesn't match the `FieldIdentifier` in the validation store.  Check the `FieldIdentifier` you're generating and logging to the validation store.  

The next article shows how to lock out the form and prevent navigation when the form is dirty.

If you've found this article well into the future, the latest version will be available [here](https://shauncurtis.github.io/articles/ValidationFormState.html)
