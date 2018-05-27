## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./output_images/success_calibration.png "Success Calibration Images "
[image2]: ./output_images/success_calibration.png "Unsuccess Calibration Images"
[image3]: ./output_images/test_calibration.png "Test Calibration"
[image4]: ./output_images/test_calibration_2.png "Test Calibration"
[image5]: ./output_images/threshold_sample.png "Threshold Sample"
[image6]: ./output_images/threshold_combined.png "Threshold Combined"
[image7]: ./output_images/warped_image.png "Warped Image"
[image8]: ./output_images/transform_plot_image.png "Transform Plot Image"
[image9]: ./output_images/rcurve_formula.png "RCurve Formula"
[image10]: ./output_images/result.png "Result"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it! :)

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first two sections of the IPhython notebook located in "./AdvLaneLine.ipynb".

I start the project by conducting **camera calibration**. In this section, I prepare the object points, which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. Nevertheless, I put the unsuccessfully detected image in `not_found`

![alt text][image1]
![alt text][image2]

Through these result, we could see that 85% (17 out of 20) was successfully recognized and only 15% (3 out of 20) was failed to be processed by the algorithm.

On the next section, **distortion correction**, `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image3]


### Pipeline (single images)

There are 2 kind of pipeline that I provide in the notebook:
1. From the third section, "Color / Gradient Threshold", until "Stack Lane Detection Information", which describe the full description and code of how I achieve the result, and
2. the "Image Pipeline" and the second section from bottom, which is the shorten version that I just call every functions to achieve the final output.

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I use the `undistort` function that I have created from the previous step and compare it with the original image. To be noted, we could see the slightly difference from the hood of the car like in the picture below

![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

In order to identify which kind of combination that would achieve a good threshold, I create various kind of thresholdings, from absolute sobel, magnitude, direction, RGB, and HLS threshold, which the unit test for each function could be seen below

![alt text][image5]

To be realized, the combination between Sobel X, Sobel Y, Magnitude, Direction, and Saturation could help me to achieve quite a good result like the picture below. I am using `test_images/test1.jpg` as the test image for this purpose.

![alt text][image6]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I create 2 functions for this section, "Transform Image". The first one is from the first project, `region_of_interest`, which helps me to apply masking for the `img`, so it only keeps the region of the `img` defined by the polygon formed from the hard-coded `vertices`. The second one is `perspective_transform`, which help me to tackle this section by transforming from the original into the bird-eye point of view. In this funcion, I decide to hard-coded the source and destination points in the following manner:
```python
img_size = (img.shape[1], img.shape[0])
src = np.float32(
    [[0, img_size[1]-offset],
     [560, 450],
     [720, 450],
     [img_size[0], img_size[1]-offset]])
dst = np.float32(
    [[200,img_size[1]],
     [200,0],
     [1160,0],
     [1160, img_size[1]]])
```

After I set the `src` and `dst`, I use it to create `M` and `MInv` rightaway for later purpose, to transform it back and apply to the original image, by using `cv2.getPerspectiveTransfor`. Lastly, I warped the test image by using `cv2.warpPerspective`. The result like in the image below

![alt text][image7]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In this section, basically I use the method that was describe in the lesson 15, which I put it all together in a function called, `detect_lane_lines`. In this function, first of all, I take the histogram of the bottom half of inputted image and create an output image to draw on and visualize the result. After that, I find the peak of the left and right halves of the histogram. These will be the starting point for the left and right lines. After I get the starting point of left and right lines, I set the number of sliding windows, height, size, and position. Then, I step it one by one.

After that, I extract the left and right line pixel positions, then I fit a second order polynomial to each. To convert it into meter, I re-calculate it again for each side like this code below

```python
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension

left_fit = np.polyfit(lefty, leftx, 2)
right_fit = np.polyfit(righty, rightx, 2)

left_fit_m = np.polyfit(lefty*ym_per_pix, leftx*xm_per_pix, 2)
right_fit_m = np.polyfit(righty*ym_per_pix, rightx*xm_per_pix, 2)
```

The result would be like the picture below

![alt text][image8]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

To calculate the radius of curvature of the lane and the minimum one, position of the vehicle with respect to the center, I create 2 functions, `calculate_lane_curvature` and `add_figures_to_image`. In `calculate_lane_curvature`, I calculate the radius of curvature by using this math formula below:

![alt text][image9]

and I transform it into code lines like below:
```python
left_curverad = ((1 + (2*left_fit_m[0]*y_eval + left_fit_m[1])**2)**1.5) / np.absolute(2*left_fit_m[0])
right_curverad = ((1 + (2*right_fit_m[0]*y_eval + right_fit_m[1])**2)**1.5) / np.absolute(2*right_fit_m[0])
```

After that, I get the curvature by summing both left and right curvature and divide it by 2 to get the mean. To get the minimum one, I use `min` function.

Then, in the 2nd function `add_figures_to_image`, I convert the curvature, minimum curvature, and center from pixel into meter. After that to know the position of the vehicle, I substract the `center` with the middle point of `x_axis`, then divide it with 100.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in 2 functions, `apply_to_original_image`, to apply the lane detection into the original image, and `add_figures_to_image`, to apply the information into the image. The result would be like the image below:

![alt text][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a result for the video output
![alt text][video1]

I try to implement it to the both challenge video, but still could get a proper lane detection. I seems that I need to tune up and create somekind adaptive thresholding for various environment.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

This pipeline still could not detect properly for the `challenge_video.mp4` and `harder_challenge_video.mp4` which is because of different environment. The thing is, this pipeline only still suitable for the environment with low-med light, so by using the saturation thresholding, it could detect the lane properly. In this situation, I think it could be improved by creating an adaptive thresholding. So, when the pipeline detect various kind of image of different environment, it could choose with kind of threshold that could detect lane lines properly.