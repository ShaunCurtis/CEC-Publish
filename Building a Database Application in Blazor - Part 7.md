# Building a Database Appication in Blazor 
# Part 7 - Some Key Project Infrastructure

This is the final article in this series and looks at various key bits of project and library infrastructure in more detail.  The articles so far are:

1. [Project Structure and Framework](https://www.codeproject.com/Articles/5279560/Building-a-Database-Application-in-Blazor-Part-1-P)
2. [Services - Building the CRUD Data Layers](https://www.codeproject.com/Articles/5279596/Building-a-Database-Application-in-Blazor-Part-2-S)
3. [View Components - CRUD Edit and View Operations in the UI](https://www.codeproject.com/Articles/5279963/Building-a-Database-Application-in-Blazor-Part-3-C)
4. [UI Components - Building HTML/CSS Controls](https://www.codeproject.com/Articles/5280090/Building-a-Database-Application-in-Blazor-Part-4-U)
5. View Components - CRUD List Operations in the UI
6. Adding New Record Types and the UI to Weather Projects
7. Some Key Project Infratructure

The sections covered in this article are:
* Components and Events
* Filtering and Paging
* Entity Framework Extensions
* Select Controls

## Sample Project and Code

The base code is here in the [CEC.Blazor GitHub Repository](https://github.com/ShaunCurtis/CEC.Weather).
The completed code for this article is in [CEC.Weather GitHub Repository](https://github.com/ShaunCurtis/CEC.Weather).

## Components and Events

Much of the project code runs asynchronously.  Async programming requires a different mindset to coding the UI for component initialisation and event handling.  The application no longer lives in a simple sequential world where data processing completes before the UI is rendered, and any updates cause a complete UI re-render.  Components are rendered before their dependant data is ready, and the UI updated piecemeal.

The UI is driven by:

1. The Renderer running *SetParametersAsync* on a component.
2. Component registered Service Events.

### UIErrorHandler

The application uses the *UIErrorHandler* component to handle data loading and UI updates.  We looked at this component article 4 when looking at UI Controls.  *UIErrorHandler* has three states - Loading, Error and Complete.  All UI components/code wrapped within the component as child content is only rendered when the state is complete.  We don't need load handling code in each component on a form - the form only gets loaded when record/recordset loading is complete.

The code below shows the control in action.  The components that depend on the record are inside the *UIErrorHandler* control, while the navigation is outside - it's not dependant on the record.

```c#
// CEC.Weather/Components/Forms/WeatherStationEditorForm.razor.cs

<UICard IsCollapsible="false">
    <Header>
        @this.PageTitle
    </Header>
    <Body>
        <CascadingValue Value="@this.RecordFieldChanged" Name="OnRecordChange" TValue="Action<bool>">
            <UIErrorHandler IsError="@this.IsError" IsLoading="this.IsDataLoading" ErrorMessage="@this.RecordErrorMessage">
                <UIContainer>
                    <EditForm EditContext="this.EditContext">
                        ......
                    </EditForm>
                </UIContainer>
            </UIErrorHandler>
            <UIContainer>
                .... //Navigation Butttons that don't depend on the record 
            </UIContainer>
        </CascadingValue>
    </Body>
</UICard>
```
The boolean properties that control *UIErrorHandler* are:

```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs

protected virtual bool IsRecord => this.Service?.IsRecord ?? false;
protected virtual bool IsError { get => !this.IsRecord; }
protected virtual bool IsDataLoading { get; set; } = true;
protected virtual bool IsLoaded => !(this.IsDataLoading) && !(this.IsError);
```

In the component *SetParametersAsync* cycle, record loading takes place in *OnParametersSetAsync* (*RecordComponentBase*).  *OnParametersSetAsync* is the last step in the *ComponentBase.SetParametersAsync* process.  Once complete, *StateHasChanged* is called on the component, precipitatiing a  re-render. The record is either in error or loaded: *UIErrorHandler* either displays the error message or renders the child content.

```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs

protected async override Task OnParametersSetAsync()
{
    await base.OnParametersSetAsync();
    // Get the record if required
    await this.LoadRecordAsync(true);
}
```
*LoadRecordAsync* looks like this. 

```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs
protected virtual async Task LoadRecordAsync(bool firstload = false)
{
    if (this.IsService)
    {
        // Set the Loading flag 
        //  call StateHasChanged only if we are responding to an event to update UIErrorHandler.  In the component loading cycle it will be called for us shortly
        this.IsDataLoading = true;
        if (!firstload) StateHasChanged();
        ......  //Record loading code
        // Set the Loading flag
        this.IsDataLoading = false;
        //  call StateHasChanged only if we are responding to an event to update UIErrorHandler.  In the component loading cycle it will be called for us shortly
        if (!firstload) StateHasChanged();
    }
}
```
During initialisation *IsDataLoading* is set to true.  If we're reponding to an event it's not, so we just set it.  We only call *StateHasChanged* if *FirstLoad* is false.  *OnParametersSetAsync* specifically calls *LoadRecordAsync(true)*.  After loading the record, we set *IsDataLoading* to false and call *StateHasChanged* if necessary.

The application uses *Querystring* to specify record IDs, it needs to handle situations where the querystring changes but the view doesn't.  The Router detects this and raises a *SameComponentNavigation* event.

We wire up this event in *RecordComponentBase* in *OnAfterRenderAsync*.  In general you should wire up events at the end of the component initialisation cycle rather than earlier to prevent "False Positives", re-re-re-....rendering/loading and sometimes infinite loops. 

```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs

protected async override Task OnAfterRenderAsync(bool firstRender)
{
    await base.OnAfterRenderAsync(firstRender);
    // Wire up the SameComponentNavigation Event - i.e. we potentially have a new record to load in the same View 
    if (firstRender) this.RouterSessionService.SameComponentNavigation += this.OnSameRouteRouting;
}
```
*OnSameRouteRouting* calls *LoadRecordAsync* as shown above.

```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs
protected async void OnSameRouteRouting(object sender, EventArgs e)
{
    System.Diagnostics.Debug.WriteLine($"RecordComponentBase IntraPage Navigation Triggered");
    // Gets the record - checks for a new ID in the querystring and if we have one loads the records
    await LoadRecordAsync();
}
```
### Service Events

The specific controller service manages the events for a record type.  There are six Contoller Service and two Router Service events used by UI components:

1. *RecordHasChanged* - *IControllerService* - triggered when a record is updated or changed
2. *ListHasChanged* - *IControllerService* - triggered when the recordset has changed
3. *FilterUpdated* - *IControllerService* - triggered when the filter has changed.  Normally triggers a recordset update and thus a ListHasChanged event.
4. *OnDirty* - *IControllerService* - triggered during editing when the record set differs from the stored record.
5. *OnClean* - *IControllerService* - triggered when the edit record set is the same as the stored record.
6. *PageChanged* -*IPagingControllerService* - triggered when the List Page has changed.
7. *SameComponentNavigation* - *RouterSessionService* - triggered when *NavigationManager* triggers a navigation event to the same View as Loaded.  
8. *NavigationCancelled* - *RouterSessionService* - triggered when a *NavigationManager* event is cancelled by the Router because the record is dirty.  Used in Edit Forms.

#### List Form Events

The important secton of code form *BaseContontrollerService* for List updates is shown below. Paging operations, triggered by the *PagingControl*, differ from recordset List operations, triggered by filter changes and editing of records in dialogs.  *GetFilteredListAsync* is called as part of the process to get a new page.  However, it only runs when *this.IsRecords* is false.  *this.Records=null* is set to trigger a recordset update whenever the application needs the recordset reloaded using the current filters.  *GetFilteredRecordListAsync* dictates what filtering is applied.  It's virtual and can be overridden.  Filtering is covered in a later section of this article.

```c#
// CEC.Blazor/Components/BaseForms/BaseControllerService.cs
public async virtual Task<bool> GetFilteredListAsync()
{
    // Check if the record set is null. and only refresh the record set if it's null
    if (!this.IsRecords)
    {
        if (this.FilterList.Load) this.Records = await this.Service.GetFilteredRecordListAsync(FilterList);
        else this.Records = new List<TRecord>();
        return true;
    }
    return false;
}
```
*IsRecords* checks the record count. -1 means the recordset is null, while 0 - means the recordset exists.  A record count of 0 is a valid record count.

```c#
// CEC.Blazor/Components/BaseForms/BaseControllerService.cs
public bool IsRecords => this.RecordCount >= 0;
public int RecordCount => this.Records?.Count ?? -1;
```


*ListComponentBase* subscribes to three events:

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs
protected override void OnAfterRender(bool firstRender)
{
    if (firstRender)
    {
        this.Paging.PageHasChanged += this.UpdateUI;
        this.Service.ListHasChanged += this.OnRecordsUpdate;
        this.Service.FilterHasChanged += this.FilterUpdated;
    }
    base.OnAfterRender(firstRender);
}
```
#### PageHasChanged

*Paging* is the *IControllerPageService* interface exposed by the Controller Service.

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs
public IControllerPagingService<TRecord> Paging => this.Service != null ? (IControllerPagingService<TRecord>)this.Service : null;
```
We register for the event so we know when paging changes the displayed page and can trigger a UI update.  UpdateUi is pretty simple - it updates the state which triggers a render of the component, dislaying he new page.

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs
protected void UpdateUI(object sender, int recordno) => this.InvokeAsync(StateHasChanged);
```

#### ListHasChanged

This gets triggered whenever the recordset changes.  We need to reset the list to the new recordset.  The code is shown below:

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs
protected async void OnRecordsUpdate(object sender, EventArgs e)
{
    if (this.IsService)
    {
        // set loading
        this.Loading = true;
        // trigger a UI update which will display loading
        this.StateHasChanged();
        // reload paging based on the new recordset and filter i.e. page 1 with the default sorting.
        await this.Paging.LoadAsync();
    }
    // finished loading
    this.Loading = false;
    // trigger a UI Update to display the new recordset with default paging settings
    this.StateHasChanged();
}
```

*LoadAsync* is shown below.  UIt triggers a further event - PageHasChanged.

```c#
// CEC.Blazor/Components/BaseForms/BaseControllerService.cs
public async Task<bool> LoadAsync()
{
    // Reset the page to 1
    this.CurrentPage = 1;
    // Check if we have a sort column, if not set to the default column
    if (!string.IsNullOrEmpty(this.DefaultSortColumn)) this.SortColumn = this.DefaultSortColumn;
    // Set the sort direction to the default
    this.SortingDirection = DefaultSortingDirection;
    // Check if we have a method loaded in the PageLoaderAsync delegate and if so run it
    if (this.PageLoaderAsync != null) this.PagedRecords = await this.PageLoaderAsync();
    // Set the block back to the start
    await this.ChangeBlockAsync(0);
    //  Force a UI update as everything has changed
    this.PageHasChanged?.Invoke(this, this.CurrentPage);
    return true;
}
```

This is registered in *PagingControl* which again triggers *StateHasChanged* forcing a component update. 

```c#
// CEC.Blazor/Components/FormControls/PagingControl.cs

protected void UpdateUI(object sender, int recordno) => InvokeAsync(StateHasChanged);
```
#### FilterHasChanged

When the filter has changed - triggered by the *FilterControl* - we need to reset the recordset.  The *FilterUpdated* method is shown below. It resets the recordset through *Service.ResetListAsync()*. 

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs

protected virtual async void FilterUpdated(object sender, EventArgs e) => await this.Service.ResetListAsync();
```
*ResetListAsync* sets the recordset to null and then triggers a *ListChanged* Event which causes the full update process described above.

```c#
// CEC.Blazor/Components/BaseForms/RecordComponentBase.cs

public async virtual Task ResetListAsync()
{
    this.Records = null;
    this.BaseRecordCount = await this.Service.GetRecordListCountAsync();
    this.TriggerListChangedEvent(this);
}
```

### List/Recordset Filtering

Filtering controls what gets displayed in recordsets.

*FilterList* implements the *IFilterList* interface.

```c#
// CEC.Blazor/Components/Interfaces/IFilterList.cs
using System;
using System.Collections.Generic;

namespace CEC.Blazor.Components
{
    public interface IFilterList
    {
        public enum FilterViewState
        {
            NotSet = 0,
            Show = 1,
            Hide = 2
        }

        /// Guid to Identify object instance
        public Guid GUID { get => Guid.NewGuid(); }

        /// Boolean to determine if the filter should be shown in the UI
        public bool Show { get => this.ShowState == 0; }

        /// Current state of the Filter
        public FilterViewState ShowState { get; set; }

        /// Dictionary of filters - key is the field/property name and value is the value to filter on
        /// </summary>
        public Dictionary<string, object> Filters { get; set; }

        /// Boolean to tell the list loader if we load an empty recordset if there are no filters set
        /// Set to false when the base recordset is very large
        public bool OnlyLoadIfFilters { get; set; }

        /// Boolean to tell the list loader if it need to load
        public bool Load { get => this.Filters.Count > 0 || !this.OnlyLoadIfFilters; }

        /// Method to reset the filter
        public void Reset() +> this.ShowState = IFilterList.FilterViewState.NotSet;

        /// Method to get a Filter value
        public bool TryGetFilter(string name, out object value)
        {
            value = null;
            if (Filters.ContainsKey(name)) value = this.Filters[name];
            return value != null;
        }

        /// Method to set a Filter
        public bool SetFilter(string name, object value)
        {
            if (Filters.ContainsKey(name)) this.Filters[name] = value;
            else Filters.Add(name, value);
            return Filters.ContainsKey(name);
        }

        /// Method to clear a filter out of the filter list
        public bool ClearFilter(string name)
        {
            if (Filters.ContainsKey(name)) this.Filters.Remove(name);
            return !Filters.ContainsKey(name);
        }
    }
}

```
```c#
using System.Collections.Generic;

namespace CEC.Blazor.Components
{
    public class FilterList : IFilterList
    {
        public Dictionary<string, object> Filters { get; set; } = new Dictionary<string, object>();

        public IFilterList.FilterViewState ShowState { get; set; } = IFilterList.FilterViewState.NotSet;

        public bool OnlyLoadIfFilters { get; set; } = false;
    }
}
```




## Filtering

Filter loading is part of *OnParametersSetAsync* process in *ListComponentBase*

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs

protected async override Task OnParametersSetAsync()
{
    await base.OnParametersSetAsync();
    // Load the page - as we've reset everything this will be the first page with the default filter
    if (this.IsService)
    {
        // Load the filters for the recordset
        this.LoadFilter();
        // Load the paged recordset
        await this.Service.LoadPagingAsync();
    }
    this.Loading = false;
}

/// Method called to load the filter
protected virtual void LoadFilter()
{
    if (IsService) this.Service.FilterList.OnlyLoadIfFilters = this.OnlyLoadIfFilter;
}
```

*WeatherReportListForm* overrides *LoadFilter* to set up the record specific filters.

```c#
// CEC.Weather/Components/Forms/WeatherReport/WeatherReportListForm.razor.cs
.....
[Parameter]
public int WeatherStationID { get; set; }
.......
/// inherited - loads the filter
protected override void LoadFilter()
{
    // Before the call to base so the filter is set before the get the list
    if (this.IsService &&  this.WeatherStationID > 0)
    {
        this.Service.FilterList.Filters.Clear();
        this.Service.FilterList.Filters.Add("WeatherStationID", this.WeatherStationID);
    }
    base.LoadFilter();
}
......
```

To digress a little, the Filter magic is again in a DbContext extension.

```c#
// CEC.Blazor/Extensions/DbContextExtensions.cs
public async static Task<List<TRecord>> GetRecordFilteredListAsync<TRecord>(this DbContext context, IFilterList filterList, string dbSetName = null) where TRecord : class, IDbRecord<TRecord>
{
    var firstrun = true;
    // Get the PropertInfo object for the record DbSet
    var propertyInfo = context.GetType().GetProperty(dbSetName ?? IDbRecord<TRecord>.RecordName);
    // Get the actual value and cast it correctly
    var dbset = (DbSet<TRecord>)(propertyInfo.GetValue(context));
    // Get a empty list
    var list = new List<TRecord>();
    // if we have a filter go through each filter
    // note that only the first filter runs a SQL query against the database
    // the rest are run against the dataset.  So do the biggest slice with the first filter for maximum efficiency.
    if (filterList != null && filterList.Filters.Count > 0)
    {
        foreach (var filter in filterList.Filters)
        {
            // Get the filter propertyinfo object
            var x = typeof(TRecord).GetProperty(filter.Key);
            // if we have a list already apply the filter to the list
            if (list.Count > 0) list.Where(item => x.GetValue(item) == filter.Value).ToList();
            // If this is the first run we query the database directly
            else if (firstrun) list = await dbset.FromSqlRaw($"SELECT * FROM vw_{ propertyInfo.Name} WHERE {filter.Key} = {filter.Value}").ToListAsync();
            firstrun = false;
        }
    }
    //  No list, just get the full recordset
    else list = await dbset.ToListAsync();
    return list;
}
```


Add a new control called *MonthYearIDListFilter*. This is used in the *WestherReport* list View to filter the records.  The code is below.

```c#
// CEC.Weather/Components/Controls/MonthYearIDListFilter.razor.cs
using CEC.Blazor.Data;
using CEC.Weather.Data;
using CEC.Weather.Services;
using Microsoft.AspNetCore.Components;
using Microsoft.AspNetCore.Components.Forms;
using System.Collections.Generic;
using System.Globalization;
using System.Threading.Tasks;

namespace CEC.Weather.Components
{
    public partial class MonthYearIDListFilter : ComponentBase
    {
        // Inject the Controller Service
        [Inject]
        private WeatherReportControllerService Service { get; set; }

        // Boolean to control the ID Control Display
        [Parameter]
        public bool ShowID { get; set; } = true;

        // Month Lookup List
        private SortedDictionary<int, string> MonthLookupList { get; set; }

        // Year Lookup List
        private SortedDictionary<int, string> YearLookupList { get; set; }

        // Weather Station Lookup List
        private SortedDictionary<int, string> IdLookupList { get; set; }

        // Dummy Edit Context for selects
        private EditContext EditContext => new EditContext(this.Service.Record);

        // privates to hold current select values
        private int OldMonth = 0;
        private int OldYear = 0;
        private long OldID = 0;

        // Month value - adds or removes the value from the filter list and kicks off Filter changed if changed
        private int Month
        {
            get => this.Service.FilterList.TryGetFilter("Month", out object value) ? (int)value : 0;
            set
            {
                if (value > 0) this.Service.FilterList.SetFilter("Month", value);
                else this.Service.FilterList.ClearFilter("Month");
                if (this.Month != this.OldMonth)
                {
                    this.OldMonth = this.Month;
                    this.Service.TriggerFilterChangedEvent(this);
                }
            }
        }

        // Year value - adds or removes the value from the filter list and kicks off Filter changed if changed
        private int Year
        {
            get => this.Service.FilterList.TryGetFilter("Year", out object value) ? (int)value : 0;
            set
            {
                if (value > 0) this.Service.FilterList.SetFilter("Year", value);
                else this.Service.FilterList.ClearFilter("Year");
                if (this.Year != this.OldYear)
                {
                    this.OldYear = this.Year;
                    this.Service.TriggerFilterChangedEvent(this);
                }
            }
        }

        // Weather Station value - adds or removes the value from the filter list and kicks off Filter changed if changed
        private int ID
        {
            get => this.Service.FilterList.TryGetFilter("WeatherStationID", out object value) ? (int)value : 0;
            set
            {
                if (value > 0) this.Service.FilterList.SetFilter("WeatherStationID", value);
                else this.Service.FilterList.ClearFilter("WeatherStationID");
                if (this.ID != this.OldID)
                {
                    this.OldID = this.ID;
                    this.Service.TriggerFilterChangedEvent(this);
                }
            }
        }

        protected override async Task OnInitializedAsync()
        {
            this.OldYear = this.Year;
            this.OldMonth = this.Month;
            await GetLookupsAsync();
        }

        // Method to get he LokkupLists
        protected async Task GetLookupsAsync()
        {
            this.IdLookupList = await this.Service.GetLookUpListAsync<DbWeatherStation>("-- ALL STATIONS --");
            // Get the months in the year
            this.MonthLookupList = new SortedDictionary<int, string> { { 0, "-- ALL MONTHS --" } };
            for (int i = 1; i < 13; i++) this.MonthLookupList.Add(i, CultureInfo.CurrentCulture.DateTimeFormat.GetMonthName(i));
            // Gets a distinct list of Years in the Weather Reports
            {
                var list = await this.Service.GetDistinctListAsync(new DbDistinctRequest() { FieldName = "Year", QuerySetName = "WeatherReport", DistinctSetName = "DistinctList" });
                this.YearLookupList = new SortedDictionary<int, string> { { 0, "-- ALL YEARS --" } };
                list.ForEach(item => this.YearLookupList.Add(int.Parse(item), item));
            }

        }
    }
}
```
```c#
// CEC.Weather/Components/Controls/MonthYearIDListFilter.razor
@using CEC.Blazor.Components.FormControls
@using Microsoft.AspNetCore.Components.Forms

@namespace CEC.Weather.Components
@inherits ComponentBase

<EditForm EditContext="this.EditContext">

    <table class="table">
        <tr>
            @if (this.ShowID)
            {
                <!--Farm-->
                <td>
                    <label class="" for="ID">Weather Station:</label>
                    <div class="">
                        <InputControlSelect OptionList="this.IdLookupList" @bind-Value="this.ID"></InputControlSelect>
                    </div>
                </td>
            }
            <td>
                <!--Month-->
                <label class="">Month:</label>
                <div class="">
                    <InputControlSelect OptionList="this.MonthLookupList" @bind-Value="this.Month"></InputControlSelect>
                </div>
            </td>
            <td>
                <!--Year-->
                <label class="">Year:</label>
                <div class="">
                    <InputControlSelect OptionList="this.YearLookupList" @bind-Value="this.Year"></InputControlSelect>
                </div>
            </td>
        </tr>
    </table>
</EditForm>
```
 The filter displays a set of dropdowns.  When you change a value, the value is added, updated or deleted in the filter list and the service FilterUpdated event is kicked off.  The List Form has registered with this event, resets and reloads the recordset (with the updated filter) and refresh the display.

```c#
// CEC.Blazor/Components/BaseForms/ListComponentBase.cs
.........
protected override void OnAfterRender(bool firstRender)
{
    if (firstRender)
    {
        this.Paging.PageHasChanged += this.UpdateUI;
        this.Service.ListHasChanged += this.OnRecordsUpdate;
        // Register the FilterChanged event hsandler with the Service FilterHasChanged event
        this.Service.FilterHasChanged += this.FilterUpdated;
    }
    base.OnAfterRender(firstRender);
}
........
protected virtual async void FilterUpdated(object sender, EventArgs e)
{
    // reset the List
    await this.Service.ResetListAsync();
    // kick off a Paging Load
    await this.Paging.LoadAsync();
    // Update the UI to show the new recordset
    await InvokeAsync(this.StateHasChanged);
}
.......
```

### Wrap Up
This article demonstrates how to build out either a Blazor WASM or Server project from the CEC.Blazor library though adding a more record types to the Weather Application.

The final article will look at deploying Blazor Applications.