---
layout: post
title: "Ultra Wide Screen Share 2.0"
categories: projects
cover-img: /assets/img/projects/ultra-wide-screen-share-2/cover-background.png
share-img: /assets/img/projects/ultra-wide-screen-share-2/cover-background.png
---
Ultra Wide Screen Share 2.0 offers seamless sharing of selected ultra wide screen area during Teams or Skype meetings. Download Ultra Wide Screen Share 2.0 from [Microsoft Store](https://apps.microsoft.com/detail/9P6J8N3K7TK4?hl=en-us&gl=US)!

![ultra-wide-screen-share](/assets/img/projects/ultra-wide-screen-share-2/cover-background.png)

## Motivation

I find my ultra wide monitor as a perfect alternative for double screen setup. I feel daily tasks performed on ultra wide monitor are much more enjoyable. There are plenty of free tools, which help to organize multiple window layout (like fancy zones https://learn.microsoft.com/en-us/windows/powertoys/fancyzones). They make the experience of ultra wide screen even better.

However, there is one big disadvantage of ultra wide monitor - sharing the screen on Teams or Zoom meeting. There are usually two main ways you can share the screen
* Share entire ultra wide monitor screen - this option could cause an issue for the spectators, since your ultra wide monitor needs to be entirely displayed on spectators standard monitor. They would most likely see zoomed out image of ultra wide screen, with black straps at the top and bottom. Of course, some applications enable spectator to zoom the presented screen in. However it is not always easy to track presenter moves and zoom in proper area.
* Share a single widow - you could also share a single window which is resized to be well displayed on spectator's standard monitors. The downside of this approach is you can share a single window at the time.

## Goal

The goal of Ultra Wide Screen Share Project was to find a way to share only the part of ultra wide monitor screen (that will be well displayed on spectator's screens), but also enable to present multiple applications windows.

## Ultra Wide Screen Share 2.0
Ultra Wide Screen Share 2.0 application was created to meet this goal. Ultra Wide Screen Share 2.0 application has a single,transparent window, which can be positioned and resized, covering the screen area desired to share. Next, Ultra Wide Screen Share 2.0 window can be selected to share in Teams or Zoom meeting. Now, the screen area in boundaries of Ultra Wide Screen Share 2.0 will be presented.

## How to get Ultra Wide Screen Share 2.0?
Application can be install via
* [Microsoft Store](https://apps.microsoft.com/detail/9P6J8N3K7TK4?hl=en-us&gl=US)
* [Github releases](https://github.com/melmanm/UltraWideScreenShare-2/releases) (Portable version - download release.zip and launch .exe file)


## Technical challenges (for developers)

The application is written in .NET Winforms. Application code is available at https://github.com/melmanm/UltraWideScreenShare-2 

### Custom Winforms Window
In order to create user friendly look of the main application window, it was required to create custom Winforms window. I summarized all challenges related with this topic in separate article: https://melmanm.github.io/misc/2024/01/10/article13-winforms-custom-window.html. 

### Transparency

Main application window seems to be transparent, since the screen area underneath is visible.

Initially I underestimated the complexity of this topic. My first idea was to simply set the background of main Winforms window to `Transparent`.

Despite the fact it is not that easy in Winforms, transparency is not supported by neither Teams nor Skype. In result, transparent window area was displayed black on spectator's screens.

In order to achieve the effect of transparency, without real transparency, I decided to make use of windows magnification API.

Windows magnifier was developed to enable users to zoom in selected screen area. Windows magnifier API and functionality is available in system's `Magnification.dll`. (https://learn.microsoft.com/en-us/previous-versions/windows/desktop/magapi/magapi-intro)

The transparency effect is achieved by displaying the magnifier window as a child control of main application window. Since the zoom of the magnifier is set to `1` (original size is preserved), and the source magnification area always corresponds with the magnifier location, a feeling of transparency is created.

Magnification window is programmed to be clickthough, so user can interact with windows displayed underneath using the mouse and keyboard.


### DPI Awareness

Magnifier window were initially displayed incorrectly on the screens, where system zoom was larger than 100%. Luckily, DPI awareness setting, influenced the magnifier and solved the original issue without additional implementation. Read more on how to set process DPI awareness on https://learn.microsoft.com/en-us/windows/win32/hidpi/setting-the-default-dpi-awareness-for-a-process.

## Contribution

If you have any suggestions or ideas related with this project feel free to open an issue or pull request on on [GitHub repo](https://github.com/melmanm/UltraWideScreenShare-2).