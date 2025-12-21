Image Processing from Buffers and Files
=======================================

@prev_tutorial{tutorial_js_canny}
@next_tutorial{tutorial_js_table_of_contents_imgproc}

Goal
----

In this tutorial you will learn how to:

-   Process images from raw buffers (Node.js/backend)
-   Achieve consistent results across different input sources
-   Apply proper preprocessing for edge detection and other gradient-based operations

Problem
-------

When processing images in OpenCV.js, you may notice inconsistent results depending on the input source:

- ✅ **HTMLImageElement**: Edge detection works well
- ❌ **Raw Buffers** (File API, Node.js): Edge detection produces poor results

This happens because browsers automatically apply contrast enhancement and gamma correction when rendering images, which raw buffer processing lacks.

Example of the Problem
----------------------

@code{.js}
// Frontend: Works well
const img = cv.imread(htmlImageElement);
const gray = new cv.Mat();
cv.cvtColor(img, gray, cv.COLOR_RGBA2GRAY);
const edges = new cv.Mat();
cv.Canny(gray, edges, 50, 150); // Clear, strong edges ✅

// Backend with raw buffer: Poor results
const { data, info } = await sharp(buffer).raw().toBuffer({ resolveWithObject: true });
const mat = new cv.Mat(info.height, info.width, cv.CV_8UC3);
mat.data.set(data);
const gray2 = new cv.Mat();
cv.cvtColor(mat, gray2, cv.COLOR_RGB2GRAY);
const edges2 = new cv.Mat();
cv.Canny(gray2, edges2, 50, 150); // Weak or no edges ❌
@endcode

Solution: Preprocessing for Consistent Results
----------------------------------------------

To achieve consistent results, apply contrast enhancement to images processed from raw buffers:

### Helper Functions

OpenCV.js provides two helper functions for this purpose:

1. **`cv.preprocessGrayscaleMat(src, useCLAHE)`** - Enhances contrast in grayscale images
2. **`cv.matFromRGBData(data, width, height, channels)`** - Creates BGR Mat from RGB(A) buffer

### Frontend Example (Browser with File API)

@code{.js}
// Load image from File input
async function processImageFile(file) {
    // Method 1: Via HTMLImageElement (recommended, automatic preprocessing)
    const img = await fileToImage(file);
    const mat = cv.imread(img);
    
    const gray = new cv.Mat();
    cv.cvtColor(mat, gray, cv.COLOR_RGBA2GRAY);
    
    const edges = new cv.Mat();
    cv.Canny(gray, edges, 50, 150);
    
    cv.imshow('canvas', edges);
    
    // Cleanup
    mat.delete();
    gray.delete();
    edges.delete();
}

function fileToImage(file) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => {
            URL.revokeObjectURL(img.src);
            resolve(img);
        };
        img.onerror = reject;
        img.src = URL.createObjectURL(file);
    });
}
@endcode

### Backend Example (Node.js with Sharp)

@code{.js}
const sharp = require('sharp');
const cv = require('@techstark/opencv-js');

async function processImageBuffer(buffer) {
    // Step 1: Decode image with Sharp
    const { data, info } = await sharp(buffer)
        .rotate()        // Handle EXIF orientation
        .removeAlpha()   // Remove alpha channel
        .raw()           // Get raw RGB pixel data
        .toBuffer({ resolveWithObject: true });
    
    // Step 2: Create BGR Mat from RGB data (handles color space conversion)
    const bgrMat = cv.matFromRGBData(data, info.width, info.height, 3);
    
    // Step 3: Convert to grayscale
    const grayMat = new cv.Mat();
    cv.cvtColor(bgrMat, grayMat, cv.COLOR_BGR2GRAY);
    
    // Step 4: Apply preprocessing (KEY STEP for consistent results!)
    const enhancedMat = cv.preprocessGrayscaleMat(grayMat, true);
    
    // Step 5: Edge detection
    const edgesMat = new cv.Mat();
    cv.Canny(enhancedMat, edgesMat, 50, 150);
    
    // Step 6: Cleanup
    bgrMat.delete();
    grayMat.delete();
    enhancedMat.delete();
    
    return edgesMat;
}
@endcode

### Unified Processing Function

For applications that need to handle both frontend and backend:

