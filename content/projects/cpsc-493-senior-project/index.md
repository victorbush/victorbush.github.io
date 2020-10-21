---
title: "CPSC 493 Senior Project"
draft: false
cover: "cpsc493-proj.jpg"
---

For my senior undergrad project, I developed a simple, 2D Android game. The game used a custom OpenGL rendering engine and custom game engine, both built from the ground up. This was my first Android application as well as my first OpenGL project. I had spent quite some time exploring the Quake 2 engine source which provided some inspiration on how to layout my game engine.

## The Game

The basic premise of the game is that you can control the direction of gravity by rotating your phone while playing. Using the combination of gravity manipulation and basic movement while on the ground, you must navigate your character through the level and find the exit.

There were a few tutorial missions to help the player get acquainted with game mechanics and two real missions the player could jump into to test out his or her skills.

{{< img src="screen_menu-1024x576.png" link="screen_menu.png" caption="Main Menu" >}}

The tutorials covered the core gameplay mechanics. The player could move the character (a secret agent Ocelot) left or right while on solid ground.

{{< img src="screen_tut_basic_move-1024x576.png" link="screen_tut_basic_move.png" caption="Basic Movement" >}}

The level was completed when the character fell through the exit.

{{< img src="screen_tut_find_exit-1024x576.png" link="screen_tut_find_exit.png" caption="Find the exit" >}}

{{< img src="screen_tut_complete-1024x576.png" link="screen_tut_complete.png" caption="Level complete screen" >}}

Key cubes were used to unlock exit doors. Key cubes were moved by simply running the character into the side of the cube.

{{< img src="screen_tut_cubes-1024x576.png" link="screen_tut_cubes.png" caption="Key cubes overview" >}}

{{< img src="screen_tut_cubes_2-1024x576.png" link="screen_tut_cubes_2.png" caption="Unlocking doors with key cubes" >}}

{{< img src="screen_tut_cubes_3-1024x576.png" link="screen_tut_cubes_3.png" caption="Moving a key cube" >}}

{{< img src="screen_tut_cubes_4-1024x576.png" link="screen_tut_cubes_4.png" caption="Key cube in the key platform; door unlocked" >}}

Controlling gravity was the core mechanic. Rotating the device would control the direction of gravity.

{{< img src="screen_tut_gravity-1024x576.png" link="screen_tut_gravity.png" caption="Controlling gravity" >}}

{{< img src="screen_tut_gravity_2-1024x576.png" link="screen_tut_gravity_2.png" caption="Example of upside-down gravity" >}}

A view mode could be toggled that would pause the game and allow the player to zoom out and look at the level to plan a strategy for beating it.

{{< img src="screen_tut_view_mode-1024x576.png" link="screen_tut_view_mode.png" caption="View mode explanation" >}}

{{< img src="screen_tut_view_mode_2-1024x576.png" link="screen_tut_view_mode_2.png" caption="View mode example" >}}

There was also a water hazard. I don't know if Ocelots can swim or not, but this one can't.

{{< img src="screen_tut_water-1024x576.png" link="screen_tut_water.png" caption="Water hazard" >}}

## The Editor

I also developed a level editor to ease the process of developing missions. The level editor was developed as a Windows application using C# and OpenTK (an OpenGL library for .NET). It allows you to place all the environment pieces, level objects, key cubes, triggers, player start, etc. and control their attributes. I'm not sure what was more fun, making the game or making the editor.

{{< img src="editor_1-1024x685.png" link="editor_1.png" caption="CPSC 493 Editor" >}}

{{< img src="editor_2-1024x685.png" link="editor_2.png" caption="CPSC 493 Editor" >}}

## Other Things

Given the fact that this was my first Android application + first OpenGL application, there were bound to be issues. Performance was not great on lower-end devices. The renderer and game engine were not exactly optimized. Also, Android would dump the OpenGL texture buffers when the app lost focus and I could not figure out how to solve the issue before the project was due. The result was that the game played fine until you hit the home button or switched apps. Upon resuming the game, all textures would be black. Then you had to manually kill the game and restart it.

Other than that, the game worked as expected. The project was a great learning experience and I had a lot of fun creating it.