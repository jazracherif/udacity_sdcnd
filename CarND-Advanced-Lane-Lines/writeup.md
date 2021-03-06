# Advanced Lane Finding Project

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

[calib]: ./output_images/calibration.png "calibration Image"
[undistorted]: ./output_images/undistorted.png "Undistorted"
[pipeline-out]: ./output_images/pipeline-out.png "Pipeline Out"
[warped]: ./output_images/warped.png "Warped picture"
[histogram]: ./output_images/histogram.png "Histogram"
[sliding-window]: ./output_images/sliding-window.png "Sliding Window"
[scatter]: ./output_images/scatter.png "Scatter plot"
[lane-window]: ./output_images/lane-window.png "Lane window"
[final]: ./output_images/final.png "Final Result"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

Relevant Files:
+ `code/calibrate.py`: Runs calibration on the checkerboad and stores the parameters in  `code/wide_dist_pickle.p`
+ `code/gradient_threshold.py`: Contians function for the Gradient thresholding logic
+ `code/pipelines.py`: Contains function for the thresholding logic including gradient and color thresholding
+ `code/lane_detection.py`: Contains function for the main procedure for detection lanes, Radius of Curvature, and Vehicle Position
+ `code/main.py`: The main entry point of the program

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the file `code/calibrate.py`

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world.

Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][calib]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I applied the distortion correction to one of the test images and got the following result:
![alt text][undistorted]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. The threshold and color transformation functions pipelines are done in `code/pipelines.py` and `code/gradient_threshold.py`.

The function pipeline() (`code/pipelines.py line 8`) runs the undistorded image into both a color and a gradient threshold transformation before combining them:

Color Threshold Transformation (`code/pipelines.py: line 20 - 28`):
1. Transform the undistored input image from RGB to HLS
2. **yellow_binary** output: the output of a color threshold on the H channel, done to capture the yellow line. (H=19)
3. **l_binary** output: the output of a thresholding on the L channel to keep lighter parts of the picture, filtering anything below 160.

Gradient Threshold transformation (combined_thresh() in `code/gradient_threshold.py: line 133`):
1. Run a Sobel operation on both x and y axis with kernel size 5 and thresholds (20,100)
2. Apply a Magnitude thresholding on both x and y axis with thresholds (30,100)
3. Apply a Direction of Gradient thresholding with angle threshold between 0.7 and 1.3
4. **sxbinary** output: Combine all 3 previous operation with an OR operations

I combined the output of the previous transformations in the following way:
1. Use the right half of the Gradient Threshold and AND it with the l_binary output
2. OR the result with the yellow_binary image which captures the left yellow line.

```python
combined_binary = np.zeros_like(sxbinary)
combined_binary[(yellow_binary==1) | ((l_binary==1) & (sxbinary_right==1))] = 1`
```
![alt text][pipeline-out]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform appears in line 195-202 in `main.py`. I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32([ [550, 470], [800, 470], [1150, 710],  [180, 710]])
dst = np.float32([[0, 0],[1280, 0], [1280, 720], [0, 720]])
```
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][warped]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The lane identication logic is implemented in file `code/lane_detection.py` and is called from within the process_image() function at lines 212-222

Two additional classes were defined to help with this:
1. A class to help maintain past information about the left and right lines that form the lane:
    ```python
    class Line()  #(main.py:line 90)
    ```
2. A class that implements exponential moving average to be applied to each of the lines polynomial coeffcients (a,b, and c in a.x^2 + b.x +c)
    ```python
    class movingAvg() #(main.py:line 11)
    ```
    
The Lane Detection logic is divided into 2 parts:
1. For the first image or whenever a problem is detected with the previous image, a call to find_lanes() (line 117) is made. This function takes as input the warped image and then applies the following logic:
    1. If previous lane data is available, generate a set of point that fit the line and add them to the warped picture. This create a simple continuity and availablility of data.
        ```python
        ploty = np.linspace(0, binary_warped.shape[0]-1, binary_warped.shape[0] )
        fitx = (current_fit.a*ploty**2 + current_fit.b*ploty + current_fit.c).astype(int)
        ```
    2. Create a histogram of the number of nonzero pixels at each x location, and find the leftmost and rightmost peaks. These peaks will correspond to the left and right line of the current lane. See an example below:

        ![alt text][histogram]
    
    3. Run a vertical sliding window around each peak capturing the nonzero pixels around a margin of 120px from the window center, and recompute the window center whenever it contains more than 50 points. There are a total of 9 windows each of 80px height. See an example of this below:

        ![alt text][sliding-window]

    4. Fit a polynomial line of degree 2 to the points detected for each line.
    5. Update the line's equation coefficients using an exponential moving average with beta=0.5
        ```python
        self.a = (self.beta_a * self.a + (1 - self.beta_a) * a )
        self.b = (self.beta_b * self.b + (1 - self.beta_b) * b )
        self.c = (self.beta_c * self.c + (1 - self.beta_c) * c )
        ```
    6. Store the line information in the line instance using function update_line_info() (`lane_detection.py: line 27`)
    
