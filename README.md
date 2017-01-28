##Writeup Template
###You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./output_images/calibration1.png "Calibration1"
[image2]: ./output_images/undistorted.png "Undistorted"
[image3]: ./output_images/video_frame.png "Original Video Frame"
[image4]: ./output_images/video_frame_undistorted.png "Undistorted Video Frame"
[image5]: ./output_images/filtered.png "Filtered"
[image6]: ./output_images/test2.png "Test 2"
[image7]: ./output_images/birdeye.png "Birdeye"
[image8]: ./output_images/peaks.png "Peaks"
[image9]: ./output_images/fits.png "Fit Curve"
[image10]: ./output_images/lane_line_curve.png "Lane Lines Drawn"
[image11]: ./output_images/info_text.png "Curvature Radius and Car Position"
[video1]: ./project_video.mp4 "Fit Visual"

## [Rubric](https://review.udacity.com/#!/rubrics/476/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

###Camera Calibration

####1. Have the camera matrix and distortion coefficients been computed correctly and checked on one of the calibration images as a test?

Code cell 2 contains the code for this step. Images of a 9x6 chessboard provided in the camera_cal folder of project repository are used for computing the calibration. 

For each image, `cv2.findChessboardCorners()` finds all chessboard corner pixels. `cv2.calibrateCamera()` computes the camera matrix and distortion coefficients by averaging the distortion found in each image.

`objp` is just a 9x6 matrix of points on the same plane the chessboard corners map to. Since all images is of the same chessboard, `objp` is static.

The result has been checked against the images used for calibration, a sample is shown below:

![alt text][image1]![alt text][image2]

###Pipeline (single images)

####1. Has the distortion correction been correctly applied to each image?

A pipeline function has been defined, which takes a video clip frame as input, and output the same image undistorted and with lane boundaries drawn out. This is the function `process_image()` in cell 14.

The first step is to call `cv2.undistort()` to undistort the input image. `mtx`, `dist` are the outputs from the camera calibration above.

![alt text][image3]![alt text][image4]

####2. Has a binary image been created using color transforms, gradients or other methods?

The next step in the pipeline is to transform the undistorted image into a binary image that outlines the lane lines. `plot_lane()` in cell 8 is the pipeline function for this purpose.

`plot_lane()` filters input image twice, once with gray scaled image, and then with only the satuation channel of the input. The results are then combined. Filtering on only the satuation channel works better for white lane lines when the road surface is light gray. 

`filter_img()` in cell 7 defines the filter. It consists of multiple gradient thresholds, including Sobel threshold (defined in `abs_sobel_thresh()` in cell 3), magnitude threshold (defined in `mag_thresh()` in cell 4), and directional threshold (defined in `dir_thresh()` in cell 5). 

![alt text][image4]![alt text][image5]

####3. Has a perspective transform been applied to rectify the image?

`process_img()` then applies perspective transform on the binary image with `warp_lane()` in cell 9.

The following source and destination points are hardcoded for the transform:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 210, 703      | 310, 703      | 
| 1094, 703     | 950, 703      |
| 578, 460      | 310, 100      |
| 706, 460      | 950, 100      |

The transform was tested against the images provided in project repositary test_images folder. Below is an example:

![alt text][image6]![alt text][image7]

####4. Have lane line pixels been identified in the rectified image and fit with a polynomial?

The pipeline function then invokes `plot_curvature()` in cell 10 to identify pixels in the rectified image. 

It first initializes the starting points of the 2 lanelines as `l_centre` and `r_centre`. These 2 points are found as the peaks of the histogram of vertical pixcel counts of the bottom half of the input image. `l_centre` is the peak in the left half of the input image, `l_centre` the peak in the right half.

Then it uses a sliding window to scan the horizontal sections of the input image. Historgram, however, is not again taken for the entire section, but a small area no more than 15 pixels from the `l_centre` and `r_centre`. This prevents noice pixcels from being identified as lane line when lane line is broken.

`l_centre` and `r_centre` is shifted towards the new peaks found in each turn. But both centres are shifted by the same direction and distance. The histogram peak that counts more pixels determines this direction and distance. This is also a measure that helps reducing the influence of noice pixcels.

With pixels on both lane lines identified, `fit_curve()` in cell 11 fits 2 polynomials in the pixels.

`draw_curve()` in cell 12 draws the polynomial curves on an image in original perspective. It first plots the curves on a blank background in the perspective where the polynomials are calculated. It then warped the curve plot back to original perspective, and overlay the result onto original image. 

Below are plots of identified lane line pixels, polynomials fitting the pixels, and image with the lane line curves drawn on top. They are generated based on one of the test images in project repository.

![alt text][image8]![alt text][image9]![alt text][image10]

####5. Having identified the lane lines, has the radius of curvature of the road been estimated? And the position of the vehicle with respect to center in the lane?

`fit_curve()` also calculates curvature radiuses of the polynomials. The curvature data are converted from pixel values to metre measures before the calculation.

`draw_info()` in cell 13 is responsible for determining lane curvature radius and car position to rectified image, and add these  to rectified image as text.

Lane curvature radius is simply taken as the average of the 2 lane line curvature radiuses. The car position, measured by how far the car is off lane centre, is calculated by how much the lane centre is off the image centre. Lane centre is the mid-point of the bottom 2  of the lane line pixels identified earlier.

---

###Pipeline (video)

####1. Does the pipeline established with the test images work to process the video?

`challenge_video()` in cell 15 utilises `clip.fl_image()` process input video frame by frame, each frame is processed with `process_image()`.

Here's a [link to video result](./project_video.mp4)


---
##Discussion

The approach taken follows mostly the steps introduced in the course lessons and labs with minor tweaks. Parameters are found empirically with many iterations of trials.

While the solutions to most steps, such as camera calibration, are quite standard, the process of highlighting and identifiying lane line has no single 'best' approach, and is discussed in the next two paragraphs.

Sobel edge detection based on gray scaled image does not perform well when the road surface is light gray in colour. White lane line is too similar to road surface, and thus hard to detect. This is resolved by applying edge detection directly to the satuation channel of image in HSV colour space. Pixels found in either detection are kept.

Lane line pixels are identfied by scanning the image in horizontal sections. This is prone to error when lane line is broken and noice pixels are present. To reduce such error, not the entire horizontal section is scanned. Instead, the scanning window has two boundaries only 20 pixels from the pixels identified in the last scan. Therefore, the scanning window moves horizontally with the lane line. If a lane line is broken, the scanning window of the broken lane line is shifted in the same direction as the scanning window of the other lane line, within which, more pixels are counted. 

The program is still far from perfect. It does not mark the lane lines in the challenge video clip correctly, where the yellow lane line's colour is faded, and there are lines of black asphalt on the road. Modifying the sobel edge detection process may not help in these situations. A smater way to indentify the lane line pixels may be needed. For example, instead of looking for an absolute peak, find a narrow band where the pixel count histogram peaks. 
