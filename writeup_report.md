## Advanced Lane Finding Writeup Report

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/chess_board_corner_10.jpeg "Corner First Example"
[image2]: ./output_images/chess_board_corner_14.jpeg "Corner Second Example"
[image3]: ./output_images/chess_board_distortion.png "Chessboard Distortion Correction"
[image4]: ./output_images/undistortion.png "Distortion Correction"
[image5]: ./output_images/combined_gradiant_threshold_bianry.jpg "Combined Threshold Binary"
[image6]: ./output_images/color_threshold_binary.jpg "Color Threshold Binary"
[image7]: ./output_images/color_gradiant_threshold.jpeg "Combined and Color Threshold Binary"
[image8]: ./output_images/birdseyeview.jpeg "Birds Eye View"
[image9]: ./output_images/histogram.jpeg "Road Transformed Histogram"
[image10]: ./output_images/slide_window.jpeg "Slide Window Example"
[image11]: ./output_images/draw_lane.jpeg "Draw Lane Example"
[image12]: ./output_images/result_2.jpeg "Result Example"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

Here are the main step I used:

**CAMERA CALIBRATION** ---> **DISTORTION CORRECTION** ---> **COLOR & GRADIENT THRESHOLD** 
---> **PERSPECTIVE TRANSFORM** ---> **FINDING THE LANE LINES** ---> **DRAW LANES**

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.
 
I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

For detected all chessboard corners in folder cameral_cal:

![alt text][image1]
![alt text][image2]

I then defined a function `distortion_correction(img, objp, imgp)` with input parameter `objpoints` and `imgpoints`, which used the above output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the chessboard image using the `cv2.undistort()` function and obtained this result:

![alt text][image3]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

**Note:**
- The road sign's angle of view and size between two pictures
- The right edge and bottom edge changes of the two pictures

![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I defined different gradient functions and apply threshold in each of them:
`abs_sobel_thresh()`: Caculate directional gradient, apply threshold
`mag_thresh()`: Caculate gradient magnitude, apply threshold
`dir_threshold()`: Caculate gradient direction, apply threshold

In order to filter noise to identify the lane lines, I defined a new function `get_combined_sobel()` to combine the thresholds.
Here is an example of my output for this step, we can find the pixels of lane-line, still a lot of noise exist:

![alt text][image5]

From the lesson of [HLS and Color Thresholds](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/d7542ed8-36ce-4407-bd0a-4a38d17d2325), it tell us color sapce like HLS(hue, lightness and saturation) can make more robust, and S-channel picks up the lines well. Therefore, I used `hls_select()` function to make better robus identificaiton of the lane lines.
Here is an example of my output for this step:

![alt text][image6]

Finally, I used a combination of color and gradient thresholds to generate a binary image.
```python
undist_test_image = distortion_correction(test_image, objpoints, imgpoints)
color_binary = hls_select(undist_test_image,thresh=(170,255))
combined_binary = get_combined_sobel(undist_test_image)
result = cv2.bitwise_or(combined_binary, color_binary)
```
Here's an example of my output:

![alt text][image7]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warped_image()`, which appears in the 5rd code cell of the IPython notebook.  The `warped_image()` function takes as inputs an combined color and gradiant image (`combined_image`), as well as image orientation (`orient='M'`), for source points and destination points.  I chose the hardcode the source and destination points in the following code, :

```python
    src = np.float32(
    [[730, 460],    
     [1100, 690],   
     [200, 695],    
     [600, 455]])   
    dst = np.float32(
    [[1120, 0],      # top right
     [1120, 680],    # bottom right
     [200, 680],     # bottom left
     [200, 0]])      # top left  
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 730, 460      | 1120, 0       | 
| 1100, 690     | 1120, 680     |
| 200, 695      | 200, 680      |
| 600, 455      | 200, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:
**Note**
The four source points in the left picture

![alt text][image8]

#### 4. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Now I have a threshold warped image and it's already to map out the lane lines(the obove right image).

##### Find the lane lines

After applying calibration, thresholding, and a perspective transform to the test image, now the lane lines stand out in binary image, then I defined `locate_lane_lines()` to explicitly find out the pixels part of the lines and which belong to the left line and which belong to the right line.

Here is a histogram for the lane lines result:

![alt text][image9]

##### Sliding the windows/Search in a margin around the line position
In order to find the lines more robust, I split the windows to 9 sub-windows, step through each sub-window to identify the good pixels in right line and right line through fit polynomials.
Optional, if we have known where the lines are in one frame of video and keep track of the lines, we can search in a margin around the line position.

Here is the visualize example result:

![alt text][image10]
![alt text][image11]


#### 5. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in 9rd code cell in IPython Notebook. I defined `draw_lanes` to draw lane area. 

Here is an example of my result on a test image:

![alt text][image12]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The result seems perform reasonably acceptable on the test video based on foudamental techniques from the lesson. However it hard to have a good result on the challenge video because of the complex road surface and some tips and tricks still neee to have a further study:

-  **Tracking:** During keep track of lane lines, I am not defined a class(such as `Class Line()`) to keep track of all interesting parameters. In the later, I planned to define a class to keep track of recent detections. This's method will benefit to perform sanity checks.
-  **Sanity Check:** There still need to find a available sanity check function to check the dection make senses, after checking that they are similar curvature, they are separated by approximately the right distance horizontally and they are roughly parallel, I believe it will have much improvement of performence in this project.

So, this writeup report doesn't means the end of the project or techniques, it's the just standing on the starting point of track, next go further.