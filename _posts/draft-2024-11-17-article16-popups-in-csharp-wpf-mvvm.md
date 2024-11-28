---
layout: post
title: "Yet another way to implement modal popups and dialogs in WPF MVVM"
categories: misc
tags:
- dotnet
- WPF
- MVVM
- .NET
- C#
- desktop
- Windows
cover-img: /assets/img/article15/cover-image.png
---

In this article I will present `pure` MVVM approach to implement modal popups in WPF applications. 

## modal dialog, popup, message box
There are some conceptual differences between popup, dialog, message box and modal (very well described at https://medium.com/design-bootcamp/popups-dialogs-tooltips-and-popovers-ux-patterns-2-939da7a1ddcd). In this article I will focus on applications's visual elements displayed on the top of the user interface, which require user interaction. I will refer them as **modal dialogs**. 

## Motivation
Over the time I saw multiple implementations and ideas on how to deal with modal dialog in WPF application. 

Lets consider a requirement to display a newsletter modal popup, which offers newsletter sign up. User can specify the email address for newsletter, but by default user's current email is used. Additionally, user has a possibility to chose `remind me later`, or `dismiss` options. Assuming the `NewsletterPopupView` is responsible for executing the sign up logic - the simplest approach to display the modal dialog would be:

```csharp
//MainViewModel

if(!User.RemindNewsletter || User.NewsletterActive)
    return;

var popupView = new NewsletterPopupView();
popupView.DataContext = new NewsletterPopupViewModel(_newsLetterService);
popupView.DataContext.UserEmail = User.Email;
if(popupView.ShowDialog())
{
    if(popupView.DataContext.Result = NewsletterPopupResult.Dismiss)
        User.RemindNewsletter = false;
    else if(popupView.DataContext.Result = NewsletterPopupResult.RemindLater)
        User.RemindNewsletter = true;
    else if(popupView.DataContext.Result = NewsletterPopupResult.SignUp)
    {
        User.NewsletterActive = true;
        User.NewsletterEmailAddress = popupView.DataContext.EmailAddress;
    }
}
```

Despite obvious simplicity, presented solution has some downsides:
* `Widndow.ShowDialog()` returns `bool?` based on the shown Window's `DialogResult` property. In cases, when modal dialog returns more complex result, it needs to be stored in dialog's DataContext, so it can be consumed by parent ViewModel after modal dialog is closed. There are also some use cases where popup returns more data than just the result.
* Displaying modal as separate window, often doesn't align with modern designs. Nowadays modal dialogs tend to have flat appearance, embedded into application main window. (See the example of Spotify desktop application below) 
   ![spotify-dialog](/assets/img/article16/spotify-dialog.png)
* Modal dialog initialization is achieved by setting DataContext's properties. (Of course property initialization could be done by `NewsletterPopupViewModel` constructor, however constructor initialization is often dedicated for dependency injection, especially when modal dialog uses some services to perform the business logic). In such case developer who uses `NewsletterPopupViewModel` is not directly instructed if any input properties are required to be set to initialize modal dialog. Moreover it is hard to distinguish which properties are required, and which are optional. 
* Parent window ViewModel becomes responsible for instantiating modal dialog's View (or at least holds modal dialog's View reference), which is questionable practice in MVVM pattern.
* `ShowDialog` will block the UI thread, There is no `ShowDialogAsync` method on available in WPF.

## Solution

### DialogView

The main element of the solution is base modal dialog UserControl (`DialogView.xaml`) and its generic ViewModel - (`DialogViewModel.cs`). 

`DialogView.xaml` is a is a visual placeholder for the popup content.

