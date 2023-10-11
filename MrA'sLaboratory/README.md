# *Mr A's Laboratory*

<img src="Images\MrA_Transfer.gif" width="100%"/>

[Itch.io page](https://yrgo-game-creator.itch.io/mr-a-vr)  

[Repository Link](https://github.com/Proliix/VrGrupp7)  

## *A brief game description*

**Mr A's Laboratory** is a VR game located on a spaceship where you need to create potions for Mr A. Get your hands dirty and extract unknown powers from alien rocks! crush and mix them to the best of your ability to please Mr A.

---



## *My contributions to this project*

Below is a summary of my code written to this game, keep in mind that this is a group effort and we co-developed a lot of features, but all the highlighted features below have been implemented by me. 

---

## - **Liquid Simulation**  

 <img src="Images\MrA_LiquidDemo.gif" alt="Liquid Simulation">


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

    //This can be used to allow liquid to pour through objects;
    [SerializeField] private LayerMask PourCollisionMask;

    [Header("Display Controls")]
    [SerializeField]
    [Range(10, 100)]
    public int LinePoints = 25;
    [SerializeField]
    [Range(0.01f, 0.25f)]
    public float TimeBetweenPoints = 0.05f;

    public int pointCount;

    private float pourStrengthLimiter = 1;
    private float liquidLost = 0;
    private bool isPouring;
    private IEnumerator couroutine_Flowing;

    [Header("If used as dispenser")]
    [SerializeField] private LiquidDispenser liquidDispenser;

    private void Awake()
    {
        splineTrajectory = new Vector3[LinePoints];
        currentTrajectory = new Vector3[LinePoints];

        if (transform.parent != null)
        {
            var dispenser = transform.parent.GetComponentInChildren<LiquidDispenser>();

            if (dispenser != null)
            {
                liquidDispenser = dispenser;
            }
        }
    }

    IEnumerator Couroutine_StartFlow(Color color)
    {
        //Wait for a liquid from the object pool
        while (liquid == null)
        {
            liquid = LiquidObjectPool.instance.GetLiquid();


            if (liquid == null)
                yield return new WaitForSeconds(TimeBetweenPoints);
            else
                Debug.Log(transform.name + " took " + liquid.transform.name + " from object pool");
        }

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
            if (liquid != null)
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

            bool collided = Physics.Raycast(lastPosition, point - lastPosition, out RaycastHit hit, (point - lastPosition).magnitude, PourCollisionMask); //Add collision Mask?

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

                //DrawDebugLines(currentTrajectory, Color.white, TimeBetweenPoints * 5);
                return;
            }
        }
    }

    public void Pour(Color color)
    {
        //Debug.Log("PourLiquid: Pour");
        couroutine_Flowing = Couroutine_StartFlow(color);
        StartCoroutine(couroutine_Flowing);
    }

    public void Stop()
    {
        isPouring = false;
        StopCoroutine(couroutine_Flowing);

        if (liquid == null)
            return;
        //Debug.Log("LiquidPour: Stopping " + liquid.transform.name);

        liquid.StopFlow();
    }

    public void ReturnLiquid()
    {
        LiquidObjectPool.instance.ReturnLiquid(liquid);
        liquid = null;
        if (this == null) { return; }
        CancelInvoke();

        pourStrengthLimiter = 1;
    }

    void DrawDebugLines(Vector3[] points, Color color, float duration)
    {
        for (int i = 0; i < points.Length; i++)
        {
            if (points[i + 1] == Vector3.zero)
                break;

            Debug.DrawLine(points[i], points[i + 1], color, duration);

        }
    }

    public void SetPourStrength(float tilt)
    {
        pourStrengthLimiter -= Time.deltaTime;
        //Magic numbers
        PourStrength = Mathf.Clamp((tilt - 1) * pourStrengthGain, 0.1f, maxPourStrength - Mathf.Clamp01(pourStrengthLimiter) * (maxPourStrength / 3));
    }

    public void UpdateLiquidLost(float lost)
    {
        liquidLost += lost;
    }

    void TryTransferLiquid(GameObject hitObject, float delay)
    {
        LiquidCatcher liquidCatcher = hitObject.GetComponent<LiquidCatcher>();

        if (liquidCatcher != null && liquidDispenser != null)
        {
            if (liquidCatcher.GetVolume() < 1)
            {
                StartCoroutine(liquidCatcher.Couroutine_AddFromDispenser(liquidDispenser.currentAttribute, liquidLost, delay));
                Invoke(nameof(StopParticle), delay);
                liquidLost = 0;
                return;
            }
        }
        else if (liquidCatcher != null)
        {
            if (liquidCatcher.GetVolume() < 1)
            {
                StartCoroutine(liquidCatcher.Couroutine_TransferLiquid(gameObject, liquidLost, delay));
                Invoke(nameof(StopParticle), delay);
                liquidLost = 0;
                return;
            }
        }
        else if (hitObject.TryGetComponent(out CanHaveAttributes canHaveAttributes))
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

    void PlayParticle()
    {
        liquid.ps_waterSplash.Play();
    }

    void StopParticle()
    {
        liquid.ps_waterSplash.Stop();
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
    [SerializeField] private float maxSpeed = 10f;

    private Vector3 targetScale;
    private float localAnimSpeed;
    private float speedCurveLerp;
    private float length;

    private Vector3 startScale;
    private float meshLength;
    private bool flowWater = true;

    public ParticleSystem ps_waterSplash;
    private Vector3 impactPos;
    private Vector3 impactUp;

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

        float contortLength = 0;
        speedCurveLerp = 0;

        while (contortLength < spline.Length)
        {
            //Retracts the splineMesh along the spline
            contortAlong.Contort((contortLength / spline.Length));

            contortLength += Time.deltaTime * localAnimSpeed * speedCurveLerp;
            speedCurveLerp = Mathf.Clamp(speedCurveLerp + speedDelta * Time.deltaTime, 0, maxSpeed);
            yield return null;

        }

        Debug.Log(gameObject.name + ": Flow Stopped");

        pourLiquid?.ReturnLiquid();
        gameObject.SetActive(false);
    }
    private void ConfigureSpline()
    {
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
        if (!flowWater) { return; }

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

## - **Tools**  

 <img src="Images\MrA_Tools.png" alt="Liquid Simulation">


### Hammer

<table>
  <tr>
    <td width="35%" valign="top" text-align="left">The hammer uses two scripts, Crusher.cs and Crushable.cs.<br><br>
Adding the Crusher script to a GameObject allows it to crush, and the Crushable script allows it to be crushed by the Crusher. <br><br> I had many versions of this, but ultimately had to disable collision between the hammer and rock. <br><br> The particle effect looks at the material color of crushable object to match the color.</td>
    <td><img src="Images\MrA_Hammer.gif" width="100%" /></td>
  </tr>
</table>

<details>
<summary>Crusher</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

[RequireComponent(typeof(Rigidbody))]
public class Crusher : MonoBehaviour
{
    [Header("Crusher settings")]
    [SerializeField] private Transform hammerHead;
    [SerializeField] private float sharpness = 1;
    private float damage;

    [Header("Sound Controls")]
    [SerializeField] private AudioSource audioSource;
    [SerializeField][Range(0, 1)] private float volumeModifier = 1;
    [SerializeField] [Range(0, 1)] private float maxVolume = 1;
    private float soundCooldownTime = 0.1f;
    private bool soundCooldown = false;

    private Vector3 oldPosition;
    private Rigidbody rb;

    void Start()
    {
        if (hammerHead == null)
            hammerHead = transform;

        rb = GetComponent<Rigidbody>();
        oldPosition = hammerHead.position;
    }

    void Update()
    {
        if (!rb.IsSleeping())
        {
            damage = GetForce();
            oldPosition = hammerHead.position;
        }
    }

    float GetForce()
    {
        return (hammerHead.position - oldPosition).magnitude / Time.deltaTime;
    }

    public float GetDamage()
    {
        if (rb.IsSleeping())
        {
            Debug.Log("Crusher is not moving: dealing 0 damage");
            return 0;
        }

        return damage * sharpness;
    }

    private void OnCollisionEnter(Collision other)
    {
        if (other.transform.TryGetComponent(out ParentCrushable parentCrushable))
        {
            parentCrushable.CollidedWithCrusher(other, GetDamage(), hammerHead.position);
            //Debug.Log("Crusher dealt " + GetDamage() + " Damage to " + other.transform.name);
        }
    }

    private void OnTriggerEnter(Collider other)
    {
        if(other.TryGetComponent(out Crushable crushable))
        {
            float damage = GetDamage();
            PlaySound(crushable.clip_soundWhenHit, damage);
            crushable.OnCollision(damage, other.ClosestPoint(hammerHead.position), hammerHead.position);
        }
    }

    private void PlaySound(AudioClip clip, float damage)
    {
        if(audioSource == null) { return; }

        if (soundCooldown) { return; }

        StartCoroutine(PlaySoundCooldown(soundCooldownTime));

        if (clip == null)
            clip = audioSource.clip;

        audioSource.volume = Mathf.Clamp(damage * volumeModifier, 0, maxVolume);
        audioSource.PlayOneShot(clip, damage);
    }

    private IEnumerator PlaySoundCooldown(float time)
    {
        soundCooldown = true;

        yield return new WaitForSeconds(time);

        soundCooldown = false;
    }
}

```
</details>

<details>
<summary>Crushable</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

[RequireComponent(typeof(Collider))]
public class Crushable : MonoBehaviour
{
    public AudioClip clip_soundWhenHit;
    public AudioClip clip_soundWhenCrushed;

    [SerializeField] private GameObject[] detatchOnDestroy;

    [HideInInspector] 
    public float startHealth;
    public float currentHealth;
    [SerializeField] 
    private float heatModifier = 3f;

    bool isInvincible = false;
    float invincibleAfterHitTime = 0.1f;

    public GameObject ParticleOnHit;

    [Header("Put this color to match the color of the object, remember to put alpha to 1!")]
    [SerializeField] private Color baseColor;
    //This value controls on how much the color varies from the base color, 0 means all particles have same color as base;
    public float particleColorGradient = 0.5f;


    void Start()
    {
        startHealth = currentHealth;
    }

    private void OnCollisionEnter(Collision other)
    {
        //Debug.Log("Crushable Collided with " + other.transform.name);

        if (other.transform.TryGetComponent(out Crusher crusher))
        {
            OnCollision(crusher.GetDamage() ,other.GetContact(0).point, other.collider.bounds.center);
        }
    }

    public void OnCollision(float damage, Vector3 hitLocation, Vector3 crusherLocation)
    {
        if (!isInvincible)
        {
            LoseHealth(damage);

            StartCoroutine(Invincible(invincibleAfterHitTime));

            SpawnParticleEffect(damage, hitLocation, crusherLocation);
        }
    }


    public void LoseHealth(float damage)
    {
        if(TryGetComponent(out Torchable torchable))
        {
            damage *= 1 + torchable.GetTemperature() * heatModifier;
        }

        currentHealth -= damage;

        if(currentHealth < 0)
        {
            Crush();
        }
    }

    void Crush()
    {
        foreach(GameObject obj in detatchOnDestroy)
        {
            if(obj == null) { continue; }

            obj.transform.parent = null;

            if(obj.TryGetComponent(out AddGrab addGrab))
            {
                addGrab.Add();
            }
        }

        if(clip_soundWhenCrushed != null)
        {
            AudioSource.PlayClipAtPoint(clip_soundWhenCrushed, transform.position, 0.5f);
        }

        Destroy(gameObject);
    }

    private IEnumerator Invincible(float time)
    {
        isInvincible = true;
        yield return new WaitForSeconds(time);
        isInvincible = false;
    }

    private void SpawnParticleEffect(float damage, Vector3 hitLocation, Vector3 crusherLocation)
    {
        //Spawn particle
        GameObject particle = Instantiate(ParticleOnHit, hitLocation, Quaternion.identity);

        particle.transform.localScale = Vector3.one * Mathf.Clamp(damage, 0, 1);

        //Get Color of our gameobject
        var color = GetComponent<Renderer>().material.color;

        if(baseColor != Color.clear)
        {
            color *= baseColor;
        }
        //Create a light/dark gradient from our gameobject color
        var gradient = new ParticleSystem.MinMaxGradient(color * (1 - particleColorGradient), color * (1 + particleColorGradient));
        var main = particle.GetComponent<ParticleSystem>().main;
        //Set particle color to gradient;
        main.startColor = gradient;

        //Rotate the particle effect to the impact direction
        particle.transform.up = crusherLocation - hitLocation;
        //Destroy after 1 second;
        Destroy(particle, 1);
    }
}

```
</details>

---

### Bloworch

<table>
  <tr>
    <td width="35%" valign="top" text-align="left">The blowtorch uses two scripts, Blowtorch.cs and Torchable.cs.<br><br> Adding Torchable to any GameObject allows it to be torched, changing the color of the object and enables the object to break. <br><br> I create the "glow effect" by changing the <a href="https://en.wikipedia.org/wiki/HSL_and_HSV">HSV</a>
of the material, reducing saturation and increasing the value. <br> For transparent materials I had to increase the  transparency to compensate for the brighter colors.  </td>
    <td><img src="Images\MrA_Blowtorch.gif" width="100%" /></td>
  </tr>
</table>


<details>
<summary>Blowtorch</summary>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Blowtorch : MonoBehaviour
{
    [SerializeField] ParticleSystem fire;
    private List<Torchable> insideTorch = new List<Torchable>();

    private bool isActive = false;

    [SerializeField] AudioClip start;
    [SerializeField] AudioClip loop;

    private AudioSource audioSource;

    private void Start()
    {
        audioSource = GetComponent<AudioSource>();
    }

    public void StartTorch()
    {
        fire.Play();
        isActive = true;

        audioSource.PlayOneShot(start);
        audioSource.clip = loop;
        audioSource.PlayDelayed(start.length - 0.5f);

        for (int i = 0; i < insideTorch.Count; i++)
        {
            insideTorch[i].OnTorchEnter();
        }
    }

    public void StopTorch()
    {
        fire.Stop();
        isActive = false;

        audioSource.Stop();

        for (int i = 0; i < insideTorch.Count; i++)
        {
            if (insideTorch[i] == null)
            {
                continue;
            }
            insideTorch[i].OnTorchExit();
        }
    }

    private void OnTriggerEnter(Collider other)
    {

        if (other.TryGetComponent(out Torchable torchable))
        {
            if (!insideTorch.Contains(torchable))
                insideTorch.Add(torchable);

            if (isActive)
                torchable.OnTorchEnter();
        }
    }

    private void OnTriggerExit(Collider other)
    {
        if (other.TryGetComponent(out Torchable torchable))
        {
            insideTorch.Remove(torchable);

            torchable.OnTorchExit();
        }
    }
}

```

</details>

<details>
<summary>Torchable</summary>

 ```csharp
using UnityEngine;
using UnityEngine.Events;

public class Torchable : MonoBehaviour
{
    public UnityEvent onBreak;
    private bool isBroken = false;

    [Header("Defaults to this gameobject")]
    [SerializeField] public GameObject torchedObject;

    [Header("Heat Controls")]
    [SerializeField] private float heatGainSpeed = 0.5f;
    [SerializeField] private float heatLossSpeed = 0.2f;
    [SerializeField] private float maxTemp = 1.5f;
    [SerializeField] private float breakTemp = 1.5f;

    [Header("Color Change Controls (Default color is red)")]
    [SerializeField] public Color maxTorchedColor;
    [SerializeField] private float saturationModifier = 0.2f;
    [SerializeField] private float valueModifier = 1;

    private Material material;
    private Color original;

    private float temp01 = 0;
    private bool insideFire = false;

    void Start()
    {
        if (torchedObject == null)
            torchedObject = gameObject;

        if (maxTorchedColor == Color.clear)
            maxTorchedColor = Color.red;

        material = torchedObject.GetComponent<Renderer>().material;
        original = material.color;

        //Debug.Log(gameObject.name +  " has Mat: " + material.name);
        this.enabled = false;
    }

    // Update is called once per frame
    void Update()
    {
        temp01 = insideFire ?
            temp01 + Time.deltaTime * heatGainSpeed :
            temp01 - Time.deltaTime * heatLossSpeed;

        temp01 = Mathf.Clamp(temp01, 0, maxTemp);

        if (temp01 >= breakTemp && !isBroken)
        {
            isBroken = true;
            onBreak.Invoke();
        }

        if (temp01 <= 0)
        {
            temp01 = 0;
            material.color = original;
            this.enabled = false;
        }

        Color.RGBToHSV(original, out float hue, out float saturation, out float value);
        Color.RGBToHSV(maxTorchedColor, out float maxH, out float MaxS, out float MaxV);

        hue = Mathf.Lerp(hue, maxH, temp01);
        saturation = Mathf.Lerp(saturation, MaxS, temp01);
        value = Mathf.Lerp(value, MaxV, temp01);
        float alpha = Mathf.Lerp(original.a, (original.a / 4f), temp01);

        Color newColor = Color.HSVToRGB(hue, saturation - (temp01 * saturationModifier), value * (1 + temp01) * valueModifier);
        newColor.a = alpha;

        material.color = newColor;
        //Debug.Log(material.name + " has temp: " + temp01 + " and color " + material.color + ". Target color: " + newColor);
    }

    public void OnTorchEnter()
    {
        this.enabled = true;
        insideFire = true;
    }

    public void OnTorchExit()
    {
        insideFire = false;
    }
    public float GetTemperature()
    {
        return temp01;
    }
}

```
</details>

---

### Mortar & Pestle

<table>
  <tr>
    <td width="35%" valign="top" text-align="left">The mortar and pestle had considerably more issues containing clean code due to the interactions with the XR Toolkit requiring workarounds. <br> For Example, the XR Socket locks the transform of attached item, making you unable to alter its scale.<br><br> You can grind a gem down  using the pestle, taking into account the circular motion for added efficiency.</td>
    <td><img src="Images\MrA_Mortar.gif" width="100%" /> </td>
  </tr>
</table>

<details>
<summary>Mortar</summary>

 ```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

public class Mortar : MonoBehaviour
{
    [SerializeField] private GameObject dustPrefab;
    [SerializeField] private Transform dustSpawnpoint;
    [SerializeField] private GameObject heldObject;

    private GameObject currentDust;
    private XRSocketInteractor socket;

    private Vector3 heldObjectOriginalScale;
    private Vector3 dustOriginalScale;
    private float lerpScale = 0;

    private Pestle pestle;
    private Crushable crushable;

    void Start()
    {
        dustOriginalScale = dustPrefab.transform.localScale;
        socket = GetComponentInChildren<XRSocketInteractor>();

    }

    private void OnTriggerEnter(Collider other)
    {
        if (other.TryGetComponent(out Pestle pestle))
        {
            this.pestle = pestle;
        }
    }

    private void OnTriggerStay(Collider other)
    {
        if(other.GetComponent<Pestle>() == null) { return; }
        if(heldObject == null) { return; }
        if(crushable == null && !heldObject.TryGetComponent(out crushable)) { return; }

        float damage = pestle.GetDamage(dustSpawnpoint.position);
        float percentageLost = damage / crushable.startHealth;

        foreach (BaseAttribute attribute in heldObject.GetComponents<BaseAttribute>())
        {
            BaseAttribute dustAttribute = (BaseAttribute)currentDust.GetComponent(attribute.GetType());
            if(dustAttribute == null)
            {
                dustAttribute = attribute.AddToOther(currentDust.transform);
            }

            dustAttribute.mass += attribute.mass * percentageLost;
        }

        crushable.LoseHealth(damage);

        lerpScale += percentageLost;
        IncreaseDustSize();
    }

    public void SocketCheck()
    {
        IXRSelectInteractable objName = socket.GetOldestInteractableSelected();
        heldObject = objName.transform.gameObject;
        heldObject.transform.localScale = heldObjectOriginalScale;

        SpawnDust();
    }

    public void SocketClear()
    {
        if(crushable != null && crushable.currentHealth <= 0)
        {
            ReleaseDust();
        }

        heldObject = null;
        crushable = null;
        Debug.Log("Socket Cleared");
    }

    void SpawnDust()
    {
        if(currentDust != null) { return; }

        currentDust = Instantiate(dustPrefab, dustSpawnpoint);
        currentDust.transform.localScale = Vector3.zero;

        foreach (Collider col in currentDust.GetComponents<Collider>())
            col.enabled = false;

        currentDust.GetComponent<Rigidbody>().isKinematic = true;

        Color color = heldObject.GetComponent<Renderer>().material.color;
        currentDust.GetComponentInChildren<Renderer>().material.SetColor("_Color", color);

        lerpScale = 0;

        if  (heldObject.TryGetComponent(out Torchable heldTorchable) &&
            currentDust.TryGetComponent(out Torchable dustTorchable))
        {
            dustTorchable.maxTorchedColor = heldTorchable.maxTorchedColor;
        }
    }

    void IncreaseDustSize()
    {
        if (currentDust == null) { return; }

        float xScale = Mathf.Clamp(lerpScale, 0, dustOriginalScale.x);
        float zScale = Mathf.Clamp(lerpScale, 0, dustOriginalScale.z);

        float yScale = Mathf.Lerp(0, dustOriginalScale.y, lerpScale);
        Vector3 newScale = new Vector3(xScale, yScale, zScale);

        currentDust.transform.localScale = newScale;

        if(lerpScale >= 1)
        {
            ReleaseDust();
            SpawnDust();
        }
    }

    void ReleaseDust()
    {
        currentDust.transform.parent = null;
        currentDust.transform.position += (currentDust.transform.up * 0.15f);

        foreach (Collider col in currentDust.GetComponents<Collider>())
            col.enabled = true;

        currentDust.GetComponent<Rigidbody>().isKinematic = false;
        currentDust.GetComponent<AddGrab>().Add();
        currentDust = null;
    }
}

```
</details>

<details>
<summary>Pestle</summary>

 ```cs

```

</details>

---