@code{.js}
async function processImageUniversal(source) {
    let bgrMat;
    
    if (source instanceof HTMLImageElement) {
        // Frontend path - browser handles preprocessing
        const rgbaMat = cv.imread(source);
        bgrMat = new cv.Mat();
        cv.cvtColor(rgbaMat, bgrMat, cv.COLOR_RGBA2BGR);
        rgbaMat.delete();
        
    } else if (Buffer.isBuffer(source)) {
        // Backend path - manual preprocessing needed
        const { data, info } = await sharp(source)
            .rotate()
            .removeAlpha()
            .raw()
            .toBuffer({ resolveWithObject: true });
        
        bgrMat = cv.matFromRGBData(data, info.width, info.height, 3);
    }
    
    // Standard processing pipeline
    const gray = new cv.Mat();
    cv.cvtColor(bgrMat, gray, cv.COLOR_BGR2GRAY);
    
    // Apply preprocessing if from buffer
    const processed = Buffer.isBuffer(source) 
        ? cv.preprocessGrayscaleMat(gray, true)
        : gray.clone();
    
    const edges = new cv.Mat();
    cv.Canny(processed, edges, 50, 150);
    
    // Cleanup
    bgrMat.delete();
    gray.delete();
    if (processed !== gray) processed.delete();
    
    return edges;
}
@endcode

Understanding the Preprocessing
-------------------------------

The `preprocessGrayscaleMat` function applies **CLAHE** (Contrast Limited Adaptive Histogram Equalization) or histogram equalization to enhance image contrast.

**Why is this needed?**

- Browsers automatically apply contrast enhancement when rendering images
- Raw buffer processing bypasses this enhancement
- Without preprocessing, edges appear weaker or disappear entirely

**When to use it:**

- ✅ Processing raw buffers from Sharp, Jimp, or similar libraries
- ✅ Node.js backend processing
- ✅ Reading images directly from File API
- ❌ Not needed for HTMLImageElement (browser already enhanced it)

Color Space Considerations
--------------------------

| Library/Source | Output Format | OpenCV Expects | Conversion Needed |
|---------------|---------------|----------------|-------------------|
| HTMLImageElement | RGBA | BGR | `COLOR_RGBA2BGR` |
| Sharp | RGB | BGR | `COLOR_RGB2BGR` |
| Canvas | RGBA | BGR | `COLOR_RGBA2BGR` |
| Jimp | RGBA | BGR | `COLOR_RGBA2BGR` |

**Key Point:** Always use `cv.matFromRGBData()` when creating Mats from RGB/RGBA buffers. It handles the color space conversion automatically.

Complete Example: ID Card Edge Detection
-----------------------------------------

@code{.js}
const express = require('express');
const multer = require('multer');
const sharp = require('sharp');
const cv = require('@techstark/opencv-js');

const app = express();
const upload = multer({ storage: multer.memoryStorage() });

app.post('/detect-edges', upload.single('image'), async (req, res) => {
    try {
        // Process uploaded image
        const { data, info } = await sharp(req.file.buffer)
            .rotate()
            .removeAlpha()
            .raw()
            .toBuffer({ resolveWithObject: true });
        
        // Create Mat with proper color space
        const bgrMat = cv.matFromRGBData(data, info.width, info.height, 3);
        
        // Convert to grayscale
        const gray = new cv.Mat();
        cv.cvtColor(bgrMat, gray, cv.COLOR_BGR2GRAY);
        
        // Enhance contrast
        const enhanced = cv.preprocessGrayscaleMat(gray);
        
        // Apply Gaussian blur
        const blurred = new cv.Mat();
        cv.GaussianBlur(enhanced, blurred, new cv.Size(5, 5), 0);
        
        // Detect edges
        const edges = new cv.Mat();
        cv.Canny(blurred, edges, 50, 150);
        
        // Convert to buffer for response
        const outputBuffer = cv.imencode('.jpg', edges);
        
        // Cleanup
        bgrMat.delete();
        gray.delete();
        enhanced.delete();
        blurred.delete();
        edges.delete();
        
        res.contentType('image/jpeg');
        res.send(Buffer.from(outputBuffer));
        
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

app.listen(3000);
@endcode

Best Practices
--------------

1. **Always clean up Mats** - Call `.delete()` on all Mat objects to prevent memory leaks
2. **Use helper functions** - `matFromRGBData()` and `preprocessGrayscaleMat()` handle common cases
3. **Handle EXIF orientation** - Use Sharp's `.rotate()` to auto-rotate based on EXIF data
4. **Apply noise reduction** - Use `GaussianBlur` before edge detection for better results
5. **Test both paths** - Ensure consistent results between frontend and backend processing

Additional Resources
--------------------

- @ref tutorial_js_canny "Canny Edge Detection"
- @ref tutorial_js_colorspaces "Color Space Conversions"
- @ref tutorial_js_filtering "Image Filtering"

Related Issue: [#27826](https://github.com/opencv/opencv/issues/27826)
```
