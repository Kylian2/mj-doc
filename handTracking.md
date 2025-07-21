# Hand Tracking

## How Hand Tracking Works in WebXR

In web XR, the hand is represented by XRHand element. This XRHand is an ordered map where the keys are the hand joints and the values an XRJointSpace.

There is 25 entries in this ordered map, which are :

```{figure} images/hand.svg
:label: Hand joints indexes
:alt: Hand joints indexes
:align: center

 Hand joints indexes
```

| Index | Nom de l'articulation              |
| ----- | ---------------------------------- |
| 0     | wrist                              |
| 1     | thumb-metacarpal                   |
| 2     | thumb-phalanx-proximal             |
| 3     | thumb-phalanx-distal               |
| 4     | thumb-tip                          |
| 5     | index-finger-metacarpal            |
| 6     | index-finger-phalanx-proximal      |
| 7     | index-finger-phalanx-intermediate  |
| 8     | index-finger-phalanx-distal        |
| 9     | index-finger-tip                   |
| 10    | middle-finger-metacarpal           |
| 11    | middle-finger-phalanx-proximal     |
| 12    | middle-finger-phalanx-intermediate |
| 13    | middle-finger-phalanx-distal       |
| 14    | middle-finger-tip                  |
| 15    | ring-finger-metacarpal             |
| 16    | ring-finger-phalanx-proximal       |
| 17    | ring-finger-phalanx-intermediate   |
| 18    | ring-finger-phalanx-distal         |
| 19    | ring-finger-tip                    |
| 20    | pinky-finger-metacarpal            |
| 21    | pinky-finger-phalanx-proximal      |
| 22    | pinky-finger-phalanx-intermediate  |
| 23    | pinky-finger-phalanx-distal        |
| 24    | pinky-finger-tip                   |

## Get Hands

Hands are inputs sources, we can access it through several manipulations :

1. First, we need to get the `XRHandState`. To get it we use the `useXRInputSourceState` hook.

```typescript
const handSourceRight = useXRInputSourceState("hand", "right");
const handSourceLeft = useXRInputSourceState("hand", "left");
```

2. After we retrieve the hand from the `XRHandInputSource` accessible through `.inputSource` property of the `XRHandState`.

```typescript
const right = handSourceRight.inputSource.hand;
const left = handSourceLeft.inputSource.hand;
```

## Get Fingers

Now we have our hands, but we also want to access to our fingers and especially their positions.

Let’s get the thumb tip and the index finger tip. To do this we need 3 things :

