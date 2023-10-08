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

    public LinkedList<Vector3[]> linkedList = new LinkedList<Vector3[]>();
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
        linkedList.AddFirst(copy);

        for (int i = 0; i < linkedList.Count && i < LinePoints; i++)
        {
            Vector3[] nthTrajectory = linkedList.ElementAt(i);
            if (nthTrajectory == null)
            {
                return;
            }

            Vector3 point = nthTrajectory[i];
            splineTrajectory[i] = point;
        }

        if (linkedList.Count > LinePoints)
        {
            linkedList.RemoveLast();
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
        if(this == null) { return;}
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
<summary>Weapon</summary>

 ```csharp
 using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class WeaponController : MonoBehaviour
{
    public int itemSlots = 3;
    [Header("Equipped Weapon")]
    public NewItemScriptableObject weapon;

    [Header("Base Weapon")]
    public NewItemScriptableObject baseWeapon;

    [Header("Equipped Items")]
    public NewItemScriptableObject[] items;

    public UIItemHolder itemHolder;

    private AudioSource sound;

    public bool isDead = false;

    [SerializeField] private Laser laserScript;
    [SerializeField] private Fire fireScript;

    void Start()
    {
        items = new NewItemScriptableObject[itemSlots];
        UpdateWeaponStats();

        sound = GetComponent<AudioSource>();
        sound.volume = AudioManager.instance.audioClips.sfxVolume;
    }

    public void AddItem(NewItemScriptableObject item)
    {
        if (isDead)
        {
            Debug.Log("Can't add item: Player is dead!");
            return;
        }

        for (int i = 0; i < items.Length; i++)
        {
            if (items[i] == null)
            {
                items[i] = Instantiate(item);
                if (itemHolder != null)
                {
                    itemHolder.AddItem(item, i);
                }
                Debug.Log("Added item: " + item.name);
                //Play pickup sound
                sound.volume = AudioManager.instance.audioClips.sfxVolume;
                sound.PlayOneShot(item.onPickup);

                UpdateWeaponStats();

                return;
            }
        }
        Debug.Log("Inventory full, can't add item: " + item.name);
    }

    public (bool, int) CanAddItem()
    {
        for (int i = 0; i < items.Length; i++)
        {
            if (items[i] == null)
            {
                return (true, i);
            }
        }

        return (false, -1);
    }


    public void RemoveAllItems()
    {
        laserScript.DiscardSuperSprite();

        for (int i = 0; i < items.Length; i++)
        {
            RemoveItem(i, false);
        }
        Debug.Log("Removed all items from weapon!");
    }

    public void RemoveItem(int index, bool playSound)
    {
        if (items[index] == null)
        {
            Debug.Log("Can't remove item at position " + index + ", item not found");
            return;
        }

        //Play item removed sound
        if (playSound)
        {
            sound.volume = AudioManager.instance.audioClips.sfxVolume;
            sound.PlayOneShot(items[index].onDestroy);
        }

        //Remove item sprite from weapon
        if (itemHolder != null)
        {
            itemHolder.RemoveItem(index, items[index]);
        }
        //Remove item from weapon
        Debug.Log("Removed item: " + items[index].name);
        items[index] = null;
        UpdateWeaponStats();
    }

    void UpdateWeaponStats()
    {
        NewItemScriptableObject newWeapon = Instantiate(baseWeapon);
        newWeapon.name = "Weapon";
        bool checkIfUltimate = true;
        for (int i = 0; i < items.Length; i++)
        {
            //Check if we have item to add to gun
            if (items[i] == null)
            {
                checkIfUltimate = false;
                continue;
            }

            NewItemScriptableObject item = items[i];

            if (i != 0)
            {
                //Check if previous item we added is the same
                checkIfUltimate = checkIfUltimate && item.name == items[i - 1].name;
            }

            //Weapon Modifiers
            if (newWeapon.fireSoundPriority < item.fireSoundPriority)
            {
                newWeapon.fireSoundPriority = item.fireSoundPriority;
                newWeapon.fire = item.fire;
            }

            if (newWeapon.bulletImpactPriority < item.bulletImpactPriority)
            {
                newWeapon.bulletImpactPriority = item.bulletImpactPriority;
                newWeapon.bulletImpactSound = item.bulletImpactSound;
            }

            newWeapon.ammo += item.ammo;
            newWeapon.ammoWeight *= item.ammoWeight;
            newWeapon.weaponDamage += item.weaponDamage;
            newWeapon.baseFireRate *= (1 + (item.fireRatePercentage / 100f));
            newWeapon.recoilModifier += item.recoilModifier;
            newWeapon.accuracy *= (item.accuracy / 100f);
            newWeapon.maxMissDegAngle += item.maxMissDegAngle;
            newWeapon.isShotgun = newWeapon.isShotgun || item.isShotgun;
            newWeapon.shotgunAmount += item.shotgunAmount;

            //Bullet Modifiers
            if (newWeapon.bulletSpritePriority < item.bulletSpritePriority)
            {
                newWeapon.bulletSpritePriority = item.bulletSpritePriority;
                newWeapon.bulletSprite = item.bulletSprite;
            }

            newWeapon.bulletVelocity += item.bulletVelocity;
            newWeapon.isBouncy = newWeapon.isBouncy || item.isBouncy;
            newWeapon.numOfBounces += item.numOfBounces;
            newWeapon.numOfPenetrations += item.numOfPenetrations;
            newWeapon.isPenetrate = newWeapon.isPenetrate || item.isPenetrate;
            newWeapon.isExplosive = newWeapon.isExplosive || item.isExplosive;
            newWeapon.explosionRadius += item.explosionRadius;
            newWeapon.explosionDamage += item.explosionDamage;
            newWeapon.isKnockback = newWeapon.isKnockback || item.isKnockback;
            newWeapon.knockbackModifier += item.knockbackModifier;
            newWeapon.isHoming = newWeapon.isHoming || item.isHoming;
            newWeapon.turnSpeed += item.turnSpeed;
            newWeapon.scanBounds += item.scanBounds;
            newWeapon.isStapler = newWeapon.isStapler || item.isStapler;
            newWeapon.stunTime += item.stunTime;
            newWeapon.speedSlowdown += item.speedSlowdown;
        }

        //Add ultimate effects
        if (checkIfUltimate)
        {
            Debug.Log("Equipped ultimate: itemName" + items[0].name);
            if (items[0].name == "Microwave(Clone)")
            {
                newWeapon.isSuperMicro = true;
            }

            if (items[0].name == "Pencil Sharpener(Clone)")
            {
                fireScript.shakeDuration = 0.4f;
                fireScript.shakeMagnitude = 0.5f;
                GetComponent<Fire>().bulletSizeMultiplier = 2f;
            }

            if (items[0].name == "Rubber(Clone)")
            {
                GetComponent<Fire>().isUltimateRubber = true;
                newWeapon.numOfBounces = 50;
            }

            if (items[0].name == "Shredder(Clone)")
            {
                fireScript.shakeDuration = 0.4f;
                fireScript.shakeMagnitude = 0.2f;
                newWeapon.shotgunAmount = 30;
                newWeapon.isSuperShredder = true;
            }

            if (items[0].name == "Stapler(Clone)")
            {
                newWeapon.stunTime = 3;
                newWeapon.isSuperStapler = true;
            }

            if (items[0].ultimateFire != null)
            {
                newWeapon.fire = items[0].ultimateFire;
                newWeapon.ultimateFire = items[0].ultimateFire;
            }
        }
        else
        {
            fireScript.shakeDuration = 0f;
            fireScript.shakeMagnitude = 0f;
            GetComponent<Fire>().bulletSizeMultiplier = 1f;
            GetComponent<Fire>().isUltimateRubber = false;
        }
        weapon = newWeapon;
        UpdateFireStats();
        UpdateAimLine();
    }

    public void LoseItemAmmo(float shots)
    {
        for (int i = 0; i < items.Length; i++)
        {
            if (items[i] != null)
            {
                items[i].ammo -= shots;
                if (items[i].ammo <= 0)
                {
                    RemoveItem(i, true);
                }
            }
        }
    }

    void UpdateFireStats()
    {
        if (GetComponent<Fire>() != null)
        {
            GetComponent<Fire>().UpdateFireModifiers();
        }
    }

    void UpdateAimLine()
    {
        if (GetComponentInChildren<AimLine>() != null)
        {
            AimLine aimLine = GetComponentInChildren<AimLine>();
            if(CanAddItem().Item2 == 0)
            {
                aimLine.laserMaxLength = 0;
            }
            else
            {
                aimLine.laserMaxLength = 3 * (weapon.bulletVelocity / baseWeapon.bulletVelocity);
            }
        }
    }

    public int NumOfItems()
    {
        int count = 0;
        foreach (var item in items)
        {
            if (item != null) { count++; }
        }
        return count;
    }

}
```

</details>

---

## - **Adding Juice**

