# text-detection-swt
Stroke Width Transform Based Text Detector

Sample Results :

Stroke width filled and run :

![alt](/characters_1.png)

Final Bounding Box:

![alt](/boundingbox_1.png)

Stroke width filled and run :

![alt](/character_2.png)

Final Bounding Box:

![alt](/boundingbox_2.png)


Steps :

1. Based on the concept of Stroke Width Transform

2. A stroke in the image is a continuous band of a nearly constant width

3. Finding the stroke width of each image pixel

4. Grouping pixels into letter candidates using their stroke width value

5. Grouping letters into bigger constructs of words and lines



Pre Process image :

1) Reducing the colors in the image using K-means clustering
2) Converting the reduced color image to grayscale image

Calculating Stroke Width Transform:

1) First, all pixels are initialized with 255 as their stroke width in a separate SWT matrix

2) Then, we calculate the edge map of the image by using the Canny edge detector

   CANNY EDGE DETECTOR :
   
   a) Filter out any noise in the original image using Gaussian filter
   
   b) Find the intensity gradients of the image using Sobel Operator
   c) Apply non-maximum suppression to get rid of spurious response to edge detection
   d) Apply double threshold to determine potential edges
   e) Track edge by hysteresis: Finalize the detection of edges by suppressing all the other edges that are weak and not connected to strong edges

3) Using Sobel Operator, we calculate gradient direction(angle) for the edge pixels


4) We start from an edge pixel and move in its gradient direction till we reach some other edge pixel.

5) If the gradient direction at the second edge pixel is same as the first edge pixel, then we assign each pixel in the ray, the distance between the two edge pixels as the stroke width, unless it already has a lower value

6) If other pixel with same gradient direction is not found, then the whole ray is discarded.

7) After getting the gradient direction, since we could only move in one of the eight adjacent pixels, we were mapping each gradient direction to four angles i.e. 0 , 45 , 90 ,135 which gave bad results.

8) Alternate traversal method was used :
    Start from an edge pixel with ray length ‘r’ set as 1
    First move in horizontal direction as r*cos(x) and then in vertical direction as r*sin(x) and increment ray length ‘r’.
    ‘x’ is the gradient direction
9) Number of pixels in a valid stroke, is assigned as the stroke width to each pixel in that ray in the SWT matrix.
   Valid stroke width is the one in which the end edge pixel has a gradient difference between –pi/2 and +pi/2 from start edge pixel
   The output of the previous steps is another image of the same size as the input image where each element contains the width of the stroke associated with the pixel.

  ISSUES And Solutions :

  1) Detects only dark text on light background
     For light text on dark background, gradient direction is outwards, wrong stroke width is assigned
     I ran the algorithm twice, once in +ve gradient direction, then in –ve gradient direction.
   
  2) Pixels in complex locations might not hold the true stroke width value

     For that reason, we pass along each non-discarded ray, where each pixel in the ray receives the minimal value between its current value, and the median value along that ray

     ![alt](/median.png)
   
10) Morphology: Dilation:
    After calculation of SWT values and normalizing them to be between 0-255
    Gaps were found between the single letters of the text preventing them from being recognized as a single connected component
    Applied dilation to fill those gaps.
    
11) FINDING THE CONNECTED COMPONENTS:
    4 connectivity is used

    Two neighboring pixels may be grouped together if  they have similar stroke width.

    The criteria for similarity is that the SWT ratio lies between 0.33 and 3(ref research paper or training results on ICDAR 2003)
    
    A queue was used to implement the above steps

    We used Breadth First Search algorithm  to access the pixels.
    
12) FINDING POSSIBLE LETTER CANDIDATES:
    Identify components that may contain text. For this we employ a small set of flexible rules like:
    The variance(σ2) of the stroke-width within a component must not be too big. Helps rejecting foliage.
    The threshold value for stroke width is obtained as:
                        Stroke width< 0.5*mean stroke width
    The aspect ratio of a component must be within a small range of values, in order to reject long and narrow components:
                        0.1< Aspect Ratio < 10
    Often times, text is surrounded by other components like frames of a sign board, number plate etc. 
    To eliminate these frames, we restrict that the bounding box of obtained components should not contain more than two other components.
    We restrict the size of our text within a range of values and hence reject very large or very small components.

    All the remaining connected components were considered as potential letter candidates.
    
13) GROUPING LETTER CANDIDATES INTO TEXT REGIONS:

    Since single letters are not expected to appear in images, group closely positioned letter candidates into regions of text
    A set of rules to group letters together into regions of text:
    Two letter candidates should have similar stroke width with ratio of median stroke width being less than 2
    Height and width of the neighboring characters should be in ratios 2 and 3 respectively.
    The candidate pairs determined above are clustered together into chains
    
14) The final step is to cluster together, all the candidate pairs determined above into chains. Multiple chains can be grouped together if they share a common end. Each hence produced chain is taken to be a text line.

PROBLEMS FACED

1) In canny edge, we had to change min max threshold according to image and not some arbitrary value
    Canny edge works on the principle of intensity gradient 
    a difference in intensity is always assumed from the implementation side
    In Otsu's method we exhaustively search for the threshold that minimizes the intra-class variance (the variance within the class)
    we were able to dynamically adjust edges by using his algorithm

2) Often times, two texts of different colors are detected in the same line and given the same bounding box if they follow the width and height ratios.
    However, these two boxes may not form a continuous line of text and can just be two separate sentences . Eg: in a newspaper color page
    To remove this problem, we compare mean component color of the two detected regions and if they differ by a ratio greater than 2, they are put in two different bounding boxes and are not merged

3) Detected region still include components which have unusual orientation because algorithm detects stroke independent of language
    To remove these components, the diameter to median stroke width ratio is changed to have a lower bound of 1.414
   
4) Morphological dilation:
  After performing dilation on the image with a normal rectangular kernel[2x2] with all values as 1, many times two letter candidates are grouped into one component
  This poses a problem for the aspect ratio conditions in many images
  The proposed solution is to use a 3x3 kernel of the form:
                                                            1   0   1
                                                            0   0   1
                                                            0   1   1
                                                            
5) Size of font:
    When the bounding box of each letter is less than 0.0025% of the area of the image, such letters maybe recognized in SWT but will not be grouped into letter candidates because of the limiting condition
  

 
