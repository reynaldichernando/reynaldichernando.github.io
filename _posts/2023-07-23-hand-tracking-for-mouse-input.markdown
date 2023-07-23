---
layout: post
title:  "Hand Tracking for Mouse Input"
date:   2023-07-23 15:55:00 +0700
categories: blog
author: Reynaldi Chernando
---
The other day I saw the launch of Apple Vision Pro, the whole thing was very interesting, but one thing that caught my attention was the finger input. It seems very intuitive, by using the finger pinching as sort of like a cursor or mouse input. I figured I want to try it out, so I took it upon myself to create it.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/fMaKvGG.jpeg">
  <source src="https://i.imgur.com/fMaKvGG.mp4" type="video/mp4">
</video>

*Apple Vision Pro input using finger pinching action*

## Game Plan

The goal here is to use the hand as an input device for computer. It should be able to handle clicking and moving the mouse cursor. In order to do this, we obviously need a camera, and for starter, let’s point the camera to face downwards, since that is basically where the hands will be when using a computer. Next we need some sort of way to detect the hand and fingers position for controlling the mouse. For this, I will be using [MediaPipe](https://developers.google.com/mediapipe){:target="_blank"} from Google. The best way I can describe MediaPipe is, a set of prebuilt solutions for ML, and it happens to have a hand landmarker feature, which is what we needed. Lastly we will need to somehow simulate the mouse input.

![MediaPipe hand landmark detection](https://i.imgur.com/P19M4Ha.jpeg)

*MediaPipe hand landmark detection*

![High level design of the system](https://i.imgur.com/CptH0au.jpeg)

*High level design of the system*

## First Trial

Starting with this, we can use the python version of MediaPipe, and have [OpenCV](https://docs.opencv.org/4.x/dd/d43/tutorial_py_video_display.html){:target="_blank"} read the camera feed and input it to MediaPipe, then utilize the hand landmarks to simulate the mouse, simple right? Except that this doesn’t work very well. I’ve only got into the OpenCV with MediaPipe and I found that it is super laggy.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/NXoHagh.jpeg">
  <source src="https://i.imgur.com/NXoHagh.mp4" type="video/mp4">
</video>

*Python version is super laggy, something to do with OpenCV*

After some researching, turns out it was because it has something to do with the OpenCV, related to how cv2.waitKey() works. I still haven’t found out the fix yet, so I suppose we’re skipping python altogether to save time and sanity.

## A Stupid Idea

During research for using MediaPipe, I found out about the web version of the library through its demo. And for some reason the web version ran very smoothly. I figured I can use the web version instead of the python version. Just one problem though, how would I control the mouse using a browser? So, I came up with a crazy idea, since I can run the MediaPipe locally, what if I have a python backend running to simulate the mouse? I just have to figure out how to communicate the MediaPipe frontend to the Python backend. 

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/8pZJXVD.jpeg">
  <source src="https://i.imgur.com/8pZJXVD.mp4" type="video/mp4">
</video>

*Web version running smoothly*

## Simulating the Mouse

For the MediaPipe frontend to communicate with the Python backend, I need to have it work through some sort of method. I thought of 3 ways, which are simple HTTP request, [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), and [gRPC](https://grpc.io){:target="_blank"} with bidirectional streaming. The HTTP request is immediately out of the picture, considering I need the latency to be as low as possible. This left me with 2 options that are both for streaming data. I'm not too familiar with gRPC, so I suppose I will just use WebSocket.

I setup a simple WebSocket server in python which will accept a JSON string message containing the x and y coordinate to move the mouse to. I then just have to connect the coordinates of one of the finger to be sent to the backend, in this case I’ll be using the thumb tip. And it worked! Surprisingly very well even, however it feels very wrong though, controlling it using the browser. In any case, the latency is not that noticeable, I suppose there might be some inefficiency. But for now let’s just ignore that.

![WebSocket server receiving information to control the mouse](https://i.imgur.com/IVFKzqI.jpeg)

*WebSocket server receiving information to control the mouse*

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/9P2Yo0J.jpeg">
  <source src="https://i.imgur.com/9P2Yo0J.mp4" type="video/mp4">
</video>

*Using a web browser to control the mouse*

Next is the clicking logic. To detect a click, we need to detect a pinch action, between the thumb and index finger. To do this, we just have to measure the distance between the thumb tip and index finger tip, this is using [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance){:target="_blank"}. If it is less than a certain threshold, we will invoke a mouse down event, and if it is greater than the threshold, we will invoke the mouse up event. Notice that we are using mouse down and up instead of click, this is to support dragging action.

This solution works fine, however, there is a problem if we were to move the hand closer to the camera. Since we are moving a 3D object in 2D space, by moving the hand closer to the camera, the distance between the finger tips also gets larger.

![Distance between finger tip, far vs close to the screen](https://imgur.com/iipvLpX.jpeg)

*Distance between finger tip, far vs close to the screen*

To solve this issue, we can implement a workaround to use a relative distance instead. By calculating the distance between the finger tips and the respective knuckles, where the distance will get larger the closer the hand is to the camera, we can compare this distance to the distance between the tip. This way we get a proper finger tips distance regardless if the hand is closer or far away from the camera.

![Calculating relative distance between finger tips](https://imgur.com/6hwT1b9.jpeg)

*Calculating relative distance between finger tips*

## The Jitter

Next problem to solve is the jitter. Notice that even when my hand is not moving, and rested on the table, the cursor is still shaking. This is inherently caused by the hand landmarker model, and the only way to really fix this is to use a better model. However not all hopes are lost. For now, we can implement a simple moving average for the cursor position, this way it doesn’t jitter so much, and the movement become smooth. The only thing is, the higher the buffer for the moving average, the higher the latency as well. While we are at this, I added buffer to the pinch state as well, because it usually does a false negative while doing pinching and moving at the same time.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/kKCgesw.jpeg">
  <source src="https://i.imgur.com/kKCgesw.mp4" type="video/mp4">
</video>

*Cursor is jittering while no hand movement*

![Using moving average to smoothen the movement](https://i.imgur.com/XnwQKkT.jpeg)

*Using moving average to smoothen the movement*

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/Vn5FfXU.jpeg">
  <source src="https://i.imgur.com/Vn5FfXU.mp4" type="video/mp4">
</video>

*Movement is much smoother with moving average*

Another improvement we can add is a safe zone for edges of the screen. As it is now, the cursor position is based on the thumb tip coordinates. Due to this, to reach the edge of the screen we need to put the hand all the way to the edge, and in doing so, the hand is no longer fully visible. To solve this, we can implement a simple linear transformation to convert the coordinates. For this, I am basically adding a padding on each side of the edges. This way we can solve the problem of the hand not being fully visible.

![Linear transformation implementation](https://i.imgur.com/dDitISx.jpeg)

*Linear transformation implementation*

![Before and after padding illustration](https://i.imgur.com/U8kTDTB.jpeg)

*Before and after padding illustration*

## A Better idea

Let’s go back to the latency inefficiency thing I mentioned earlier. Not only am I facing this issue, but also the fact that I need to have the tab always open for the cursor to function. We want to maintain using the web version of MediaPipe and also have it running even when the tab is not active. After some researching, one suitable option is to use [Tauri](https://tauri.app), a framework for building desktop apps using web technology and Rust, because it can run the web frontend as a standalone program and the Rust backend can be used to communicate with the frontend much more efficiently. This can be used to simulate the mouse input we had in python. I just have to adjust some of my code. Not knowing any Rust, I did use Google and ChatGPT for implementing this.

![Tauri Github Page](https://i.imgur.com/dT9Xwma.jpeg)

*Tauri Github Page*

![Tauri Github Page](https://i.imgur.com/FTP4cGn.jpeg)

*Rust backend to control the mouse*

![Javascript code calling the Rust backend](https://i.imgur.com/EhBP6Yt.jpeg)

*Javascript code calling the Rust backend*


Putting it all together, here is the result.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/FKLTpZ8.jpeg">
  <source src="https://i.imgur.com/FKLTpZ8.mp4" type="video/mp4">
</video>

*Using hand tracking to simulate mouse input*

## Developing another mode

At this point, the project should’ve been finished, but I took some time to watch YouTube video related to hand tracking input. I found that Meta Quest has a hand gesture input, which is equally cool. If in the Apple Vision Pro, you need to have the eye tracking sensor to know where the cursor is pointed, in the Meta Quest, you just have to point your hand or finger to the “screen”. I figured, why not also add this mode so the camera can face forward, and user just have to point their finger towards the screen.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/BoBUSZA.jpeg">
  <source src="https://i.imgur.com/BoBUSZA.mp4" type="video/mp4">
</video>

*Meta Quest hand tracking feature. credit: “Tricks Tips Fix” on Youtube*

In order to determine where the finger is pointing, we can’t only use where it is located in the x and y coordinates like we did for the down facing mode. Instead, first we need to know what angle is the finger pointing at. Second, we also need to know the Z distance between the camera and the finger, these values will be used for calculating where the cursor will “land” on the screen basically.

![Illustration on determining where the cursor will land on the screen](https://i.imgur.com/2VuVSZe.jpeg)

*Illustration on determining where the cursor will land on the screen*

For the angle, we can simply pick 2 points from the hand landmark, and by using some trigonometry we can determine the angle. We do this for the YZ angle and XZ angle to have both the horizontal and vertical angle. For the distance, this is quite tricky, because the Z axis in the hand landmark does not mean how far it is from the screen, it is just the Z distance between the finger point and the wrist point. So in order to determine distance, we will have to play around with the scale. Remember earlier where I mentioned about distance between finger points is larger when hand is closer to the screen, and smaller when the hand is further from the screen. I’m basically using this information to calculate the distance from the camera to the hand.

![Example for Y axis, using the angle of the finger, along with the distance to screen, we can find the cursor Y value](https://i.imgur.com/GxyTr84.jpeg)

*Example for Y axis, using the angle of the finger, along with the distance to screen, we can find the cursor Y value*

![The Formula](https://imgur.com/BwT0mAR.jpeg)

*The Formula*

Testing the front facing mode, I am experiencing more shaking than the down facing mode. Even with the moving average, the shaking is still unbearable. So I seek a better alternative to this method, and found the [One Euro Filter](https://gery.casiez.net/1euro/){:target="_blank"}. It is a type of low pass filter, which basically does smoothing for noisy input, that is our cursor. I did have to tweak some parameter to make it usable as we are adjusting for lower jittering but also reasonable latency. In addition to that, I added some thresholding for the angle in order to further reduce the jittering. And it is somewhat usable now! I also ended up changing the moving average into One Euro Filter for the down facing mode for better latency.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/Q2B3lO3.jpeg">
  <source src="https://i.imgur.com/Q2B3lO3.mp4" type="video/mp4">
</video>

*Mouse is jittering like crazy even with moving average*

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/rHn03Mm.jpeg">
  <source src="https://i.imgur.com/rHn03Mm.mp4" type="video/mp4">
</video>

*After adding One Euro Filter and thresholding, the mouse movement is relatively tame, but still jittery*

In the end, however, it is not all sunshine and roses, as there are some major problems for the front facing mode. The first biggest one is the shaking input. At certain position and angle, for some reason the MediaPipe readings will be very shaky, rendering all the filter and smoothing useless. The second one is when pinching the fingers together, there is a slight drift, making it hard to click something as the cursor just drifts away. These issues are inherently present within the model, and I figured no amount of smoothing and filtering can fix this besides fixing the actual model.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/7AC0EdT.jpeg">
  <source src="https://i.imgur.com/7AC0EdT.mp4" type="video/mp4">
</video>

*Certain position of the hand where the hand landmarks result is very shaky*

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/3lKsaiJ.jpeg">
  <source src="https://i.imgur.com/3lKsaiJ.mp4" type="video/mp4">
</video>

*Mouse is drifting when fingers are pinched*

## Conclusion

In this project, I did what I set out to make, which is to create a cursor or mouse input like the one from the Apple Vision Pro, plus the Meta Quest headset. I had fun working on this project as I get to try some cool tech like MediaPipe, Tauri, Rust and doing some maths for this to work. You can find the project in the repository below. I’ve only tested this on Windows, it may or may not work on Linux or MacOS.

[reynaldichernando/pinch](https://github.com/reynaldichernando/pinch){:target="_blank"}

Here are some video demos for the final result.

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/QW5PoP8.jpeg">
  <source src="https://i.imgur.com/QW5PoP8.mp4" type="video/mp4">
</video>

*Interacting with the down facing mode*

<video autoplay loop width="100%" controls muted preload="none" poster="https://i.imgur.com/SbxrnMf.jpeg">
  <source src="https://i.imgur.com/SbxrnMf.mp4" type="video/mp4">
</video>

*Interacting with the front facing mode*

All in all, I would say the down facing mode works quite reliably, while the front facing mode is still unstable given the issues I mentioned.

Some notes

- To minimize some of the drift when pinching, resting the thumb on the side of the middle finger tip can help.
- For the angle calculation in front facing mode, especially vertical angle, I added an offset, due to the camera being placed at the top of the screen, this is to correct the cursor being lower than where we are pointing.
- I have yet to find a way to make the Tauri app work in background, so minimizing the window will stop the cursor. So, to change window, just click the desired window without minimizing the Tauri window.