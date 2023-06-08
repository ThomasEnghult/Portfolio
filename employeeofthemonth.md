# *Employee of the Month*

## *A brief game description*
Employee of the month is an 4 player arena shooter where you pick up office supplies and modify your improvised handmade weapon to shoot at your co-workers.  
Different office supplies gives unique effects and alter your weapon accordingly. Each round you have only one life and the last man standing wins the round.

---

## *My contributions to this project*

### - Creating the weapon

<table>
  <tr>
    <td><img src="Images\EOTM_weapon-pickup.gif" /></td>
    <td><img src="Images\EOTM_scriptable_objects.png" /></td>
  </tr>
</table>


The weapon and its modifiers are created using Scriptable Objects. Each office supply that you can pick up is a scriptable object, this enables fast iteration and addition of items.

<details>
<summary>Want to ruin the surprise?</summary>
<br>

    ```csharp
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
    ```
</details>
