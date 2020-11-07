# A Simple Bootstrap Modal Dialog for Blazor

## Overview

If you want a web based SPA [Single Page Application] to look like a real application you need modal dialogs.  This article shows how to build a modal dialog wrapper for Blazor IComponents using the Bootstrap Modal Dialog CSS Framework.

## Code and Examples

There's no specific project code for this article. The component is part of *CEC.Blazor* Library avaliable on Github at [CEC.Blazor](https://github.com/ShaunCurtis/CEC.Blazor).  You can get the source from here, and see a live example in action. To review the code , start at *CEC.Blazor.Server/Routes/WeatherForecast/WeatherForecastListModalView*.

## The Modal Dialog Classes

The Modal Dialog control is a Blazor Component.

There are three classes and one Enum:

1. BootStrapModal
2. BootStrapModalOptions
1. BootStrapModalResult
1. BootStrapModalResultType

#### BootStrapModalResultType

```c#
// Defines the types for exiting the dialog
public enum BootstrapModalResultType
{
    NoSet,
    OK,
    Cancel,
    Exit
}
```

#### BootStrapModalResult

*BootstrapModalResult* is passed back by *BootStrapModal* when the modal closes.

```c#
public class BootstrapModalResult
{
    // The closing type
    public BootstrapModalResultType ResultType { get; private set; } = BootstrapModalResultType.NoSet;

    // Whatever object you wish to pass back
    public object Data { get; set; } = null;

    // A set of static methods to build a BootstrapModalResult

    public static BootstrapModalResult OK() => new BootstrapModalResult() {ResultType = BootstrapModalResultType.OK };

    public static BootstrapModalResult Exit() => new BootstrapModalResult() {ResultType = BootstrapModalResultType.Exit};

    public static BootstrapModalResult Cancel() => new BootstrapModalResult() {ResultType = BootstrapModalResultType.Cancel };

    public static BootstrapModalResult OK(object data) => new BootstrapModalResult() { Data = data, ResultType = BootstrapModalResultType.OK };

    public static BootstrapModalResult Exit(object data) => new BootstrapModalResult() { Data = data, ResultType = BootstrapModalResultType.Exit };

    public static BootstrapModalResult Cancel(object data) => new BootstrapModalResult() { Data = data, ResultType = BootstrapModalResultType.Cancel };
}
```
#### BootStrapModalOptions

*BootstrapModalOptions* is an options class passed to the Modal Dialog class when opening the Dialog.  The properties are pretty self explanatory.  
```c#
public class BootstrapModalOptions
{
    public string Title { get; set; } = "Modal Dialog";

    public string ContainerCSS { get; set; }

    public string ModalCSS { get; set; }

    public string ModalHeaderCSS { get; set; }

    public string ModalBodyCSS { get; set; }

    public bool ShowCloseButton { get; set; }

    public bool HideHeader { get; set; }

    // Generic name/object list to pass any object to the dialog
    public Dictionary<string, object> Parameters { get; set; } = new Dictionary<string, object>();
}
```
#### BootStrapModal

The Razor Markup for *BootstrapModal* replicates the Bootstrap markup.  We don't need to worry about toggling the container *display* mode as we don't generate any markup if *ShowDialog* is false.  We cascade the ModalDialog so that the child form has direct access to it. 
```c#
@inherits ComponentBase

@namespace CEC.Blazor.Components.UIControls

@if (this._ShowModal)
{
    <CascadingValue Value="this">
        <div class="@this._ContainerCss" data-backdrop="static" tabindex="-1" role="dialog" aria-modal="true" style="display:block;">
            <div class="@this._ModalCss">
                <div class="modal-content">
                    @if (!this.Options.HideHeader)
                    {
                        <div class="@this._ModalHeaderCss">
                            <h5 class="modal-title">@this.Options.Title</h5>
                            @if (this.Options.ShowCloseButton)
                            {
                                <button type="button" class="close" data-dismiss="modal" aria-label="Close" @onclick="this.Dismiss">
                                    <span aria-hidden="true">&times;</span>
                                </button>
                            }
                        </div>
                    }
                    <div class="@this._ModalBodyCss">
                        @this._Content
                    </div>
                </div>
            </div>
        </div>
    </CascadingValue>
}
```

Some key points:
1. The component is initialised when the View is created and added to the RenderTree.
2. There's no need for multiple copies of the component on a page.  Call *Show*, supplying the component type to display it.
3. The component hides itself.  Either the "child" calls the *BootstrapModal* function *Close* or *BootstrapModal* itself calls  *Dismiss*.
3. The component uses a *TaskCompletionSource* object to manage the async behaviour of the component and communicate task status back to the caller.