- the hand (we have it)
- the frame (XRFrame), we can get it from useFrame() hook, we explain it in [Accessing Headset Orientation](movements.md#accessing-headset-orientation)
- the reference space (XRReferenceSpace) : [Accessing Reference Space](movements.md#accessing-reference-space)

On the XRFrame instance, we should have a `getJointPose` method wo return the space of our joint (XRJointSpace) which contains orientation and position of it.

_Be careful, `getJointPose` can be undefined_

```typescript
const thumbTip = hand.get("thumb-tip");
const indexTip = hand.get("index-finger-tip");

const thumbPose = frame.getJointPose(thumbTip, referenceSpace);
const indexPose = frame.getJointPose(indexTip, referenceSpace);
```

Then on our fingers pose we have the .transform property which give us the XRRigidTransform which let us access the .position. But .position does not return a Vector3 (which is specific to ThreeJS), it returns a DOMPointReadOnly which need to be converted to a Vector3.

```typescript
const thumbPos = DOMPointReadOnlyToVector3(thumbPose.transform.position);
const indexPos = DOMPointReadOnlyToVector3(indexPose.transform.position);
```

_Function to convert DOMPointReadOnly to Vector3_

```typescript
function DOMPointReadOnlyToVector3(entry: DOMPointReadOnly) {
  return new THREE.Vector3(entry.x, entry.y, entry.z);
}
```

## Detect a pinch

Now we know all those things we can make a function to detect a pinch :

It needs the XRHand we want to test for the pinch, the XRFrame and the XRReferenceSpace.

```typescript
/**
 * Detect if the hand make a pinch
 * @param hand - XRHand
 * @param frame - XRFrame
 * @param referenceSpace - ReferenceSpace
 * @param threshold under this distance (in meter) it's detected. Default = 0.025 (2.5 cm)
 * @returns boolean
 */
export function isPinching(
  hand: XRHand | undefined,
  frame: XRFrame | undefined,
  referenceSpace: XRReferenceSpace | undefined,
  threshold: number = 0.025
): boolean {
  if (!(hand && frame && frame.getJointPose && referenceSpace)) return false;

  const thumbTip = hand.get("thumb-tip");
  const indexTip = hand.get("index-finger-tip");

  if (!thumbTip || !indexTip) {
    return false;
  }

  const thumbPose = frame.getJointPose(thumbTip, referenceSpace);
  const indexPose = frame.getJointPose(indexTip, referenceSpace);

  if (!thumbPose || !indexPose) {
    return false;
  }

  const thumbPos = DOMPointReadOnlyToVector3(thumbPose.transform.position);
  const indexPos = DOMPointReadOnlyToVector3(indexPose.transform.position);

  const distance = thumbPos.distanceTo(indexPos);
  return distance < threshold;
}
```

# Hand Detection Module

HandState is the module where we developed the hand tracking, including function to detect hand actions and an event dispatcher.

## Gesture Detection Functions

**`isPinching(hand, frame, referenceSpace, threshold?): boolean`**

Detects if the hand is performing a pinch gesture between thumb and index finger.

**Parameters:**

- `hand`: XRHand object (can be undefined, a security check is made in the function)
- `frame`: Current XR frame
- `referenceSpace`: XR reference space
- `threshold`: Distance threshold in meters (default: 0.025m = 2.5cm)

**Logic:** Calculates the distance between thumb tip and index finger tip. If below threshold, considers it a pinch.

**`isPinchingMiddle(hand, frame, referenceSpace, threshold?): boolean`**

Detects pinching between thumb and middle finger.

**Parameters:** Same as `isPinching`

**Usage:** Useful for gesture differentiation or alternative commands.

**`isOpenHand(hand, frame, referenceSpace, threshold?): boolean`**

Detects if the hand is open by measuring average distance between palm and fingertips.

**Parameters:**

- `threshold`: Distance threshold in meters (default: 0.08m = 8cm)

**Logic:**

- Uses middle finger metacarpal as palm reference.
- Calculates distance to each fingertip.
- If average distance exceeds threshold, hand is considered open

**`isCloseHand(hand, frame, referenceSpace, threshold?): boolean`**

Detects if the hand is closed. Unlike `!isOpenHand()`, this function checks that the hand is defined to prevent false positives.

## Main Class HandState

### Types and Interfaces

```typescript
type HandActionEvents = "pinch" | "pinch-middle" | "opened" | "closed";

interface HandActionEvent {
  hand: XRHand;
  side: "left" | "right";
}
```

### Update Method

**Role:** Main method to call each frame to detect gestures and emit events.

**Detection Logic:**

1. For each hand (right and left) and for each gesture type (pinch, pinch-middle, opened, closed)
2. Check current gesture state
3. Compare with previous state
4. If transition from `false` to `true`, emit event
5. Update previous state

**Events Emitted:**

- `"pinch"`: When pinching begins
- `"pinch-middle"`: When middle finger pinching begins
- `"opened"`: When hand opens
- `"closed"`: When hand closes

## Usage Example

```typescript
// Initialization
const handState = new HandState({
  rightHand: xrInputSource.hand,
  leftHand: xrInputSource.hand,
  pinchThreshold: 0.03, // 3cm
});

// Event listening
handState.addEventListener("pinch", (event) => {
  console.log(`Pinch detected on ${event.side} hand`);
});

handState.addEventListener("opened", (event) => {
  console.log(`${event.side} hand opened`);
});

// In render loop
useFrame((_, __, frame) => {
  handState?.update(frame, referenceSpace);
});
```

All the source code is available [here](ressources/handState.ts)
