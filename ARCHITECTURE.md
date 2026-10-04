# System Design Specification: AI-Based University Automated Vehicle Access & Security System (AU-AVAS)

**Student Name:** Muhammad Umar  
**Roll No:** 231200  
**Class:** BSAI-A-F23  
**Course:** AI Project Design & Development Lab (Lab 02)  

---

## Task 1: Functional & Non-Functional Requirements Breakdown

### 1.1 Functional Requirements (FR)

| Req ID | Requirement Title | Description | Target Metric / Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| **FR-01** | Vehicle & Plate Detection | Detect oncoming vehicle and crop the License Plate region from RTSP stream. | Detection confidence threshold >= 0.85. |
| **FR-02** | Optical Character Recognition (OCR) | Extract alphanumeric license plate characters from the cropped plate image. | Character recognition accuracy >= 95% on clean plates. |
| **FR-03** | Database Verification | Query the local campus whitelist database for matching registration status. | Query response latency < 100 ms. |
| **FR-04** | Hardware Barrier Trigger | Signal microcontroller (ESP32/Arduino) via relay/serial to open/close barrier arm. | Hardware trigger pulse sent within 200 ms of verification. |
| **FR-05** | Audit Logging & Alerts | Record timestamped entry log with plate image; flag unauthorized entries to dashboard. | 100% of access attempts logged to SQLite/PostgreSQL. |

### 1.2 Non-Functional Requirements (NFR)

| Req ID | Quality Attribute | Technical Specification & Operational Bound |
| :--- | :--- | :--- |
| **NFR-01** | **Latency** | End-to-end processing pipeline latency (Frame Capture -> OCR -> Barrier Open) must not exceed 1.2 seconds. |
| **NFR-02** | **Throughput / FPS** | Ingestion pipeline must maintain a minimum frame rate of 20 FPS at 1080p stream resolution without frame drop. |
| **NFR-03** | **Accuracy Thresholds** | Overall vehicle recognition precision >= 98%, False Acceptance Rate (FAR) < 0.1%. |
| **NFR-04** | **Edge Resource Usage** | Edge computing unit (Jetson/Local Host) peak memory footprint <= 3.5 GB RAM, GPU utilization <= 75%. |
| **NFR-05** | **Data Privacy & Security** | Vehicle snapshot data and database access encrypted at rest (AES-256) with strict role-based access control (RBAC). |

---

## Task 2: System Boundary, User Persona, & Input/Output Mapping

### 2.1 User Personas & Actors

| Actor | Role Type | Description & Responsibilities |
| :--- | :--- | :--- |
| **Campus Security Guard** | Operational Actor | Monitors gate status via visual interface; handles fallback manual barrier override. |
| **Campus Administrator** | Administrative Actor | Enrolls student/faculty vehicle plates into database; audits entry/exit logs. |
| **Hardware Controller** | Automated Actor | ESP32/Relay module receiving serial/MQTT signals to drive barrier actuator motor. |

### 2.2 System Boundary & Input/Output Specification

| Subsystem Component | Input Signature / Source | Output Signature / Destination | Operational Constraints |
| :--- | :--- | :--- | :--- |
| **Ingestion Engine** | RTSP H.264 stream (1920x1080 @ 30 FPS) from IP Camera. | Raw frame tensor (NumPy array (1080, 1920, 3)). | Max packet drop < 1%, network bandwidth <= 8 Mbps. |
| **Plate Detector (YOLO)**| Normalized Image Tensor ((640, 640, 3)). | Bounding box coordinates [x1, y1, x2, y2], confidence float. | Model parameter size <= 30 MB, inference time <= 35 ms. |
| **OCR Module** | Cropped license plate ROI ((H, W, 3)). | Alphanumeric string (e.g., "ICT-ABC-123"). | Max character string length <= 10. |
| **Decision & Actuator** | Recognized alphanumeric string. | Relay signal (HIGH/LOW) via USB-UART / GPIO. | 5V TTL pulse width 500 ms. |
| **Security Logger** | Event metadata (Timestamp, Plate ID, Image Path, Status). | Database Row entry; JSON payload for unauthorized UI alert. | Database connection pool <= 10 concurrent clients. |

---

## Task 3: Data-Flow Diagrams (DFD)

### 3.1 Level-0 Context Diagram (System Level)

```
       +---------------------------------------------+
       |             IP Surveillance Camera          |
       +---------------------------------------------+
                              |
                              | [RTSP Video Stream]
                              v
       +---------------------------------------------+
       |                                             |
       |             AU-AVAS CORE ENGINE             |<====> [Campus Whitelist DB]
       |                                             |        (Query / Verification)
       +---------------------------------------------+
              |                               |
              | [Barrier Pulse Signal]        | [Real-time Logs & Alerts]
              v                               v
+-------------------------------+   +---------------------------------+
| Barrier Controller (ESP32)    |   |     Security Guard Dashboard    |
+-------------------------------+   +---------------------------------+
```

