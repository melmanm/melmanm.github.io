---
layout: post
title: "Yet another way to implement modal dialogs in WPF using MVVM pattern"
categories: misc
tags:
- dotnet
- WPF
- MVVM
- .NET
- C#
- desktop
- Windows
cover-img: /assets/img/article16/cover-image.png
---

In this article I will present cool, MVVM based, approach to implement modal dialogs in WPF applications. 

The complete code, including example, is available at [my github](https://github.com/melmanm/FlatWpfDialog).

The code presented in this article uses [CommunityToolkit.Mvvm](https://www.nuget.org/packages/CommunityToolkit.Mvvm) nuget package, but general concepts are not dependant on it.

## Table of contents <!-- omit from toc -->
- [modal dialog, popup, message box](#modal-dialog-popup-message-box)
- [Motivation](#motivation)
- [Solution](#solution)
  - [DialogView](#dialogview)
  - [DialogViewModel](#dialogviewmodel)
- [Solution Pros](#solution-pros)
- [Solution Cons](#solution-cons)
- [Usage](#usage)
- [Full example](#full-example)

## modal dialog, popup, message box
There are some conceptual differences between popup, dialog, message box and modal (very well described at https://medium.com/design-bootcamp/popups-dialogs-tooltips-and-popovers-ux-patterns-2-939da7a1ddcd). In this article I will present interesting approach on how to implement, applications's visual elements displayed on the top of the user interface, which require user interaction. I will refer them as **modal dialogs**. 

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

To address the downsides of above implementation, I came up with following solution.

### DialogView

The main element of the proposal is base modal dialog UserControl (`DialogView.xaml`) and its generic ViewModel - (`DialogViewModel.cs`). 

`DialogView.xaml` is just is a visual placeholder for the popup content.

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

Using DataTemplates, `DialogView.xaml` renders actual dialog content, based on content ViewModel type, specified in `DialogView.xaml` DataContext.

`DialogView` can be used as the last element of MainWindow View.

```xml
<Window ..>
    <Grid>

    <!-- MainWindowContent -->

    <views:DialogView/>
    </Grid>
</Window>
```

When `DialogView.xaml`'s DataContext has `IsVisible` property set to `true`, dialog is displayed, on the top of MainWindow's content.


![dialog-view-placeholder](/assets/img/article16/dialogview-placeholder.png)


### DialogViewModel

`DialogViewModel.cs` is the DataContext bound to the `DialogView.xaml`. It has two crucial properties
* `bool IsVisible` - controls if popup is displayed or not (collapsed)
* `object? DialogContentViewModel` - ViewModel which represents the DataContext for the dialog content View

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

It is generic function, which returns object of type `IDialogContentOutput`. `IDialogContentOutput` is just a marker interface, for an implementation representing the result of user interaction with the dialog. (It can be used for setting initial dialog properties values.)

```csharp
public interface IDialogContentOutput;
```

As the first input argument it takes the implementation of `IDialogContentInput` placeholder interface, which represents the input to the dialog. 

```csharp
public interface IDialogContentOutput;
```

Another `ShowAsync`  input argument is the ViewModel representing the DataContext of the actual content View, which will be displayed in the `DialogView`. Passed ViewModel needs to implement `IDialogContentViewModel<TInput, TOutput>` interface. 

```csharp
public interface IDialogContentViewModel<TInput, TOutput> where TInput : IDialogContentInput where TOutput : IDialogContentOutput
{
    void Initialize(TInput parameters, TaskCompletionSource<TOutput> taskCompletionSource);
}
```
`IDialogContentViewModel` defines `Initialize` method which takes the dialog content input, and `TaskCompletionSource` that can return an output. 

Thanks to `TaskCompletionSource` usage, the implementation of `IDialogContentViewModel` can set the result using 
`TaskCompletionSource.SetResult<T>(T result)` method. Until that happens `DialogViewModel` waits for the result asynchronously using `TaskCompletionSource.Task.WaitAsync(CancellationToken ct)` method.

The logic of `ShowAsync` method can be summarized in following points
* make the modal dialog visible
* initialize modal dialog content ViewModel with input values, and provide `TaskCompletionSource` to it
* wait until `TaskCompletionSource` is set, by the content view model, providing the output
* hide the modal dialog
* return dialog output

## Solution Pros

* Modal dialogs input and output is clearly defined
* `ShowAsync` method returns user defined dialog output, the output type is specific for each dialog
* UI thread is not blocked, since dialog waits for the interaction output asynchronously

## Solution Cons

* ViewModel - the DataContext of the dialog content View - is not obligated to call `TaskCompletionSource.SetResult<T>(T result)` on the `TaskCompletionSource` passed in `Initialize` function. In result function might be never called, leaving the dialog in open state.

## Usage

Having base code in place, lets go back to the original example and create the newsletter dialog.

First, lets create a `NewsletterDialogContentView.xaml`:

```xml
<UserControl x:Class="FlatWpfDialog.Views.NewsletterDialogContentView"
    ...>
    <UserControl.Resources>

    </UserControl.Resources>
    <StackPanel>
        <StackPanel>
            <Label FontSize="14" FontWeight="Bold">Would you like ty sign up for newsletter?</Label>
            <Label>newsletter will be sent to below email:</Label>
            <TextBox Text="{Binding UserEmail}" Margin="5"/>
            <StackPanel Orientation="Horizontal" HorizontalAlignment="Right">
                <Button Content="Sign Up" Command="{Binding SignUpCommand}" />
                <Button Content="Dissmiss" Command="{Binding DismissCommand}" />
                <Button Content="Remind me later" Command="{Binding RemindLaterCommand}"/>
            </StackPanel>
        </StackPanel>
    </StackPanel>
</UserControl>
```

Next, lets think of what should be the newsletter dialog input and output. 

```csharp
public class NewsletterDialogInput(string userEmail) : IDialogContentInput
{
    public string UserEmail { get; } = userEmail;
}

public class NewsletterDialogOutput(NewsletterDialogResult dialogResult, string newsletterEmailAddress) : IDialogContentOutput
{
    public NewsletterDialogResult DialogResult { get; } = dialogResult;
    public string NewsletterEmailAddress { get; } = newsletterEmailAddress;
}
```
> Remember to mark dialog input with `IDialogContentInput`, and output with `IDialogContentOutput` inteface.


Now, it is the time for ViewModel - `NewsletterDialogContentViewModel.cs`, which implements `IDialogContentViewModel<NewsletterDialogInput, NewsletterDialogOutput>` interface.

```csharp
public partial class NewsletterDialogContentViewModel : ObservableObject, IDialogContentViewModel<NewsletterDialogInput, NewsletterDialogOutput>
{
    private TaskCompletionSource<NewsletterDialogOutput>? _tcs;

    [ObservableProperty]
    private string _userEmail = string.Empty;

    public ICommand? SignUpCommand { get; private set; } = null;
    public ICommand? DismissCommand { get; private set; } = null;
    public ICommand? RemindLaterCommand { get; private set; } = null;

    public NewsletterDialogContentViewModel()
    {
        SignUpCommand = new RelayCommand(() => /*buisness logic can go here;*/ _tcs?.SetResult(new(NewsletterDialogResult.SignUp, UserEmail)));
        DismissCommand = new RelayCommand(() => /*buisness logic can go here;*/ _tcs?.SetResult(new(NewsletterDialogResult.Dismiss, UserEmail)));
        RemindLaterCommand = new RelayCommand(() => /*buisness logic can go here;*/ _tcs?.SetResult(new(NewsletterDialogResult.RemidLater, UserEmail)));
    }

    public void Initialize(NewsletterDialogInput parameters, TaskCompletionSource<NewsletterDialogOutput> tcs)
    {
        UserEmail = parameters.UserEmail;

        _tcs = tcs;
    }
}
```

Now, lets go back to generic `DialogView.xaml` and register the DataTemplate for newsletter dialog content

```xml
<DataTemplate DataType="{x:Type viewModels:NewsletterDialogContentViewModel}">
    <views:NewsletterDialogContentView/>
</DataTemplate>
```

Finally, newsletter modal dialog can be shown using  `ModalDialog`'s `ShowAsync` method. For instance

```csharp
public MainWindowViewModel(DialogViewModel dialogViewModel, NewsletterDialogContentViewModel newsletterDialogViewModel)
    {
        OpenDialogCommand = new AsyncRelayCommand(async () =>
        {
            var dialogOutput = await dialogViewModel.ShowAsync(newsletterDialogViewModel, new ("melmanm@melmanm.github.io"));
            LastDialogResult = dialogOutput.DialogResult;
            LastNewsletterEmailAddress = dialogOutput.NewsletterEmailAddress;
        });
    }
```

![newsletter-dialog](/assets/img/article16/newsletter-dialog.png)


## Full example

The complete code, including Newsletter modal dialog example, is available at [my github](https://github.com/melmanm/FlatWpfDialog).