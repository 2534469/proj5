## Writeup Template
### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Vehicle Detection Project**

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector. 
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[image1]: ./output_images/example_classes.png
[image2]: ./output_images/hog_images.png
[image3]: ./output_images/box_example.png
[image4]: ./output_images/pipeline_test.png
[image5]: ./output_images/test_heat_map0.png
[image6]: ./output_images/test_heat_map1.png
[image7]: ./output_images/test_heat_map3.png
[video1]: ./project_video.mp4

## [Rubric](https://review.udacity.com/#!/rubrics/513/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Vehicle-Detection/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

The code for this step is contained in the first code cell of the IPython 'Solution.ipynb' notebook under Hog Features .  

I started by reading in all the `vehicle` and `non-vehicle` images.  Here is an example of one of each of the `vehicle` and `non-vehicle` classes:

![alt text][image1]

I then explored different color spaces and different `skimage.hog()` parameters (`orientations`, `pixels_per_cell`, and `cells_per_block`).  I grabbed random images from each of the two classes and displayed them to get a feel for what the `skimage.hog()` output looks like with `orient = 9`, `pixels_per_cell=(8, 8)`,`cells_per_block=(2, 2)`,`hog_channel = 0`.
I tried out different color spaces as YCrCb, RGB, LUV, HLS, YUV to stack as well as a different size for color bins. The best ones were LUV and YCrCb, tha lates I used in my pipeline.



![alt text][image2]

#### 2. Explain how you settled on your final choice of HOG parameters.

I tried various combinations of parameters and have chosen to use all hog channels, as well as to encrease a spatial size to (32,32) and the number of bins for color histogram.

```python
    color_space= 'YCrCb'
    orient = 9
    pix_per_cell = 8
    cell_per_block = 2
    hog_channel = 'ALL'
    spatial_size = (32,32)
    hist_bins = 32
    spatial = True
    hist = True
    hog = True
```

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

I trained a linear SVM using a LinearSVM, see section Training SVM and calculating accuracy. As features I stacked the spatial, color histogram and HOG features as one vector and sclaed it using StandardScaler:
```python
scaler = StandardScaler().fit(X_train)
```
I trained a LinearSVC using those features. Before that I split the whole set into training and test set:
```python
svc = LinearSVC()
start = time.time()
svc.fit(X_train, y_train)

```

### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

I started with naive approach from the video and used just a sliding window between y_start and y_stop, what in my case were (400, 656). But this approach was extremely slow. That's why I swithced to the approach of calculating the HOG features for the whole image and then extract the partial features for an image-patch and calculating a color and spatial feature for this patch upscaling it with the scale factor.
Please, see function `def get_heatmap_and_decorated(img, scale = 1.5, cells_per_step = 2, y_start=400, y_stop=656, confidence_threshold = 0.01)`
Here is the example of windows found using this approach and corresponding heatmaps:

![alt text][image3]

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

Ultimately I searched on two scales using YCrCb 3-channel HOG features plus spatially binned color and histograms of color in the feature vector, which provided a nice result, please see function `process_image`.
I also used calls of process_image with different scales and starting/ending y. The idea is to search smaller objects next to the horizont, where they appear to be and bigger near to the ego-car. I als used a confidence-threshold, which is a distance to the hyperplane in the linear SVN. The closer the point appears to the 0(the plane), less confident is the classifier about it's predicted class. 

![alt text][image4]
---

### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)
Here's a [link to my video result](./output.mp4)


#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

I recorded the positions of positive detections in each frame of the video.  From the positive detections I created a heatmap and then thresholded that map to identify vehicle positions.  I then used `scipy.ndimage.measurements.label()` to identify individual blobs in the heatmap.  I then assumed each blob corresponded to a vehicle.  I constructed bounding boxes to cover the area of each blob detected.  

Here's an example result showing the heatmap from a series of frames of video, the result of `scipy.ndimage.measurements.label()` and the bounding boxes then overlaid on the last frame of video.
Plese, mention, these results show only one(not combined) call of get_heatmap_and_decorated.
For the video, I used the last 8 frames to evarage the heatmaps and then filter out those under the heatmap threshold:
```python
class Detection():
    def __init__(self, queue_length, heap_thr):
        self.all_detected_rects = []
        self.queue_length = queue_length
        self.heap_thr = heap_thr
        
    def put_boxes(self, boxes):
        self.all_detected_rects.append(boxes)
        if (len(self.all_detected_rects) > self.queue_length + 1) :
            self.all_detected_rects.pop(0)
            
    def get_accumulated_heatmap(self, img):
        heatmap = np.zeros_like(img[:,:,0])
        for rectangles in self.all_detected_rects:
            heatmap = add_heat(heatmap, rectangles)
        return apply_threshold(heatmap, self.heap_thr)
```
### Here is the frames:

![alt text][image5]

### Here is the output of `scipy.ndimage.measurements.label()` on the integrated heatmap:
![alt text][image6]

### Here the resulting bounding boxes are drawn onto the last frame in the series:
![alt text][image7]



---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The problems I was facing that the cars in the far background were not detected, or there were a lot of false positives, especially in the area of the shadows on the road or in the trees next to the roads. Probably, one could use more than feature based on color and use different color spaces together. Another my idea was touse the confidence of SVC, I was hoping that false positives were appaering with a low confidence, but it was not always the case.
Probably one could try different scale/region combination more exastively, but this would take much more time or one should parallelize it using GPU.


