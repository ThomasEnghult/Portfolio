# *Dusty Deliveries*

<img src="Images\DustyDeliveries_01.png" width="100%"/>

[Itch.io page](https://yrgo-game-creator.itch.io/dusty-deliveries)  

[Repository Link](https://github.com/Llrac/ProjectBoat)  

## *A brief game description*

**Dusty Deliveries** is a CO-OP adventure game where you together walk around on a ship. Set the course and engage the thrusters and explore the desert landscape!

---

## *My contributions to this project*

Below is a summary of my code written to this game, keep in mind that this is a group effort and we co-developed a lot of features, but all the highlighted features below have been implemented by me. 

---

## *Physics for a Boat in a Desert*

To start off, I wanted to try make my own implementation of a physics system, partly because of the unique challenge of having a boat travel on top of sand, and partly because I wanted to learn more about using physics, without for example the Chaos Vehicle system in Unreal Engine.

I came across this [tutorial](https://www.youtube.com/watch?v=GJ9Y9Z6Axss) about a hovercraft, containing a hover component and a lateral stabilizer that prevents sideways drift. The hover component acts like a spring and pushes the vehicle up. Attaching one hover component in each corner stabilizes the craft and creates the hover effect.

<table>
  <tr>
     <td width="30%" valign="top" text-align="left"> This shows the boat travelling forward without any stabilizing effects. <br><br> The boat is hovering above ground being pushed forward by the thrusters and sideways by gravity </td>
    <td ><img src="Images\DustyDeliveries_NoStabilizer.gif" /> <br> <i>No stabilizer</td>
  </tr>
    <tr>
     <td width="30%" valign="top" text-align="left"> Enabling the <b> Lateral Stabilizer </b> shows significant improvement, but it still has major flaws. <br><br> It keeps going off course and also tipping over due to the back hovers still having contact with the ground for a split second after jumping, causing the boat to pitch down.   </td>
    <td ><img src="Images\DustyDeliveries_LateralStabilizer.gif" /><br> <i>With Lateral stabilizer</td> 
  </tr>
      <tr>
     <td width="30%" valign="top" text-align="left"> To keep it from spinning out of control, I added a <b> Yaw Stabilizer</b>. <br><br> This keeps the boats course straight. But we still have the problem of the boat tipping over causing hard crashes into the sand. </td>
    <td ><img src="Images\DustyDeliveries_YawStabilizer.gif" /><br> <i>With Lateral stabilizer and Yaw Stabilizer</td>
  </tr>
    </tr>
      <tr>
     <td width="30%" valign="top" text-align="left"> Finally, I added an <b> Air Stabilizer </b> that tries to keep the boat upright. Also to handle the rough landings this component looks at the grounds normal and tries to align accordingly. <br> It does this by projecting each hover components distance to a plane, then feeding the error into a <a href="https://en.wikipedia.org/wiki/Proportional%E2%80%93integral%E2%80%93derivative_controller">PID Controller</a> to apply a countering force. </td>
    <td ><img src="Images\DustyDeliveries_AirStabilizer.gif" /> <br><i>With Lateral stabilizer, Yaw Stabilizer and Air Stabilizer (and more speed)</td>
  </tr>
</table>

### Links to blueprints

[Hover Component (Containing the PID Controller)](https://blueprintue.com/blueprint/a656i1au/)  
[Lateral Stabilizer](https://blueprintue.com/blueprint/4wzatnq9/)  
[Yaw Stabilizer](https://blueprintue.com/blueprint/c4n3r2i1/)  
[Air Stabilizer (Generating Error margins for Hover Component)](https://blueprintue.com/blueprint/kojjz_3c/)  

### Major flaws with my design...

Physics Simulations in video games are really hard, I'd imagine you've seen vehicles even in triple A titles spinning out of control and flying away, mine is no exception.  
The most fatal flaw in my design is doing physics calculations on a per frame basis. In testing this worked flawlessly due to the framerate never dropping below 60 fps, but when testing in a more performance intense environment the cracks started to show.  
If the framerate is running low and your physics calulations are using delta time this means that it will apply a major force the next frame causing the system to run out of control, creating a negative feedback loop.  
I never got around to fixing this issue due to time constraints and the next time I'll tackle vehicle physics I'll probably use the Chaos Vehicles instead!

---

## Creating the level

<img src="Images\DustyDeliveries_Level.gif" width=100%/>

To build a level of this scale, I needed tools to speed up the process, by using [landscape splines](https://docs.unrealengine.com/5.1/en-US/landscape-splines-in-unreal-engine/) to deform the terrain it allowed me to deform large amounts of terrain, but I still needed a way to place a boat-load of mountains. 



### Placing meshes along a Spline

<table>
  <tr>
    <td width="50%" valign="top" text-align="left"><img src="Images\DustyDeliveries_Level.png"/></td>
    <td><img src="Images\DustyDeliveries_Spline.gif"/> <i>The meshes spawn when the spline is extended </td>
  </tr>
</table>


To place all the mountains, I used a spline to place meshes, based off this [tutorial](https://www.youtube.com/watch?v=OdjvlvGRYRE) from Unreal Engine.  
I added some custom features to the mesh generator to improve it:

- Modified it to take multiple meshes as input.
- Added Custom spacing to more accurately merge the meshes into each other.
- Preventing the same mesh from being used twice in a row.
- Randomly flip the mesh by 180°, creating a more diverse look.
- Allowing you to change the scale.
- Creating a random offset in scale to create a height difference.
- Allowing you to set the seed, enabling the possibility to randomize and choose your favourite seed.


[Link To Spline Blueprint](https://blueprintue.com/blueprint/5xjso_mm/)


<table>
  <tr>
    <td width="65%" valign="top" text-align="left"><img src="Images\DustyDeliveries_SplineSettings.gif"/> <i> Modifying the random scale offset and seed</td>
    <td><img src="Images\DustyDeliveries_SplineSettings.png"/></td>
  </tr>
</table>

---

## Main Menu

<img src="Images\DustyDeliveries_MainMenu.gif" width="100%"/>

I love 3D Menu's and this [tutorial](https://www.youtube.com/watch?v=4sxxe9_w9Zs) gave me a great starting point, unfortunately it lacked controller support and I failed to find a built in function for navigating the menu so I added that together with our character running around when you navigate to a different menu.  


[Link to button navigation Blueprint](https://blueprintue.com/blueprint/2mcqe4ay/)  
*If the button is focused and you navigate up/down/left/right it will change focus if there's a button assigned to that direction*