# *Employee of the Month*

<img src="Images\EOTM_menu.jpg" width="100%"/>

[Itch.io page](https://yrgo-game-creator.itch.io/employee-of-the-month)  

[Repository Link](https://github.com/MikaelahJ/EmployeeOfTheMonth/tree/main)  

## *A brief game description*

Employee of the month is a 2D, 4 player arena shooter where you pick up office supplies and modify your improvised handmade weapon to shoot at your co-workers.  
Different office supplies gives unique effects and alter your weapon accordingly. Each round you have only one life and the last man standing wins the round.

---



## *My contributions to this project*

Below is a summary of my code written to this game, keep in mind that this is a group effort and we co-developed a lot of features, but all the highlighted features below have been implemented by me. 

---

## - **Creating the weapon**

<table>
  <tr>
    <td><img src="Images\EOTM_weapon-pickup.gif"/>
  <br> <i>Shredder adds more bullets, Pencil Sharperner  pierce glass</i></td>
    <td><img src="Images\EOTM_scriptable_objects.png" /></td>
  </tr>
</table>


The weapon and its modifiers are created using Scriptable Objects. Each office supply that you can pick up is a scriptable object, this enables fast iteration and addition of items.  
Each time you pick up or lose an item the weapon recalibrates the combined stats and then updates the other scripts.

Click the dropdown arrow or click the link below to see the code

[NewItemScriptableObject.cs](https://github.com/MikaelahJ/EmployeeOfTheMonth/blob/main/Employee%20of%20the%20month/Assets/Scripts/Weapon%20Scripts/WeaponController.cs)  
[WeaponController.cs](https://github.com/MikaelahJ/EmployeeOfTheMonth/blob/main/Employee%20of%20the%20month/Assets/Scripts/Weapon%20Scripts/WeaponController.cs)  

<details>
<summary>Item ScriptableObject</summary>

 ```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "NewItem", menuName = "ScriptableObjects/NewItemScriptableObject", order = 1)]

//Add variables in the list below, please set the value to 0 or equivalent
//After you added a variable, update the UpdateWeaponStats Function in WeaponController
public class NewItemScriptableObject : ScriptableObject
{
    [Header("Item On Weapon Sprite")]
    public Sprite itemIcon;
    public Sprite itemBrokenOnGround;
    public bool brokenItemSticksToWall = false;

    [Header("In Game Sprite")]
    public Sprite sprite;

    [Header("Sounds")]
    public AudioClip onRespawn;
    public AudioClip onPickup;
    public AudioClip onDestroy;
    [Tooltip("Will always play if you have three of this item")]
    public AudioClip ultimateFire;
    public AudioClip fire;
    [Tooltip("Highest priority overrides the current fire sound")]
    public int fireSoundPriority = 0;
    public AudioClip bulletImpactSound;
    [Tooltip("Highest priority overrides the current bullet impact sound")]
    public int bulletImpactPriority = 0;

    [Header("Weapon modifiers")]
    [Tooltip("Total Ammo before item breaks")]
    public float ammo = 0;
    [Tooltip("ammo consumed per shot, ammoweight of 0.5 fires 2 bullets per 1 ammo. Three ammoweight 0.5 mods gives 0.125 ammoweight, 8 shots per 1 ammo. Three ammoweight 1.5 mods gives 3.4 ammoweight, 1 shot per 3.4 ammo")]
    public float ammoWeight = 1;
    [Tooltip("modifier is additive")]
    public float weaponDamage = 0;
    [Tooltip("modifier is additive")]
    public float recoilModifier = 0;
    [Tooltip("Base rate of Bullets fired per second, can only be set on base weapon")]
    public float baseFireRate = 0;
    [Tooltip("% Increase of fire rate, use negative values for a decrease in fire rate")]
    public float fireRatePercentage = 0;
    [Tooltip("3 items with 100% accuracy gives weapon 100% accuracy, 3 items with 80% accuracy gives 0.8*0.8*0.8 = 51% accuracy")]
    [Range(0, 100)]public float accuracy = 100;
    [Tooltip("The spread of missed bullets, 0% accuracy with a 45 degree miss angle allows you to miss in a 90 degree cone")]
    [Range(0, 90)] public float maxMissDegAngle = 0;
    public bool isShotgun = false;
    public bool isSuperShredder = false;

    public int shotgunAmount = 0;

    [Header("Bullet modifiers")]
    public Sprite bulletSprite;
    [Tooltip("Highest priority overrides the current bullet sprite")]
    public int bulletSpritePriority = 0;

    [Tooltip("modifier is additive")]
    public float bulletVelocity = 0f;
    [Header("Bouncy")]
    public bool isBouncy = false;
    public int numOfBounces = 0;
    [Header("Penetration")]
    public bool isPenetrate = false;
    public int numOfPenetrations = 0;
    [Header("Explosive")]
    public bool isSuperMicro = false;
    public bool isExplosive = false;
    public float explosionRadius = 0f;
    public float explosionDamage = 0f;
    [Header("Knockback")]
    public bool isKnockback = false;
    public float knockbackModifier = 0f;
    [Header("Homing")]
    public bool isHoming = false;
    public float turnSpeed = 0f;
    public float scanBounds = 0f;
    [Header("Stapler")]
    public bool isSuperStapler = false;
    public bool isStapler = false;
    public float stunTime = 0f;
    public float speedSlowdown = 0;
    [Header("Animations")]
    public bool hasAnimations = false;
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

We had a lot of fun adding all kinds of juice to get the game as fun as possible.  
My favourite contributions to juicing the game is the following:  
(Keep in mind everything is in 2D) 

<table>
  <tr>
    <td><img src="Images\EOTM_pencil_wall.gif" /> <br> <i>Pencil gets stuck on the wall</td>
    <td><img src="Images\EOTM_items.gif" />
  <br> <i>Item pickup and dropping</td>
  </tr>
  <tr>
    <td><img src="Images\EOTM_aimline.gif" />
  <br> <i>The laser extends based on bullet velocity</td>
    <td><img src="Images\EOTM_masking.gif" />
  <br> <i>Player sprite is masked by the green area</td>
  </tr>
</table>

Click the dropdown arrow below to see the code

<details>
<summary>Shooting pencils sticks to walls</summary>
<br>

 ```csharp
using UnityEngine;

//This object gets instanced when a pencil bullet hits a wall
//All functions gets called by the bullet before the bullet is destroyed

public class PencilStuckInWall : MonoBehaviour
{
    public GameObject pencil;
    public GameObject crack;

    public void SetPencilPosition(Collision2D wall)
    {
        Vector2 conctactPoint = wall.GetContact(0).point;
        Vector2 wallNormal = wall.GetContact(0).normal;


        float penDistanceFromCenter = Vector2.Dot(conctactPoint, wallNormal);
        float wallDistanceFromCenter = Vector2.Dot(wall.transform.position, wallNormal);

        float distanceUpWall = penDistanceFromCenter - wallDistanceFromCenter;
        //puts the pen 1/4th of the wall up from contact point
        Vector2 offset = wallNormal * 0.5f * distanceUpWall;
        transform.position = conctactPoint - offset;
    }

    public void SetPencilRotation(Quaternion rotation)
    {
        pencil.transform.rotation = rotation;
    }

    public void SetCrackTransform(Collision2D wall)
    {
        //Vector perpendicular to wall
        Vector2 wallNormal = wall.GetContact(0).normal;
        //Vector towards wall from collision point
        Vector2 wallDirection = new Vector2(wallNormal.y, wallNormal.x);

        //The centerpoint where all walls are facing towards this point
        Vector3 wallGraphicsCenterPoint = new Vector3(0, -1, 0);

        //Checks distance from Vector3.zero, the dot product of the normal vector compares only the distance in relation to the walls face
        //a wall along the x-axis will have a normal of (0, 1) and then we compare only the y value distance from center to figure out if the pen is above or below the wall
        float itemDistanceFromCenter = Mathf.Abs(Vector2.Dot((transform.position - wallGraphicsCenterPoint), wallNormal));
        float wallDistanceFromCenter = Mathf.Abs(Vector2.Dot((wall.transform.position - wallGraphicsCenterPoint), wallNormal));

        if (wallDistanceFromCenter < itemDistanceFromCenter)
        {
            //Item is behind a wall
            pencil.GetComponent<SpriteRenderer>().sortingOrder = -2;
            Destroy(crack);
            return;
        }
        else
        {
            //Debug.Log("Put item on wall");
            //Set the items angle to 70deg relative to the walls face
            crack.transform.localEulerAngles = new Vector3(wallDirection.x * -70, wallDirection.y * 70, transform.localEulerAngles.z);
            crack.GetComponent<SpriteRenderer>().sortingOrder = 1;
        }
    }
}
```
</details>


<details>
<summary>Item magnets to weapon slot on pickup</summary>
<br>

 ```csharp
    //Negative base speed makes item go away from target at start
    public float baseSpeed = -3f;
    public float acceleration = 15;

    public void MoveTowardsTarget()
        {
            Vector3 itemSlotPosition = itemSlot.transform.position;
            Vector3 moveDirection = itemSlotPosition - transform.position;

            if(moveDirection.magnitude < 0.2f)
            {
                targetWeapon.GetComponent<WeaponController>().AddItem(item);
                Destroy(gameObject);
            }

            transform.position += baseSpeed * Time.deltaTime * moveDirection.normalized;
            baseSpeed += Time.deltaTime * acceleration;
        }

```
</details>

<details>
<summary>Item drops to the ground and breaks when removed from weapon</summary>
<br>

 ```csharp
 using UnityEngine;

public class ItemRemovedFromGun : MonoBehaviour
{
    public Sprite sprite;
    public Sprite brokenSprite;
    public Vector3 moveDirection;

    public bool sticksToWalls = false;

    public float throwDistance = 5f;
    public float throwHeight = 1;
    public float animationSpeed = 4;
    public float breakSize = 1f;

    private Vector3 startLocalScale;
    private Rigidbody2D rb2d;

    private float timer = 0;
    private bool isBroken = false;

    void Start()
    {
        rb2d = GetComponent<Rigidbody2D>();
        GetComponent<SpriteRenderer>().sprite = sprite;
        startLocalScale = transform.localScale;

        float offset = 0.1f;
        float randomDistance = throwDistance + throwDistance * Random.Range(-throwDistance * offset, throwDistance * offset);
        rb2d.AddForce(moveDirection * randomDistance, ForceMode2D.Impulse);
    }

    void FixedUpdate()
    {
        if (isBroken) { return; }

        //For Kinematic Mode
        //Vector3 moveDistance = throwDistance * moveDirection * Time.deltaTime;
        //rb2d.MovePosition(transform.position + moveDistance);

        float size = Mathf.Sin(timer);

        transform.localScale = startLocalScale + startLocalScale * throwHeight * size;

        timer += (Time.deltaTime * animationSpeed);

        if(breakSize - 1 > size) 
        {
            BreakItem(null);
        }
    }

    void BreakItem(Collision2D wall)
    {
        isBroken = true;

        GetComponent<CircleCollider2D>().enabled = false;
        GetComponent<SpriteRenderer>().sortingOrder = -1;
        GetComponent<SpriteRenderer>().sprite = brokenSprite;
        GetComponent<Rigidbody2D>().velocity = Vector2.zero;

        transform.localScale = new Vector3(breakSize, breakSize, breakSize);

        //If not broken by wall
        if (wall == null)
            return;

        //Vector perpendicular to wall
        Vector2 wallNormal = wall.GetContact(0).normal;
        //Vector towards wall from collision point
        Vector2 wallDirection = new Vector2(wallNormal.y, wallNormal.x);

        float itemDistanceFromCenter = Mathf.Abs(Vector2.Dot((Vector2)transform.position, wallNormal));
        float wallDistanceFromCenter = Mathf.Abs(Vector2.Dot((Vector2)wall.transform.position, wallNormal));

        if (wallDistanceFromCenter > itemDistanceFromCenter)
        {
            Debug.Log("Put item on wall");
            //Set the items angle to 70deg relative to the walls face
            transform.localEulerAngles = new Vector3(wallDirection.x * -70, wallDirection.y * 70, transform.localEulerAngles.z);
            GetComponent<SpriteRenderer>().sortingOrder = 1;

            float wallX = Mathf.Abs(wallNormal.x);
            float wallY = Mathf.Abs(wallNormal.y);

            //Move the item "up the wall" from collision point
            Vector2 offset = wallNormal / 8;
            transform.position -= (Vector3)offset;
        }
        else
        {
            //Item is behind a wall
            Destroy(gameObject);
        }
    }

    private void OnCollisionEnter2D(Collision2D other)
    {
        if(!sticksToWalls) { return; }

        Debug.Log("Collided with " + other.transform.name);
        if (other.gameObject.CompareTag("HardWall") || other.gameObject.CompareTag("SoftWall"))
        {
            Debug.Log("Item Collided with wall");
            BreakItem(other);
        }
    }
}
```
</details>

<details>
<summary>Laser guided Aim</summary>
<br>

 ```csharp
using UnityEngine;

[RequireComponent(typeof(LineRenderer))]
public class AimLine : MonoBehaviour
{
    private LineRenderer aimLine;
    public float laserWidth = 0.1f;
    public float laserMaxLength = 5f;
    public bool isBlocked = false;

    void Start()
    {
        aimLine = GetComponent<LineRenderer>();
        Vector3[] initLaserPositions = new Vector3[2] { Vector3.zero, Vector3.zero };
        aimLine.SetPositions(initLaserPositions);
    }

    void Update()
    {
        if (!isBlocked)
        {
            aimLine.enabled = true;
            ShootLaserFromTargetPosition(transform.position, transform.up, laserMaxLength);
        }
        else
        {
            aimLine.enabled = false;
        }
    }

    void ShootLaserFromTargetPosition(Vector3 targetPosition, Vector3 direction, float length)
    {
        Vector3 endPosition = targetPosition + (length * direction);

        RaycastHit2D raycastHit = Physics2D.Raycast(targetPosition, direction, length, LayerMask.GetMask("HardWall", "SoftWall"));
        
        if(raycastHit)
            endPosition = raycastHit.point;

        aimLine.SetPosition(0, targetPosition);
        aimLine.SetPosition(1, endPosition);
    }
}
```
</details>

<details>
<summary>The player sprite is masked when sprite is clipping through walls</summary>
<br>

 ```csharp
using System.Collections.Generic;
using UnityEngine;

public class RoomMaskManager : MonoBehaviour
{
    [SerializeField] private List<SpriteMask> rooms;
    public string layerName;

    void Start()
    {
        foreach (var spriteMask in rooms)
        {
            spriteMask.frontSortingLayerID = SortingLayer.NameToID(layerName);
            spriteMask.backSortingLayerID = SortingLayer.NameToID(layerName);
            spriteMask.GetComponent<RoomMask>().playerSpriteName = layerName;
        }
    }
}

```
 ```csharp
using UnityEngine;

 public class RoomMask : MonoBehaviour
{
    public string playerSpriteName;
    bool startCollision;

    void Start()
    {
        //Fixes bug where player is hidden at start because game time is paused before collision triggers to play the 3..2..1 countdown.
        Invoke(nameof(DisableAllMasks), 0.1f);
    }

    void DisableAllMasks()
    {
        if(!startCollision)
        GetComponent<SpriteMask>().enabled = false;
    }

    private void OnTriggerEnter2D(Collider2D collision)
    {
        startCollision = true;
        //Debug.Log("Trigger Entered with " + collision.name);
        if(collision.name != playerSpriteName) { return; }
        //Debug.Log("Enabled mask: " + collision.name);
        GetComponent<SpriteRenderer>().enabled = false;
        GetComponent<SpriteMask>().enabled = true;
    }

    private void OnTriggerExit2D(Collider2D collision)
    {
        //Debug.Log("Trigger Left: " + collision.name);
        if (collision.name != playerSpriteName) { return; }

        if (collision.transform.parent.TryGetComponent<HasHealth>(out HasHealth health))
        {
            if (health.isDead)
            {
                Debug.Log("RoomMask: Player died");
                return;
            }
        }

        Debug.Log("Disabled mask: " + collision.name);
        GetComponent<SpriteRenderer>().enabled = true;
        GetComponent<SpriteMask>().enabled = false;
    }
}
 ```
</details>

---

## - **Camera Controller**

The Camera Controller adjusts both the position of the camera and the zoom in factor and can be adjusted with the exposed variables.  
It captures all objects added to the tracking list, and when an object is removed from tracking it smoothly pans over to highlight the remaining tracked objects.  
The camera stays within the bounds of the map and can do so while zooming in and out.

<table>
  <tr>
    <td><img src="Images\EOTM_CameraController.gif" /> 
  <br> <i>Adjusting camera settings</td>
    <td><img src="Images\EOTM_04.gif" /> 
  <br> <i>The Camera follows the action and stays within the map</td>
  </tr>

</table>

[CameraController.cs](https://github.com/MikaelahJ/EmployeeOfTheMonth/blob/main/Employee%20of%20the%20month/Assets/Scripts/CameraController.cs)  

<details>
<summary>CameraController</summary>
<br>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class CameraController : MonoBehaviour
{
    public GameObject map;
    public GameObject[] players;
        
    [SerializeField] [Range(0, 10)]   private float moveSpeed = 2;
    [SerializeField] [Range(0, 10)]   private float zoomSpeed = 1.5f;
    [SerializeField] [Range(-10, 10)] private float zoomOffset = 1;
    [SerializeField] [Range(0, 10)]   private float zoomScale = 2.8f;

    [SerializeField] [Range(0, 10)]   private float minOrthograpic = 4;
    [SerializeField] [Range(0, 100)]  private float maxOrthograpic = 15;

    private float xMin, xMax, yMin, yMax;
    private Camera cam;
    private int numOfPlayers = 0;

    void Awake()
    {
        players = new GameObject[4];
        if(TryGetComponent(out AudioSource audio))
        {
            audio.volume = AudioManager.instance.audioClips.characterVolume;
        }
    }
    // Start is called before the first frame update
    void Start()
    {
        cam = GetComponent<Camera>();
        //Lock the camera to the map
        if (map.TryGetComponent<BoxCollider>(out BoxCollider mapBounds))
        {
            if(mapBounds.bounds.extents.x/cam.aspect > mapBounds.bounds.extents.y)
            {
                xMin = -mapBounds.bounds.extents.x + mapBounds.bounds.center.x;
                xMax = mapBounds.bounds.extents.x + mapBounds.bounds.center.x;
                yMin = xMin / cam.aspect;
                yMax = xMax / cam.aspect;
                //Debug.Log("Camera limited by Width");
            }
            else
            {
                yMin = -mapBounds.bounds.extents.y + mapBounds.bounds.center.y;
                yMax = mapBounds.bounds.extents.y + mapBounds.bounds.center.y;

                xMin = yMin * cam.aspect;
                xMax = yMax * cam.aspect;
                //Debug.Log("Camera limited by height");
            }

            //Set the max cam size to view whole map
            maxOrthograpic = yMax;
        }
    }

    void Update()
    {
        if(numOfPlayers != 0)
            MoveCameraOrthographic(players);
        if(numOfPlayers == 1)
        {
            moveSpeed = 8;
        }
    }

    public void AddCameraTracking(GameObject player)
    {
        for (int i = 0; i < players.Length; i++)
        {
            if (players[i] == null)
            {
                players[i] = player;
                Debug.Log("Added " + player.name + " to camera tracking");
                numOfPlayers++;
                return;
            }
        }
        Debug.Log("Tracking list full! Couldn't add " + player.name + " to camera tracking");
    }

    public void RemoveCameraTracking(GameObject player)
    {
        for (int i = 0; i < players.Length; i++)
        {
            if (players[i] == player)
            {
                players[i] = null;
                Debug.Log("Removed " + player.name + " from camera tracking");
                numOfPlayers--;
                return;
            }
        }
        Debug.Log("Can't Remove " + player.name + " in camera tracking list, does not exist in tracking");
    }


    void MoveCameraOrthographic(GameObject[] players)
    {
        //Set Size
        float currentSize = cam.orthographicSize;
        float targetSize = zoomOffset + GetCameraTargetSize(players);
        float increment = (targetSize - currentSize) * Time.deltaTime * zoomSpeed;

        cam.orthographicSize = Mathf.Clamp(currentSize + increment, minOrthograpic, maxOrthograpic);

        //Move camera
        Vector3 middle = GetCameraDestination(players);
        middle.z = 0;
        transform.position += middle * Time.deltaTime * moveSpeed;

        if(map != null)
            transform.position = ClampCameraToMap();
    }

    Vector3 ClampCameraToMap()
    {
        float height = cam.orthographicSize;
        float width = height * cam.aspect;

        float camX = Mathf.Clamp(transform.position.x, xMin + width, xMax - width);
        float camY = Mathf.Clamp(transform.position.y, yMin + height, yMax - height);
        return new Vector3(camX, camY, -10);
    }

    float GetCameraTargetSize(GameObject[] players)
    {
        float aspect = GetComponent<Camera>().aspect;
        List<Vector3> playerPositions = new List<Vector3>();

        //Scale distance in proportions to the camera aspect
        for (int i = 0; i < players.Length; i++)
        {
            if(players[i] != null)
                playerPositions.Add(new Vector3(players[i].transform.position.x, players[i].transform.position.y * aspect));
        }

        float maxDistance = minOrthograpic;
        //Calculate the max distance of all player pairs
        foreach (Vector3 position in playerPositions)
        {
            for (int i = 1; i < playerPositions.Count; i++)
            {
                float distance = Vector2.Distance(position, playerPositions[i]);
                if (maxDistance < distance)
                {
                    maxDistance = distance;
                }
            }
        }

        return maxDistance/zoomScale;
    }

    Vector3 GetCameraDestination(GameObject[] players)
    {
        Vector3 destination = Vector2.zero;
        foreach (GameObject player in players)
        {
            if(player != null)
                destination += (player.transform.position - transform.position);
        }

        destination = destination / players.Length;

        return destination;
    }

    void MoveCameraPerspective(Vector3 player, Vector3 cube)
    {
        Vector3 middle = ((player - transform.position) + (cube - transform.position)) / 2;

        transform.position += middle * Time.deltaTime * moveSpeed;
        transform.position = Vector3.Lerp(transform.position, new(transform.position.x, transform.position.y, -(zoomOffset + Vector2.Distance(player, cube))), Time.deltaTime);
    }
}

```
</details>

---

## - **Gamemodes**

Adding gamemodes to this game really enhances the replay value and allows you to team up.  
The King of the Hill gamemode shown below is by far my favourite mode.  
It was quite a challenge to implement due to it being added late into development and we had only scoped for the Free For All mode to begin with.

<table>
  <tr>
    <td><img src="Images\EOTM_gamemodes.gif"/> </td>
    <td><img src="Images\EOTM_02.gif"/>
  <br> <i> 4 Player Free for All King of the Hill gamemode</td>
  </tr>
</table>

Click the link below to view the code, it's too long to include in a dropdown menu.

[GamemodeManager.cs](https://github.com/MikaelahJ/EmployeeOfTheMonth/blob/main/Employee%20of%20the%20month/Assets/Scripts/GameModeManager.cs) 

<details>
<summary>King of the Hill zone</summary>
<br>

 ```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System.Linq;
using System.Text.RegularExpressions;
using TMPro;

public class KingOfTheHillScript : MonoBehaviour
{
    ParticleSystem sys;
    ParticleSystemRenderer myParticleSystemRenderer;
    BoxCollider2D myCollider;
    LineRenderer myLine;
    [SerializeField] private TextMeshPro scoreText;

    List<Team> teamsInZone = new List<Team>();
    int currentPoints = 0;

    void Start()
    {
        scoreText.text = "";
        myParticleSystemRenderer = GetComponent<ParticleSystemRenderer>();
        sys = GetComponent<ParticleSystem>();
        myCollider = GetComponent<BoxCollider2D>();
        myLine = GetComponent<LineRenderer>();
        SetLine();
    }

    private void SetLine()
    {
        if (teamsInZone.Count == 0)
        {
            SetBorderColors();
        }
        else
        {
            SetBorderColors(teamsInZone[0]);
        }

        Mesh mesh = myCollider.CreateMesh(false, false);
        mesh.Optimize();

        Vector3[] positions = mesh.vertices;
        positions = positions.OrderBy(pos => Vector3.SignedAngle(pos.normalized, Vector3.up, Vector3.forward)).ToArray();

        myLine.positionCount = positions.Length;
        myLine.SetPositions(positions);
    }

    private void SetBorderColors(Team team = null)
    {
        Color32 teamColor = new Color32(255, 255, 255, 255);
        if (team != null)
            teamColor = team.GetColor();

        Material newLineMaterial = new Material(myLine.material);
        newLineMaterial.color = teamColor;
        myLine.material = newLineMaterial;

        var mainModule = sys.main;
        ParticleSystem.MinMaxGradient colGrad = new ParticleSystem.MinMaxGradient(teamColor);
        mainModule.startColor = colGrad;
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        if (other.gameObject.CompareTag("Player"))
        {
            teamsInZone.Add(other.GetComponentInParent<HasHealth>().team);
            OnlyOneTeamInZone();
        }
    }

    private void OnTriggerStay2D(Collider2D other)
    {
        if (other.gameObject.CompareTag("Player"))
        {
            if (OnlyOneTeamInZone())
            {
                Team team = other.GetComponentInParent<HasHealth>().team;
                team.AddPoints(Time.deltaTime);
                if (team.GetPoints() > currentPoints + 1)
                {
                    DisplayTeamText(team);
                }
            }
        }
    }

    private void OnTriggerExit2D(Collider2D other)
    {
        if (other.gameObject.CompareTag("Player"))
        {
            teamsInZone.Remove(other.GetComponentInParent<HasHealth>().team);
            OnlyOneTeamInZone();
        }
    }

    private bool OnlyOneTeamInZone()
    {
        if (teamsInZone.Count == 0)
        {
            scoreText.text = "";
            SetLine();
            return false;
        }
        Team lastTeam = teamsInZone[0];
        for (int i = 1; i < teamsInZone.Count; i++)
        {
            if (lastTeam != teamsInZone[i])
            {
                return false;
            }
        }
        DisplayTeamText(lastTeam);
        return true;

    }

    private void DisplayTeamText(Team team)
    {
        string AddSpaceBeforeNumbers = string.Join(" ", Regex.Split(team.GetTeamName().ToString(), @"(?<!^)(?=[0-9])"));
        currentPoints = Mathf.FloorToInt(team.GetPoints());
        string displayText = AddSpaceBeforeNumbers + "\n" + "Score: " + currentPoints;
        scoreText.text = displayText;
        Color32 teamColor = team.GetColor();
        scoreText.color = teamColor;
        SetLine();
    }
}
```
</details>