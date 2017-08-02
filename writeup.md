## Finding Lane Lines on the Road

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report

---

### Reflection
The pipeline consisted of the following steps:

1. Convert an image to [grayscale](#grayscale) using custom weights
2. Apply [gaussian blur](#gaussian-blur) filter
3. Invoke [Canny edge detection](#canny-edges-detection) algorithm
4. Apply [region of interest](#region-of-interest) filter
5. Apply Hough transformation
6. Calculate lanes [directions from Hough segments](#hough-analysis)
7. Draw lanes using [_averager_](#averager) on the original image

#### Grayscale
[low_conrast]: ./test_images/challengeLowContrast.jpg "Original"
[standard_gs]: ./writeup/challengeLowContrast.stadndard.jpg "Standard Grayscale"
[cutsom_gs]: ./test_images_output/challengeLowContrast.gray.jpg "Custom Grayscale"

I used custom channel weights while mixing channels to create grayscale image.
The main reason to use custom weights - increase contrast of lanes on
_challenge video_. The idea as simple as overweight light colors of lanes keeping
surface color mostly untouched. As a result of overflow lanes become black
on gray or light gray surface, hopefully this will give big enough gradients
for Canny edge detection algorithm.
Bellow result of standard and custom grayscale transformation.

Original image
![alt text][low_conrast]

Standard grayscale
![alt text][standard_gs]

Cutsom grayscale
![alt text][cutsom_gs]

##### Shortcomings
Using custom weights and just integer overflow to increase contrast looks naïve
and too simple. This may lead to unstable work of the pipeline in different
conditions eg. different light sources, night time, tunnels, different road surfaces, different lane colors.

##### Improvements
The main vector of improvements in this area - find suitable contrast enhancement
algorithm which will work stable enough in wide range of light conditions.

#### Gaussian Blur
One of downsides of grayscale conversion algorithm - it may introduce high
gradients in unexpected spots. In order to minimize negative effect gaussian
blur filter could be used.

The pipeline sets kernel to 7.

##### Shortcomings
The kernel is too big, which may lead to lower gradients (and as a result to
  missed edges) in some spots.

##### Improvements
Together with changing grayscale algorithm it's better to test the pipeline with
lower kernel value.

#### Canny Edges Detection
[canny_standard_gs]: ./writeup/challange.branches.png "Canny against standard grayscale"
[canny_custom_gs]: ./test_images_output/challengeShadows.canny.jpg "Canny against custom grayscale"

The pipeline uses aggressive parameters for canny edges detection.
Proposed grayscale conversion algorithm allows to do so. One of the reasons behind
this - attempt to avoid edges produced by shadows.

Fragment of detection using standard grayscale transformation
![alt text][canny_standard_gs]

Detection using custom grayscale transformation
![alt text][canny_custom_gs]

Parameters of the pipeline:
* Low threshold - 120
* high threshold - 350

#### Region of Interest
[roi]: ./writeup/roi.png "Region of Interest shape"

The pipeline uses following ROI shape.
![alt text][roi]

There is an obvious advantage of such a narrow RoI - more edges which doesn't
belong to lanes filtered out. On other hand there are plenty of disadvantages.

##### Shortcomings
Narrow RoI is not that good choice since it:
* sensitive to camera position in the car - RoI is symmetric so camera should
be mounted at the middle of the vehicle.
* sensitive to camera parameters (such as focal distance) - the same RoI could
not be reused for a different camera
* sensitive to the position on the road - changing lanes or even driving not along to the center of the lane could lead to unstable detection
* suitable for highway-style roads - cross roads, turns, uphills and downhills may not be handled well using given RoI

##### Improvements
The most important improvement (together with contrast enhancement algorithm) could be changing RoI shape. It could be either asymmetric wide region without blind zone in the middle, or even more complex polygon (two trapezoids with
  different angles?).

#### Hough Analysis
[hough_edges]: ./writeup/challengeEdge.hough.jpg "Hough Edges"
[filtered_edges]: ./writeup/challengeEdge.new.jpg "Filtered Edges"

The pipeline uses complex algorithm to convert segments found by Hough transformation into lane lines.
1. Invoke Hough transformation
2. Filter out irrelevant segments
    1. Evaluate segment slope
    2. Classify segment as belong to left (negative slope) or right lane
    3. Drop segment if slope doesn't belong to the slope range (slopes of inner and outer edges of the region) of corresponding part of RoI.
  ![alt text][hough_edges]
  ![alt_text][filtered_edges]
3. For each side
    1. Calculate mean slope
    2. Find segment with slope closest to mean value
    3. Take middle point of the segment
    4. Find highest (in term of vertical axis) segment
    5. Evaluate appropriate x-coordinate using point from 3 and slope from 1
    6. Evaluate lane position on the bottom of image using point from 5 and slope from 1

##### Shortcomings
Straight line sometimes not that good approximation for road lane lines.

#### _Averager_
In some cases detection could be not perfect, it may lead to "jumping" lane on
the video stream. To avoid such behavior and provide smooth lane tracking the _averager_ (not the best name actually) was introduced.

The main idea of the component - to track current position and instead of applying the whole delta at once go step by step to the target value.

##### Shortcomings
The _averager_ works well if only some frames have lane detection problems. In case if long video fragment has some anomaly it will take more time to recover even when detection perfectly done.