2. In case a lane has been detected, don't run the slow window sliding logic of 1.III but instead run a faster search algorithm by looking at points close to the previously detected line. The funtion process_next_image() (`lane_detection.py: line 258`) is executed and the following steps are followed:
    1. Retrieve each lane's fitted equations and keep points which are within a 100pixel margin from the line's center. The green region in the picture below illustrate the searched window.
    
        ![alt text][lane-window]

    2. To the detected points add an additional set which correspond to the line's equation in order to add robustness and continuity (similar to step 1-I).
    3. Concatenate the last 10 sets of observation together and fit a polynomial of degree 2. This is done in update_line_fit() (`lane_detection.py: line 7`). Below is a scatter plot of the left and right lines captured at the 4th image of the project_video.mp4 file:
    
        ![alt text][scatter]
        
    4. Update the current lane coefficients using exponential moving average.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The **Radius of Curvature** is calculated in function get_radius() in `lane_detection.py: line 71`.

1. First, for each line, I rescale all points from pixel to meter space by using the following factors:
    ```python
    ym_per_pix = 30/720 # meters per pixel in y dimension
    xm_per_pix = 3.7/1100 # meters per pixel in x dimension
    ```
2. I then fit a polynomial line of degree 2 and I calculate the angle of the tangent at the bottom of the picture, i.e, at pixel value 719. Here's the code for the left line from `lane_detection.py: lines 90-91`.
    ```python
    y_eval = 719
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    ```
    The same is done for the right line.

The **position offset** of the vehicle with respect to the center of the lane is calculated in function get_distance_from_center() `lane_detection.py: line 52`.

1. First, for each line, I rescale all points from pixel to meter space using the same factors as above.
2. Then I fit a polynomial line to each set and I calculate the x-value at the bottom of the picture (y=719)
3. The 2 points calculated correspond to the left and right corners of the lane, the center of which is calculated and then compared to the center of the picture itself. See below
    ```python
    center = (left_pt + right_pt) / 2
    offset = (1280/2 - center) * xm_per_pix
    ```
    if the offset is negative, the car is on the **left** of the center by a value of the offset in meters.
    if the offset is positive, the car is on the **right** of the center by a value of the offset in meters


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines 227 through 284 in my code in `code/main.py` in the function `process_image()`.  Here is an example of my result on a test image:

![alt text][final]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

+ Here's a link to my [project_video result](./videos/project_video_out.mp4).
+ Here's a link to my [challenge_video result](./videos/challenge_video_out.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The most difficult part of this project has been lane stabilitzation, particularly as observed in the video challenge_video.mp4.

The approach I initially took relied on fitting only data for the current frame and updating the coefficients of the line's equation using exponential moving average. However, as observed in some conditions, the equation can vary a lot and the **beta** parameter may not be enough to smoothen the change. Too small and the shape will change too quickly, too large and it will take too long a time to adapt to changes.

In order to mitigate the shortcomings of the beta parameter, I added a threshold mechanism, increasing the beta value when the delta changes for the equation's coeffcient became large enough, and even disregarding the change altogether if the delta was above 100x of the last value. I also had different threshold and beta values for each coefficient, however choosing the right thresholds proved difficult as it worked in some conditions but not in others.

In order to resolve this satisfactorily, I ultimatly resolved to maintain a cache of the last 10 observed frames and fit a polynomial on all those points in order to increase robustness. This worked better than the thresholding mechanism. On top of that I combined the current frame observation with points generated using the current fitted line in order to always have a minimum number of points in case nothing is detected in the picture. This helped deal with cases such as going under the bridge when the lane marking become too dark to be detected.

Finally, I added a fallback mechanism in case the lane detection fails by going back to finding the lanes using the vertical sliding window logic.

More work needs to be done to make this module more robust, in particular:
+ The left mark is currently expected to be a yellow line and this must be improved for middle and right lanes situations.
+ The hardcoded *src* and *dst* values for the perspective transform are hardcoded for the current video format and may not be optimal
+ The scaling factors from pixels to meters for the radius of curvature and vehicle position needs tuning
+ More testing is needed in different lighting conditions: for example, the system may fail in the dark
+ More testing is needed on windy road in order to evaluate how quickly the system is able to follow the curves. Tuning the beta smoothing parameter and the catching mechanism will be needed.
