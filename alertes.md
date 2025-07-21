# Alerts

The alert system allows you to anticipate an action that will be performed a few moments later by the simulator. For example, you want to be alerted half a second before each ball throw.

The system allows alerting in two different ways:

- by interval: an event is sent x seconds before and x seconds after the moment when the target action is performed.
- instantly: an event is sent when the action is performed.

To do this, you need to create an alert timeline:

```typescript
const alertesTimeline = new AlertsTimeline();
```

Add timelines to it, specifying or not a gap defining the interval around the action:

```typescript
alertesTimeline.addTimeline(ball.timeline, 0.2); //defines a 0.2 second interval around the action

alertesTimeline.addTimeline(ball.timeline); //defines an alert when the action is performed
```

Next, you can create your Alert object that will send you events. You need to pass the alert timeline and clock as parameters:

```typescript
let alertes = new Alerts(alertesTimeline, clock);
```

Once this is done, you can define event listeners. You have three possible events:

- `inf`: the lower bound of the interval
- `sup`: the upper bound of the interval
- `instant`: the moment of the action (when no interval is defined)

The callback function takes 2 elements as parameters:

- The first is the event targeted by the alert, it contains its details. It is of type `AlertEvent`.
- The second is the time when the alert was triggered.

ex:

```typescript
alertes.addEventListener("inf", (e: AlertEvent, time: number) => {
  //code
});
```

## Animation During Alert Interval

To be able to create an animation that runs during the alert interval, you need to define progress based on time. This way, if the player navigates or pauses at a moment when the animation is playing, it will be synced to the clock time.

Example of a scaling animation:

To do this, we will need to add two elements to our ball (via UserData):

- a boolean `isScaling` indicating if the animation is running
- a number `startScalingTime` keeping the animation start date

We also need the animation duration, its initial state and its final state.

Here is the function that handles the animation:

```typescript
const scale = (ball: THREE.Object3D) => {
  if (!ball.userData.isScaling) {
    return;
  }

  const scaleDuration = 0.5;
  const startScale = 1;
  const targetScale = 2;

  const scaleAvencement =
    (clock.getTime() - ball.userData.startScalingTime) / scaleDuration;

  const currentScale =
    startScale + (targetScale - startScale) * scaleAvencement;

  ball.scale.setScalar(currentScale);
};
```

To start the animation, you need to set isScaling to true and give the start date. This is done in the event listener.

```typescript
alertes.addEventListener("inf", (e: AlertEvent, time: number) => {
  const ballModel = e._ballRef.deref();
  const ball = ballsRef.current.get(ballModel.id);
  if (!ball) {
    return;
  }
  if (e.actionDescription === "tossed") {
    ball.userData.startScalingTime = time;
    ball.userData.isScaling = true;
  }
});
```
