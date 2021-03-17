---
title: The Blazor Inline Dialog Control
oneliner: A Blazor inline dialog control to lock all page controls except those within the form.
precis: The third article in a series describing how to build Blazor edit forms/controls with state management, validation and form locking.  This article focuses on form locking.
date: 2021-03-17
published: 2021-03-11
---

# The Inline Dialog Control

Date: 2021-03-17

This is the third in a series of articles describing a set of useful Blazor Edit controls that solve some of the current shortcomings in the out-of-the-box edit experience without the need to buy expensive toolkits.

This article describes how to build a component that disables links, buttons the URL bar,... : everywhere except the content within the component.  While it can't stop the user navigating through the browser controls, it turns on the browser `beforeunload` event to force the "Do you really want to leave this site?" dialog box.  Everything is implemented with a relatively simple standard Blazor component and a small `js` file. 

![Inline Dialog](https://shauncurtis.github.io/siteimages/Articles/Editor-Controls/inlinedialog.png)

## Code Repository

The repository contains a project that implements the controls for all the articles in this series.  You can find it [here](https://github.com/ShaunCurtis/Blazor.Database).

The example site is here [https://cec-blazor-database.azurewebsites.net/](https://cec-blazor-database.azurewebsites.net/).

The example form described at this end of this article can be seen at [https://cec-blazor-database.azurewebsites.net//inlinedialogeditor](https://cec-blazor-database.azurewebsites.net//inlinedialogeditor).

## Overview

If you want to see the component in action, go to [this page on my Demo Site](https://cec-blazor-database.azurewebsites.net//inlinedialogeditor).  It's a basic mockup to demonstrate the functionality and extends the form used in the last two articles.  There's a typical Edit Form with the two extra controls covered in the previous articles:

1. `EditFormState` monitors the edit state of the `Model` data.
2. `ValidationFormState`- a form validator.

The key bit of action is hooking up the `InlineDialog` control `Lock` to the form state.  `EditFormState` monitors the form state and invokes the  EventCallback `EditStateChanged` whenever a change takes place.  The page `EditStateChanged` event handler is registered with the `EditFormState.EditStateChanged` and updates `_isDirty` whenever the state changes.  If `EditFormState' is dirty, `InlineDialog` is locked. 
```html
@using Blazor.Database.Data
@page "/inlinedialogEditor"

    <InlineDialog Lock="this._isDirty" Transparent="false">
        <EditForm Model="@Model" OnValidSubmit="@HandleValidSubmit" class="p-3">
            <EditFormState @ref="editFormState" EditStateChanged="this.EditStateChanged"></EditFormState>
            <ValidationFormState @ref="validationFormState"></ValidationFormState>

            <label class="form-label">ID:</label> 
            <InputNumber class="form-control" @bind-Value="Model.ID" />
            <label class="form-label">Date:</label> 
            <InputDate class="form-control" @bind-Value="Model.Date" />
            <ValidationMessage For="@(() => Model.Date)" />
            <label class="form-label">Temp C:</label>
            <InputNumber class="form-control" @bind-Value="Model.TemperatureC" />
            <ValidationMessage For="@(() => Model.TemperatureC)" />
            <label class="form-label">Summary:</label>
            <InputText class="form-control" @bind-Value="Model.Summary" />
            <ValidationMessage For="@(() => Model.Summary)" />

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
    </InlineDialog>
}
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
        {
            this.editFormState.UpdateState();
        }

        private void EditStateChanged(bool editstate)
            => this._isDirty = editstate;
    }
```

## InLineDialog

Lets look at the Parameters and public Properties first.
1. We capture added attributes, though we only use `class`.
2. `Cascade` turns on/off parameter cascading of `this` i.e. the instance of `InlineDialog`. Default is `true`.
3. `Transparent` sets the background to either transparent or translucent.  The demo is set to transluscent so you can see it switch in and out.
4. `ChildContent` is the content between `<InlineDialog>` and `</InlineDialog>`.
5. `IsLocked` is a read only Property for checking the component state.

```csharp
[Parameter(CaptureUnmatchedValues = true)] public IDictionary<string, object> AdditionalAttributes { get; set; } = new Dictionary<string, object>();
[Parameter] public bool Cascade { get; set; } = true;
[Parameter] public bool Transparent { get; set; } = true;
[Parameter] public RenderFragment ChildContent { get; set; }
public bool IsLocked => this._isLocked;
```
The private properties:
1. Inject `IJSRuntime` to give access to the Javascript Interop and set/unset the browser `BeforeUnload` event.
2. `CssClass` builds the Html  `class` attribute for the component, combining any entered classes with those built by the component.
3. The Css properties define the various Css options for `class`.
4. `_isLocked` in the private field for controlling lock state.


```csharp
[Inject] private IJSRuntime _js { get; set; }

private string CssClass => (AdditionalAttributes != null && AdditionalAttributes.TryGetValue("class", out var obj))
    ? $"{this.frontcss} { Convert.ToString(obj, CultureInfo.InvariantCulture)}"
    : this.frontcss;

private string backcss = string.Empty;
private string frontcss = string.Empty;
private string _backcss => this.Transparent ? "back-block-transparent" : "back-block";
private string _frontcss => this.Transparent ? "fore-block-transparent" : "fore-block";
private string __backcss => string.Empty;
private string __frontcss => string.Empty;
private bool _isLocked;
```

There are two public methods: `Lock` and `Unlock`.  These change the Css classes.  `SetPageExitCheck` interfaces with the Javascript functions to add or remove the `beforeunload` event on `Window`.  The code is show below.

```csharp
public void Lock()
{
    this._isLocked = true;
    this.backcss = this._backcss;
    this.frontcss = this._frontcss;
    this.SetPageExitCheck(true);
    this.InvokeAsync(StateHasChanged);
}

public void Unlock()
{
    this._isLocked = false;
    this.backcss = this.__backcss;
    this.frontcss = this.__frontcss;
    this.SetPageExitCheck(false);
    this.InvokeAsync(StateHasChanged);
}

private void SetPageExitCheck(bool action)
    => _js.InvokeAsync<bool>("cecblazor_setEditorExitCheck", action);
```

The Javascript in *site.js*:

```js
window.cecblazor_setEditorExitCheck = function (show) {
    if (show) {
        window.addEventListener("beforeunload", cecblazor_showExitDialog);
    }
    else {
        window.removeEventListener("beforeunload", cecblazor_showExitDialog);
    }
}

window.cecblazor_showExitDialog = function (event) {
    event.preventDefault();
    event.returnValue = "There are unsaved changes on this page.  Do you want to leave?";
}
```

Moving on to the Razor for the component:
1. We add a `div` with the Css class `_backcss`: this is either *back-block-transparent* or *back-block* when `Locked` or empty when `Unlocked`.
2. We add a `div` with the Css class `_frontcss`: this is either *fore-block-transparent* or *fore-block* when `Locked` or empty when `Unlocked` combined with any `class` attribute value we have added to the component.
2. We cascade `this` if `Cascade` is true.

```html
<div class="@this.backcss"></div>

<div class="@this.CssClass">
    @if (this.Cascade)
    {
        <CascadingValue Value="this">
            @this.ChildContent
        </CascadingValue>
    }
    else
    {
        @this.ChildContent
    }
</div>
```

Moving on to the component Css, which is where the magic happens.  We implement a similar CSS technique to that used in modal dialogs, adding a transparent or translucent layer over the page content to *block* content below the layer, and place the contents of `InlineDialog` in front of that layer.  If you use a lot on z-index layers in your application, you may need to tweak the Z-index to make sure it sits on top.

```css
div.back-block {
    display: block;
    position: fixed;
    z-index: 1; /* Sit on top */
    left: 0;
    top: 0;
    width: 100%; /* Full width */
    height: 100%; /* Full height */
    overflow: auto; /* Enable scroll if needed */
    background-color: RGBA(224, 224, 224, 0.4);/* the translucent effect*/
}

div.back-block-transparent {
    display: block;
    position: fixed;
    z-index: 1; /* Sit on top */
    left: 0;
    top: 0;
    width: 100%; /* Full width */
    height: 100%; /* Full height */
    overflow: auto; /* Enable scroll if needed */
    background-color: transparent; 
}

div.fore-block-transparent {
    display: block;
    position: relative;
    z-index: 2; /* Sit on top */
}

div.fore-block {
    display: block;
    position: relative;
    z-index: 2; /* Sit on top */
    background-color: RGB(255, 255, 255);/* need to set the colour, adjust as necessary */
}
```

## Wrap Up

This solution uses the same techniques used by modal dialogs in placing a barrier between the controls on the page and the contents of the control.  It's an *in place* modal dialog.  `Lock` inserts the barrier and `Unlock` removes it.  We add the Javascript Interop to turn on add off the `beforeunload` event on the browser.  Choose between a transparent or transluscent layer, or code your own CSS.

Having developed many solutions to solve this problem, and written articles about them, I'm a little flabbergasted to finally find a solution that's this easy.  The best solutions are always the simplest!
