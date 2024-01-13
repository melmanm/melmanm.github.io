---
layout: post
title: "Winforms - How to create a custom window with resize drag and snap functionalities?"
categories: misc
tags:
- Winforms
- Forms
- dotnet
- .NET
- C#
cover-img: /assets/img/article12/cover.png
---

In this article i will describe how to create custom Winforms window, which can be resized, dragged, and snapped, providing the same experience as standard window.

A motivation to create custom Winforms window is, in most cases, application design, which requires a layout not supported by standard system window. For instance, in my recent project i needed to implement a design, where window navigation buttons are placed in the "notch" style.

![ultra-wide-screen-share](/assets/img/article13/ultra-wide-screen-share.png)

I decided to create this article, since I didn't find any comprehensive material covering this topic.

## Removing window tittle bar and borders
In order to design custom window (and its tittle bar) it is required to get rid of default windows tittle bar it can be done by setting `FormBorderStyle` property of the window form.
```csharp
FormBorderStyle = FormBorderStyle.None;
```
This setting hides the widow tittle bar and borders. Custom tittle bar and border can be now implemented and styled inside the window using Winforms controls.

Unfortunately, once `FormBorderStyle.None` is set, window loses resizing, dragging and snapping functionalities.

Let's add it back!

## Window dragging
Window dragging can be normally achieved by pressing left mouse button a on a window tittle bar, and moving the window while the button is pressed.

Since default window tittle bar was removed in previous step, let's create a custom tittle bar, by placing `Panel` control at top of the `Form`.

![custom-tittle-bar](/assets/img/article13/custom-tittle-bar.png)

Next panel's MouseDown event needs to be handled, to detect user left mouse button is pressed in the panel area, and start dragging the window.

```csharp
const uint WM_NCLBUTTONDOWN = 0xA1;
const uint HT_CAPTION = 0x2;

private void menuPanel_MouseDown(object sender, MouseEventArgs e)
{
    if (e.Button == MouseButtons.Left)
    {
        PInvoke.ReleaseCapture();
        PInvoke.SendMessage(new HWND(Handle), WM_NCLBUTTONDOWN, new WPARAM(HT_CAPTION), new LPARAM());
    }
}
```

This method implementation requires a comment. `ReleaseCapture` and `SendMessage` functions are implemented in Windows native `user32.dll`. To invoke them in C# code, marshalling is required. You can use `DllImport` attribute to specify these functions signatures, or, as I did, make use of the [Microsoft.Windows.CsWin32 nuget package](https://www.nuget.org/packages/Microsoft.Windows.CsWin32/), which does the marshalling for you.

### ReleaseCapture
Normally the the mouse is captured by control when user presses a mouse button over it and released when button is up. `ReleaseCapture` function does not wait for mouse up event, but releases the capture instantly. It is a preparation for the next step.

### SendMessage
`SendMessage` with `WM_NCLBUTTONDOWN` and `HT_CAPTION` function is used to inform the windows to that, mouse button was pressed over the original window title bar. Thanks to this trick, windows starts dragging action. Since the mouse is already done dragging continues as usual, until it is released by the user.

![dragging-gif](/assets/img/article13/dragging-gif.gif)

## Window Resizing
After styling main window with `FormBorderStyle.None` window lost borders and ability to use them to resize the window.

In order to implement this functionality we will try to capture `WM_NCHITTEST` message sent by the windows to Winforms window, whenever cursor moves or mouse button is clicked. Windows sends this message to the window, expecting that window responds with information about the window element which is pointed by mouse or clicked.

The plan is simple. Fist `WM_NCHITTEST` message needs to be captured. Next we check if cursor is pointing at the specific widow edge (left, right, top, bottom or corner). If it is, we generate the response, containing corresponding border identifier. Thanks to this trick, windows start to behave like original window border (which is hidden by `FormBorderStyle.None`) is clicked on hovered.

Following code-behind shows how to implement this feature for right window edge.

```csharp
const int WM_NCHITTEST = 0x0084;
const int HTRIGHT = 11;
const int BorderSize = 4;
protected override void WndProc(ref Message m)
{
    if (m.Msg == WM_NCHITTEST)
    {
        base.WndProc(ref m);

        //cursor position
        var cursorPosition = PointToClient(Cursor.Position);
        //border area
        Rectangle rightBorderRectangle  = new Rectangle(ClientSize.Width - BorderSize, 0, BorderSize, ClientSize.Height);

        if (rightBorderRectangle.Contains(cursorPosition))
            m.Result = (IntPtr)HTRIGHT;
    }
}
```

> This approach can be extended for other edges and corners, by using return values defined at https://learn.microsoft.com/en-us/windows/win32/inputdev/wm-nchittest

> Note that `BorderSize` can be adjusted to your needs. It defines the width of the area, which will be considered as window border, given in px.

![resizing-gif](/assets/img/article13/resizing-gif.gif)

## Window snapping to edges of the screen