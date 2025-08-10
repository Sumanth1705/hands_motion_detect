# hands_motion_detect
This is a python project that uses to detect the motion of hand and gives result how many fingers opened or closed. 
import cv2
import mediapipe as mp
import time

# --- Setup MediaPipe Hand Module ---
# This initializes the MediaPipe Hands solution.
# static_image_mode=False: Treat the input images as a video stream.
# max_num_hands=2: Detects up to two hands.
# min_detection_confidence=0.7: Minimum confidence value for hand detection.
# min_tracking_confidence=0.7: Minimum confidence value for hand tracking.
mp_hands = mp.solutions.hands
hands = mp_hands.Hands(
    static_image_mode=False,
    max_num_hands=4,
    min_detection_confidence=0.9,
    min_tracking_confidence=0.9
)

# --- Setup MediaPipe Drawing Utilities ---
# This module provides drawing functions for landmarks and connections.
mp_drawing = mp.solutions.drawing_utils
mp_drawing_styles = mp.solutions.drawing_styles

# --- Webcam Setup ---
# 0 represents the default webcam. Change if you have multiple cameras.
cap = cv2.VideoCapture(0)

# Check if webcam opened successfully
if not cap.isOpened():
    print("Error: Could not open webcam.")
    exit()

# --- Frame Rate Calculation Variables ---
pTime = 0  # Previous time
cTime = 0  # Current time

print("Webcam started. Press 'q' to quit.")

while True:
    # Read a frame from the webcam
    success, img = cap.read()
    if not success:
        print("Ignoring empty camera frame.")
        continue

    # Flip the image horizontally for a mirror effect, which is more intuitive
    img = cv2.flip(img, 1)

    # Convert the BGR image to RGB (MediaPipe requires RGB)
    imgRGB = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)

    # Process the image to find hand landmarks
    results = hands.process(imgRGB)

    # Check if hands are detected
    if results.multi_hand_landmarks:
        # Iterate over each detected hand
        for hand_landmarks in results.multi_hand_landmarks:
            # Draw the hand landmarks and connections on the original BGR image
            mp_drawing.draw_landmarks(
                img,
                hand_landmarks,
                mp_hands.HAND_CONNECTIONS,
                mp_drawing_styles.get_default_hand_landmarks_style(),
                mp_drawing_styles.get_default_hand_connections_style()
            )

            # --- Simple Hand Motion Identification (Open vs. Closed Hand) ---
            # This is a very basic example. More complex gestures require more logic.
            # We'll use the Y-coordinates of specific finger tips and knuckles.

            # Store landmark coordinates for easier access
            # Each landmark has x, y, z coordinates (z is depth).
            # Indices for landmarks:
            # TIP OF THUMB: 4
            # TIP OF INDEX FINGER: 8
            # TIP OF MIDDLE FINGER: 12
            # TIP OF RING FINGER: 16
            # TIP OF PINKY FINGER: 20
            # BASE OF FINGERS (MCP joints): 5, 9, 13, 17
            # WRIST: 0

            # Get y-coordinates (vertical position) of relevant landmarks
            # Note: In image coordinates, y increases downwards.
            thumb_tip_y = hand_landmarks.landmark[mp_hands.HandLandmark.THUMB_TIP].y
            thumb_ip_y = hand_landmarks.landmark[mp_hands.HandLandmark.THUMB_IP].y # Inner thumb joint

            index_tip_y = hand_landmarks.landmark[mp_hands.HandLandmark.INDEX_FINGER_TIP].y
            index_mcp_y = hand_landmarks.landmark[mp_hands.HandLandmark.INDEX_FINGER_MCP].y # Base of index finger

            middle_tip_y = hand_landmarks.landmark[mp_hands.HandLandmark.MIDDLE_FINGER_TIP].y
            middle_mcp_y = hand_landmarks.landmark[mp_hands.HandLandmark.MIDDLE_FINGER_MCP].y

            ring_tip_y = hand_landmarks.landmark[mp_hands.HandLandmark.RING_FINGER_TIP].y
            ring_mcp_y = hand_landmarks.landmark[mp_hands.HandLandmark.RING_FINGER_MCP].y

            pinky_tip_y = hand_landmarks.landmark[mp_hands.HandLandmark.PINKY_TIP].y
            pinky_mcp_y = hand_landmarks.landmark[mp_hands.HandLandmark.PINKY_MCP].y

            # Determine if fingers are extended (open) or curled (closed)
            # A finger is considered extended if its tip's y-coordinate is significantly
            # higher (smaller y value) than its base's y-coordinate.
            # Thumb check is slightly different because of its orientation.

            fingers_extended = []

            # Thumb: Check if thumb tip is above (smaller y) the inner thumb joint.
            # A small threshold is used to account for natural variations.
            if thumb_tip_y < thumb_ip_y - 0.03: # Adjust threshold as needed
                fingers_extended.append(True)
            else:
                fingers_extended.append(False)

            # Other fingers: Check if tip is above MCP joint
            if index_tip_y < index_mcp_y:
                fingers_extended.append(True)
            else:
                fingers_extended.append(False)

            if middle_tip_y < middle_mcp_y:
                fingers_extended.append(True)
            else:
                fingers_extended.append(False)

            if ring_tip_y < ring_mcp_y:
                fingers_extended.append(True)
            else:
                fingers_extended.append(False)

            if pinky_tip_y < pinky_mcp_y:
                fingers_extended.append(True)
            else:
                fingers_extended.append(False)

            # Count how many fingers are extended
            num_fingers_up = sum(fingers_extended)

            gesture_text = ""
            if num_fingers_up >= 4: # If 4 or 5 fingers are up (allowing for thumb nuance)
                gesture_text = "OPEN HAND"
            elif num_fingers_up == 0:
                gesture_text = "CLOSED FIST"
            else:
                gesture_text = f"{num_fingers_up} FINGERS UP" # E.g., for peace sign, etc.

            # Display the detected gesture on the image
            cv2.putText(img, gesture_text, (50, 100), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2, cv2.LINE_AA)


    # --- Calculate and Display FPS ---
    cTime = time.time()
    fps = 1 / (cTime - pTime)
    pTime = cTime
    cv2.putText(img, f'FPS: {int(fps)}', (20, 50), cv2.FONT_HERSHEY_PLAIN, 2, (255, 0, 0), 2)

    # Display the processed frame
    cv2.imshow('Hand Motion Detection', img)

    # Break the loop if 'q' is pressed
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

# --- Release resources ---
cap.release()
cv2.destroyAllWindows()
print("Webcam released and windows closed.")