```xml
<UserControl x:Class="FlatWpfDialog.Views.DialogView"
            ...
            >
    <Grid Visibility="{Binding IsVisible, Converter={StaticResource BooleanToVisibilityConverter}, FallbackValue=Collapsed}">
        <Grid.Background>
            <SolidColorBrush Color="White" Opacity="0.8" />
        </Grid.Background>

        <Border HorizontalAlignment="Center" VerticalAlignment="Center" 
                Background="White" BorderBrush="DarkGray" BorderThickness="1" Padding="20">
            <ContentControl Focusable="False" Content="{Binding DialogContentViewModel}">
                <FrameworkElement.Resources>
                    <ResourceDictionary>

                        <DataTemplate DataType="{x:Type viewModels:NewsletterDialogContentViewModel}">
                            <views:NewsletterDialogContentView/>
                        </DataTemplate>
                        
                        <!-- data templates of other dialog contents -->

                    </ResourceDictionary>
                </FrameworkElement.Resources>
            </ContentControl>
        </Border>

    </Grid>
</UserControl>
```
`DialogView` can be now used as the last element of MainWindow View.

```xml
<Window ..>
    <Grid>

    <!-- MainWindowContent -->

    <views:DialogView/>
    </Grid>
</Window>
```

When `DialogViewModel`, bound to the `DialogView` has `IsVisible` property set to `true`, the popup is displayed.

### DialogViewModel

`DialogViewModel.cs`,  bound to above `DialogView`, has two crucial properties
* `bool IsVisible` - bind to UserControl's Visibility - controls if popup is displayed or not
* `object? DialogContentViewModel` - ViewModel which represents the DataContext for the View that will be displayed as the `DialogView` content (since `DialogView` can be treated as a placeholder for dialog content View)

Generic dialog can be displayed using `ShowAsync` function 

```csharp
public partial class DialogViewModel : ObservableObject
{
    [ObservableProperty]
    private object? _dialogContentViewModel;

    [ObservableProperty]
    private bool _isVisible = false;

    public async Task<TOutput> ShowAsync<TInput, TOutput>(IDialogContentViewModel<TInput, TOutput> dialogContentViewModel, TInput input) where TInput : IDialogContentInput where TOutput : IDialogContentOutput
    {
        IsVisible = true;
        DialogContentViewModel = dialogContentViewModel;

        var taskCompletionSource = new TaskCompletionSource<TOutput>();
        
        dialogContentViewModel.Initialize(input, taskCompletionSource);
        
        var result = await taskCompletionSource.Task.WaitAsync(CancellationToken.None);

        IsVisible = false;

        return result;
    }
    
}
```

`ShowAsync` method requires some explanation.

It is generic function, which returns object of type `IDialogContentOutput`. `IDialogContentOutput` is just a marker interface, for an implementation representing the result of interacting with the dialog.

As an input it takes the implementation of `IDialogContentInput` placeholder interface, which represents the input to the dialog. Additionally it takes the ViewModel representing the DataContext of the actual content View, which will be displayed in the `DialogView`. Passed ViewModel needs to implement `IDialogContentViewModel<TInput, TOutput>` interface. 

```csharp
public interface IDialogContentViewModel<TInput, TOutput> where TInput : IDialogContentInput where TOutput : IDialogContentOutput
{
    void Initialize(TInput parameters, TaskCompletionSource<TOutput> taskCompletionSource);
}
```
`IDialogContentViewModel` defines `Initialize` method which takes the dialog content input, and `TaskCompletionSource` that can return and output. 

Thanks to `TaskCompletionSource` usage, the implementation of `IDialogContentViewModel` can set the result using 
`TaskCompletionSource.SetResult<T>(T result)` method. Until that happens `DialogViewModel` waits for the result asynchronously using `TaskCompletionSource.Task.WaitAsync(CancellationToken ct)` method.

The logic of `ShowAsync` method can be summarized in following points
* make dialog visible
* initialize content ViewModel with input values, and provide `TaskCompletionSource`
* wait until `TaskCompletionSource` is set with content output
* hide dialog
* return dialog output

> The complete code, including Newsletter modal dialog example, is available at [my github](https://github.com/melmanm/FlatWpfDialog).

## Solution Pros

* Modal dialog input and output is clearly defined
* UI thread is not blocked, since dialog waits for the interaction output asynchronously

## Solution Cons

* ViewModel - the DataContext of the dialog content View - is not obligated to call `TaskCompletionSource.SetResult<T>(T result)` on the `TaskCompletionSource` passed in `Initialize` function. In result function might be never called, leaving the dialog in open state.
