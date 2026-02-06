# Core Components for Chess Position Detection

This document outlines the core components of the project that can be reused to determine a chess position from an image and output it in a usable format (FEN).

## Overview

The process of converting an image of a chess board to a FEN string involves several steps:
1.  **Model Loading**: Loading the pre-trained ONNX models for piece and corner detection.
2.  **Corner Detection**: Identifying the four corners of the chessboard to establish the grid geometry.
3.  **Perspective Transformation**: Warping the image or mapping coordinates to align with a standard 8x8 grid.
4.  **Piece Detection**: Identifying chess pieces and their locations in the image.
5.  **Square Mapping**: Mapping detected pieces to specific squares on the board.
6.  **FEN Generation**: Converting the board state into the Forsyth-Edwards Notation (FEN).

## Core Files and Functions

The following files in `src/utils/` contain the reusable logic:

### 1. `src/utils/loadModels.tsx`
*   **Purpose**: Loads the TensorFlow.js graph models.
*   **Key Function**: `LoadModels(piecesModelRef, xcornersModelRef)`
*   **Reusability**: Essential for initializing the inference engine. It loads `480M_pieces_float16/model.json` and `480L_xcorners_float16/model.json`.

### 2. `src/utils/detect.tsx`
*   **Purpose**: Handles input preprocessing and low-level detection logic.
*   **Key Functions**:
    *   `getInput(videoRef, ...)`: Prepares input tensors (resize, pad) from an image/video source.
    *   `getBoxesAndScores(...)`: Extracts bounding boxes and scores from model predictions.
*   **Reusability**: Used by both piece and corner detection to process model outputs.

### 3. `src/utils/findCorners.tsx`
*   **Purpose**: Detects the chessboard corners and determines orientation.
*   **Key Functions**:
    *   `runXcornersModel(...)`: Runs the corner detection model.
    *   `findCornersFromXcorners(...)`: Reconstructs the board grid from detected corner points using geometry and heuristics (Delaunay triangulation, quad scoring).
    *   `calculateKeypoints(...)`: Identifies specific corners (a1, h1, a8, h8) based on the positions of white and black pieces.
*   **Reusability**: Critical for establishing the board geometry from a raw image.

### 4. `src/utils/findPieces.tsx`
*   **Purpose**: Detects pieces and maps them to board squares.
*   **Key Functions**:
    *   `detect(...)`: Runs the piece detection model.
    *   `getSquares(...)`: Maps detected pieces to board squares using the perspective transformation.
    *   `getUpdate(...)`: Updates the internal state probabilities based on detection results.
*   **Reusability**: Core logic for "seeing" the pieces on the board.

### 5. `src/utils/findFen.tsx`
*   **Purpose**: Converts the internal state to a FEN string.
*   **Key Functions**:
    *   `setFenFromState(...)`: Greedily assigns pieces to squares based on accumulated probabilities to construct the board state.
    *   `getFenAndError(...)`: Validates the board state and generates the FEN string.
*   **Reusability**: The final step to get the usable data format (FEN).

### 6. `src/utils/warp.tsx`
*   **Purpose**: Handles geometric transformations.
*   **Key Functions**:
    *   `getPerspectiveTransform(...)`: Calculates the transformation matrix.
    *   `getInvTransform(...)`: Calculates the inverse transformation matrix from keypoints.
    *   `transformCenters(...)`: Maps ideal square centers to image coordinates.
*   **Reusability**: Required for mapping between 2D image coordinates and the logical 8x8 board grid.

### 7. `src/utils/constants.tsx`
*   **Purpose**: Defines constants used across the application.
*   **Key Constants**: `MODEL_WIDTH`, `MODEL_HEIGHT`, `SQUARE_NAMES`, `PIECE_SYMBOLS`.
*   **Reusability**: Provides necessary configuration for models and logic.

## Dependencies

*   **@tensorflow/tfjs-core`, `@tensorflow/tfjs-converter`, `@tensorflow/tfjs-backend-webgl`**: For running the deep learning models.
*   **chess.js**: For chess logic validation and FEN generation.
*   **delaunator**: Used in `findCorners.tsx` for triangulation of detected points.
*   **vectorious**: Linear algebra library used in `warp.tsx` for matrix operations.

## Reusability Strategy

To implement a standalone "Image to FEN" function, one would:

1.  **Initialize**: Call `LoadModels` to warm up the models.
2.  **Process Image**:
    *   Create a `videoRef` or similar object containing the input image.
    *   Call `_findCorners` (or adapt its logic) to get the `keypoints` (a1, h1, h8, a8 coordinates).
3.  **Analyze Position**:
    *   Use the `keypoints` to call `_findFen` (or adapt its logic).
    *   `_findFen` will internally call `detect` (to find pieces), `getSquares` (to map them), and `setFenFromState` (to generate FEN).
4.  **Output**: The result will be a FEN string representing the position in the image.
