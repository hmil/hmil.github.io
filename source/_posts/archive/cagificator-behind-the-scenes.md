---
title: 'Cagificator: behind the scenes'
id: 42
categories:
  - computer vision
  - hacking
date: 2014-11-19 18:33:58
tags:
---

After a few successful cagifications of my friend's profile pictures I felt the need for an automated tool to save me some time. So I quickly hacked [a cagifier app](http://cage.hmil.fr "The cagificator") for that purpose. In this article I will explain a little bit of what's behind this app and how it could be improved.

To get started, I analyzed how a manual cagification is done and how it could be automated. When you manually cage someone, you usually follow these steps:
1\. Find a good candidate face replacement
2. Align it with the original face's features (eyes, mouth, ...)
3\. Adjust the hue/saturation/value
4\. fade out the edges to get a smooth transition
<del>5\. upload the image as the victim's facebook profile picture</del>

We quickly see what the main challenges are. First one must identify faces in the picture and extract features positions. Second, a powerful color correction algorithm must be applied. And finally we must be able to choose a suitable candidate from a replacement picture database.

To make things easier, I decided to manually create a collection of replacements masks. These masks are face pictures with smooth edges and cut just the right way to avoid showing face borders (hair, chin, ears, ...). Now on to the serious stuff:

## Face detection

[![landmarks](/assets/archive/2014/11/landmarks.jpg)](/assets/archive/2014/11/landmarks.jpg)
There's this awesome tool called [flandmarks](http://cmp.felk.cvut.cz/~uricamic/flandmark/ "flandmarks") which allows you to extract facial landmarks from an image. But first you need to get a bounding box for the face. For that I chose to use OpenCV's haar cascades. There's a good tutorial on that [here](http://docs.opencv.org/trunk/doc/py_tutorials/py_objdetect/py_face_detection/py_face_detection.html). Then let flandmarks do its magic and you've got the data you need. Flandmarks comes with an example showing how to do that with OpenCV.

This needs to be computed once for all replacement pictures and then once for every picture submited. To align the faces I just compute a transformation matrix that has one translation, one scaling and a rotation such that the leftmost and rightmost points (edge of the eyes) are aligned and the face "height" (y offset between the eyes and the mouth) is the same. This is basic linear algebra that won't be discussed here.

## Color correction

The color correction tries to mimic the way I would do it by hand.
First I compute an HSV histogram for the target and replacement face. To get the target's histogram I use the alpha mask of the candidate such that both histograms are computed on the same set of pixels.

The hue is the easiest part : I compute the mean HUE and then I shift the HUE of every pixel in the replacement picture by the offset between both mean values.
For Value and Saturation, it gets a little more tricky. I first tried to multiply V and S of every pixel by some coefficient such that the mean Saturation of the replacement matches the mean Saturation of the target. But this tends to attenuate contrast and the difference in contrast is highly visible.
So i tried another way, hopefully a more successful one. If i want to keep the same contrast as the original level, I would actually need to identify the "range" in which most of the histogram bars are and stretch the replacement image histogram to fit the target's one.
The easiest way to do that is compute a linear function y = ax + b that's applied to the target's pixels. With a well chosen function we can have the target and replacement histograms overlap as shown in the following picture.

[caption id="attachment_44" align="alignnone" width="1195"][![histograms](/assets/archive/2014/11/histograms.png)](/assets/archive/2014/11/histograms.png) From left to right: masked target image, replacement image, color-corrected replacement ; Hue (H), Saturation (S) and Value (V) plots. 
 The plot shows: the target image histograms (red) and the corrected replacement (green). We can see that the replacement's V histogram has been stretched so that it occupies all of the available space (replacement has less contrast than target) and that its S plot was compressed near zero (conversion to black and white). 
 Hue is pointless since this is B&amp;W. The horizontal red line is the threshold.[/caption]

To determine the coefficients a and b, we pick an arbitrary threshold. All bars lower than this threshold in the histogram are ignored. We can then get the minimum and maximum values for which there is something in the histogram. We do this for both the target and the replacement and then it's high school math to find a and b.This work is done independently for the S and V channels as they are independent variables.

In the end, the following correction is applied to each pixel :
```
  dstS = srcS * aS + bS
  dstV = srcV * aV + bV
  // Hue is cyclic so instead of capping max and min values, use a modulo
  dstH = srcH + (targetMeanH - replacementMeanH) % 255
```

## Candidate picking

The last big feature to implement is choosing the right replacement image. Now I was a bit tired of this project when reaching this point so I definitely chose the lazy option. That is, to compute the transformations mentioned above for each picture, then compute the sum of the absolute color difference for each pixel of the image. The replacement that has the lowest sum is the best.

This is definitely not optimal since it favors overall low absolute difference but does not deal with face alignments. One could hope that if for instance the mouth is not well aligned, it will yield a higher score and the candidate will be rejected. However it is not the case and many results suffer from alignment problems. Also this solution is computationally intensive and you can see it starts lagging seriously when your image contains multiple faces.

&nbsp;

Overall it's been a fun experiment and although this is far from perfect, I'm amazed to see that for some particular images I wouldn't have done better myself.
Be sure to [try it out](http://cage.hmil.fr) and if you obtain nice results, feel free to link the picture in the comments !