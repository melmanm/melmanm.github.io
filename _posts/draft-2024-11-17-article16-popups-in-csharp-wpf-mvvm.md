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
There are some conceptual differences between popup, dialog, message box and modal (very well described at https://medium.com/design-bootcamp/popups-dialogs-tooltips-and-popovers-ux-patterns-2-939da7a1ddcd). In this article I will be focusing on applications's visual elements displayed on the top of the user interface, which require user interaction. I will refer them as **modal dialogs**. 

## Motivation
Over the time I saw multiple implementations and ideas on how to deal with modal dialog in WPF application. Lets consider a requirement to display a newsletter modal popup, which offers newsletter sign up. To sign up it requires the user to fill in the email address. Additionally, user has a possibility to chose `remind me later`, or `dismiss` options. Assuming the `NewsletterPopupView` is responsible for executing the sign up logic - the simplest approach to display the modal dialog would be:

```csharp
//MainViewModel

if(!User.RemindNewsletter || User.NewsletterActive)
    return;

var popupView = new NewsletterPopupView();
popupView.DataContext = new NewsletterPopupViewModel(_newsLetterService);
popupView.DataContext.UserName = User.Name;
if(popupView.ShowDialog())
{
    if(popupView.DataContext.Result = NewsletterPopupResult.Dismiss)
        User.RemindNewsletter = false;
    else if(popupView.DataContext.Result = NewsletterPopupResult.RemindLater)
        User.RemindNewsletter = true;
    else if(popupView.DataContext.Result = NewsletterPopupResult.Ok)
        User.NewsletterActive = true;
}
```

Despite obvious simplicity, presented solution has some downsides:
* `Widndow.ShowDialog()` returns `bool?` based on the shown Window's `DialogResult` property. In case (like ours), when parent needs modal dialog result, and the result itself is extended to `Ok` `Dismiss` and `RemindLater` options, it needs to be stored in Popup's DataContext, so parent ViewModel can access it after modal dialog is closed. There are also some use cases where popup returns more data than just the result.
* Displaying modal as separate window, often doesn't align with modern designs. Nowadays modal dialogs tend to have flat appearance, embedded into application main window. (Like on below example from Spotify application): 
   ![spotify-dialog](/assets/img/article16/spotify-dialog.png)
* Modal dialog initialization is achieved by setting DataContext's properties. (Of course property initialization could be done by `NewsletterPopupViewModel` constructor, but constructor it is often dedicated for dependency injection, especially when modal dialog uses services to perform the business logic). In such case developer who uses `NewsletterPopupViewModel` is not directly instructed if any input properties are required to be set to initialize modal dialog. Moreover it is hard to distinguish which properties are required, and which are optional. 
* Parent window ViewModel becomes responsible for instantiating modal dialog's View. or at least holds modal dialog's View reference, which is questionable practice in MVVM pattern.
* `ShowDialog` will block the UI thread, There is no `ShowDialogAsync` method on available in WPF.
