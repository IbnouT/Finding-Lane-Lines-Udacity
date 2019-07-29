# **Finding Lane Lines on the Road**

## Project1 for Self-Driving Cars Nanodegree @Udacity
---

## **Overview**

The objective of this project is to build an image processing pipeline for identifying lane lines on the road, initially on a serie of test images and then later on some video files.

The pipeline I've considered is composed of the following steps:
1. Color selection of the lane lines
2. Gray scaling of the images
3. Noise reduction with Gaussian blurring
4. Canny Edges detection
5. Selection of the region of interest
6. Lines detection with Hough tranform

Final results of the videos processing can be found in the folder [test_videos_output](test_videos_output/):
* Raw lane lines:
  - [Raw solidWhiteRight.mp4](test_videos_output/solidWhiteRight0.mp4)
  - [Raw solidYellowLeft.mp4](test_videos_output/solidYellowLeft0.mp4)

* Full extend of the lane lines:
  - [solidWhiteRight.mp4](test_videos_output/solidWhiteRight.mp4)
  - [solidYellowLeft.mp4](test_videos_output/solidYellowLeft.mp4)

* Challenge lane lines:
  - [Challenge.mp4](test_videos_output/challenge.mp4)


[//]: # (Image References)

[pipeline]: ./images/pipeline.png "Pipeline"
[original]: ./test_images/solidYellowLeft.jpg "Original Image"
[color_selection]: ./test_images_output/1_color_select_solidYellowLeft.jpg "Selection of White/Yellow color"
[gray_scaled]: ./test_images_output/2_gray_scaled_solidYellowLeft.jpg "Gray Scaled"
[blur_grayed]: ./test_images_output/3_blur_grayed_solidYellowLeft.jpg "Noise Reduction"
[edges]: ./test_images_output/4_edges_solidYellowLeft.jpg "Edges Detection"
[edges_region]: ./test_images_output/5_masked_edges_solidYellowLeft.jpg "Region Selection 2"
[rawlines_1]: ./images/LinesYellowLeft.png "raw lines - YellowLeft"
[rawlines_2]: ./images/LinesSolidWhiteRight.png "raw lines - SolidWhiteRight"
[line_ex_1]: ./test_images_output/6_lines_detection_solidYellowLeft.jpg "Extended lines - - YellowLeft"
[line_ex_2]: ./test_images_output/6_lines_detection_solidWhiteRight.jpg "Extended lines - - SolidWhiteRight"
[region_selection]: ./images/region_select_solidYellowLeft.png "Region Selection 1"
[example_widget]: ./images/example_widget.png "Example Interactive Widget"
---

## **Reflection**

### 1. The Pipeline

![alt text][pipeline]


#### Step 1: Color Selection
The first step I considered in this pipeline was to select white and yellow colors from the image. This helped remove most contents of the image that are not relevant.

One important decision I had to make here was to decide which color scheme to use (RGB, HSV, HLS). I found out that RGB colors was Ok to use with the test images, but at later stage when processing the video for the challenge I noticed that light and shadow had huge impact on the accuracy of the yellow color selection.

Finally, with some googling help, I decided to go with HLS as the hue (H) channel keeps almost same value for the range of yellow color variation, and the lightness (L) channel was useful for detecting the white color. For the color ranges I used this [online tool](http://colorizer.org/) to try several combinations.
```
# White color ranges
HLS_LOW_WHITE = np.array([0, 200, 0])
HLS_UPPER_WHITE = np.array([255,255,255])

# Yellow color ranges
HLS_LOW_YELLOW = np.array([10, 0, 100])
HLS_UPPER_YELLOW = np.array([40, 255, 255])
```
| Original Image | Selecting White & Yellow color |
| :---: |:---:|
| ![alt text][original]  | ![alt text][color_selection]  |

#### Step 2: Gray-Scaling
This step was necessary to prepare the image for later edge detection.

| White & Yellow Selection | Gray Scaled |
| :---: |:---:|
| ![alt text][color_selection]  | ![alt text][gray_scaled]  |

#### Step 3: Noise Reduction
Before detecting edges it is usually good idea to reduce noise on the image. This is done here using Gaussian blurring with a kernel size of 5.
I did try several kernel values to see the effect, while trying to keep it as low as possible for performance reason.

| Gray Scaled | Blurred Gray |
| :---: |:---:|
| ![alt text][gray_scaled]  | ![alt text][blur_grayed]  |

**On a general note**: most of the processing methods used in this pipeline require to test with different parameters and figure out what works best. To make that easy I added for those steps interactive widgets that allow to change the parameters with sliders and visually see the effects on the image being processed. example_widget

![alt text][example_widget]

#### Step 4: Canny Edge Detection
This is one of the most important step, using Canny edge detection algorithm find the edges on our pre-processed image. The result of this process is dependent on the low and high threshold values used. But since it is recommended to use a ratio of 2:1 or 3:1 for high/low threshold then I had to just play with the low threshold value and compute the corresponding high value based on the ratio I fixed to 3:1.

Again here also I used some interactive widgets to change the parameters and see immediate effects on the image being processed. However I had to change initial threshold values I selected after reassessing the result of the end to end pipeline processing.

| Blurred Gray | Canny Edge Detection |
| :---: |:---:|
| ![alt text][blur_grayed]  | ![alt text][edges]  |

#### Step 5: Selection of the region of interest
As we can see in the result of the previous step, we get the edges of our lane lines but also many other edges we are not interested on. Taking into consideration the fact that the camera taking these pictures has a fixed position and assuming that the car is always driving at the center between 2 lane lines, then we can consider those lane lines will most likely be always in the same area of the image.

I selected a fixed area for all images, with fixed coordinates. The selection looks like as following (area is the inside of blue border lines):

![alt text][region_selection]

It worked almost fine for all test images and 2 of the test videos provided [solidWhiteRight.mp4](./test_videos/solidWhiteRight.mp4) and [solidYellowLeft.mp4](./test_videos/solidYellowLeft.mp4). For the challenge video, I was not capturing the right area! I found out it was due to that video having a different size of 1280x720, while other assets was sized 960x540. This reminded me of am important step of pre-processing which is to ensure that all images have same size. Fortunately the challenge video were proportional to the size of previous images tested, so I had to be careful on defining my selection area to use proportional distance.

The result of the region selection applied to the edges detection looks like this:

| Canny Edge Detection | Selecting Region of Interest |
| :---: |:---:|
| ![alt text][edges]  | ![alt text][edges_region]  |

#### Step 6: Line Detection and drawing onto the original image
##### a. Detecting the lines:
I applied the Hough Transform to the selected edges from previous step. This detected a list of lines depending on the tuning parameters of the algorithm:
* rho : distance resolution in pixels of the Hough grid
* theta : angular resolution in radians of the Hough grid
* threshold : minimum number of votes (intersections in Hough grid cell)
* min_line_length : minimum number of pixels making up a line
* max_line_gap : maximum gap in pixels between connectable line segments

For `rho` and `theta` I fixed the values respectively to `1` and `pi/180`, and had to play with the other parameters to fine tune and see how to constantly detect lines aligned with the road marking. Here also using interactive widgets helped a lot.

##### b. Drawing the lines:
A default function to draw the detected lines onto an image was provided in this project. Just applying that function we could see the multiple line segments around the edges of some lane lines on the road.

| Raw Lines Detection (YellowLeft.jpg) | Raw Lines Detection (SolidWhiteRight) |
| :---: |:---:|
| ![alt text][rawlines_1]  | ![alt text][rawlines_2]  |

I modified this `draw_lines` function to get one single line fully extended on each side of the road, that map with the lane marking, in the following way:
* Separate left-side and right-side lines by calculating the line's slope:
  - if slope > 0, line should be on right side
  - if slope < 0, line should be on left side
* For each group of lines (right and left), compute:
  - the average slope
  - the average y-intercept
* The resulting right line (respectively left line) will be defined by the average right-slope value and average y-intercept value
* Draw each resulting line segments starting from the bottom on the image to about the middle of the image. More precisely I drawn the lines for the following for the points defined by:
  - y = image_height
  - y = image_height * 0.6

The results look like this:

| Raw Lines Detection (YellowLeft.jpg) | Raw Lines Detection (SolidWhiteRight) |
| :---: |:---:|
| ![alt text][line_ex_1]  | ![alt text][line_ex_2]  |

### 2. Potential shortcomings with the current pipeline


TBD


### 3. Some possible improvements

TBD
