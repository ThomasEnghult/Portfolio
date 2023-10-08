# *Mr A's Laboratory*

<img src="Images\EOTM_menu.jpg" width="100%"/>

[Itch.io page](https://yrgo-game-creator.itch.io/employee-of-the-month)  

[Repository Link](https://github.com/Proliix/VrGrupp7)  

## *A brief game description*

**Mr A's Laboratory** is a VR game located on a spaceship where you need to create potions for Mr A. Get your hands dirty and extract unknown powers from alien rocks! crush and mix them to the best of your ability to please Mr A.

---



## *My contributions to this project*

Below is a summary of my code written to this game, keep in mind that this is a group effort and we co-developed a lot of features, but all the highlighted features below have been implemented by me. 

---

## - **Liquid Simulation**  

 <img src="Images\MrA_LiquidDemo.gif" alt="Liquid Simulation" width="100%">


To emulate a liquid simulation I used [Spline Mesh](https://assetstore.unity.com/packages/tools/modeling/splinemesh-104989) from the Unity Asset store based on this [tutorial](https://www.youtube.com/watch?v=3CcWus6d_B8) that helped me get started.  
By calculating the current trajectory of the liquid using a [projectile motion](https://en.wikipedia.org/wiki/Projectile_motion) and then storing that trajectory I can then "move" a spline point along that trajectory untill it hits the ground.  
Below is a demonstration of the trajectories used to calculate the spline.



<table>
  <tr>
    <td><img src="Images\MrA_Liquid.gif"/></td>
    <td><img src="Images\MrA_LiquidWithGizmos.gif" /></td>
  </tr>
</table>


The trajectory is shown in blue and after it's been traversed shown in red.  
The white dots are the spline points and the black dots are used to calculate the direction of the previous point.

Click the dropdown arrow or click the link below to see the code

[PourLiquid.cs](https://github.com/Proliix/VrGrupp7/blob/main/VrGrupp7/Assets/Scripts/Pour/PourLiquid.cs)  
[Liquid.cs](https://github.com/Proliix/VrGrupp7/blob/main/VrGrupp7/Assets/Scripts/Pour/Liquid.cs)  

<details>
<summary>PourLiquid</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System.Linq;

public class PourLiquid : MonoBehaviour
{
    private Liquid liquid;

    public LinkedList<Vector3[]> trajectories = new LinkedList<Vector3[]>();
    public Vector3[] splineTrajectory;
    public Vector3[] currentTrajectory;

    [Range(0.5f, 2f)] public float simulationSpeed = 2f;

    [Header("Pour Controls")]
    [SerializeField]
    public Transform pourPosition;
    [SerializeField]
    [Range(1, 10)]
    private float pourStrengthGain = 3;
    [SerializeField]
    [Range(1, 5)]
    private float maxPourStrength = 2;
    private float PourStrength = 2f;

    [Header("Display Controls")]
    [SerializeField]
    [Range(10, 100)]
    public int LinePoints = 25;
    [SerializeField]
    [Range(0.01f, 0.25f)]
    public float TimeBetweenPoints = 0.05f;

    public int pointCount;

    float pourStrengthLimiter = 1;

    float liquidLost = 0;

    //This can be used to allow liquid to pour through objects;
    [SerializeField] private LayerMask PourCollisionMask;

    bool isPouring;

    private IEnumerator couroutine_Flowing;

    [SerializeField] private LiquidDispenser liquidDispenser;

    private void Awake()
    {

        splineTrajectory = new Vector3[LinePoints];
        currentTrajectory = new Vector3[LinePoints];


        if(transform.parent != null)
        {
            var dispenser = transform.parent.GetComponentInChildren<LiquidDispenser>();

            if(dispenser != null)
            {
                liquidDispenser = dispenser;
            }
        }
    }

    IEnumerator Couroutine_StartFlow(Color color)
    {
        //Wait for a liquid from the object pool
        while(liquid == null)
        {
            liquid = LiquidObjectPool.instance.GetLiquid();

             
            if (liquid == null)
                yield return new WaitForSeconds(TimeBetweenPoints);
            else
                Debug.Log(transform.name + " took " + liquid.transform.name + " from object pool");
        }


        //Debug.Log("PourLiquid: " + liquid.transform.name + "Starting Flow");

        //set the liquids pourLiquid reference to this script
        liquid.pourLiquid = this;

        isPouring = true;

        //Record a trajectory for the liquid to follow
        RecordPositions();
        //Temporarly assign the current trajectory to the splineTrajectory to let the liquid have a trajectory to follow
        splineTrajectory = currentTrajectory;
        //Start the flow
        liquid.StartFlow(color);

        while (isPouring)
        {
            yield return new WaitForSeconds(TimeBetweenPoints / simulationSpeed);

            RecordPositions();

            //Sometimes is called after we cancel the couroutine
            if(liquid != null)
                liquid.UpdateSpline();
        }


    }


    void RecordPositions()
    {
        DrawProjection();

        Vector3[] copy = (Vector3[])currentTrajectory.Clone();
        trajectories.AddFirst(copy);

        for (int i = 0; i < trajectories.Count && i < LinePoints; i++)
        {
            Vector3[] nthTrajectory = trajectories.ElementAt(i);
            if (nthTrajectory == null)
            {
                return;
            }

            Vector3 point = nthTrajectory[i];
            splineTrajectory[i] = point;
        }

        if (trajectories.Count > LinePoints)
        {
            trajectories.RemoveLast();
        }
    }

    private void DrawProjection()
    {
        float mass = 1;

        Vector3 startPosition = pourPosition.position;
        Vector3 startVelocity = PourStrength * transform.up / mass;
        int i = 0;

        currentTrajectory[i] = startPosition;

        for (float time = TimeBetweenPoints; i < LinePoints - 1; time += TimeBetweenPoints)
        {
            i++;
            Vector3 point = startPosition + time * startVelocity;
            point.y = startPosition.y + startVelocity.y * time + (Physics.gravity.y / 2f * time * time);

            currentTrajectory[i] = point;

            Vector3 lastPosition = currentTrajectory[i - 1];

            bool collided = Physics.Raycast(lastPosition, point - lastPosition, out RaycastHit hit, (point - lastPosition).magnitude, PourCollisionMask);

            if (collided && hit.collider.gameObject != gameObject)
            {

                currentTrajectory[i] = hit.point;
                i++;

                pointCount = i;

                TryTransferLiquid(hit.collider.gameObject, time);

                // Clear Unused LinePoints
                while (LinePoints > i)
                {
                    currentTrajectory[i] = Vector3.zero;
                    i++;
                }

                return;
            }
        }
    }

    public void Pour(Color color)
    {
        couroutine_Flowing = Couroutine_StartFlow(color);
        StartCoroutine(couroutine_Flowing);
    }

    public void Stop()
    {
        isPouring = false;
        StopCoroutine(couroutine_Flowing);
        

        if (liquid == null)
            return;

        liquid.StopFlow();
    }

    public void ReturnLiquid()
    {
        LiquidObjectPool.instance.ReturnLiquid(liquid);
        liquid = null;
        if(this == null) { return;}
        CancelInvoke();

        pourStrengthLimiter = 1;
    }

    public void SetPourStrength(float tilt)
    {
        pourStrengthLimiter -= Time.deltaTime;
        //Magic numbers
        PourStrength = Mathf.Clamp((tilt - 1) * pourStrengthGain, 0.1f, maxPourStrength - Mathf.Clamp01(pourStrengthLimiter)* (maxPourStrength / 3));
    }

    public void UpdateLiquidLost(float lost)
    {
        liquidLost += lost;
    }

    void TryTransferLiquid(GameObject hitObject, float delay)
    {
        LiquidCatcher liquidCatcher = hitObject.GetComponent<LiquidCatcher>();
        
        if(liquidCatcher != null && liquidDispenser != null)
        {
            if (liquidCatcher.GetVolume() < 1)
            {
                StartCoroutine(liquidCatcher.Couroutine_AddFromDispenser(liquidDispenser.currentAttribute, liquidLost, delay));
                Invoke(nameof(StopParticle), delay);
                liquidLost = 0;
                return;
            }
        }
        else if(liquidCatcher != null)
        {
            if (liquidCatcher.GetVolume() < 1)
            {
                StartCoroutine(liquidCatcher.Couroutine_TransferLiquid(gameObject, liquidLost, delay));
                Invoke(nameof(StopParticle), delay);
                liquidLost = 0;
                return;
            }
        }
        else if(hitObject.TryGetComponent(out CanHaveAttributes canHaveAttributes))
        {
            StartCoroutine(Couroutine_TransferAttributes(canHaveAttributes, liquidLost, delay));
            Invoke(nameof(PlayParticle), delay);
            liquidLost = 0;
            return;
        }

        Invoke(nameof(PlayParticle), delay);
        BaseAttribute[] attributes = GetComponents<BaseAttribute>();

        for (int i = 0; i < attributes.Length; i++)
        {
            attributes[i].LoseMass(liquidLost);
        }

        Debug.Log("Tried to tranfer to " + hitObject.name);

        liquidLost = 0;
    }

    IEnumerator Couroutine_TransferAttributes(CanHaveAttributes canHaveAttributes, float liquidLost, float delay)
    {
        yield return new WaitForSeconds(delay);

        Debug.Log("Transferred Attributes to: " + canHaveAttributes.gameObject.name);
        canHaveAttributes.AddAttributes(gameObject, liquidLost);
    }

    private void OnDisable()
    {
        if (isPouring)
        {
            Debug.Log("PourLiquid Disabled: Stopping pour on " + transform.name);
            Stop(); 
        }
    }
}
```
</details>

<details>
<summary>Liquid</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using SplineMesh;

public class Liquid : MonoBehaviour
{
    public PourLiquid pourLiquid;

    [SerializeField] Vector3 scale;

    [SerializeField] Material material;
    [SerializeField] Spline spline;
    [SerializeField] ExampleContortAlong contortAlong;

    [SerializeField] float speedDelta;
    [SerializeField] float animationSpeed;
    private float localAnimSpeed;

    private Vector3 targetScale;
    private float speedCurveLerp;
    private float length;

    private Vector3 startScale;
    private float meshLength;
    [SerializeField] private float maxSpeed = 10f;
    private bool flowWater = true;

    public ParticleSystem ps_waterSplash;
    Vector3 impactPos;
    Vector3 impactUp;

    public void StartFlow(Color color)
    {
        Debug.Log(gameObject.name + ": Starting Flow");
        StopAllCoroutines();
        flowWater = true;

        SetColor(color);
        Init();
        StartCoroutine(Coroutine_WaterFlow());
    }

    public void StopFlow()
    {
        Debug.Log(gameObject.name + ": Stopping flow");
        flowWater = false;
    }


    void Init()
    {
        //Debug.Log("Init");
        ConfigureSpline();

        contortAlong.Init();

        meshLength = contortAlong.MeshBender.Source.Length;
        meshLength = meshLength == 0 ? 1 : meshLength;

        localAnimSpeed = animationSpeed / scale.x;

        UpdateScale();

        speedCurveLerp = 0;
        length = 0;

        gameObject.SetActive(true);
    }

    public void UpdateScale()
    {
        //Scale the mesh to the length of the spline
        float scaleFactor = spline.Length * (1f / meshLength);

        Vector3 localScale = scale;
        localScale.x = scale.x * scaleFactor;

        startScale = localScale;
        startScale.x = 0;
        targetScale = localScale;
    }

    IEnumerator Coroutine_WaterFlow()
    {
        //Debug.Log(gameObject.name + ": Flow Started");
        while (flowWater)
        {
            //Moves the mesh along the spline by scaling it, targetScale updates depending on the length of the spline
            contortAlong.ScaleMesh(Vector3.Lerp(startScale, targetScale, length / meshLength));

            //Increments the lerp value if we haven't reached a lerp value of 1;
            if (length < meshLength)
            {
                length += Time.deltaTime * localAnimSpeed * speedCurveLerp;
                speedCurveLerp = Mathf.Clamp(speedCurveLerp + speedDelta * Time.deltaTime, 0, maxSpeed);
            }
            else
            {
                length = meshLength;
            }

            ps_waterSplash.transform.position = impactPos;
            ps_waterSplash.transform.up = impactUp;

            yield return null;
        }

        float count = 0;
        speedCurveLerp = 0;

        while (count < spline.Length)
        {
            //Retracts the splineMesh along the spline
            contortAlong.Contort((count / spline.Length));

            count += Time.deltaTime * localAnimSpeed * speedCurveLerp;
            speedCurveLerp = Mathf.Clamp(speedCurveLerp + speedDelta * Time.deltaTime, 0, maxSpeed);
            yield return null;

        }

        Debug.Log(gameObject.name + ": Flow Stopped");

        pourLiquid?.ReturnLiquid();
        gameObject.SetActive(false);
    }
    private void ConfigureSpline()
    {
        Vector3[] points = pourLiquid.splineTrajectory;

        Vector3 targetDirection = (pourLiquid.transform.up);
        transform.forward = new Vector3(targetDirection.x, 0, targetDirection.z).normalized;

        List<SplineNode> nodes = new List<SplineNode>(spline.nodes);
        for (int i = 2; i < nodes.Count; i++)
        {
            spline.RemoveNode(nodes[i]);
        }

        UpdateSpline();
    }

    public void UpdateSpline()
    {
        if (!flowWater)
        {
            return;
        }

        Vector3[] points = pourLiquid.splineTrajectory;

        //If the spline trajectory is the shortest possible, containing only a start point and end point, we process it manually
        if (points[2] == Vector3.zero)
        {
            Vector3 point = points[0];
            Vector3 myAngle = points[0] - points[1];

            Vector3 normal = Quaternion.Euler(myAngle) * myAngle;
            Vector3 direction = point - (normal / 3f);

            spline.nodes[0].Position = transform.InverseTransformPoint(points[0]);
            spline.nodes[0].Direction = transform.InverseTransformPoint(direction);

            spline.nodes[1].Position = transform.InverseTransformPoint(points[1]);
            spline.nodes[1].Direction = transform.InverseTransformPoint(direction);

            RemoveUnusedSplines(2);

            impactPos = spline.nodes[1].Position;
            impactUp = spline.nodes[1].Direction;

            return;
        }

        int maxPoints = pourLiquid.LinePoints;
        int nodeCount = 0;

        for (int i = 0; i < maxPoints; i += 2, nodeCount++)
        {
            Vector3 point = points[i];

            //Logic for detecting when we reached the end of the spline trajectory
            if (point == Vector3.zero)
            {
                //If we stopped on an uneven point (we have i += 2) we register the middle step as a node instead
                if (points[i - 1] != Vector3.zero)
                {
                    impactPos = points[i - 1];
                    impactUp = points[i - 2] - points[i - 1];
                    i--;
                    maxPoints = 0;
                }
                //We have stopped at correct point and just need to get the impact position
                else
                {
                    impactPos = points[i - 2];
                    impactUp = points[i - 3] - points[i - 2];
                    break;
                }

            }

            if (spline.nodes.Count <= nodeCount)
            {
                spline.AddNode(new SplineNode(Vector3.zero, Vector3.forward));
            }

            if (point == Vector3.zero && i > 2)
            {
                point = points[i - 1];
                point += point - points[i - 2];
            }

            Vector3 myAngle;

            if (points[i + 1] == Vector3.zero && i > 0)
                myAngle = -(points[i] - points[i - 1]);
            else
                myAngle = points[i] - points[i + 1];

            Vector3 normal = Quaternion.Euler(myAngle) * myAngle;
            Vector3 direction = point - (normal / 2f);


            spline.nodes[nodeCount].Position = transform.InverseTransformPoint(point);
            spline.nodes[nodeCount].Direction = transform.InverseTransformPoint(direction);
        }

        RemoveUnusedSplines(nodeCount);

        UpdateScale();
    }

    void RemoveUnusedSplines(int nodeCount)
    {
        for (int i = spline.nodes.Count - 1; i > nodeCount - 1; i--)
        {
            spline.RemoveNode(spline.nodes[i]);
            //Debug.Log("points: " + line.positionCount);
            //Debug.Log("Removing node: " + i);
        }
    }

    public void SetColor(Color color)
    {
        Material newMaterial = new Material(material);

        newMaterial.SetColor("_Color", color);

        contortAlong.material = newMaterial;

        if (ps_waterSplash != null)
        {
            float particleColorGradient = 0.2f;
            //Create a light/dark gradient from our gameobject color
            var gradient = new ParticleSystem.MinMaxGradient(color * (1 - particleColorGradient), color * (1 + particleColorGradient));
            var main = ps_waterSplash.main;
            //Set particle color to gradient;
            main.startColor = gradient;

        }
    }
}

```

</details>

---

## - **Adding Juice**

