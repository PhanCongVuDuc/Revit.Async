# Revit.Async
Use Task-based asynchronous pattern (TAP) to run Revit API code from any execution context.

[中文说明](说明.md)

[![NuGet Link](https://img.shields.io/nuget/v/Revit.Async)](https://www.nuget.org/packages/Revit.Async/)
# Background

If you've encountered the Revit API exception "Cannot execute Revit API outside of Revit API context"—often when running code from a modeless window—this library may help.

A typical solution is to wrap Revit API code in an `IExternalEventHandler`, register it with Revit, and trigger it using an `ExternalEvent`. However, once triggered, you cannot easily determine when the handler executes or retrieve its results from the same context. Manually yielding control back to the calling context is usually required.

This process resembles the "Promise" mechanism in JavaScript ES6. Using the task-based asynchronous pattern (TAP) in .NET—commonly known as `Task<T>`—Revit.Async internally wraps your code in an `IExternalEventHandler` and returns results to the calling context. This allows you to run Revit API code naturally from any context.

For more on the task-based asynchronous pattern (TAP), see these Microsoft resources:

- [Task-based asynchronous pattern (TAP)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Task asynchronous programming model](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model)

Here is a [diagram comparing the Revit API external event mechanism with Revit.Async](https://drive.google.com/file/d/1sb6Yrlt6zjkE9XBh4UB5sWV_i8nTpkmG/view?usp=sharing) and
screenshots of the two main parts:

## Revit API External Event

![Revit API](RevitExternalEvent.png)

## Revit.Async
![Revit.Async](Revit.Async.png)

# Revit.Async and Multithread
I am often asked if Revit.Async runs the Revit API in a background thread. The answer is **no**. The term "Async" is misleading; it originates from .NET's naming convention for certain multithreaded methods, but it does not mean multithreading in this context.

To clarify, we must distinguish between asynchronous and multithreaded programming. As a Stack Overflow answer explains: "Threading is about workers; asynchrony is about tasks." Consider this analogy:

*   **Synchronous:** You cook eggs, then you cook toast.
*   **Asynchronous, single-threaded:** You start the eggs and set a timer. You start the toast and set a timer. While they cook, you clean the kitchen. When the timers go off, you serve the food.
*   **Asynchronous, multithreaded:** You hire two additional cooks to handle the eggs and toast separately, which requires coordination and resource management.

The common misconception that "asynchronous equals multithreaded" arises because multithreading operations often use asynchronous. In most UI applications, a background thread runs a task, but the results must be sent back to the main UI thread for presentation. The asynchronous pattern handles this "go back" phase.

This is consistent across frameworks:
*   In Windows Forms, you use `Invoke` to queue a delegate to the main thread.
*   In WPF, you use the `Dispatcher` to queue a delegate to the main thread.

Similarly, the Revit API requires all model updates to be executed on the main thread for thread safety. To update models from a worker thread, you must use an `ExternalEvent` to queue (via `Raise()`) an `IExternalEventHandler` instance, which will call the Revit API on the main thread. This is the asynchronous pattern Revit provides.

Revit.Async is simply a wrapper around this built-in pattern. Its goal is to provide an out-of-the-box experience for asynchronous Revit API calls. There is **no** multithreading involved in Revit.Async.

# Usage
## Initialize
In any valid Revit API context, initialize RevitTask before you use any functionality of RevitTask.
```csharp
RevitTask.Initialize(app);
```

Some of the valid Revit API contexts are:
* IExternalCommand.Execute method
* IExternalApplication.OnStartup method
* IExternalEventHandler.Execute method
* Revit API event handlers
* IUpdater

## RunAsync

The core functionality of Revit.Async is provided by the `RevitTask.RunAsync()` method, which has several overloads.

### Execute Synchronous Code
*   **Without Return Value**
    *   `Task RunAsync(Action action)`
    *   `Task RunAsync(Action<UIApplication> action)`
    ```csharp
    // Without UIApplication
    await RevitTask.RunAsync(() => { /* sync code */ });

    // With UIApplication
    await RevitTask.RunAsync((uiApp) => { /* sync code with Revit DB access */ });
    ```

*   **With Return Value**
    *   `Task<T> RunAsync<T>(Func<T> func)`
    *   `Task<T> RunAsync<T>(Func<UIApplication, T> func)`
    ```csharp
    // Without UIApplication
    var result = await RevitTask.RunAsync(() => { return 0; });

    // With UIApplication
    var result = await RevitTask.RunAsync((uiApp) => { return 0; });
    ```

### Execute Asynchronous Code
*   **Without Return Value**
    *   `Task RunAsync(Func<Task> func)`
    *   `Task RunAsync(Func<UIApplication, Task> func)`
    ```csharp
    // Without UIApplication
    await RevitTask.RunAsync(async () => { /* async code */ });

    // With UIApplication
    await RevitTask.RunAsync(async (uiApp) => { /* async code with Revit DB access */ });
    ```

*   **With Return Value**
    *   `Task<T> RunAsync<T>(Func<Task<T>> func)`
    *   `Task<T> RunAsync<T>(Func<UIApplication, Task<T>> func)`
    ```csharp
    // Without UIApplication
    var result = await RevitTask.RunAsync(async () =>
    {
        var httpResponse = await http.Get("server api url");
        return httpResponse;
    });

    // With UIApplication
    var result = await RevitTask.RunAsync(async (uiApp) =>
    {
        var httpResponse = await http.Get("server api url");
        return httpResponse;
    });
    ```

# Examples

## Standard approach (without Revit.Async)

```csharp

[Transaction(TransactionMode.Manual)]
public class MyRevitCommand : IExternalCommand
{
    public static ExternalEvent SomeEvent { get; set; }
    public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
    {
        //Register MyExternalEventHandler ahead of time
        SomeEvent = ExternalEvent.Create(new MyExternalEventHandler());
        var window = new MyWindow();
        //Show modeless window
        window.Show();
        return Result.Succeeded;
    }
}

public class MyExternalEventHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        //Running some Revit API code here to handle the button click
        //It's complicated to accept argument from the calling context and return value to the calling context
        var families = new FilteredElementCollector(app.ActiveUIDocument.Document)
                            .OfType(typeof(Family))
                            .ToList();
        //ignore some code
    }
}

public class MyWindow : Window
{
    public MyWindow()
    {
        InitializeComponents();
    }

    private void InitializeComponents()
    {
        Width                 = 200;
        Height                = 100;
        WindowStartupLocation = WindowStartupLocation.CenterScreen;
        var button = new Button
        {
            Content             = "Button",
            Command             = new ButtonCommand(),
            VerticalAlignment   = VerticalAlignment.Center,
            HorizontalAlignment = HorizontalAlignment.Center
        };
        Content = button;
    }
}

public class ButtonCommand : ICommand
{    
    public bool CanExecute(object parameter)
    {
        return true;
    }

    public event EventHandler CanExecuteChanged;

    public void Execute(object parameter)
    {
        //Running Revit API code directly here will result in a "Running Revit API outside of Revit API context" exception
        //Raise a predefined ExternalEvent instead
        MyRevitCommand.SomeEvent.Raise();
    }
}
```

## Revit.Async Approach

```csharp
[Transaction(TransactionMode.Manual)]
public class MyRevitCommand : IExternalCommand
{
    public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
    {
        //Always initialize RevitTask ahead of time within Revit API context
        // version 1.x.x
        // RevitTask.Initialze();

        // version 2.x.x
        RevitTask.Initialize(commandData.Application);
        var window = new MyWindow();
        //Show modeless window
        window.Show();
        return Result.Succeeded;
    }
}

public class MyWindow : Window
{
    public MyWindow()
    {
        InitializeComponents();
    }

    private void InitializeComponents()
    {
        Width                 = 200;
        Height                = 100;
        WindowStartupLocation = WindowStartupLocation.CenterScreen;
        var button = new Button
        {
            Content             = "Button",
            Command             = new ButtonCommand(),
            CommandParameter    = true,
            VerticalAlignment   = VerticalAlignment.Center,
            HorizontalAlignment = HorizontalAlignment.Center
        };
        Content = button;
    }
}

public class ButtonCommand : ICommand
{    
    public bool CanExecute(object parameter)
    {
        return true;
    }

    public event EventHandler CanExecuteChanged;

    public async void Execute(object parameter)
    {
        //.NET 4.5 supported keyword, use ContinueWith if using .NET 4.0
        var families = await RevitTask.RunAsync(
            app => 
            {
                //Run Revit API code here
                
                //Taking advantage of the closure created by the lambda expression,
                //we can make use of the argument passed into the Execute method.
                //Let's assume it's a boolean indicating whether to filter families that is editable
                if(parameter is bool editable)
                {
                    return new FilteredElementCollector(app.ActiveUIDocument.Document)
                        .OfType(typeof(Family))
                        .Cast<Family>()
                        .Where(family => editable ? family.IsEditable : true)
                        .ToList();
                }
                
                return null;
            });
        
        MessageBox.Show($"Family count: {families?.Count ?? 0}");
    }
}
```

## Define Your Own Handler

Tired of the limited `IExternalEventHandler` interface? Switch to `IGenericExternalEventHandler<TParameter, TResult>`. This interface lets you pass arguments to a handler and receive a result upon completion.

For best results, use the predefined abstract classes, which manage argument passing and result returning for you.

| Class                                                   | Description                       |
| ------------------------------------------------------- | --------------------------------- |
| `AsyncGenericExternalEventHandler<TParameter, TResult>` | Use to execute asynchronous logic |
| `SyncGenericExternalEventHandler<TParameter, TResult>`  | Use to execute synchronize logic  |

```csharp
[Transaction(TransactionMode.Manual)]
public class MyRevitCommand : IExternalCommand
{
    public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
    {
        //Always initialize RevitTask ahead of time within Revit API context
        // version 1.x.x
        // RevitTask.Initialze();

        // version 2.x.x
        RevitTask.Initialize(commandData.Application);
        //Register SaveFamilyToDesktopExternalEventHandler ahead of time
        RevitTask.RegisterGlobal(new SaveFamilyToDesktopExternalEventHandler());
        var window = new MyWindow();
        //Show modeless window
        window.Show();
        return Result.Succeeded;
    }
}

public class MyWindow : Window
{
    public MyWindow()
    {
        InitializeComponents();
    }

    private void InitializeComponents()
    {
        Width                 = 200;
        Height                = 100;
        WindowStartupLocation = WindowStartupLocation.CenterScreen;
        var button = new Button
        {
            Content             = "Save Random Family",
            Command             = new ButtonCommand(),
            CommandParameter    = true,
            VerticalAlignment   = VerticalAlignment.Center,
            HorizontalAlignment = HorizontalAlignment.Center
        };
        Content = button;
    }
}

public class ButtonCommand : ICommand
{    
    public bool CanExecute(object parameter)
    {
        return true;
    }

    public event EventHandler CanExecuteChanged;

    public async void Execute(object parameter)
    {
        var savePath = await RevitTask.RunAsync(
            async app =>
            {
                try
                {
                    var document = app.ActiveUIDocument.Document;
                    var randomFamily = await RevitTask.RunAsync(
                        () =>
                        {
                            var families = new FilteredElementCollector(document)
                                .OfClass(typeof(Family))
                                .Cast<Family>()
                                .Where(family => family.IsEditable)
                                .ToArray();
                            var random = new Random(Environment.TickCount);
                            return families[random.Next(0, families.Length)];
                        });

                    //Raise your own handler
                    return await RevitTask.RaiseGlobal<SaveFamilyToDesktopExternalEventHandler, Family, string>(randomFamily);
                }
                catch (Exception)
                {
                    return null;
                }
            });
        var saveResult = !string.IsNullOrWhiteSpace(savePath);
        MessageBox.Show($"Family {(saveResult ? "" : "not ")}saved:\n{savePath}");
        if (saveResult)
        {
            Process.Start(Path.GetDirectoryName(savePath));
        }
    }
}

public class SaveFamilyToDesktopExternalEventHandler : 			
	SyncGenericExternalEventHandler<Family, string>
{
    public override string GetName()
    {
        return "SaveFamilyToDesktopExternalEventHandler";
    }

    protected override string Handle(UIApplication app, Family parameter)
    {
        //write sync logic here
        var document       = parameter.Document;
        var familyDocument = document.EditFamily(parameter);
        var desktop = Environment.GetFolderPath(Environment.SpecialFolder.DesktopDirectory);
        var path = Path.Combine(desktop, $"{parameter.Name}.rfa");
        familyDocument.SaveAs(path, new SaveAsOptions {OverwriteExistingFile = true});
        return path;
    }
}
```

# Todo

- Check current context to decide whether to create an IExternalEventHandler or to run code directly
- Support progress
- Support cancellation

# Issues

Feel free to contact me by 303353762@qq.com if you have any issue using this library.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=KennanChan/Revit.Async&type=Date)](https://star-history.com/#KennanChan/Revit.Async&Date)

