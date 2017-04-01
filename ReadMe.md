##Advanced Lane Finding Project

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

[image1]: ./output_images/undistorted.png "Undistorted"
[image2]: ./output_images/undistort5.jpg "Road Transformed"
[image3]: ./output_images/threshold0.jpg "Threshold"
[image4]: ./output_images/track0.jpg "Warp"
[image5]: ./output_images/fit0.jpg "Fit Visual"
[image6]: ./output_images/fin0.jpg "Output"
[image7]: ./output_images/warped.jpg "Warped"
[image8]: ./output_images/fin0.jpg "Final"

[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "Calibration.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result and finally wrote them to a pickle file: 

![alt text][image1]

```
objpoints = [] # 3d points in real world space
imgpoints = [] # 2d points in image plane.

/# Make a list of calibration images
images = glob.glob('camera_cal/calibration*.jpg')


/# Step through the list and search for chessboard corners
for idx, fname in enumerate(images):

    img = cv2.imread(fname)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # Find the chessboard corners
    ret, corners = cv2.findChessboardCorners(gray, (9,6), None)

    # If found, add object points, image points
    if ret == True:
        objpoints.append(objp)
        imgpoints.append(corners)
```

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]
#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image (function color_threshold() and abs_sobel_thresh()).  For color thresholding I used hls and hsv schemes with filters on s and v channels. Here's an example of my output for this step.  

![alt text][image3]

```
	gradx = abs_sobel_thresh(img, orient = 'x', thresh=(12,255))
    grady = abs_sobel_thresh(img, orient = 'y', thresh=(25,255))
    cbinary = color_threshold(img, thresh1=(100,255), thresh2=(50,255))
    preprocessImg[((gradx == 1) & (grady == 1) | (cbinary == 1))] = 255

```

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes steps in `process_img()`, which appears in lines 1 through 8 in the file `AdvancedLane.ipython`  in the 2nd code cell of the IPython notebook).  The `process_img()` function takes as inputs an image (`img`).  The source area we are trying to focus on is in the shape of a trapezoid. So we want to find out the four points in the trapezoid and transform them to our destination points. For the bottom of the trapezoid I am using the bot_trim to cut off the car hood. I also assumed the center of the picture is the center of the trapezoid and we are taking a certain percent off the middle(mid_width/2 and bot_width/2) to calculate x coordinates  of the four points. In addition, taking a certain percentage(height_pct, bot_width) of the picture height to get the y of the top of the trapezoid  :

```
    src = np.float32([[img.shape[1]*(.5-mid_width/2), img.shape[0]*height_pct], 
                      [img.shape[1]*(.5+mid_width/2), img.shape[0]*height_pct],
                      [img.shape[1]*(.5+bot_width/2), img.shape[0]*bot_trim],
                      [img.shape[1]*(.5-bot_width/2), img.shape[0]*bot_trim]
                      
                      ])

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 589, 447      | 320, 0        | 
| 691, 446      | 960, 0      |
| 1126, 673     | 960, 720      |
| 154, 673      | 320, 720        |

Below is the result for the transformed image for image 0.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I used sliding window medthod to apply convolution to maximize hot pixels in each window and thresholded the left and right lanes. The are included in the AdvancedLane.ipython file in the process_img function. The window fitting result and codes are pasted below:

![alt text][image5]


```
  for level in range(0,len(centroids)):
        leftx.append(centroids[level][0])
        rightx.append(centroids[level][1])
        # Window_mask is a function to draw window areas
        l_mask = curve_centers.window_mask(width,height,warped,centroids[level][0],level)
        r_mask = curve_centers.window_mask(width,height,warped,centroids[level][1],level)
        # Add graphic points from window mask here to total pixels found 
        left[(left == 255) | ((l_mask == 1) ) ] = 255
        right[(right == 255) | ((r_mask == 1) ) ] = 255

```



#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code of identifying lane curature is in the process_img() function with curverad being the variable of curve radius.


```
ploty = range(0, warped.shape[0])
res = np.arange(warped.shape[0] - (height/2),0,-height)

curve_fit_cr = np.polyfit(np.array(res, np.float32)*ym_per_pix, np.array(leftx, np.float32)*xm_per_pix,2)
    curverad = ((1+ (2*curve_fit_cr[0]*ploty[-1]*ym_per_pix + curve_fit_cr[1])**2)**1.5)/np.absolute(2*curve_fit_cr[0])
    
    
```



#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.
Below is an exmaple out put of the test images.

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://youtu.be/lNIP5YH7fqc)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I spent quite some time on identifying the source and destination points for the trapezoid area. I followed the Q&A session and obtained the points through certain percent of the height and width. However, this is still hardcoding since  we know farely well where the sky/ car hood is. To improve this process we can try to automatically identify the trapezoid area.

