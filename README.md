# Building a Database Appication in Blazor 
## Part 3 - View Components - CRUD Edit and View Operations in the UI

## Introduction

This is the third in a series of articles looking at how to build and structure a real Database Application in Blazor. The articles so far are:


1. [Project Structure and Framework](https://www.codeproject.com/Articles/5279560/Building-a-Database-Application-in-Blazor-Part-1-P)
2. [Services - Building the CRUD Data Layers](https://www.codeproject.com/Articles/5279596/Building-a-Database-Application-in-Blazor-Part-2-S)
3. View Components - CRUD Edit and View Operations in the UI

* Further articles wil look at List Operations in the UI
* UI Components - Building HTML/CSS Controls
* A walk through detailing how to add more records to the application - in this case weather stations and weather station data.

This article looks in detail at building reusable CRUD presentation layer components, specifically Edit and View functionality - and using them into both Server and WASM projects.

### Sample Project and Code

See the [CEC.Blazor GitHub Repository](https://github.com/ShaunCurtis/CEC.Blazor) for the libraries and sample projects.

### The Base Forms

All CRUD UI components inherit from *OwningComponentBase*.  We use *OwningComponentBase* in preference to  *ComponentBase* because it gives control over the scope of Scoped Services. Not all the code is shown in the article.  Some class are too big.  All source files can be viewed on the Github site, and I include references or links to specific code files at appropriate places in the article.  Much of the information detail is in the comments in the code sections.

#### ApplicationComponentBase

[*ApplicationComponentBase*](https://github.com/ShaunCurtis/CEC.Blazor/blob/master/CEC.Blazor/Components/BaseForms/ApplicationComponentBase.cs) is the base component and contains all the common client application code.  It provides:

  1. Injection of common services, such as Navigation Manager, AuthenicationState, RouterSessionService and Application Configuration.
  2. Authentication and user management.
  3. Navigation and Routing through *NavigateTo*.
  4. Core modal dialog event handling (for when a component is wrapping in a Modal Dialog).
  5. State Updating.
  6. A set of Common Properties that are used by th inheriting classes.

#### ControllerServiceComponent and Its Children

[*ControllerServiceComponentBase*](https://github.com/ShaunCurtis/CEC.Blazor/blob/master/CEC.Blazor/Components/BaseForms/ControllerServiceComponentBase.cs) is the base CRUD component.

There are three inherited classes for specific CRUD operations:
1. [*ListComponentBase*](https://github.com/ShaunCurtis/CEC.Blazor/blob/master/CEC.Blazor/Components/BaseForms/ListComponentBase.cs) for all list pages
2. [*RecordComponentBase*](https://github.com/ShaunCurtis/CEC.Blazor/blob/master/CEC.Blazor/Components/BaseForms/RecordComponentBase.cs) for displaying individual records.
3. [*EditComponentBase*](https://github.com/ShaunCurtis/CEC.Blazor/blob/master/CEC.Blazor/Components/BaseForms/EditRecordComponentBase.cs) for CUD *Create/Update/Delete* operations.

Common code resides in *ControllerServiceComponent*, specific code in the inherited class.

We'll examine the functionality in these components as look at their deployment in the sample projects, and step down into the base components to see how their specific functionility is implemented.

## Implementing Edit Pages

### The View

The routed view is a very simple.  It contains the routes and the component to load.  The editor component is implemented separately so it can be used in both the WASM and Server projects and in the modal dialog editor.

```c#
// CEC.Blazor.WASM.Client/Routes/WeatherForecastEditorView.razor
@page "/WeatherForecast/New"
@page "/WeatherForecast/Edit"
@inherits ApplicationComponentBase
@namespace CEC.Blazor.WASM.Client.Routes

<WeatherEditorForm></WeatherEditorForm>
```

### The Form

The code file is relatively simple, with most of the detail in the razor markup.

```C#
// CEC.Weather/Components/Forms/WeatherForecastEditorForm.razor
public partial class WeatherEditorForm : EditRecordComponentBase<DbWeatherForecast, WeatherForecastDbContext>
{
    [Inject]
    public WeatherForecastControllerService ControllerService { get; set; }

    private string CardCSS => this.IsModal ? "m-0" : "";

    protected async override Task OnInitializedAsync()
    {
        // Assign the correct controller service
        this.Service = this.ControllerService;
        await base.OnInitializedAsync();
    }
}
```

This gets and assigns the specific ControllerService through DI to the *IContollerService Service* Property.

The Razor Markup below is an abbreviated version of the full file.  This makes extensive use of UIControls which will be covered in detail in a later article.  See the comments for detail. 
```C#
// CEC.Weather/Components/Forms/WeatherForecastEditorForm.razor.cs
// UI Card is a Bootstrap Card
<UICard IsCollapsible="false">
    <Header>
        @this.PageTitle
    </Header>
    <Body>
        // Cascades the Event Handler in the form for RecordChanged.  Picked up by each FormControl and fired when a value changes in the FormControl
        <CascadingValue Value="@this.RecordFieldChanged" Name="OnRecordChange" TValue="Action<bool>">
            // Error handler - only renders it's content when the record exists and is loaded
            <UIErrorHandler IsError="@this.IsError" IsLoading="this.IsDataLoading" ErrorMessage="@this.RecordErrorMessage">
                <UIContainer>
                    // Standard Blazor EditForm control
                    <EditForm EditContext="this.EditContext">
                        // Fluent ValidationValidator for the form
                        <FluentValidationValidator DisableAssemblyScanning="@true" />
                        .....
                        // Example data value row with label and edit control
                        <UIFormRow>
                            <UILabelColumn Columns="4">
                                Record Date:
                            </UILabelColumn>
                            <UIColumn Columns="4">
                                // Note the Record Value bind to the record shadow copy to detect changes from the orginal stored value
                                <FormControlDate class="form-control" @bind-Value="this.Service.Record.Date" RecordValue="this.Service.ShadowRecord.Date"></FormControlDate>
                            </UIColumn>
                        </UIFormRow>
                        ..... // more form rows here
                    </EditForm>
                </UIContainer>
            </UIErrorHandler>
            // Container for the buttons - not record dependant so outside the error handler to allow navigation if UIErrorHandler is in error.
            <UIContainer>
                <UIRow>
                    <UIColumn Columns="7">
                        // Bootstrap alert to display any messages
                        <UIAlert Alert="this.AlertMessage" SizeCode="Bootstrap.SizeCode.sm"></UIAlert>
                    </UIColumn>
                    <UIButtonColumn Columns="5">
                        ....
                        // UIButton is a Bootstrap button.  Show controls whether it's displayed.
                        // For example Save is displayed when the Service Record is Dirty and the record has loaded. 
                        <UIButton Show="(!this.IsClean) && this.IsLoaded" ClickEvent="this.Save" ColourCode="Bootstrap.ColourCode.save">Save</UIButton>
                        <UIButton Show="this.ShowExitConfirmation && this.IsLoaded" ClickEvent="this.ConfirmExit" ColourCode="Bootstrap.ColourCode.danger_exit">Exit Without Saving</UIButton>
                        <UIButton Show="(!this.NavigationCancelled) && !this.ShowExitConfirmation" ClickEvent="(e => this.NavigateTo(PageExitType.ExitToList))" ColourCode="Bootstrap.ColourCode.nav">Exit To List</UIButton>
                        <UIButton Show="(!this.NavigationCancelled) && !this.ShowExitConfirmation" ClickEvent="this.Exit" ColourCode="Bootstrap.ColourCode.nav">Exit</UIButton>
                    </UIButtonColumn>
                </UIRow>
            </UIContainer>
        </CascadingValue>
    </Body>
</UICard>
```
### Base Form Code

At this point we step down from project specific code, to generic library base forms.  Each function/code block is annotated with the name of the source component.  Code blocks/Methods/Base Methods are ordered in the order in which they are executed.

#### Component Event Code

##### OnInitializedAsync

The code block below shows the two OnInitializedAsync methods.

*OnInitializedAsync* is implemented from top down (local code is run before calling the base method).

```c#
// CEC.Weather/Components/Forms/WeatherEditorForm.razor.cs
protected async override Task OnInitializedAsync()
{
    // Assign the correct controller service
    this.Service = this.ControllerService;
    // Set the delay on the record load as this is a demo project
    this.DemoLoadDelay = 250;
    await base.OnInitializedAsync();
}

// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs
protected async override Task OnInitializedAsync()
{
    // Resets the record to blank 
    await this.Service.ResetRecordAsync();
    await base.OnInitializedAsync();
}

// CEC.Blazor/Components/BaseForms/ApplicationComponentBase.cs
protected async override Task OnInitializedAsync()
{
    // Gets the user if we have an AuthenticationState
    if (this.AuthenticationState != null) await this.GetUserAsync();
    await base.OnInitializedAsync();
}
```

##### OnParametersSetAsync

*OnParametersSetAsync* is implemented from bottom up (base method called before any local code).

```C#
// CEC.Blazor/Components/BaseForms/ApplicationComponentBase.cs
protected async override Task OnParametersSetAsync()
{
    await base.OnParametersSetAsync();
    // Get the record if required - see below for method detail
    await this.LoadRecordAsync();
}
```

##### LoadRecordAsync

Record loading code is broken out so it can be used outside the component lifecycle methods.  It's implemented from bottom up (base method is called before any local code).

The primary record load functionaility is in *RecordComponentBase* which gets and loads the record based on ID.  *EditComponentBase* adds the extra functionality for editing rather than just viewing the record. 

```C#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs
protected virtual async Task LoadRecordAsync()
{
    if (this.IsService)
    {
        // Set the Loading flag and call StateNasChanged to force UI changes 
        // in this case making the UIErrorHandler show the loading spinner 
        this.IsDataLoading = true;
        StateHasChanged();

        // Check if we have a query string value in the Route for ID.  If so use it
        if (this.NavManager.TryGetQueryString<int>("id", out int querystringid)) this.ID = querystringid > -1 ? querystringid : this._ID;

        // Check if the component is a modal.  If so get the supplied ID
        else if (this.IsModal && this.Parent.Options.Parameters.TryGetValue("ID", out object modalid)) this.ID = (int)modalid > -1 ? (int)modalid : this.ID;

        // make this look slow to demo the spinner
        if (this.DemoLoadDelay > 0) await Task.Delay(this.DemoLoadDelay);

        // Get the current record - this will check if the id is different from the current record and only update if it's changed
        await this.Service.GetRecordAsync(this._ID, false);

        // Set the error message - it will only be displayed if we have an error
        this.RecordErrorMessage = $"The Application can't load the Record with ID: {this._ID}";

        // Set the Loading flag and call statehaschanged to force UI changes 
        // in this case making the UIErrorHandler show the record or the error message 
        this.IsDataLoading = false;
        StateHasChanged();
    }
}

// CEC.Blazor/Components/BaseForms/EditComponentBase.cs
protected async override Task LoadRecordAsync()
{
    await base.LoadRecordAsync();

    //set up the Edit Context
    this.EditContext = new EditContext(this.Service.Record);

    // Get the actual page Url from the Navigation Manager
    this.RouteUrl = this.NavManager.Uri;
    // Set up this page as the active page in the router service
    this.RouterSessionService.ActiveComponent = this;
    // Wire up the router NavigationCancelled event
    this.RouterSessionService.NavigationCancelled += this.OnNavigationCancelled;
}
```

##### OnAfterRenderAsync

*OnAfterRenderAsync* is implemented from bottom up (base called before any local code is executed).

```C#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs
protected async override Task OnAfterRenderAsync(bool firstRender)
{
    await base.OnAfterRenderAsync(firstRender);
    // Wire up the SameComponentNavigation Event - i.e. we potentially have a new record to load in the same View 
    if (firstRender) this.RouterSessionService.SameComponentNavigation += this.OnSameRouteRouting;
}
```

#### Event Handlers

There are three event handlers wired up in the Component load events.

```c#
// CEC.Blazor/Components/BaseForms/EditComponentBase.cs
// Event handler for a navigation cancelled event raised by the router
protected virtual void OnNavigationCancelled(object sender, EventArgs e)
{
    // Set the boolean properties
    this.NavigationCancelled = true;
    this.ShowExitConfirmation = true;
    // Set up the alert
    this.AlertMessage.SetAlert("<b>THIS RECORD ISN'T SAVED</b>. Either <i>Save</i> or <i>Exit Without Saving</i>.", Bootstrap.ColourCode.danger);
    // Trigger a component State update - buttons and alert need to be sorted
    InvokeAsync(this.StateHasChanged);
}
```
```c#
// CEC.Blazor/Components/BaseForms/EditComponentBase.cs
// Event handler for the Record Form Controls FieldChanged Event
// wired to each control through a cascaded parameter
protected virtual void RecordFieldChanged(bool isdirty)
{
    if (this.EditContext != null)
    {
        // Sort the Service Edit State
        this.Service.SetClean(!isdirty);
        // Set the boolean properties
        this.ShowExitConfirmation = false;
        this.NavigationCancelled = false;
        // Sort the component state based on the edit state
        if (this.IsClean)
        {
            this.AlertMessage.ClearAlert();
            this.RouterSessionService.SetPageExitCheck(false);
        }
        else
        {
            this.AlertMessage.SetAlert("The Record isn't Saved", Bootstrap.ColourCode.warning);
            this.RouterSessionService.SetPageExitCheck(true);
        }
        // Trigger a component State update - buttons and alert need to be sorted
        InvokeAsync(this.StateHasChanged);
    }
}
```
```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs
// Event handler for SameRoute event raised by the router.  The ID Querystring may have changed and we need to load a new record
protected async void OnSameRouteRouting(object sender, EventArgs e)
{
    // Gets the record - checks for a new ID in the querystring and if we have one loads the new record
    await LoadRecordAsync();
}
```

#### Action Button Events

There are four event methods triggered by actionbutton clicks (Save,..).

```c#
// CEC.Blazor/Components/BaseForms/EditRecordComponentBase.cs
/// Save Method called from the Button
protected virtual async Task<bool> Save()
{
    var ok = false;
    // Validate the EditContext
    if (this.EditContext.Validate())
    {
        // Save the Record
        ok = await this.Service.SaveRecordAsync();
        if (ok)
        {
            // Reset the EditContext State to clean
            this.EditContext.MarkAsUnmodified();
            // Set the boolean properties
            this.ShowExitConfirmation = false;
            // Sort the Router session state to clean
            this.RouterSessionService.NavigationCancelledUrl = string.Empty;
        }
        // Set the alert message to the return result
        this.AlertMessage.SetAlert(this.Service.TaskResult);
        // Trigger a component State update - buttons and alert need to be sorted
        this.UpdateState();
    }
    else this.AlertMessage.SetAlert("A validation error occurred.  Check individual fields for the relevant error.", Bootstrap.ColourCode.danger);
    return ok;
}
```
```c#
// CEC.Blazor/Components/BaseForms/EditRecordComponentBase.cs
/// Save and Exit Method called from the Button
protected virtual async void SaveAndExit()
{
    if (await this.Save()) this.ConfirmExit();
}
```
```c#
// CEC.Blazor/Components/BaseForms/EditRecordComponentBase.cs
/// Exit Method called from the Button
protected virtual void Exit()
{
    // Check if we are free to exit (we have a clean record) or need confirmation
    if (this.IsClean) ConfirmExit();
    else this.ShowExitConfirmation = true;
}
```
```c#
// CEC.Blazor/Components/BaseForms/EditRecordComponentBase.cs
/// Confirm Exit Method called from the Button - bail out regardless
protected virtual void ConfirmExit()
{
    // Override the sate to clean - the router only lets us escape if the state is clean.
    this.Service.SetClean();
    // Sort the Router session state
    this.RouterSessionService.NavigationCancelledUrl = string.Empty;
    //turn off page exit checking
    this.RouterSessionService.SetPageExitCheck(false);
    // Sort the exit strategy - where does the user want to exit to.
    if (this.IsModal) ModalExit();
    else
    {
        // Check if we have a Url the user tried to navigate to - default exit to the root
        if (!string.IsNullOrEmpty(this.RouterSessionService.NavigationCancelledUrl)) this.NavManager.NavigateTo(this.RouterSessionService.NavigationCancelledUrl);
        else if (!string.IsNullOrEmpty(this.RouterSessionService.ReturnRouteUrl)) this.NavManager.NavigateTo(this.RouterSessionService.ReturnRouteUrl);
        else this.NavManager.NavigateTo("/");
    }
}
```
```c#
// CEC.Blazor/Components/BaseForms/EditRecordComponentBase.cs
// Cancel Method called from the Button
protected void Cancel()
{
    // Set the boolean properties
    this.ShowExitConfirmation = false;
    this.NavigationCancelled = false;
    // Sort the Router session state
    this.RouterSessionService.NavigationCancelledUrl = string.Empty;
    // Sort the component state based on the edit state
    if (this.IsClean) this.AlertMessage.ClearAlert();
    else this.AlertMessage.SetAlert($"{this.Service.RecordConfiguration.RecordDescription} Changed", Bootstrap.ColourCode.warning);
    // Trigger a component State update - buttons and alert need to be sorted
    this.UpdateState();
}
```

#### Navigation Buttons

*NavigateTo* provides a structured approach to navigation between, and exiting from, forms.  Exit buttons are wired directly to *NavigateTo*, and buttons such as SaveAndExit call it after completing their actions.  

```C#
// CEC.Blazor/Components/BaseForms/ControllerServiceComponentBase.cs
protected virtual void NavigateTo(PageExitType exittype)
{
    this.NavigateTo(new EditorEventArgs(exittype));
}

protected override void NavigateTo(EditorEventArgs e)
{
    if (IsService)
    {
        //check if record name is populated and if not populate it
        if (string.IsNullOrEmpty(e.RecordName)) e.RecordName = this.Service.RecordConfiguration.RecordName;

        // check if the id is set for view or edit.  If not, sets it.
        if ((e.ExitType == PageExitType.ExitToEditor || e.ExitType == PageExitType.ExitToView) && e.ID == 0) e.ID = this._ID;
        base.NavigateTo(e);
    }
}
```
These propagate down to *NavigateTo* in *ApplicationComponentBase*
```c#
// CEC.Blazor/Components/BaseForms/ApplicationComponentBase.cs
protected virtual void NavigateTo(EditorEventArgs e)
{
    switch (e.ExitType)
    {
        case PageExitType.ExitToList:
            this.NavManager.NavigateTo($"/{e.RecordName}/");
            break;
        case PageExitType.ExitToView:
            this.NavManager.NavigateTo($"/{e.RecordName}/View?id={e.ID}");
            break;
        case PageExitType.ExitToEditor:
            this.NavManager.NavigateTo($"/{e.RecordName}/Edit?id={e.ID}");
            break;
        case PageExitType.SwitchToEditor:
            this.NavManager.NavigateTo($"/{e.RecordName}/Edit?id={e.ID}");
            break;
        case PageExitType.ExitToNew:
            this.NavManager.NavigateTo($"/{e.RecordName}/New?qid={e.ID}");
            break;
        case PageExitType.ExitToLast:
            if (!string.IsNullOrEmpty(this.RouterSessionService.ReturnRouteUrl)) this.NavManager.NavigateTo(this.RouterSessionService.ReturnRouteUrl);
            this.NavManager.NavigateTo("/");
            break;
        case PageExitType.ExitToRoot:
            this.NavManager.NavigateTo("/");
            break;
        default:
            break;
    }
}
```
## Implementing Viewer Pages

### The View

The routed view is a very simple.  It contains the routes and the component to load.

```c#
// CEC.Blazor.WASM.Client/Routes/WeatherForecastViewerView.razor
@page "/WeatherForecast/View"
@namespace CEC.Blazor.WASM.Client.Routes
@inherits ApplicationComponentBase

<WeatherViewerForm></WeatherViewerForm>
```

### The Form

The code file is relatively simple, with most of the detail in the razor markup.

```C#
// CEC.Weather/Components/Forms/WeatherViewerForm.razor
public partial class WeatherViewerForm : RecordComponentBase<DbWeatherForecast, WeatherForecastDbContext>
{
    [Inject]
    private WeatherForecastControllerService ControllerService { get; set; }

    public override string PageTitle => $"Weather Forecast Viewer {this.Service?.Record?.Date.AsShortDate() ?? string.Empty}".Trim();

    protected async override Task OnInitializedAsync()
    {
        this.Service = this.ControllerService;
        // Set the delay on the record load as this is a demo project
        this.DemoLoadDelay = 250;
        await base.OnInitializedAsync();
    }

    // Demo code to move between record and demo samerouterouting i.e. only the querystring changing 
    protected void NextRecord(int increment) 
    {
        var rec = (this._ID + increment) == 0 ? 1 : this._ID + increment;
        rec = rec > this.Service.BaseRecordCount ? this.Service.BaseRecordCount : rec;
        this.NavManager.NavigateTo($"/WeatherForecast/View?id={rec}");
    }
}
```

This gets and assigns the specific ControllerService through DI to the *IContollerService Service* Property.

The Razor Markup below is an abbreviated version of the full file.  This makes extensive use of UIControls which will be covered in detail in a later article.  See the comments for detail. 
```C#
// CEC.Weather/Components/Forms/WeatherViewerForm.razor.cs
// UI Card is a Bootstrap Card
<UICard IsCollapsible="false">
    <Header>
        @this.PageTitle
    </Header>
    <Body>
        // Error handler - only renders it's content when the record exists and is loaded
        <UIErrorHandler IsError="@this.IsError" IsLoading="this.IsDataLoading" ErrorMessage="@this.RecordErrorMessage">
            <UIContainer>
                    .....
                    // Example data value row with label and edit control
                    <UIRow>
                        <UILabelColumn Columns="2">
                            Date
                        </UILabelColumn>

                        <UIColumn Columns="2">
                            <FormControlPlainText Value="@this.Service.Record.Date.AsShortDate()"></FormControlPlainText>
                        </UIColumn>

                        <UILabelColumn Columns="2">
                            ID
                        </UILabelColumn>

                        <UIColumn Columns="2">
                            <FormControlPlainText Value="@this.Service.Record.ID.ToString()"></FormControlPlainText>
                        </UIColumn>

                        <UILabelColumn Columns="2">
                            Frost
                        </UILabelColumn>

                        <UIColumn Columns="2">
                            <FormControlPlainText Value="@this.Service.Record.Frost.AsYesNo()"></FormControlPlainText>
                        </UIColumn>
                    </UIRow>
                    ..... // more form rows here
            </UIContainer>
        </UIErrorHandler>
        // Container for the buttons - not record dependant so outside the error handler to allow navigation if UIErrorHandler is in error.
        <UIContainer>
            <UIRow>
                <UIColumn Columns="6">

                    <UIButton Show="this.IsLoaded" ColourCode="Bootstrap.ColourCode.dark" ClickEvent="(e => this.NextRecord(-1))">
                        Previous
                    </UIButton>
                    <UIButton Show="this.IsLoaded" ColourCode="Bootstrap.ColourCode.dark" ClickEvent="(e => this.NextRecord(1))">
                        Next
                    </UIButton>
                </UIColumn>
                <UIButtonColumn Columns="6">
                    <UIButton Show="!this.IsModal" ColourCode="Bootstrap.ColourCode.nav" ClickEvent="(e => this.NavigateTo(PageExitType.ExitToList))">
                        Exit To List
                    </UIButton>
                    <UIButton Show="!this.IsModal" ColourCode="Bootstrap.ColourCode.nav" ClickEvent="(e => this.NavigateTo(PageExitType.ExitToLast))">
                        Exit
                    </UIButton>
                    <UIButton Show="this.IsModal" ColourCode="Bootstrap.ColourCode.nav" ClickEvent="(e => this.ModalExit())">
                        Exit
                    </UIButton>
                </UIButtonColumn>
            </UIRow>
        </UIContainer>
    </Body>
</UICard>
```
### Base Form Code

At this point we step down from project specific code, to generic library base forms.  Everything is a simplified version of the Editor Code without the *EditRecordComponentBase* events.


### Wrap Up
That wraps up this article.  We've looked at the Editor code in detail to see how it works, and then taken a quick look at the Viewer code.  We'll look in more detail at the List components in a separate article.   
Some key points to note:
1. The differences in code between a Blazor Server and a Blazor WASM project are very minor.
2. Almost all the functionality needed is implemented in the library components.  Most of the application code is Razor markup for the individual record fields.
3. Extensive use of Async functionality in the components and CRUD data access.
