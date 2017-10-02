
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

[image1]: ./examples/undist_chess_board.png "Undistorted"
[image2]: ./test_images/straight_lines1.jpg "Test image"
[image3]: ./examples/thresholded_binary_image.png "Binary Example"
[image4]: ./examples/warped_image.png "Warp Example"
[image5]: ./examples/warped_binary_image.png "Binary warped Visual"
[image6]: ./examples/histogram.png "Histogram"
[image7]: ./examples/sliding_windows.png "Sliding windows"
[image8]: ./examples/margin_search.png "Margin search"
[image9]: ./output_images/final_image.png "Test image output"
[image10]: ./output_images/test_images_out.png "Test images output"

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the fourth code cell of the IPython notebook located in "lane_lines.ipynb".  

I started by preparing "object points", which were the (x, y, z) coordinates of the chessboard corners in the world. I assumed that the chessboard was fixed on the (x, y) plane at z=0, such that the object points were the same for each calibration image.  The variable `objp` is just a replicated array of coordinates, and `objpoints` were appended with a copy of it every time I successfully detected all chessboard corners in a test image.  `imgpoints` was appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I applied the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
 
I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in the 5th cell  in `lane_lines.ipynb` file). After a lot of trial with different color spaces I found that the L Channel from the `LUV` color space, with a minimim threshold of 225 and a maximum threshold of 255, did a very good job of idetifying the white lane lines while ignoring the yellow lines.  

The B channel from the `Lab` color space, with a minimum threshold of 155 and a maximum threshold of 200, did a better job than the S channel in identifying the yellow lines and ignored the white lines.  

I applied the gradient thresholds in the x direction, y direction but they did not add much information and hence I did not include them in my pipeline.

Here's an example of my output for this step.  

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in the 7th code cell in `lane_lines.ipynb`.  The `warp()` function takes as inputs an image (`img`), and the source (`src`) and destination (`dst`) points are set inside the function.  I chose to hardcode the source and destination points in the following manner:

```python
src = np.float32([[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
                 [((img_size[0] / 5) - 50), img_size[1]],
                 [(img_size[0] * 5 / 6) + 35, img_size[1]],
                 [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
    
dst = np.float32([[(img_size[0] / 4), 0],
                 [(img_size[0] / 4), img_size[1]],
                 [(img_size[0] * 3 / 4), img_size[1]],
                 [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 206, 720      | 320, 720      |
| 1101, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I obtained a perspective transform of the the camera images converted into binary images using the `warp()` function. The lanes lines can be easily separated as shown in the figure below.

![alt text][image5]

Then I did the following steps to draw the lane-lines:  

* Plot the histogram of the binary warped image which gives a plot of the frquency distribution of the non-zero pixels in the image. The histogram provides information regarding the position of the lane-lines. The x and y position where the lane lines are present are identified and stored in the `leftx_base` and `rightx_base`. 

![alt text][image6]

* Follow a sliding window search strategy to identify the x and y positions of all non-zero pixels in the image in each window and search within a margin around the `leftx_base` and `rightx_base` positions. 9 sliding windows are used to search along the height of the image. This is implemented inside the `sliding_window_search()` function in the 10th code cell of the `lane_line.ipynb` notebook.

![alt text][image7]

* A margin search is used to apply the sliding window within a margin of +/- 100 pixels around the `leftx_base` and `rightx_base` found in the particular sliding window. This assumption is practical since lane lines are not found randomly across the image. I stored the positions of the left lane line points and the right lane points and fit them using a 2nd degree polynomial. This is achieved inside the `margin_search()` function in the 11th code cell of the `lane_line.ipynb` notebook.

![alt text][image8]

The code for these steps have been written in the 10th and 11th cells of `lane_lines.ipynb`.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

For calculating the radius of curvature it is necessary to find a relationship between the distance between 2 pixels in the image and the actual distance between 2 points on the road. Assuming the meters per pixel in the y direction and x direction to be 30/720 and 3.7/700 respectively the value of the lane line points on the road was obtained. The points were fit into a 2nd degree polynomial and the radius of curvature was calculated using the formula mentioned [here](https://www.intmath.com/applications-differentiation/8-radius-curvature.php).   

The position of the vehicle with respect to the center was calculated by subtracting the mid-point of the width of image from the mid-point of the x-values of the left lane and right lane. This is achived in the `center_offset()` function written in the 12th code cell of `lane_lines.ipynb`. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The region between the 2 lane lines was colored in green so that the lane area was identified clearly.  Here is an example of my result on the test image:

![alt text][image9]

Here is the output on a few more test images:

![alt text][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a sample implementation of the pipeline run on the project video. 

![](./output_images/sample.gif)


Here's a [link to my video result](https://youtu.be/06kv6iewOg8) on YouTube. 

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I faced a few issues when I hadn't averaged the lane lines obtained over a few frames. In a few cases the lane lines jumped around a bit and didn't look stable. After averaging over 10 frames, the lane lines looked fairly stable.  

My pipeline would suffer if there is occlusion on the lane lines, that is if a car in the front moves on the lane lines. Also, it would be difficult to track the lanes if the car changes lanes. As I figured out my pipeline fails in the harder challenge video where the lane lines have a lot more bends. The assumption of fitting the lane lines using a quadratic polynomial is not robust here. My pipeline also suffers when there are shadows on the road or there is bright sunlight on the road as it can't distinguish between the white lane lines and white reflection from the road. To summarize I have learned that it is relatively easy to finetune a pipeline to work well in ideal road and weather conditions, but it is really challenging to find single combination which works robustly in any condition.  

I feel that using a 2 stream ConvNet suitable for videos, where one stream identifies the lane lines and the other stream takes advantage of the temporal information could be useful. Also, since videos are sequential data Recurrent Neural Networks could be used as they work well in sequence analysis. I would love to continue working on the project and explore these techniques.


```python

```
