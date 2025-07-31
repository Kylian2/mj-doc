# Movements

In the context of developing this game, we have developed scripts allowing the player to move in ways other than movement in the real world.

These movements are:

- horizontal
- vertical
- rotations
- jumping

These components are available for download here: link.

To use movements, you need to include the components within the scene.

```typescript
export default function App() {
  const xrOrigin: any = useRef(null);
  const [scene, setScene] = useState < string > "danubebleu";

  return (
    <div className="canvas-container">
      <button onClick={() => store.enterVR()}>Enter VR</button>
      <Canvas style={{ background: "skyblue" }}>
        <XR store={store}>
          <XROrigin ref={xrOrigin} />

          <MovePlayer xrOrigin={xrOrigin} />
          <FlyPlayer xrOrigin={xrOrigin} />
          <RotatePlayer />
          <JumpPlayer xrOrigin={xrOrigin} />
        </XR>
      </Canvas>
    </div>
  );
}
```

## MovePlayer

This component allows the player to move in a way similar to walking.

## FlyPlayer

This component allows horizontal movement of the player (similar to flying).

It takes the following options:
| option | description | default value |
|----------|------------------------------------------------------------------------------------------------------|-------------------|
| always | allows enabling or disabling always active mode | true |
| joystick | allows selecting the button managing movement. true = right joystick, false = A and B buttons | true |

## RotatePlayer

This component allows the player to rotate on themselves.

To reduce simulator sickness, two modes are available:

- continuous rotation with progressive acceleration and deceleration of movement.
- discrete rotation, incrementing rotation 30 degrees by 30 degrees.

It takes the following option:
| option | description | default value |
|----------|------------------------------------------------------------------------------------------------------|-------------------|
| discrete | selects rotation mode: discrete / continuous | false (continuous) |

## JumpPlayer

This component allows the player to jump.

Warning, it is controlled by the A key which can conflict with flying. You must therefore manage both together.

| option             | description                                          | default value |
| ------------------ | ---------------------------------------------------- | ------------- |
| enable             | allows enabling or disabling jumping                 | true          |
| alwaysFly (unused) | allows indicating if flight mode is always activated | false         |

## Things to Know

### Accessing Reference Space

To access the reference space, you need to use the WebGL rendering component.

```typescript
const { gl } = useThree() as { gl: WebGLRenderer & { xr: any } };

const referenceSpace = gl.xr.getReferenceSpace();
```

### Accessing Controller Buttons

To get the values of player controller buttons, you need to get the controller then its "gamepad" where inputs are exposed:

```typescript
const controller = useXRInputSourceState("controller", "left"); //get the controller

const thumbstickState = controller.gamepad?.["xr-standard-thumbstick"]; //Access joystick state on this controller
const buttonA = controller.gamepad?.["a-button"]?.button; //access A button
```

Buttons are booleans set to true if pressed and false otherwise.

### Accessing Headset Orientation

To access headset orientation, you will need the webGL renderer and its xr property.

```typescript
const frame = gl.xr.getFrame();
if (!frame) return;

const viewerPose = frame.getViewerPose(referenceSpace);
if (!viewerPose) return;

const headQuaternion = new Quaternion(
  viewerPose.transform.orientation.x,
  viewerPose.transform.orientation.y,
  viewerPose.transform.orientation.z,
  viewerPose.transform.orientation.w
);
```

The quaternion contains the headset orientation. You can then apply it to correctly orient the player's movement.

In React Three Fiber, this way to get the frame `const frame = gl.xr.getFrame();` works only in `useFrame()` hook. There is an other way, maybe more conventionnal in R3F to get it : through useFrame parameters.

```typescript
useFrame((_, __, frame) => {

  //some code

  const viewerPose = frame.getViewerPose(referenceSpace);
  if (!viewerPose) return;

  const headQuaternion = new Quaternion(
    viewerPose.transform.orientation.x,
    viewerPose.transform.orientation.y,
    viewerPose.transform.orientation.z,
    viewerPose.transform.orientation.w
  );

  //some code

}
```

# How to Make the Player Move

There are two ways to make the player move. First, using the XRReferenceSpace: we make the space move around the player who always stays at the origin. Second, a specific way of React Three Fiber: the XROrigin which represents the virtual position of the player.

One issue that appears is that moving only the XROrigin makes the player's real and virtual positions unsynchronized, which creates problems when we want to access the positions of certain elements (like hand position) which are in the "real world / XRReferenceSpace" coordinate space. You might say "then let's use only the XRReferenceSpace then", but no: when using it we have a one-frame delay between the movements of the player and the movements of the controllers.

**We must investigate to find the better way to make a movement**

## How to Move with XROrigin

Before, make sure you have access to an XROrigin reference.

```typescript
useFrame(() => {
  if (xrOrigin?.current == null || controller == null) {
    return;
  }

  // Retrieve controller thumbstick
  const thumbstickState = controller.gamepad?.["xr-standard-thumbstick"];
  if (thumbstickState == null) {
    return;
  }

  const referenceSpace = gl.xr.getReferenceSpace();
  if (!referenceSpace) {
    return;
  }

  const moveX: number = thumbstickState.xAxis ?? 0;
  const moveZ: number = thumbstickState.yAxis ?? 0;

  const speed: number = 0.03; // meter or unit per frame

  /* --- Claim headset orientation --- */
  //
  /* ----------------------------------------- */

  // Apply orientation
  const forward = new Vector3();
  forward.set(0, 0, -1).applyQuaternion(headQuaternion);
  forward.y = 0;
  forward.normalize();

  const right = new Vector3();
  right.set(1, 0, 0).applyQuaternion(headQuaternion);
  right.y = 0;
  right.normalize();

  const moveDirection = new Vector3();
  moveDirection.addScaledVector(forward, -moveZ);
  moveDirection.addScaledVector(right, moveX);

  // Update position of xrOrigin
  xrOrigin.current.position.x += moveDirection.x * speed;
  xrOrigin.current.position.z += moveDirection.z * speed;
});
```

## How to Move with XRReferenceSpace

To move this XRReferenceSpace, we need to apply a translation to it. The base code of movement is the same as with xrOrigin (get joystick value and head orientation and then apply translation). Here is how to apply translation:

```typescript
// Base code of movement

const offsetTransform = new XRRigidTransform(
  {
    x: -moveDirection.x * speed,
    y: 0,
    z: -moveDirection.z * speed,
  },
  { x: 0, y: 0, z: 0, w: 1 } // Optional (a quaternion for reference space rotation, here it's a null rotation just to show)
);

try {
  const newReferenceSpace =
    referenceSpace.getOffsetReferenceSpace(offsetTransform);
  this.context.renderer.xr.setReferenceSpace(newReferenceSpace);
} catch (e) {
  console.error("Error during referenceSpace modification", e);
}
```
