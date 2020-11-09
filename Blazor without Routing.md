# An Alternative Approach to Building a Blazor Application

SPA's and routing seem intrinsically joined at the hip.  This articles looks at an alterenative approach to building a Blazor SPA application without a router or URL in sight.

An SPA is an application, not a web site.  Yet we let the web paradigm constrain our thinking.  In a standard desktop application we don't use anything as clumsy as a URL to move around the application, open new forms, edit information,...  So why are we doing it in a SPA, there are alternatives, you just need to start thinking outside the box.

You can see a routerless version of the standard Blazor Weather Application in action at these sites:

[WASM Weather Version](https://cec-blazor-wasm.azurewebsites.net/)

[Blazor Server Version](https://cec-blazor-server.azurewebsites.net/)

There's a GitHub repository [here](https://github.com/ShaunCurtis/CEC.Blazor/tree/Experimental).  Note you need to be on the Experimental Branch.

## Terminology

To be clear about the terms I use in this article:

1. **Page** - this is used sparsely.  A page is a classic HTML web page served up by a web server.  The only page in an application of the initial _Host.cshtml in Server and index.html in WASM.
2. **View** - a View is what gets loaded into App by the Router.  It actually gets loaded into the Layout component.
3. **Layout** - is the Layout component specified by the View.  If none is specified, the default is used.
4. **Form** - is a unit that contains a set of Control components to display information.   Editors/Viewers/Lists are classic forms.  A View can contain one or more Forms, and a Form can be shown as a modal dialog.
5. **Control** - is a component that displays something. Buttons, edit boxes, switches and dropdown lists are all controls.

## App.razor

The Blazor Application runs inside the *\<app\>\</app\>* HTML tags in the base web page.

In a routed Blazor application the *Router* component is at the top of the RenderTree in *App.razor*.  URL navigation is managed by the *NavigationManager*. The router hooks into the *NavigationManager* and receives notifications when a navigation event occurs.  It works out which component needs loading and loads it into the *Layout* component.

```html
<Router AppAssembly="@typeof(Program).Assembly">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
    </Found>
    <NotFound>
        <LayoutView Layout="@typeof(MainLayout)">
            <p>Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>
```
The new App.razor looks like this:

```html
<ViewManager DefaultViewData="@viewData" ModalType="typeof(CEC.Blazor.Components.UIControls.BootstrapModal)" DefaultLayout="@typeof(BaseLayout)">
</ViewManager>

@code {
    public ViewData viewData = new ViewData(typeof(CEC.Weather.Components.Views.Index), null);
}
```

The router has been replaced with a *ViewManager* component.  This becomes the root component of the RenderTree.  All the parameters are *Types*.

1. *DefaultViewData* is the default View for the application - this is the View that gets loaded at startup and is the "Home" View.  All Views must implement *IView*.  The ViewData is constructed in the code as a Property.
2. *ModalType* is the modal dialog frame component we will use for display forms in modal dialog format.  It must implement *IModal*.  More about this later.
3. *DefaultLayout* is the default layout to use for the View.  This is a standard Blazor Layout.

## ViewData

The *ViewData* class holds the data required to render a View.

```c#
public sealed class ViewData
{
    /// The type of the View.
    public Type PageType { get; private set; }

    /// Parameter values to add to the View when created
    public Dictionary<string, object> ViewParameters { get; private set; } = new Dictionary<string, object>();

    /// View values that can be used by the view and subcomponents
    public Dictionary<string, object> ViewFields { get; private set; } = new Dictionary<string, object>();

    /// Constructs an instance of <see cref="ViewData"/>.
    public ViewData(Type pageType, Dictionary<string, object> viewValues)
    {
        if (pageType == null) throw new ArgumentNullException(nameof(pageType));
        if (!typeof(IView).IsAssignableFrom(pageType)) throw new ArgumentException($"The view must implement {nameof(IView)}.", nameof(pageType));
        this.PageType = pageType;
        this.ViewParameters = viewValues;
    }
    .....
    // various methods to update and get Parameters and Fields from ViewParameters and ViewFields
}
```

## ViewManager

We'll look at ViewManager in sections, the full class file is available in the Repo.

```c#
public class ViewManager : IComponent
{

    [Inject] private IJSRuntime _js { get; set; }

    [Parameter] public ViewData DefaultViewData { get; set; }

    [Parameter] public Type ModalType { get; set; } = typeof(BootstrapModal);

    [Parameter] public Type DefaultLayout { get; set; }

    public IModal ModalDialog { get; protected set; }

```
The *ViewManager* implements IComponent directly.  The first section gets the JSRuntime Interop through DI and declares the three parameter based properties.


```c#
        public ViewData ViewData
        {
            get
            {
                if (this._ViewData == null) this._ViewData = this.DefaultViewData;
                return this._ViewData;
            }
            protected set
            {
                this.LastViewData = this._ViewData;
                this._ViewData = value;
            }
        }

        private ViewData _ViewData { get; set; }

        public ViewData LastViewData { get; protected set; }
```

We manage the Viewdata through the *ViewData* property, keeping the previous ViewData so we can go back to it when we exit a form. 

```c#
private readonly RenderFragment _componentRenderFragment;

private bool _RenderEventQueued;

private RenderHandle _renderHandle;

public ViewManager() => _componentRenderFragment = builder =>
{
    this._RenderEventQueued = false;
    BuildRenderTree(builder);
};

public void Attach(RenderHandle renderHandle)
{
    _renderHandle = renderHandle;
}
```

Class initialization builds the component render fragment.  This is passed to the renderer whenever the component needs rendering. *Attach* saves the RenderHandle - it gets called by the Renderer when the component is attaches to the RenderTree.

The RenderTree Builder process looks like this:

```c#
/// Renders the component.
protected virtual void BuildRenderTree(RenderTreeBuilder builder)
{
    // Adds cascadingvalue for the ViewManager
    builder.OpenComponent<CascadingValue<ViewManager>>(0);
    builder.AddAttribute(1, "Value", this);
    // Get the layout render fragment
    builder.AddAttribute(2, "ChildContent", this._layoutViewFragment);
    builder.CloseComponent();
}
```
The component creates a cascading value of itself.  This can be used by all sub components to access the ViewManager.  The *ChildContent* is the Layout either defined by the View or the default Layout.

```c#
private RenderFragment _layoutViewFragment =>
    builder =>
    {
        // Adds the Modal Dialog infrastructure
        var pageLayoutType = ViewData?.PageType?.GetCustomAttribute<LayoutAttribute>()?.LayoutType ?? DefaultLayout;
        builder.OpenComponent(0, ModalType);
        builder.AddComponentReferenceCapture(1, modal => this.ModalDialog = (IModal)modal);
        builder.CloseComponent();
        // Adds the Layout component
        if (pageLayoutType != null)
        {
            builder.OpenComponent<LayoutView>(2);
            builder.AddAttribute(3, nameof(LayoutView.Layout), pageLayoutType);
            // Adds the view render fragment into the layout component
            if (this._ViewData != null)
                builder.AddAttribute(4, nameof(LayoutView.ChildContent), this._viewFragment);
            else
            {
                builder.AddContent(2, this._fallbackFragment);
            }
            builder.CloseComponent();
        }
        else
        {
            builder.AddContent(0, this._fallbackFragment);
        }
    };
```

The Layout fragment - the child content of the *ViewManager* - consists of the Modal Dialog component and the Layout.  The View is the *ChildComponent* of the Layout. 

```c#
/// Render fragment that renders the View
private RenderFragment _viewFragment =>
    builder =>
    {
        // Adds the defined view with any defined parameters
        builder.OpenComponent(0, _ViewData.PageType);
        if (this._ViewData.ViewParameters != null)
        {
            foreach (var kvp in _ViewData.ViewParameters)
            {
                builder.AddAttribute(1, kvp.Key, kvp.Value);
            }
        }
        builder.CloseComponent();
    };
```

The final method defines a fallback fragment to display if there's problems with the Layout or View.
```c#
/// Fallback render fragment if there's no Layout or View specified
private RenderFragment _fallbackFragment =>
    builder =>
    {
        builder.OpenElement(0, "div");
        builder.AddContent(1, "This is the ViewManager's fallback View.  You have no View and/or Layout specified.");
        builder.CloseElement();
    };
```
The "render" method is *ViewHasChanged*.

```c#
public void ViewHasChanged()
{
    if (!this._RenderEventQueued)
    {
        this._RenderEventQueued = true;
        _renderHandle.Render(_componentRenderFragment);
    }
}
```
It queues the component render fragment on to the Renderer's render queue.  Note *_RenderEventQueued* is set to true when a render event is queued, and set to false when the render fragment is actually run.

```c#
public Task SetParametersAsync(ParameterView parameters)
{
    parameters.SetParameterProperties(this);
    this.ViewHasChanged();
    return Task.CompletedTask;
    //return this.LoadView();
}
```
The other side of the *IComponent* interface.  This only gets called on startup as none of the parameters are ever changed once attached to the RenderTree.

The View is updated by form or control components by calling *LoadViewAsync* on the cascaded instance of *ViewManager*.  There are various versions of *LoadViewAsync*, constructing the *ViewData* in various ways.  They all call the core method shown below:

```c#
public Task LoadViewAsync(ViewData viewData = null)
{
    // can be locked by a component if it has a dirty dataset
    if (!this.IsLocked)
    {
        if (viewData != null) this.ViewData = viewData;
        if (ViewData == null)
        {
            throw new InvalidOperationException($"The {nameof(ViewManager)} component requires a non-null value for the parameter {nameof(ViewData)}.");
        }
        this.ViewHasChanged();
    }
    return Task.CompletedTask;
}
```
The method updates the *ViewData* property and then calls *ViewHasChanged* to queue a re-render of the component.

### UIViewLink

*UIViewLink* is similar to *NavLink*.  It constructs a clickable link to navigate to a defined View.

```c#
    public class UIViewLink : UIBase
    {
        /// View Type to Load
        [Parameter] public Type ViewType { get; set; }

        /// View Paremeters for the View
        [Parameter] public Dictionary<string, object> ViewParameters { get; set; } = new Dictionary<string, object>();

        /// Cascaded ViewManager
        [CascadingParameter] public ViewManager ViewManager { get; set; }

        /// Boolean to check if the ViewType is the current loaded view
        /// if so it's used to mark this component's CSS with "active" 
        private bool IsActive => this.ViewManager.IsCurrentView(this.ViewType);

        /// Builds the render tree for the component
        protected override void BuildRenderTree(RenderTreeBuilder builder)
        {
            var css = string.Empty;
            var viewData = new ViewData(ViewType, ViewParameters);

            if (AdditionalAttributes != null && AdditionalAttributes.TryGetValue("class", out var obj))
            {
                css = Convert.ToString(obj, CultureInfo.InvariantCulture);
            }
            if (this.IsActive) css = $"{css} active";
            this.UsedAttributes.Add("@onclick");
            this.UsedAttributes.Add("onclick");
            this.ClearDuplicateAttributes();
            builder.OpenElement(0, "a");
            builder.AddAttribute(1, "class", css);
            builder.AddMultipleAttributes(2, AdditionalAttributes);
            builder.AddAttribute(3, "onclick", EventCallback.Factory.Create<MouseEventArgs>(this, e => this.ViewManager.LoadViewAsync(viewData)));
            builder.AddContent(4, ChildContent);
            builder.CloseElement();
        }
    }
```
### LeftNavMenu

```html
....
    <li class="nav-item px-3">
        <UIViewLink class="nav-link" ViewType="typeof(WeatherForecastListView)">
            <span class="oi oi-cloudy" aria-hidden="true"></span> Normal Weather
        </UIViewLink>
    </li>
    <li class="nav-item px-3">
        <UIViewLink class="nav-link" ViewType="typeof(WeatherForecastListModalView)">
            <span class="oi oi-cloud-upload" aria-hidden="true"></span> Modal Weather
        </UIViewLink>
    </li>
.....
```
```c#
 @code {
    [CascadingParameter] public ViewManager ViewManager { get; set; }
}
```