### 3.2 Level-1 Detailed Data Flow Diagram

```
[ IP Camera Stream ]
        |
        v
+-------------------------------------------------------+
|  1.0 Data Ingestion Module                            |
|      - Frame buffer capture via RTSP                  |
+-------------------------------------------------------+
        |  (Raw Image Frame)
        v
+-------------------------------------------------------+
|  2.0 Image Preprocessing Module                       |
|      - Letterbox resize to 640x640 & normalization    |
+-------------------------------------------------------+
        |  (Normalized Tensor)
        v
+-------------------------------------------------------+
|  3.0 YOLOv8 Plate Detector Module                     |
|      - Locates and crops License Plate ROI            |
+-------------------------------------------------------+
        |  (Cropped Plate Image)
        v
+-------------------------------------------------------+
|  4.0 OCR Extraction Engine                            |
|      - Character segmentation & text parsing          |
+-------------------------------------------------------+
        |  (Alphanumeric String: e.g. "ICT-ABC-123")
        v
+-------------------------------------------------------+
|  5.0 Database Verification Engine                     |<====> [( Campus Vehicle DB )]
|      - Queries whitelist record                       |
+-------------------------------------------------------+
        |
        +-----> [ If Match Found ] ----> 6.0 Actuation Engine ----> [ Barrier Servo / Motor ]
        |
        +-----> [ Log All Events ] ----> 7.0 Logging Service  ----> [ SQLite / UI Alerts ]
```

---

## Task 4: Modular Software Architecture Blueprint

```python
"""
AU-AVAS Modular Software Architecture Blueprint
Defines module boundaries, class contracts, typing, and method signatures.
"""

from typing import Tuple, Optional, Dict, Any
import numpy as np


class DataIngestion:
    """Handles RTSP video stream capture and buffer management."""
    def __init__(self, rtsp_url: str, target_fps: int = 25):
        self.rtsp_url: str = rtsp_url
        self.target_fps: int = target_fps
        self.is_connected: bool = False

    def connect_stream(self) -> bool:
        """Establishes connection with IP camera."""
        pass

    def fetch_frame(self) -> Tuple[bool, Optional[np.ndarray]]:
        """Retrieves latest video frame from buffer."""
        pass

    def release_stream(self) -> None:
        """Gracefully disconnects video stream."""
        pass


class ImagePreprocessor:
    """Performs image normalization, color conversion, and resizing."""
    def __init__(self, target_resolution: Tuple[int, int] = (640, 640)):
        self.target_resolution: Tuple[int, int] = target_resolution

    def preprocess_for_detection(self, frame: np.ndarray) -> np.ndarray:
        """Resizes and letterboxes frame to YOLO model dimensions."""
        pass

    def crop_bounding_box(self, frame: np.ndarray, bbox: Tuple[int, int, int, int]) -> np.ndarray:
        """Crops Region of Interest (ROI) containing detected number plate."""
        pass


class ModelInferenceEngine:
    """Manages YOLO plate localization and OCR text recognition."""
    def __init__(self, yolo_weights_path: str, conf_threshold: float = 0.85):
        self.weights_path: str = yolo_weights_path
        self.conf_threshold: float = conf_threshold

    def detect_plate(self, input_tensor: np.ndarray) -> Tuple[bool, Optional[Tuple[int, int, int, int]], float]:
        """Runs object detection to locate license plate."""
        pass

    def extract_text(self, plate_crop: np.ndarray) -> Tuple[str, float]:
        """Extracts alphanumeric characters using OCR."""
        pass


class AlertLogger:
    """Handles database authorization queries, gate actuation, and audit logging."""
    def __init__(self, db_connection_str: str, controller_port: str = "/dev/ttyUSB0"):
        self.db_uri: str = db_connection_str
        self.serial_port: str = controller_port

    def verify_plate(self, plate_number: str) -> bool:
        """Queries database to verify if vehicle is registered."""
        pass

    def trigger_barrier(self, action: str) -> bool:
        """Sends command signal ('OPEN'/'CLOSE') to microcontroller."""
        pass

    def log_event(self, plate_number: str, is_authorized: bool, metadata: Dict[str, Any]) -> bool:
        """Persists access attempt with timestamp and status to audit database."""
        pass
```
    def log_event(self, plate_number: str, is_authorized: bool, metadata: Dict[str, Any]) -> bool:
        """Persists access attempt with timestamp and status to audit database."""
        pass
