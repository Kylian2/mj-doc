# Navigating Between Scenes

To navigate between scenes (without going through a URL link that would cause a page reload), you simply need to define which one you want to display.

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

          {/* Reset space - See next section */}
          <XRSpaceManager scene={scene} xrOrigin={xrOrigin} />

          {/* Scene selector */}
          {scene === "home" && <HomeScene scene={[scene, setScene]} />}
          {scene === "danubebleu" && <DanubeBleu scene={[scene, setScene]} />}
        </XR>
      </Canvas>
    </div>
  );
}
```

Each scene from which you want to be able to navigate must know the `scene` state and be able to modify it.

Example of a button to change scenes:

```typescript
<Button
  onClick={() => {
    if (scene && setScene) {
      setScene(scene);
    }
  }}
  marginTop={12}
  backgroundColor={"white"}
>
  <Text color={"black"}>Play</Text>
</Button>
```

## Reset Orientation When Changing Scenes

To reset the orientation and position when changing scenes, you need to use the following function. This function saves the reference space at the beginning of the game and restores it during the transition.

```typescript
/**
 * Handle position reset when changing scene
 */
function XRSpaceManager({
  scene,
  xrOrigin,
}: {
  scene: string;
  xrOrigin: React.RefObject<any>;
}) {
  const { gl } = useThree() as { gl: WebGLRenderer & { xr: any } };
  const initialReferenceSpace = useRef<XRReferenceSpace | null>(null);
  const isInitialized = useRef(false);
  const lastScene = useRef(scene);

  // save initial reference space
  useFrame(() => {
    if (!isInitialized.current && gl.xr.isPresenting) {
      const currentReferenceSpace = gl.xr.getReferenceSpace();
      if (currentReferenceSpace) {
        initialReferenceSpace.current = currentReferenceSpace;
        isInitialized.current = true;
      }
    }
  });

  useEffect(() => {
    if (lastScene.current !== scene && isInitialized.current) {
      if (initialReferenceSpace.current && gl.xr.isPresenting) {
        try {
          gl.xr.setReferenceSpace(initialReferenceSpace.current);
        } catch (e) {
          console.warn("Error during reference space change", e);
        }
      }

      if (xrOrigin.current) {
        xrOrigin.current.position.set(0, 0, 0);
      }

      lastScene.current = scene;
    }
  }, [scene, gl, xrOrigin]);

  return null;
}
```
