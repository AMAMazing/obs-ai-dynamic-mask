# OBS Interactive AI Masker & Trainer

This project provides an interactive tool to create and refine alpha masks for OBS Studio, with an AI-assisted starting point and a built-in mechanism to collect training data for improving the AI over time.

Instead of relying on a perfectly pre-trained AI or a static mask, this tool allows you:
1.  To get an initial AI-suggested mask for your object (e.g., a DJ controller).
2.  To manually adjust this mask using simple control points.
3.  To send the finalized mask to OBS Studio.
4.  To save the manually corrected mask and the corresponding camera frame as training data, enabling you to train a progressively better AI model.

## Core Idea

[Diagram: Camera Feed -> (Optional) AI Suggests Mask -> User Adjusts Mask (4-point polygon) -> Mask sent to OBS & Saved as Training Data]

## Features

*   **Interactive Mask Adjustment:** Manually define or refine a quadrilateral (4-point polygon) mask.
*   **AI-Assisted Start (Future Goal):** A button to have an AI attempt an initial mask.
*   **Real-time OBS Update:** Send the current mask to OBS via an image file.
*   **Integrated Training Data Collection:** Save the adjusted mask and the corresponding frame with a button press.
*   **Iterative AI Improvement:** Use the collected data to train/re-train your custom AI model periodically.

## Project Phases & Plan

This project will be developed iteratively.

### Phase 1: Manual Masking & OBS Integration (The "MVP")

**Goal:** Create a tool that allows a user to draw a 4-point polygon on a live camera feed, convert this polygon into a black and white mask image, and have OBS use this image. A button will trigger the update to OBS.

**Steps:**

1.  **Setup Basic Python Environment:**
    *   Libraries: `opencv-python`, `numpy`.
2.  **Camera Feed Display:**
    *   Create a Python script that captures video from a camera and displays it in an OpenCV window.
3.  **Interactive 4-Point Polygon Drawer:**
    *   Implement mouse click handling on the OpenCV window.
    *   Allow the user to click 4 points to define a quadrilateral.
    *   Store the coordinates of these 4 points.
    *   (Optional) Allow dragging/adjusting these points after placement.
4.  **Mask Generation from Polygon:**
    *   Create a function that takes the 4 polygon points and the frame dimensions.
    *   Generates a new black image (same size as the camera frame).
    *   Draws a filled white polygon using the 4 points onto this black image. This is your mask.
5.  **"Send to OBS" Functionality:**
    *   Add a button (can be a key press initially, e.g., 's' for send).
    *   When pressed, the current generated mask image is saved to a predefined file path (e.g., `dynamic_mask.png`).
    *   **OBS Setup:**
        *   Configure an "Image Mask/Blend" filter in OBS to point to this `dynamic_mask.png` file (absolute path).
6.  **"Clear Points" Functionality:**
    *   A button/key press (e.g., 'c') to clear the current 4 points and start over.

**Outcome of Phase 1:** A user can manually define a quadrilateral mask on their live camera feed, and with a key press, this mask is applied in OBS. This already solves the "fixed position" problem when adjustments are needed.

### Phase 2: Training Data Collection

**Goal:** Add functionality to save the current camera frame and its corresponding manually defined mask for future AI training.

**Steps:**

1.  **"Save for Training" Button:**
    *   Add a new button/key press (e.g., 't' for train).
2.  **Data Saving Logic:**
    *   When pressed:
        *   Save the current camera frame as an image (e.g., `dataset/images/frame_XXXX.png`).
        *   Save the current generated mask (the black and white polygon) as a corresponding mask image (e.g., `dataset/masks/frame_XXXX_mask.png`).
        *   Ensure unique filenames (e.g., using a timestamp or incrementing counter).
        *   Organize these into `images` and `masks` subdirectories within a `dataset` folder.
3.  **User Interface:** Clearly indicate when data is saved.

**Outcome of Phase 2:** Users can easily collect paired image-mask data every time they adjust and are happy with a mask.

### Phase 3: Initial AI Integration (AI-Assisted Mask Suggestion)

**Goal:** Integrate a pre-trained generic segmentation model (like YOLOv8-seg) to provide an *initial guess* for the mask, which the user can then refine.

**Steps:**

1.  **Add AI Library:**
    *   Install `ultralytics`.
2.  **Load Pre-trained Model:**
    *   Load a model like `yolov8n-seg.pt`.
3.  **"AI Suggest Mask" Button:**
    *   Add a new button/key press (e.g., 'a' for AI).
4.  **AI Prediction Logic:**
    *   When pressed:
        *   Take the current camera frame.
        *   Feed it to the YOLOv8 model.
        *   Identify a target object (e.g., the largest detected object, or try to find a 'laptop'/'keyboard' if your DDJ resembles one to the generic model).
        *   Convert the AI's segmentation mask for that object into a 4-point bounding quadrilateral (e.g., by finding the min/max x/y of the contour, or using `cv2.minAreaRect` and getting its 4 corner points). *This is a simplification of the AI mask to fit the 4-point editing system.*
        *   Set these 4 points as the current interactive points, allowing the user to immediately adjust them.
5.  **Refine Workflow:** The user clicks 'a', sees the AI's 4-point guess, adjusts it, then can 's' (send to OBS) and 't' (save for training).

**Outcome of Phase 3:** The AI provides a starting point, potentially speeding up the manual adjustment process. Users are still collecting high-quality (manually corrected) training data.

### Phase 4: Custom AI Model Training & Integration

**Goal:** Periodically use the collected training data to train a custom YOLOv8 segmentation model specifically for the user's object (e.g., DDJ FLX4).

**Steps:**

1.  **Data Preparation:**
    *   The `dataset/images` and `dataset/masks` folders now contain your training data.
    *   You'll need to convert these bitmap masks into the format YOLOv8 expects for training segmentation models (usually polygon coordinates in text files). *This is a step that requires an external script or process initially if not building the conversion tool directly into this app.*
    *   Alternatively, if the AI in Phase 3 directly outputs segmentations, you could save *those* full segmentations (not just the 4-point approximation) alongside the original image when the user confirms a good AI suggestion.
2.  **Train Custom Model (External Process Initially):**
    *   Using the Ultralytics YOLOv8 command-line tools, train a segmentation model with your collected dataset.
    *   `yolo segment train data=your_custom_data.yaml model=yolov8n-seg.pt epochs=50 imgsz=640`
3.  **Update AI in Tool:**
    *   Modify the Python script (from Phase 3) to load your newly trained custom model (`best.pt`) instead of the generic `yolov8n-seg.pt`.
    *   The "AI Suggest Mask" button should now use your custom model, which will (hopefully) provide much more accurate initial suggestions for your specific DDJ.

**Outcome of Phase 4:** The "AI Suggest Mask" feature becomes significantly more accurate for the target object, reducing manual effort. The user continues to refine and collect more data, allowing for further re-training and improvement.

## Future Considerations / Advanced Features

*   More sophisticated mask editing tools (e.g., more points, curve adjustments).
*   Direct conversion of bitmap masks from Phase 2 to YOLO training format within the tool.
*   Automated retraining pipeline.
*   Better UI than just OpenCV windows and key presses (e.g., using PyQt or Tkinter).

## Tech Stack (Initial)

*   **Python**
*   **OpenCV-Python:** For camera access, image display, drawing, mouse events.
*   **NumPy:** For numerical operations on images/arrays.
*   **(Phase 3+): Ultralytics:** For YOLOv8 AI model.

This iterative plan allows for progressive development, delivering value at each stage and building towards a more intelligent system over time.