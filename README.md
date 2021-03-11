---
title: The Blazor EditFormState Control
date: 2020-10-23
---

# The Blazor EditFormState Control

## Overview

This is the first in a series of articles describing a set of useful Blazor Edit controls that solve some current shortcomings in the out-of-the-box edit experience without buying into some expensive toolkits.

## Code and Examples

The repository contains a project that implements the controls for all the articles in this series.  You can find it [here](https://github.com/ShaunCurtis/Blazor.Database).

The example site is here [https://cec-blazor-database.azurewebsites.net/](https://cec-blazor-database.azurewebsites.net/).

> It's still a Work In Progress for future articles so will change and develop.

## The Blazor Edit Setting

To begin lets look at the current form controls and how they work together.  A classic form looks something like this:

```html
<EditForm Model="@exampleModel" OnValidSubmit="@HandleValidSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <InputText id="name" @bind-Value="exampleModel.Name" />
    <ValidationMessage For="@(() => exampleModel.Name)" />

    <button type="submit">Submit</button>
</EditForm>
```

#### EditForm

`EditForm` is the overall wrapper. It:

1. Creates the html `Form` context.
2. Hooks up any `Submit` buttons - i.e. buttons with their `type` set to `submit` within the form.
3. Creates/manages the `EditContext`.
4. Cascades the `EditContext`.  All the controls within `EditForm` capture and use it in one shape or form.
4. Provides callback delegates for the submission process - `OnValidSubmit`, `OnSubmit` and `OnInvalidSubmit` -  for the parent control wiring.

#### EditContext

`EditContext` is the class at the heart of the edit process, providing overall management.  It's created from a `model`: a standard `object`.  It can be any class, but in practice it'll be a data class of some type.  The only pre-requisite is that fields used in the form are declared as public read/write properties.

The `EditContext` cascaded by `EditForm` is either:
 - passed directly to `EditForm` as the `EditContext` parameter,
 - or the model is passed as the `Model` parameter and `EditForm` creates a `EditContext` instance from it.

An important point to make here is you shouldn't change out the EditContext model for another object once you've created it.  While it may be possible, DON'T.  To change out the model, code to refresh the whole form: better safe than ...!

#### FieldIdentifier

A `FieldIdentifier` is a partial "serialization" of a model property.  It's how `EditContext` identifies and tracks individual properties.  `Model` is the class instance that owns the property and `FieldName` is the property name obtained through reflection.

#### Input Controls

`InputText` and `InputNumber` and the other `InputBase` controls capture the cascaded `EditContext`.  Any value changes are pushed up to `EditContext` by calling `NotifyFieldChanged` with their `FieldIdentifier`.

#### EditContext Revisited

The `EditContext` maintains a list of `FieldIdentifier`s internally.  The `FieldIdentifier` provided in calls to `NotifyFieldChanged` is logged to the list.  `EditContext` triggers `OnFieldChanged` whenever `NotifyFieldChanged` is called.

You can check the modification state of the EditContext at any time by calling `IsModified` which checks the `FieldIdentifier` collection for any logged field changes. `MarkAsUnmodified` lets you reset an individual `FieldIdentifier` or all the `FieldIdentifiers` in the collection.

`EditContext` also contains the functionality to manage validation, but not actually do it.  When a user clicks on a submit button, `EditForm` does one of two actions:

1. If a delegate is registered with `OnSubmit`, it triggers it.  This ignores validation.
2. If no `OnSubmit` delegate is registered, it runs any registered validation by calling `EditContext.Validate` and then either triggers `OnValidSubmit` or `OnInvalidSubmit` depending on the result.

`EditContext.Validate` fires `OnValidationRequested` synchronously (if there's a delegate registered) and then checks if there are any messages in the `ValidationMessageStore`.  If it's empty, the form passed validation.  

Validators are wired to this event to handle validation.   There's no `interface` to conform to, Validators just need to be wired into `OnValidationRequested` and interact with the `ValidationMessageStore` of `EditContext`. It validates whatever it's configured to validate, and logs any error messages to the `ValidationMessageStore`.  Finally it calls `NotifyValidationStateChanged` on `EditContext`. This triggers `OnValidationStateChanged`.

#### Validation Controls

Controls such as `ValidationMessage` and `ValidationSummary` capture the cascaded `EditContext` and wire into the `OnValidationStateChanged` to manage and display their messages.

## EditFormState Control

The `EditFormState` control, like the input controls, is wired into the cascaded `EditState`.  What it does is:

1. Builds a list of public properties exposed by the `Model` and maintains the edit state on each - a comparison of the original value with the edited value.
2. Updates the state on each change in a field value.
3. Exposes the state through a readonly property.
4. Calls an EventCallback delegate whenever the edit state is updated.

Before we look at the control let's look at the Model - `WeatherForecast` - and some of the supporting classes.

### WeatherForecast

`WeatherForecast` is a typical data class.  
1. Each field is declared as a property with default values.
2. `Validate` implements `IValidation`.  Ignore this for the moment we'll look at validation in subsequent article.  I've shown it as you'll see it in the Repo code.


```csharp
public class WeatherForecast : IValidation
{
    public int ID { get; set; } = -1;
    public DateTime Date { get; set; } = DateTime.Now;
    public int TemperatureC { get; set; } = 0;
    [NotMapped] public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
    public string Summary { get; set; } = string.Empty;

    /// Ignore for now, but as you'll see it in the example repo it's shown
    public bool Validate(ValidationMessageStore validationMessageStore, string fieldname, object model = null)
    {
        ....
    }
}
```

### EditField

`EditField` is our class for "serialization" of properties from the model.

1. The base fields are records - they can only be set on initialization.
2. `EditedValue` carries the current value of the field.
3. `IsDirty` is used to test if the field is dirty.

```csharp
public class EditField
{
    public string FieldName { get; init; }
    public Guid GUID { get; init; }
    public object Value { get; init; }
    public object Model { get; init; }
    public object EditedValue { get; set; }
    public bool IsDirty
    {
        get
        {
            if (Value != null && EditedValue != null) return !Value.Equals(EditedValue);
            if (Value is null && EditedValue is null) return false;
            return true;
        }
    }

    public EditField(object model, string fieldName, object value)
    {
        this.Model = model;
        this.FieldName = fieldName;
        this.Value = value;
        this.EditedValue = value;
        this.GUID = Guid.NewGuid();
    }

    public void Reset()
        => this.EditedValue = this.Value;
}
```

### EditFieldCollection

`EditFieldCollection` is an `IEnumerable` collection of `EditField`s.  The class provides a set of controlled setters and getters for the collection and implments the necessary methods for the `IEnumerable` interface.

```csharp
    public class EditFieldCollection : IEnumerable
    {
        private List<EditField> _items = new List<EditField>();
        public int Count => _items.Count;
        public Action<bool> FieldValueChanged;
        public bool IsDirty => _items.Any(item => item.IsDirty);

        public void Clear()
            => _items.Clear();

        public void ResetValues()
            => _items.ForEach(item => item.Reset());

        public IEnumerator GetEnumerator()
            => new EditFieldCollectionEnumerator(_items);

        public T Get<T>(string FieldName)
        {
            var x = _items.FirstOrDefault(item => item.FieldName.Equals(FieldName, StringComparison.CurrentCultureIgnoreCase));
            if (x != null && x.Value is T t) return t;
            return default;
        }

        public T GetEditValue<T>(string FieldName)
        {
            var x = _items.FirstOrDefault(item => item.FieldName.Equals(FieldName, StringComparison.CurrentCultureIgnoreCase));
            if (x != null && x.EditedValue is T t) return t;
            return default;
        }

        public bool TryGet<T>(string FieldName, out T value)
        {
            value = default;
            var x = _items.FirstOrDefault(item => item.FieldName.Equals(FieldName, StringComparison.CurrentCultureIgnoreCase));
            if (x != null && x.Value is T t) value = t;
            return x.Value != default;
        }

        public bool TryGetEditValue<T>(string FieldName, out T value)
        {
            value = default;
            var x = _items.FirstOrDefault(item => item.FieldName.Equals(FieldName, StringComparison.CurrentCultureIgnoreCase));
            if (x != null && x.EditedValue is T t) value = t;
            return x.EditedValue != default;
        }

        public bool HasField(EditField field)
            => this.HasField(field.FieldName);

        public bool HasField(string FieldName)
        {
            var x = _items.FirstOrDefault(item => item.FieldName.Equals(FieldName, StringComparison.CurrentCultureIgnoreCase));
            if (x is null | x == default) return false;
            return true;
        }

        public bool SetField(string FieldName, object value)
        {
            var x = _items.FirstOrDefault(item => item.FieldName.Equals(FieldName, StringComparison.CurrentCultureIgnoreCase));
            if (x != null && x != default)
            {
                x.EditedValue = value;
                this.FieldValueChanged?.Invoke(this.IsDirty);
                return true;
            }
            return false;
        }

        public bool AddField(object model, string fieldName, object value)
        {
            this._items.Add(new EditField(model, fieldName, value));
            return true;
        }

```

The `Enumerator` support class.

```csharp
        public class EditFieldCollectionEnumerator : IEnumerator
        {
            private List<EditField> _items = new List<EditField>();
            private int _cursor;

            object IEnumerator.Current
            {
                get
                {
                    if ((_cursor < 0) || (_cursor == _items.Count))
                        throw new InvalidOperationException();
                    return _items[_cursor];
                }
            }
            public EditFieldCollectionEnumerator(List<EditField> items)
            {
                this._items = items;
                _cursor = -1;
            }
            void IEnumerator.Reset()
                => _cursor = -1;

            bool IEnumerator.MoveNext()
            {
                if (_cursor < _items.Count)
                    _cursor++;
                return (!(_cursor == _items.Count));
            }
        }
    }
```

### EditFormState

The main control is declared as a component and implements `IDisposable`.

```csharp
public class EditFormState : ComponentBase, IDisposable
```

The properties are:
1. Pick up the `EditContext` from the cascade.
2. Provide a `EditStateChanged` callback as an pseudo event to the parent control to tell it the edit state has changed.
3. Provide a `Reset` Parameter that the parent can use to tell the control to clear/reset it's state to clean.
4. Provide a readonly Property `IsDirty` for any controls that have a `@ref` to check the control state.
5. `EditFields` is the internal edit field collection we populate and use to manage the state.
6. `disposedValue` is part of the `IDisposable` implementation.

```csharp
        /// EditContext - cascaded from EditForm
        [CascadingParameter] public EditContext EditContext { get; set; }

        /// EventCallback for parent to link into for Edit State Change Events
        /// passes the the current Dirty state
        [Parameter] public EventCallback<bool> EditStateChanged { get; set; }

        /// Pseudo Property to force the control to do a state and validation
        [Parameter]
        public bool Reset
        {
            get => false;
            set
            {
                if (value) this.Clear();
            }
        }

        /// Property to expose the Edit/Dirty state of the control
        public bool IsDirty => EditFields?.IsDirty ?? false;

        private EditFieldCollection EditFields = new EditFieldCollection();
        private bool disposedValue;
```
When the component is initialized we get the model and capture all the model properties and populate the `EditFields` collection with our initial data.  Finally we wire up to `EditContext.OnFieldChanged` to handle any data changes the user makes.

```csharp
        protected override Task OnInitializedAsync()
        {
            Debug.Assert(this.EditContext != null);
            if (this.EditContext != null)
            {
                // Gets the model from the EditContext and populates the EditFieldCollection
                var model = this.EditContext.Model;
                var props = model.GetType().GetProperties();
                foreach (var prop in props)
                {
                    var value = prop.GetValue(model);
                    EditFields.AddField(model, prop.Name, value);
                }
                // Wires up to the EditContext OnFieldChanged event
                this.EditContext.OnFieldChanged += FieldChanged;
            }
            return Task.CompletedTask;
        }

```
The `FieldChanged` event handler looks up the `EditField` from the `EditFields` collection and sets its `EditedValue` through `SetField` to the new value.  It then triggers the `EditStateChanged` callback.  This will normally be wired up by the main edit form so it can take whatever action it needs when the editstate is dirty.  We'll see that in action in a subsequent article.

```csharp
        /// Event Handler for Editcontext.OnFieldChanged
        private void FieldChanged(object sender, FieldChangedEventArgs e)
        {
            // Get the PropertyInfo object for the model property
            // Uses reflection to get property and value
            var prop = e.FieldIdentifier.Model.GetType().GetProperty(e.FieldIdentifier.FieldName);
            if (prop != null)
            {
                // Get the value for the property
                var value = prop.GetValue(e.FieldIdentifier.Model);
                // Sets the edit value in the EditField
                EditFields.SetField(e.FieldIdentifier.FieldName, value);
                // Invokes EditStateChanged
                this.EditStateChanged.InvokeAsync(EditFields?.IsDirty ?? false);
            }
        }

```

Finally we have some utility methods and `IDisposable` implementation.

```csharp
        /// Method to clear the Validation and Edit State 
        public void Clear()
            => this.EditFields.ResetValues();

        // IDisposable Implementation
        protected virtual void Dispose(bool disposing)
        {
            if (!disposedValue)
            {
                if (disposing)
                {
                    if (this.EditContext != null)
                        this.EditContext.OnFieldChanged -= this.FieldChanged;
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
    }
```

## A Simple Implementation

![EditForm](../Images/Articles/Editor-Controls/EditformState.png)

To test the component, here's a simple implementation test page.

Change the temperature up and down and you should see the State button change colour and Text.

```html
@using Blazor.Database.Data
@page "/test"

<EditForm Model="@Model" OnValidSubmit="@HandleValidSubmit">
    <EditFormState EditStateChanged="this.EditStateChanged"></EditFormState>

    <label class="form-label">ID:</label> <InputNumber class="form-control" @bind-Value="Model.ID" />
    <label class="form-label">Date:</label> <InputDate class="form-control" @bind-Value="Model.Date" />
    <label class="form-label">Temp C:</label> <InputNumber class="form-control" @bind-Value="Model.TemperatureC" />
    <label class="form-label">Summary:</label> <InputText class="form-control" @bind-Value="Model.Summary" />

    <div class="text-right mt-2">
        <button class="btn @btncolour">@btntext</button>
        <button class="btn btn-primary" type="submit">Submit</button>
    </div>

    <div>
    </div>
</EditForm>
```

```csharp
@code {
    protected bool _isDirty = false;
    protected string btncolour => _isDirty ? "btn-danger" : "btn-success";
    protected string btntext => _isDirty ? "Dirty" : "Clean";

    private WeatherForecast Model = new WeatherForecast()
    {
        ID = 1,
        Date = DateTime.Now,
        TemperatureC = 22,
        Summary = "Balmy"
    };

    private void HandleValidSubmit() { }

    private void EditStateChanged(bool editstate)
        => this._isDirty = editstate;
}
```

## Wrap Up

While the real benefits of this control may not be immediately obvious if you haven't had to implement such functionality before, we will use this control in the following articles to build a production level editor form.  The next article will look at how to use this control to lock out the form and prevent navigation when the form is dirty.
