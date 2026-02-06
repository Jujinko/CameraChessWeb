# Chess Analyzer Python App - Detailed Design Document

## 1. Executive Summary

This document details the design and implementation plan for a Python-based web application that analyzes chess positions from images or videos. The application will leverage the existing computer vision logic from the TypeScript/React frontend, ported to a Python backend using Flask, OpenCV, and ONNX Runtime. The goal is to provide a robust tool for digitizing physical chess games, enabling analysis, and fostering a social chess environment for hybrid office settings.

## 2. System Architecture

### 2.1 Technology Stack

*   **Backend Framework:** Flask (Python)
*   **Computer Vision:**
    *   `opencv-python` (Image processing, perspective transforms)
    *   `onnxruntime` (Inference for Piece and Corner detection models)
    *   `scipy` (Delaunay triangulation for grid detection)
    *   `numpy` (Matrix operations)
*   **Chess Logic:** `python-chess` (FEN handling, move validation, game state)
*   **Frontend:** HTML5, CSS3, JavaScript (Minimal, utilizing Jinja2 templates for rendering)
*   **Storage:** Temporary filesystem storage for uploads (can be upgraded to database/cloud storage).

### 2.2 High-Level Data Flow

1.  **User Action:** User uploads an image or video via the Web UI.
2.  **Preprocessing:** Backend receives the file, resizes/pads it to match model input requirements (480x288).
3.  **Inference:**
    *   **Piece Detection:** The image is passed to the Piece Detection Model (ONNX) to find bounding boxes and classes of chess pieces.
    *   **Corner Detection:** The image (and potentially piece locations) is passed to the Corner Detection Model (ONNX).
4.  **Geometry Reconstruction:**
    *   **Grid Finding:** Delaunay triangulation is used on detected corners to find the best-fitting 8x8 grid structure.
    *   **Perspective Transform:** A homography matrix is calculated to map the image board to a flat 8x8 coordinate system.
5.  **State Estimation:** Detected pieces are mapped to the logical board squares using the perspective transform.
6.  **FEN Generation:** A standard FEN string is generated from the board state.
7.  **Response:** The server returns the FEN, an annotated image, and the current board state to the frontend.
8.  **Interaction:** User validates the board, makes a move (physically and uploads new image, or virtually on screen).

## 3. Detailed Implementation Strategy

This section describes how to port the core logic from `src/utils/` (TypeScript) to Python.

### 3.1 Dependencies

Create a `requirements.txt`:
```text
flask
opencv-python-headless
numpy
scipy
onnxruntime
python-chess
```

### 3.2 Vision Module (`app/vision.py`)

This module is the heart of the application.

#### 3.2.1 Constants (Ported from `src/utils/constants.tsx`)
```python
MODEL_WIDTH = 480
MODEL_HEIGHT = 288
SQUARE_SIZE = 128
BOARD_SIZE = 8 * SQUARE_SIZE
# ... Piece symbols, etc.
```

#### 3.2.2 Helper Functions (Ported from `detect.tsx`)

*   **`get_input(image)`**:
    *   Read image using `cv2.imread`.
    *   Resize while maintaining aspect ratio to fit within `MODEL_WIDTH` x `MODEL_HEIGHT`.
    *   Pad with neutral color (114, 114, 114) to fill the dimensions.
    *   Normalize (divide by 255.0) and transpose to `(1, 3, H, W)` for ONNX.

*   **`process_boxes(outputs, ...)`**:
    *   Decode YOLO/LeYOLO outputs. The output shape is typically `(Batch, N_Anchors, 4+Classes)`.
    *   Convert `[x_center, y_center, width, height]` to `[x1, y1, x2, y2]`.
    *   Scale coordinates back to original image size.
    *   Apply Non-Max Suppression (NMS) using `cv2.dnn.NMSBoxes`.

#### 3.2.3 Core Logic Classes

**Class `ChessVision`**:

1.  **Initialization**:
    ```python
    def __init__(self, piece_model_path, corner_model_path):
        self.piece_session = onnxruntime.InferenceSession(piece_model_path)
        self.corner_session = onnxruntime.InferenceSession(corner_model_path)
    ```

2.  **Piece Detection** (Porting `findPieces.tsx`):
    ```python
    def detect_pieces(self, image):
        input_tensor = self.preprocess(image)
        outputs = self.piece_session.run(None, {self.piece_input_name: input_tensor})
        boxes, scores, class_ids = self.postprocess(outputs)
        return boxes, scores, class_ids
    ```

3.  **Corner Detection** (Porting `findCorners.tsx`):
    *   Similar to pieces, but runs the corner model.
    *   **Crucial Step:** Unlike the TS version which might use piece locations to crop/focus (ROI), the initial Python version can run on the full image or implement the same ROI logic if `keypoints` (board corners) are estimated from piece density.