```c#
public partial class BootstrapModal : ComponentBase
{
    public BootstrapModalOptions Options { get; set; } = new BootstrapModalOptions();

    private RenderFragment _Content { get; set; }

    private bool _ShowModal { get; set; }

    private Task modalResult { get; set; }

    private string _ContainerCss => $"modal fade show {this.Options.ContainerCSS}".Trim();

    private string _ModalCss => $"modal-dialog {this.Options.ModalCSS}";

    private string _ModalHeaderCss => $"modal-header {this.Options.ModalHeaderCSS}".Trim();

    private string _ModalBodyCss => $"modal-body {this.Options.ModalBodyCSS}".Trim();

    // object we use to communicate back to the caller on the task state
    private TaskCompletionSource<BootstrapModalResult> _modalcompletiontask { get; set; } = new TaskCompletionSource<BootstrapModalResult>();

    //  Method used to Show/Hide the Dialog
    // Note that TModal must implement IComponent
    public Task<BootstrapModalResult> Show<TModal>(BootstrapModalOptions options) where TModal: IComponent 
    {
        this.Options = options;
        this._modalcompletiontask = new TaskCompletionSource<BootstrapModalResult>();
        var i = 0;
        this._Content = new RenderFragment(builder =>
        {
            builder.OpenComponent(i++, typeof(TModal));
            builder.CloseComponent();
        });
        this._ShowModal = true;
        InvokeAsync(StateHasChanged);
        return _modalcompletiontask.Task;
    }

    // Method to update the dialog display options
    public void Update(BootstrapModalOptions options = null)
    {
        this.Options = options??= this.Options;
        InvokeAsync(StateHasChanged);
    }

    // method called by the dismiss button in the dialog header to close the dialog
    public async void Dismiss()
    {
        _ = _modalcompletiontask.TrySetResult(BootstrapModalResult.Cancel());
        this._ShowModal = false;
        this._Content = null;
        await InvokeAsync(StateHasChanged);
    }

    // Method called by the child form to close the dialog
    public async void Close(BootstrapModalResult result)
    {
        _ = _modalcompletiontask.TrySetResult(result);
        this._ShowModal = false;
        this._Content = null;
        await InvokeAsync(StateHasChanged);
    }
}
```
## Implementing Bootstrap Modal

### The YesNoModal

The *YesNoModal* is a simple "Are You Sure" modal form.  It captures the cascaded parent object reference as *Parent* and calls the *Parent.Close()* to hide the dialog.  It calls one of the *BootstrapModalResult* constructor methods to build the correct *BootstrapModalResult* response. 

```html
@inherits ComponentBase
@namespace CEC.Blazor.Components.UIControls
<UIContainer>
    <UIRow>
        <UIColumn Columns="12">
            @((MarkupString)this.Message)
        </UIColumn>
        <UIButtonColumn Columns="12">
            <button type="button" class="btn btn-danger" @onclick="(e => this.Close(true))">Exit</button>
            <button type="button" class="btn btn-success" @onclick="(e => this.Close(false))">Cancel</button>
        </UIButtonColumn>
    </UIRow>
</UIContainer>
```

```c#
public partial class YesNo : ComponentBase
{
    [CascadingParameter]
    public BootstrapModal Parent { get; set; }

    [Parameter]
    public string Message { get; set; } = "Are You Sure?";

    public void Close(bool state)
    {
        if (state) this.Parent.Close(BootstrapModalResult.Exit());
        else this.Parent.Close(BootstrapModalResult.Cancel());
    }
}
```

### Form using BootstrapModal

Declare the component in your form - I normally do it at the bottom.  The button control is just to "demo" activating the *YesNoDialog*.
```c#
<div>
    <button type="button" class="btn btn-danger" @onclick="(e => this.AreYouSure())">Ask Are You Sure</button>
</div>

<BootstrapModal @ref="this._BootstrapModal"></BootstrapModal>
```
Define a referenced Property in your form.
```c#
protected BootstrapModal _BootstrapModal { get; set; }
```
And implement an action handler to open the modal dialog. This:

1. Creates a ModalOptions object to set up the modal dialog appearance.
2. Adds any necessary parameters.
3. Calls and awaits *Show* on the *BootstrapModal* instance to turn on the Modal, passing the component to display as generic T.
4. Responds to the result when the modal is closed.
```c#
protected async void AreYouSureAsync()
{
    var modalOptions = new BootstrapModalOptions()
    {
        ModalBodyCSS = "p-0",
        ModalCSS = "modal-xl",
        HideHeader = true,
    };
    modalOptions.Parameters.Add("NotUsedParameter", -1);
    var result = await this._BootstrapModal.Show<YesNoModal>(modalOptions);
    if (result.ResultType == BootstrapModalResultType.Cancel) {
        //Do something to stop
    }
}
```
## Wrap Up

While the modal dialog is implemented using Bootstrap, you can change the Markup to fit any other CSS framework or your own custom CSS.

If your looking for a more complex Modal Dialog with more features take a look at [Blazored Modal Dialog](https://github.com/Blazored/Modal).

```c#
```

```c#
```