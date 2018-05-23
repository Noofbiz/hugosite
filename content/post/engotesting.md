---
title: Engo Testing
date: 2018-02-19T18:02:22Z
lastmod: 2018-02-19T18:02:22Z
author: Jerry Caligiure
cover: /images/engotesting/cover.jpeg
categories: ["engo", "testing", "programming"]
tags: ["engo", "ecs", "testing", "go"]
---

Testing Systems for `engo`

<!--more-->

I've been writing test cases for my [engoBox2dSystem](https://github.com/Noofbiz/engoBox2dSystem) lately, and have been also adding functionality to `engo` to help with this. Maybe this blog post can provide some insight into writing more tests of systems.

First, testing systems will require using the testing package, as well as `engo`'s headless mode. To showcase these, this is the test for adding entities to the mouse system:

```go
func TestMouseSystemAdd(t *testing.T) {
	engo.Run(engo.RunOptions{
		Width:        100,
		Height:       100,
		NoRun:        true,
		HeadlessMode: true,
	}, &MouseTestScene{5})

	if len(sys.entities) != 5 {
		t.Errorf("Entity count does not match number added, have: %d, want: %d",
		len(sys.entities), 5)
	}
}
```

The options for `engo.Run` for HeadlessMode just initalizes engo for use without an OpenGL surface to draw to. NoRun is another useful testing option shown here. It bypasses `engo`'s main loop, allowing the test full control of when the world/system updates. The other thing of inerest is the scene passed to `engo.Run`

```go
type MouseTestScene struct {
	entityCount int
}

func (*MouseTestScene) Preload() {}

func (s *MouseTestScene) Setup(w *ecs.World) {
	// Add systems to the world
	sys = &MouseSystem{}
	w.AddSystem(&common.CameraSystem{})
	w.AddSystem(sys)

	//Add some entities
	basics = make([]ecs.BasicEntity, 0)
	for i := 0; i < s.entityCount; i++ {
		basic := ecs.NewBasic()
		basics = append(basics, basic)
		entity := mouseEntity{&basic, &MouseComponent{}, &common.SpaceComponent{}, nil,
								&Box2dComponent{}}
		entity.SpaceComponent = &common.SpaceComponent{
			Position: engo.Point{X: float32(i * 20), Y: 0},
			Width:    10,
			Height:   10,
		}
		entityBodyDef := box2d.NewB2BodyDef()
		entityBodyDef.Type = box2d.B2BodyType.B2_dynamicBody
		entityBodyDef.Position = Conv.ToBox2d2Vec(entity.SpaceComponent.Center())
		entityBodyDef.Angle = Conv.DegToRad(entity.SpaceComponent.Rotation)
		entity.Box2dComponent.Body = World.CreateBody(entityBodyDef)
		var entityShape box2d.B2PolygonShape
		entityShape.SetAsBox(Conv.PxToMeters(entity.SpaceComponent.Width/2),
			Conv.PxToMeters(entity.SpaceComponent.Height/2))
		entityFixtureDef := box2d.B2FixtureDef{
			Shape:    &entityShape,
			Density:  1,
			Friction: 1,
		}
		entity.Box2dComponent.Body.CreateFixtureFromDef(&entityFixtureDef)
		for _, system := range w.Systems() {
			switch sys := system.(type) {
			case *MouseSystem:
				sys.Add(entity.BasicEntity, entity.MouseComponent, entity.SpaceComponent,
						nil, entity.Box2dComponent)
			}
		}
	}
}

func (*MouseTestScene) Type() string { return "MouseTestScene" }
```

This is the scene used for testing. All it does is create an entity and add it to the system. The system itself has an entityCount, which is how many entities the system generates.

A few other useful testing techniques I found are to utilize changing around the camera system, I made this function:

```go
func cameraShift(origX, origY, transX, transY, transZ, rotation float32) (expectedX, expectedY float32) {
	engo.Input.Mouse.X = origX
	engo.Input.Mouse.Y = origY
	expectedX = origX
	expectedY = origY

	if transZ != 0 {
		engo.Mailbox.Dispatch(common.CameraMessage{
			Axis:  common.ZAxis,
			Value: transZ,
		})
		expectedX = expectedX*sys.camera.Z()
					- (engo.GameWidth() * (sys.camera.Z() - 1) / 2)
		expectedY = expectedY*sys.camera.Z()
					- (engo.GameHeight() * (sys.camera.Z() - 1) / 2)
	}

	if transX != 0 {
		engo.Mailbox.Dispatch(common.CameraMessage{
			Axis:        common.XAxis,
			Value:       transX,
			Incremental: true,
		})
		expectedX += transX
	}

	if transY != 0 {
		engo.Mailbox.Dispatch(common.CameraMessage{
			Axis:        common.YAxis,
			Value:       transY,
			Incremental: true,
		})
		expectedY += transY
	}

	if rotation != 0 {
		engo.Mailbox.Dispatch(common.CameraMessage{
			Axis:  common.Angle,
			Value: rotation,
		})
		sin, cos := math.Sincos(rotation * math.Pi / 180)
		expectedX, expectedY = expectedX*cos+expectedY*sin, expectedY*cos-expectedX*sin
	}

	return
}
```

This function takes in the original X and Y position of the mouse, and how you translate (X, Y, and Z) and rotate the camera, then returns the expected X and Y position of the mouse.

Finally, if testing log outputs, you can change what stdout is in the log package:

```go
func TestMouseSystemNewNoCamera(t *testing.T) {
	var str bytes.Buffer
	log.SetOutput(&str)

	expected := "ERROR: CameraSystem not found - have you added the `RenderSystem` before the `MouseSystem`?\n"

	w := ecs.World{}
	w.AddSystem(&MouseSystem{})

	if !strings.HasSuffix(str.String(), expected) {
		t.Errorf("Log did not recieve correct message suffix, have: %v, wanted (suffix): %v",
		str.String(), expected)
	}
}
```

This way your log prints to the output you set it to, rather than stdout. This will help test if logs and errors are actually printing correctly ^_^

That's about all the neat testing stuff I've been finding out lately. Hope you had some fun reading it!
