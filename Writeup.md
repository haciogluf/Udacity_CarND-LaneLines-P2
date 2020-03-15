# **Project 2: Advanced Lane Finding** 

## **Project Summary**

This project aims to detect lane lines on the road. A pipeline which finds the lines will be presented below.

### **Content:**

**1.** Camera Calibration

**2.** Pipeline Explanation

**3.** Possible Shortcomings & Improvement Points

[//]: # (Image References)

[image1]: ./examples/CameraCalibration.png "CameraCalibration"
[image2]: ./examples/UndistortedImage.png "Undistorted"
[image3]: ./examples/BinaryImage.png "Binary"
[image4]: ./examples/WarpedImage.png "Warped"
[image5]: ./examples/Histogram.png "Histogram"
[image6]: ./examples/LanePixels.png "LanePixels"
[image7]: ./examples/LanePixelsPrior.png "LanePixelsPrior"
[image8]: ./examples/WarpedOriginalImage.png "WarpedOriginalImage"
[image9]: ./examples/Result.png "Result"

---

## **Reflection**

### **1. Camera Calibration**

Camera calibration is done by using 9x6 chessboard images which are provided in the project sandbox. First, findChessboardCorners function of CV2 is used to detect corners on chessboard images, and all valid image/object points are appended in an array. Then, the image and object points are used as input for calibrateCamera function of CV2 to undistord an grayscale image. Additionally, the image and object points data is stored in CalibrationData.p file.

![alt text][image1]

### **2. Pipeline Explanation**

***Undistort an image*** 

A distortion correction function is defined during camera calibration *"UndistortImage"*. This function is used by loading the *"CalibrationData.p"* to undistort an image.

![alt text][image2]

***Create a binary image by using gradient and clolor transformations***

A combination of gradient and color transformations is appied to the undistorted image. First, x and y axes gradients are obtained by sobel transformation. Absolute value of x and y gradients are used in AND logic to create a binary. 

Then magnitude based on the x and y gradients is calculated by the formula: sqrt(x^2+y^2). Additionally, since the lane lines should have high gradient in a particular orientatin, the direction of the gradients is calculated by the formula: arctan(y/x). Logical AND of magnitude and direction binaries are combined with the previously created binary (x/y gradients)

Finally, another binary is created by checking HLS color transformation to be able to detect different colored lines under different lighting conditions. This binary is also combined with the previously created binary. All can be seen below in the picture.

x axis gradient thresholds = (30,100)

Y axis gradient thresholds = (50,100)

Magnitude thresholds = (80,150), kernel = 9

Direction gradient thresholds = (0.8,1.2), kernel = 15

HLS s-channel thresholds = (150,200)

![alt text][image3]

***Create a bird-eye view image by perspective transformation***

A function is defined to warp the binary image by perspective transformation. First the vertices are defined to detemine the area which will be transformed. The Vertices are the corner points of a trapezoidal shape, which is basically the region of interest. 

The region of interest is warped to a rectangle. X points of the rectange corners are determined as average of the x points (separated for left and right).
e.g: x_left = (vertice_x_bottom_left + vertice_x_top_left)/2 + offset. Here, the offset is an optional calibration parameter, and its value is zero for this project.

y points are defined as maximum and minimum y values of the image.

![alt text][image4]

***Finding Lane Pixels***

*Histogram View*

Histogram of the bottom half of the image helps to identify the binary activations of the image. Peaks in the histogram view is used to defined where to start looking for pixels of the lines.

![alt text][image5]

*Sliding Window Method*

Bottom points of the left-side and right-side peaks of the histogram are determined as starting points for the sliding window search method. Sliding window method starts from the bottom of the image and continues to top. 

Window number is defined as 9 for this project. The height of the windows will be height of the image divided by the number of the windows. Window margin is defined as 100; so,the width of the windows are equal to 200. Once the pixels in the window are found, the mean of the pixels are calculated an used as the center of the next window. Therefore, if there is a curve on the road, the rectange windows will follow the curve throuhg the frame.

After finding all pixels for the left and right lines, second order polynomials (x = ay^2+by+c) are fitted to the (x,y) points representing the pixels. 

![alt text][image6]

*Search from Prior Method*

Search from prior method is a way to used last (x,y) points to identify searching area instead of sliding window method. Search from prior method uses left and right curve parameters (a,b,c) and calculates new x values bases on the new binary image. Margin of the search area is defined as 100 for this project; so, the width of the search area is equal to 200. The method first finds the activated pixels around the new x values. Then fits new curves to those pixels. 

This method can only be used if a curve is fitted for the previous frame. In this document, the same image(binary) is used for both sliding window method and search from prior method to demonstrate/compare the results. A selection method for these 2 methods will be presented later. 

![alt text][image7]

***Extracting Lane Data***

*Finding Curvature Radius & Finding Vehicle Position on the Lane*

Curve radius is used as a feedback to control vehicle steering angle to keep the vehicle between lines. The curve radius can be calculated by using first and second order derivative of the curves. In this project, the average of the x values from left and right lines are used to fit another curve between 2 lines. y value, to calculate radius of the curvature, is selected as the bottom of the image, which is the closest point to the vehicle.

![alt text][image8]

Here the units are always pixels. The data needs to be converted to the real world units, like meters. In order to convert the data from pixels to meter, the warped image can be taken as reference. X axis difference between 2 lines accoring to wrapped binary image is roughly 500 pixels. Based on the information that the width of the lanes are 3.7 meters. Therefore, the conversion coefficient for x axis is equal to 3.7/500. The dashed lines are 3 meters long. It is assumed that the height of the picture below corresponds to 40 meters in real world. Therefore, the conversion coefficient for x axis is equal to 40/720.

The curve equation: x= ay^2+by+c in pixel can be converted to meter by the equation: x= mx/(m^2)a(y^2)+(mx/my)by+c

Vehicle position on the lane is found by comparing the middle x point of the image and the bottom x point of the fitted curve(average).

Curvature data based on the example image:

Radius of Curvature: 5107m
Vehicle is 0.04m left of center

***Visualization***

After the detection and data extraction, the lane and the informations are projected to the original image.

![alt text][image9]

***Pipeline***

*Lane Class and Sanity Check*

A class for the detected lane is defined to keep the latest data. This data is used for sanity checks. 

Lane class includes the following information:

* Latest curve parameters (left and right)
* Latest x points (left and right)
* Latest y points
* Latest curvature radius
* Latest drift value
* Latest lane detection information
* Latest sanity check information

Sanity check is conducted by the following 3 steps:

* Change of radius is below a threshold.
* Change of horizontal drift is below a threshold.
* Lanes are roughly parallel: Multiple points along the lines are checked.

*Pipeline Logic Flow*

* Identify the lane lines (Always sliding window method is used for the first step)
    * If the lane is detected use search from prior method
* Sanity check after lane detection
* If Sanity check fails 3 times in a row OR lane is not detected in the previous cycle, use sliding window method for the next fram
    * For the current frame, take the average of current curve data and the latest valid curve data

*Example Video:*

See: [*Project2_Delivery_Report*](https://github.com/haciogluf/Udacity_CarND-LaneLines-P2/blob/master/Project2_Delivery_Report.html)

### 3. Possible Shortcomings & Improvement Points

There is a lot of calibration parameters in the algoritm, such as gradient threshold, color thresholds or region of interest for wrapping. If these parameter are not calirated well, it is always possible to detect undesired pixels in the image. This would lead wrong fitted curves. Calibration effort can be considered as a shortcoming, but it is inevidible. Therefore, the algoritm needs to be tested in very different road and lighting conditions, such as rainy weather or night time. 

Additionally, the current algorithm smooths the lane detection only if the detected lane does not pass sanity checks. Even in that case, smoothing is conducted with last valid detection and the latest detection. Improvement on this topic may increase robustness under different road and lighting conditions.
