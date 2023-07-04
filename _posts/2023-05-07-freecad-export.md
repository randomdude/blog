Hi all. Today I thought I'd share my soloution for "building" a FreeCAD object, which will consume a CAD design and output gcode for feeding to my lasercutter, along with a visual diff against the previous build. This process involves a few discrete steps - adding tabs to the object, for example, and placing the resultant 2D shapes on the Z-plane ready for cutting.

For the purposes of this post, I'll be using a simple enclosure I whipped up for my CheapBMC board. It's designed to fit in a 3.5" drive bay and have a couple of PCBs screwed to it. I'm not going to focus on the enclosure itself, but here's an image to show you what I mean:

![A little boxy thing](images/2023-05-07-source.png)

This is what I'll refer to as the 'source'. It's what I've designed in FreeCAD and what is stored in Git.

## Tabs

If you've done a lot of lasercutting, you might notice a lack of 'tabs' on the box. What I mean by this is the interlocking teeth that allow glue to get a good purchase and fasten the sides together. This is typically added using a script such as [LCInterlocking](https://github.com/randomdude/LCInterlocking), a third-party package which I have somewhat modified. This operation, however, is not parametric - meaning that if you were to 'tab' your object, and then modify the object, the changes would not be reflected in the 'tabbed' version. I felt this would cause problems later, as I would inevitably change the source object and forget to update the tabbed version, and so I decided to do this as a 'build time' step.

Just to make sure you follow, here's a version of the above, with tabs added. I've hidden the front so that you can see what's going on a little better.

![The box above, with tabs](images/2023-05-07-source-tabbed.png)

To run in an automated 'build time' fashion, it's clear that some code is going to be needed. Fortunately, FreeCAD exposes a rich Python-based scripting engine, and so it is relatively easy to load the LCInterlocking script and run it. We do, however, need to specify the faces which require tabs. It would seem intuitive to do this by listing the name of each face which requires tabs, but unfortunately face names are not stable and do change occasionaly, so I decided on describing these faces instead. I wrote some code that allowed a face to be specified by a combination of the parent object's name, and the direction which the face, uh, faces. We specify the 'narrower' side, and LCInterlocking takes care of not only adding tabs, but also adding the corresponding holes to any objects which it touches.

So, for example, the 'sides' of the object - these ones - 

![Sides](images/2023-05-07-source-sides.png)

- require two faces of tabs, both on the downward-facing faces. We specify this in code:
```
	builder.createTabsByFaceNormal("sidesWithHoles", FreeCAD.Vector( 0, 0, -1))```
Note the vector, specifying a negative Z direction. We can do the same for the other two sides, the front and back, which are modelled as two seperate objects:
```
	builder.createTabsByFaceNormal("front", FreeCAD.Vector( 0, 0, -1))
	builder.createTabsByFaceNormal("back",  FreeCAD.Vector( 0, 0, -1))```
And similarly, for these two sides, we can add tabs on the faces that point in either X-direction, so that the lock into the sides.
```
	for objName in ["front", "back"]:
		builder.createTabsByFaceNormal(objName, FreeCAD.Vector( +1,  0,  0))
		builder.createTabsByFaceNormal(objName, FreeCAD.Vector( -1,  0,  0))```
And call the `execute` method to create the actual tabs.
```
	tabbedObjects = builder.execute()
	```

This yields a nice tabbed object, ready for the laser cutter:

![Tabs again!](images/2023-05-07-source-tabbed.png)

.. or is it?

## Positioning and gcode

Well, actually, no, we can't use FreeCAD's g-code generation facilities just yet. First, we need to take each peice, and orient it so that it is flat on the Z axis. Fortunately, I've done all the hard work (very badly!), so this is a simple call:

```
	# Align our tabbed object to the z plane
	exporter = exportutils(tabbedObjects, builder.material)
	exporter.rotateAndPositionAllObjectsOnZ()

	# Now we can place our objects one after another, in a neat row.
	exporter.placeInRow(tabbedObjects, startPosX = 10, startPosY = 10)
```

This will go from our tabbed object to a flat representation, like this:

![flattened to Z=0](images/2023-05-07-flattened.png)

Neat, huh? We can then use FreeCAD's CAM code to do the hard work for us

```
	exporter.execute()
	exporter.saveGCode()
```

This produces the gcode file we can feed straight to the laser cutter. All done? Almost!

## Diffing

This is far from a perfect system, and while it may look comprehensive for a trivial example such as this, for complex projects I am left wanting. For example, I would often find that I had hidden an object in FreeCAD and committed to the build server, which would then duly remove that object from the built gcode, and I would be left with a useless cut object.
To try to cut down on the amount of problems like this, the build server will take a screenshot of the result of each build, in a 2D projection, and archive it along with the rest of the build artifacts. Then, on each subsequent build, the build server performs a visual 'diff' with the previous build, to highlight changes. This is done via imagemagick, and yields a result something like this (taken from a more complex project):

Here, I've rotated the connectors in the moddle by 180 degrees. You can see the old outline in blue, and the new in red.

![ro-ro-rotate your SCART](images/2023-05-07-diff.png)


## Building

It's nice to have a build server that co-ordinates all these things and ensures I am cutting the correct version of the design. It's too easy to make a mistake when cutting manually, a process which involves:

1) Performing tedious 'build-time' tasks such as adding tabs 
2) Export G-code, ensuring the correct settings (such as speed and laser power) are used
3) Locating the built G-code and copying to an SD card
4) Putting the SD card into the laser cutter, and selecting the correct file

Repetitive tasks are more suited to Jenkins than myself. My present workflow involves simply pushing my design to a git repository, where Jenkins builds it. Once the build is complete, I examine the diff myself, and then turn on the laser cutter. The laser cutter has a small 'orange pi' SBC which will then download the latest artifacts for all projects, and at that point I can simply select the project I want to cut on the laser cutter's UI and hit 'OK' to cut.

## Optimisation

One big feature I'm missing is an intelligent way to reduce wastage. For example, going back to our 2D output above, we can see that the front and back have been placed on end-to-end, meaning that a lot of wood is wasted. While I don't have a nice automated way of optimising this, it _can_ be done manually, so that's what I do. Unfortunately, the tabbing process has renamed all our objects, so we can no longer refer to them by name. Instead, we observe the bounding-box of each object and rotate based on that. Let's modify our code above:

```
	# Align our tabbed object to the z plane
	exporter = exportutils(tabbedObjects, builder.material)
	exporter.rotateAndPositionAllObjectsOnZ()

	# Rotate the two supports so the result causes less wastage
	for obj in tabbedObjects:
		if obj.Shape.BoundBox.YLength < 40:
			obj.Placement.Rotation = obj.Placement.Rotation.multiply(FreeCAD.Rotation(FreeCAD.Base.Vector(0,1,0),90))

	# Now we can place our objects one after another, in a neat row.
	exporter.placeInRow(tabbedObjects, startPosX = 10, startPosY = 10)
```

The result is much more effecient.

![images/2023-05-07-flattened-optimised.md]


### Conclusion

I hope this has been a useful insight into what can be achieved if one ventures into FreeCAD's - and Jenkins' - scripting languages. While it was a little difficult to concoct the scripts that power this build process, and they are brittle and break often, I'm sharing them in the hope that they can be used to inspire further designs. I'd love to hear what you create (and love to get a pull request even more!). I have a very small audience on this blog, so if there's anything you want to know more about, like the underlying scripts, just give me a shout and I'll get round to doing a post about it sometime. :)