4.  **Geometry & Warping** (Porting `warp.tsx` and `findCorners.tsx`):
    *   **Triangulation:** Use `scipy.spatial.Delaunay` on the detected corner points.
    *   **Quad Finding:** Iterate through triangles to form quadrilaterals (pairs of triangles sharing an edge).
    *   **Scoring:** Score each quad by how well it expands to form a regular 8x8 grid that aligns with other detected corners.
        *   Use `cv2.findHomography` or `cv2.getPerspectiveTransform`.
        *   Transform all corners using the matrix and check alignment errors.
    *   **Orientation:** Use piece colors (White vs Black clusters) to determine board orientation (A1 vs H8).
        *   Compute centroid of white pieces vs black pieces.
        *   Orient the grid so white is at ranks 1-2 and black at 7-8.

5.  **Square Mapping** (Porting `findPieces.tsx`):
    *   Generate "Ideal Centers" for 64 squares in the unwarped coordinate system.
    *   Use the inverse homography to map these centers onto the original image.
    *   For each square center on the image, find the closest detected piece bounding box (using Euclidean distance to box center).
    *   Assign piece if distance < threshold.

6.  **FEN Construction** (Porting `findFen.tsx`):
    *   Create a `chess.Board`.
    *   Populate it based on the mapped pieces.
    *   Handle conflicts (e.g., multiple pieces mapping to one square) by taking the highest confidence score.
    *   Return `board.fen()`.

### 3.3 Application Logic (`app/routes.py`)

*   **`GET /`**: Render the home page (Upload form).
*   **`POST /analyze`**:
    *   Accepts image file.
    *   Saves temp file.
    *   Calls `ChessVision.analyze(image_path)`.
    *   Returns JSON: `{ "fen": "...", "image_url": "...", "board_state": [...] }` or renders a result template.
*   **`POST /move`**:
    *   Accepts current FEN and a move (LAN or SAN).
    *   Uses `python-chess` to validate and apply the move.
    *   Returns new FEN.

## 4. Web Interface Design

### 4.1 Layout
*   **Header:** "Social Chess Analyzer"
*   **Main Area:**
    *   **Left Panel:** Live Camera Feed or Image Upload Zone.
    *   **Center:** The Analyzed Image with overlay (SVG/Canvas) showing detected grid and pieces.
    *   **Right Panel:** The "Digital Board" (using `chessboard.js` or similar) showing the resulting FEN state.
*   **Controls:** "Analyze", "Confirm Position", "Make Move", "Reset".

### 4.2 Interaction Flow
1.  User uploads photo of physical board.
2.  App displays the photo with an overlay of what it "sees" (green boxes for pieces, red grid lines).
3.  App displays the digital board representation next to it.
4.  User can drag/drop on the digital board to correct any mistakes.
5.  User clicks "Confirm & Play".
6.  User makes a move on the physical board, takes a new photo, and uploads.
7.  App calculates the difference (Move Detection) or just re-analyzes the whole board to find the new state.
    *   *Note:* Re-analyzing from scratch is stateless and easier. Comparing Previous FEN vs New FEN helps validity checking.

## 5. Social & Comfort Features (Brainstorming)

To make this a "Social Office Experience":

1.  **The "Watercooler" Dashboard:**
    *   A TV screen in the office displaying the current state of active games.
    *   "Remote vs Office" tournament bracket.

2.  **Slack/Teams Bot:**
    *   Notifications: "White has moved! It's Black's turn."
    *   Interactive moves: `/chess move e4` (for remote players playing against the physical board).
    *   Polls: "What is the best move?" (Crowdsourced games).

3.  **QR Code Handoff:**
    *   The web interface generates a QR code encoding the current game ID/FEN.
    *   A colleague walking by can scan it, view the board on their phone, and suggest a move or "take over" the seat.

4.  **Async/Correspondence Mode:**
    *   Players don't need to be online simultaneously.
    *   Upload a move, tag the opponent. The opponent gets an email/ping to make their move whenever (next day/hour).

5.  **Game History & Replay:**
    *   Auto-generate a GIF of the game progression from the uploaded photos.
    *   Export PGN for analysis in engines (Stockfish).

6.  **"Blunder Check" Integration:**
    *   After a move is detected, run a quick Stockfish evaluation.
    *   If the evaluation drops significantly, pop up a "Are you sure?" (optional "Training Mode").

## 6. Project Structure

```
chess_analyzer/
├── app/
│   ├── __init__.py
│   ├── routes.py
│   ├── vision.py        # Core logic
│   ├── chess_logic.py   # python-chess wrappers
│   ├── static/
│   │   ├── css/
│   │   ├── js/
│   │   └── uploads/
│   └── templates/
│       ├── index.html
│       └── analyze.html
├── models/
│   ├── pieces.onnx      # Downloaded/Symlinked
│   └── corners.onnx     # Downloaded/Symlinked
├── requirements.txt
├── config.py
└── run.py
```
