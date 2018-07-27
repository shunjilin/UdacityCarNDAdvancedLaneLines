# Advanced Lane Finding Project

Shunji Lin

---

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

[image1]: ./writeup_images/undistort.png "Undistorted"
[image2]: ./writeup_images/undistortroad.png "Undistorted Road"
[image3]: ./writeup_images/colorgradientthreshold.png "Threshold"
[image4]: ./writeup_images/sourcedest.png "Source and Destination Points"
[image5]: ./writeup_images/warpparallel.png "Warp Parallel Lanes"
[image6]: ./writeup_images/polynomialfitted.png "Polynomial Fit"
[image7]: ./writeup_images/projectlane.png "Project Lane Lines"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

All implentations/code are located in [./advanced_lane_lines.ipynb](./advanced_lane_lines.ipynb)

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. All chessboard corners were successfully detected in 17 out of 20 of the calibration images given in ./camera_cal directory.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color (HLS) and gradient (sobel in x-axis) thresholds to generate a binary image. The thresholds were fine tuned to create clear lane lines with minimal noise on the test images for ./project_video.mp4. Here's an example of my output for this step:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`. The `warp()` function takes as inputs an image (`img`), as well as source (`src_pts`), destination (`dst_pts`) points, and a 3x3 transformation matrix (`M`) that will be used by [cv2.warpPerspective()](https://docs.opencv.org/2.4/modules/imgproc/doc/geometric_transformations.html?) to transform the image. I chose the hardcode the source and destination points in the following manner:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 590, 455      | 220, 0        | 
| 696, 455      | 1088, 0       |
| 220, 719      | 220, 719      |
| 1088, 719     | 1088, 719     |

The source points are in red, and destination points are in blue:

![alt text][image4]

I verified that my perspective transform was working as expected by drawing the `src_pts` and `dst_pts` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I identify the lane lines using sliding windows across the y-plane and identifying where the activated pixels were, taking their mean, and ensuring that each window falls within some margin of the previous windows. This is shown in the function `find_lane_pixels()`. Another method is used to prevent the reconstruction of lane lines on a new frame if a prior (cached) lane line pair is given. This function `search_around_poly()` does the aforementioned, constructing the line around the margins of the prior's lane lines. Finally the function `fit_polynomial()` encapsulates the above two functions and returns the appropriate fitted 2nd order polynomial coefficients for the left and right lane lines, selecting the appropriate operation according to whether the prior lines are given.

The fitted 2nd order polynomial looks like this:

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

If the 2nd order polynomial of the lane line is of the form `x = a*y**2 + b*y + c`,
the curvature radius can be calculated by the formula `1 + (2*a*y + b)**2)**1.5) / abs(2*a)`,
where y is the y value at which we want to evaluate the curvature of the lane line.

The function to calculate the curvature is given by `measure_curvature_real()` and care have to be taken
to ensure that it is given the 2nd order polynomial adjusted for meters instead of pixels.

For the offset (position of the vehicle with respect to center), I had to calculate 2 points:
1) Center of the lane, which is given by `left_lane_position + (right_lane_position - left_lane_position)/2`,
which can be calculated from the polynomial coefficients.
2) Center of the car, which is approximated by taking `image.shape[1] / 2` (half of the width of the image)
Then the offset is given by `offset = center_of_car - center_of_lane`.
Again, care must be taken to make sure everything is adjusted from pixels to meters.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Here is an example of the lane line being plotted back to the road image, along with the curvature radius and offset values:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

The final pipeline involves applying the transformations in the order: undistort, color and gradient threshold, warp. It also involves projecting the lane lines onto the frame, along with the curvature radius and offset values.

In addition, I implemented a Line class to store/cache important states in order to improve the performance of lane detection on the project video.

This class keeps track of a buffer of (max) 10 valid pairs (frames) of lane lines, so that an average of these lane lines can be taken to be projected onto the video. This smoothens the lane line projections, and reduces jitter.

I also have a tolerance factor of 5 for each coefficient of the polynomial for the lane lines, such that the difference between the current frame's lane lines' coefficients and the prior lane lines' coefficients (average of the lane lines in the buffer) cannot be more than 5x the current lane lines' coefficients. Otherwise, the frame's lane lines are dropped. This removes some unwanted noisy lines. Finally a limit of 5 consecutive drops is allowed until the buffers are cleared and new lines have to be redrawned, so that the buffer is too far back in terms of frames.

Overall the above operations allowed the lane line projection to work reasonably well on the project video.

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The main issue with my implementation as is is that it is currently overfitting the project video, and unable to deal with the harder videos. For the challenge video, the pipeline is sometimes mistaking the middle of the lane as a lane line, and one way to deal with that would be to ensure that the lane lines are consistently apart by a minimum distance. For the harder challenge video, the lighting varies quite significantly, and getting the pipeline to work on that video would probably require further fine tuning the gradient and color thresholds.
