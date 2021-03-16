---
title: The Blazor EditFormState Control
date: 2021-03-10
oneliner: A Blazor control to manage and monitor edit state in a form.
precis: The first article in a series looking at how to build Blazor edit forms/controls with state management, validation and form locking.  This article focuses on edit state.
published: 2021-03-10
---

# The Blazor EditFormState Control

Publish Date: 2021-03-10
Last Updated: 2021-03-15

## Overview

This is the first in a series of articles describing a set of useful Blazor Edit controls that solve some of the current shortcomings in the out-of-the-box edit experience without the need to buy expensive toolkits.

![EditForm](https://shauncurtis.github.io/siteimages/Articles/Editor-Controls/EditFormState.png)

## Code and Examples

The repository contains a project that implements the controls for all the articles in this series.  You can find it [here](https://github.com/ShaunCurtis/Blazor.Database).

The example site is here [https://cec-blazor-database.azurewebsites.net/](https://cec-blazor-database.azurewebsites.net/).

You can see the test form described later at [https://cec-blazor-database.azurewebsites.net//testeditor](https://cec-blazor-database.azurewebsites.net//testeditor).

> The Repo is a Work In Progress for future articles so will change and develop.

## The Blazor Edit Setting

To begin, let's look at the current form controls and how they work together.  A classic form looks something like this:

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
4. Cascades the `EditContext`.  All controls within `EditForm` capture and use it in one way or another.
4. Provides callback delegates to the parent control for the submission process - `OnSubmit`, `OnValidSubmit` and `OnInvalidSubmit`.

#### EditContext

`EditContext` is the class at the heart of the edit process, providing overall management.  The data class it operates on is the `model`: defined as an `object` type.  It can be any object, but in practice will be a data class of some type.  The only pre-requisite is that fields used in the form are declared as public read/write properties.

The `EditContext` is either:
 - passed directly to `EditForm` as the `EditContext` parameter,
 - or the object instance of the model is set as the `Model` parameter and `EditForm` creates an `EditContext` instance from it.

An important point to remember is don't change out the EditContext model for another object once you've created it.  While it may be possible, it's not advisable.  If the model needs to be changed out, code to refresh the whole form: better safe than ...!

#### FieldIdentifier

The `FieldIdentifier` class represents a partial "serialization" of a model property.  The `EditContext` tracks and identifies individual properties throughj their `FieldIdentifier`.  `Model` is the object that owns the property and `FieldName` is the property name obtained through reflection.

#### Input Controls

`InputText` and `InputNumber` and the other `InputBase` controls capture the cascaded `EditContext`.  Any value changes are pushed up to `EditContext` by calling `NotifyFieldChanged` with their `FieldIdentifier`.

#### EditContext Revisited

The `EditContext` maintains a `FieldIdentifier` list internally.  `FieldIdentifier` objects are passed around in various methods and events to identify specific fields.  Calls to `NotifyFieldChanged` add `FieldIdentifier` objects to the list.  `EditContext` triggers `OnFieldChanged` whenever `NotifyFieldChanged` is called.

`IsModified` provides access to the state of the list or an individual `FieldIdentifier`. `MarkAsUnmodified` resets an individual `FieldIdentifier` or all the `FieldIdentifiers` in the collection.

`EditContext` also contains the functionality to manage validation, but not actually do it.  We'll look at the validation process in the next article.  

## EditFormState Control

The `EditFormState` control, like all edit form controls, captures the cascaded `EditState`.  What it does is:

1. Builds a list of public properties exposed by the `Model` and maintains the edit state of each - an equality check of the original value against the edited value.
2. Updates the state on each change in a field value.
3. Exposes the state through a readonly property.
4. Provides a EventCallback delegate which is triggered whenever the edit state is updated.

Before we look at the control let's look at the Model - in our case `WeatherForecast` - and some of the supporting classes.

### WeatherForecast

`WeatherForecast` is a typical data class.  
1. Each field is declared as a property with default values.
2. `Validate` implements `IValidation`.  Ignore this for the moment we'll look at validation in the next article.  I've shown it as you'll see it in the Repo code.


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

`EditField` is our class for "serializing" out properties from the model.

1. The base fields are *records* - they can only be set on initialization.
2. `EditedValue` carries the current value of the field.
3. `IsDirty` tests equality between `Value` and `EditedValue`.

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

`EditFieldCollection` is an `IEnumerable` collection of `EditField`.  The class provides a set of controlled setters and getters for the collection and implements the necessary methods for the `IEnumerable` interface.  It also provides an `IsDirty` property to expose the state of the collection.

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

Now we've seen the support classes, On to the main control.

### EditFormState

`EditFormState` is declared as a component and implements `IDisposable`.

```csharp
public class EditFormState : ComponentBase, IDisposable
```

The properties are:
1. Pick up the `EditContext` from the cascade.
2. Provide a `EditStateChanged` callback to the parent control to tell it the edit state has changed.
4. Provide a readonly Property `IsDirty` for controls using `@ref` to check the control state.
5. `EditFields` is the internal `EditFieldCollection` we populate and use to manage the edit state.
6. `disposedValue` is part of the `IDisposable` implementation.

```csharp
        /// EditContext - cascaded from EditForm
        [CascadingParameter] public EditContext EditContext { get; set; }

        /// EventCallback for parent to link into for Edit State Change Events
        /// passes the the current Dirty state
        [Parameter] public EventCallback<bool> EditStateChanged { get; set; }

        /// Property to expose the Edit/Dirty state of the control
        public bool IsDirty => EditFields?.IsDirty ?? false;

        private EditFieldCollection EditFields = new EditFieldCollection();
        private bool disposedValue;
```

When the component initializes it captures the `Model` properties and populates `EditFields` with the initial data.  The last step is to wire up to `EditContext.OnFieldChanged` to `FieldChanged`, so `FieldChanged` gets called whenever a field value changes.

```csharp
    protected override Task OnInitializedAsync()
    {
        Debug.Assert(this.EditContext != null);
        if (this.EditContext != null)
        {
            // Populates the EditField Collection
            this.GetEditFields();
            // Wires up to the EditContext OnFieldChanged event
            this.EditContext.OnFieldChanged += FieldChanged;
        }
        return Task.CompletedTask;
    }

    /// Method to populate the edit field collection
    protected void GetEditFields()
    {
        // Gets the model from the EditContext and populates the EditFieldCollection
        this.EditFields.Clear();
        var model = this.EditContext.Model;
        var props = model.GetType().GetProperties();
        foreach (var prop in props)
        {
            var value = prop.GetValue(model);
            EditFields.AddField(model, prop.Name, value);
        }
    }
```

The `FieldChanged` event handler looks up the `EditField` from `EditFields` and sets its `EditedValue` by calling `SetField`.  It then triggers the `EditStateChanged` callback, with the current dirty state.

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
    /// Method to Update the Edit State to current values 
    public void UpdateState()
    {
        this.GetEditFields();
        this.EditStateChanged.InvokeAsync(EditFields?.IsDirty ?? false);
    }

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

To test the component, here's a simple test page.

![EditForm](https://shauncurtis.github.io/siteimages/Articles/Editor-Controls/EditFormState.png)

Change the temperature up and down and you should see the State button change colour and Text.

You can see this example in action at [https://cec-blazor-database.azurewebsites.net/editstateeditor](https://cec-blazor-database.azurewebsites.net/editstateeditor).

```html
@using Blazor.Database.Data
@page "/test"

<EditForm Model="@Model" OnValidSubmit="@HandleValidSubmit">
    <EditFormState @ref="editFormState" EditStateChanged="this.EditStateChanged"></EditFormState>

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
    protected EditFormState editFormState { get; set; }

    private WeatherForecast Model = new WeatherForecast()
    {
        ID = 1,
        Date = DateTime.Now,
        TemperatureC = 22,
        Summary = "Balmy"
    };

    private void HandleValidSubmit()
    {
        this.editFormState.UpdateState();
    }

    private void EditStateChanged(bool editstate)
        => this._isDirty = editstate;
}
```

## Wrap Up

While the real benefits of this control may not be immediately obvious if you haven't implementede such functionality before, we'll use it in the follow on articles to build an editor form.  The next article looks at the validation process and how to build a simple custom validator. The third article looks at form locking, using this control as part of the process.

If you've found this article well into the future, the latest version will be available [here](https://shauncurtis.github.io/articles/EditFormState.html)

