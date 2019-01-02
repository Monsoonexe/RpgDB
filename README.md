# RPG Database for Unity

This is a simple database builder for use in RPG style games. The goal of this asset is to provide a simple solution for translating information from a human-readable spreadsheet into programmable game data. In thirty seconds a spreadsheet of data from your favorite pen and paper RPG can be converted to JSON and saved in a directory that can be parsed by this asset.

That being said, this asset may require some customization to be useful in specific situations. It was built using Starfinder (a D&D derivative) as inspiration, but with a few small tweaks the data format can be massively changed to fit any situation.

The following README contains development notes to make customizations painless.

### Requirements

JSON.net for Unity
https://assetstore.unity.com/packages/tools/input-management/json-net-for-unity-11347


## Overview


### Search

There is currently one primary search function that can be used in the code: 

`RpgDB.XXXDatabase.GetByName()`

This returns an object of the type that is specific to the database, such as `class Weapon`

**Example Search:**
```
Weapon debugWeapon = allWeapons.GetByName("Bow");
PropertyInfo[] properties = typeof(Weapon).GetProperties();
foreach (PropertyInfo property in properties)
{
Debug.Log(property.Name + ": " + property.GetValue(debugWeapon, null));
}
```


### GameDatabase.cs
`public class GameDatabase : MonoBehaviour`

After attaching this script to a GameObject, this script will initialize at runtime with several components that hold parsed data. These are held as components so that the JSON is parsed only once, at runtime.

The components are:
- Weapons
- Ammunition
- Armor
- Upgrades
- Classes


## Constructors

### Constructors/Database.cs
`public abstract class Database : MonoBehaviour`

This MonoBehaviour is the base class for the components that are built by `RpgDB.GameDatabase`, and provides all common functionality for parsing the JSON data. 

The following section of notes is intended to support case-by-case customization of RpgDB. 

When using RpgDB for alternate datasources, most of the work will be done in the `Constructors`, and the requirements should be intuitive: change the constructor class fields and methods to match the data you're providing.

However, there may some situations where your solution will require cautiously modifying the `RpgDB.Database` constructor. Hopefully these notes you in that process:

- **RpgDB.Database.JsonHome**

	Location of directory containing JSON files relative to project home.

	Default: `public static string JsonHome = @"Assets/RpgDB/json/"`

- **RpgDB.Database.AddObject(JToken item, List<IRpgDBEntry> list, string category=null)**
	
	Void class that is defined individually within all subsequent _database_ classes. 

	Example from `Constructors/Equipment/WeaponDatabase.cs`:
	```
	public override void AddObject(JToken item, List<IRpgDBEntry> list, string category)
	{
	    Weapon weapon = new Weapon();
	    weapon.ConvertObject(item, category);
	    list.Add(weapon);
	}
	```

	This function takes in a list of `RpgDB.IRpgDBEntry` objects and converts them to their appropriate status using `RpgDB.RpgDBObject.ConvertObject`, then appends the newly converted object to the provided list.

- **RpgDB.Database.GetJsonFromFile(string file_path)**

	Takes in a simple file path string, then prepends it with `RpgDB.Database.JsonHome`, appends it with `.json`. Once the file path is complete, this function will use the disposable `System.IO.StreamReader` to convert the file contents into a string variable: `file_output`.

	`Newtonsoft.Json.Linq.JObject.Parse(file_output)` will load a JToken from a string that contains JSON. This JToken is necessary for further datahandling, and is returned by the function upon its creation.

- **RpgDB.Database.LoadDataFromJson(string category, List<IRpgDBEntry> list)**

	Uses the string `category` as the `file_path` for `RpgDB.Database.GetJsonFromFile(string file_path)`. The object that is returned will hold a single "parent" JToken, which will contain a series of "child" JTokens as defined in the JSON file.

	This function will loop through each child item, running `RpgDB.Database.AddObject` to convert the JToken into an `RpgDB.IRpgDBEntry` that may be added to the provided list.

- **RpgDB.Database.LoadData(string[] categories, List<IRpgDBEntry> list)**

	Intermediary between `RpgDB.Database.LoadDataFromJson` and the array of category names that is provided when run by the child classes of `RpgDB.Database`.

- **RpgDB.Database.LoadData(string category, List<IRpgDBEntry> list)**

	Intermediary between `RpgDB.Database.LoadDataFromJson` and the child classes of `RpgDB.Database`.

	This overload allows category arrays and single categories to be handled identically.


### Constructors/IRpgDBEntry.cs
`public interface IRpgDBEntry`

This interface allows dynamic functionality for helper functions that are not benefitted by static typing.

This is used most frequently by the children of the `RpgDB.Database` class, in order to allow common functionality to be passed up to the parent. We are also able to use this interface to create an inventory system that is simply a `List<IRpgDBEntry>`.

_Development Note: This item needs be upgraded to not rely on an interface._


## Constructors/Character

The following constructors are used to parse JSON data related to character class, skills, abilities, proficiencies, feats, and modifiers. 

Character Race is scheduled but not yet featured.

### Constructors/Character/ClassDatabase.cs
`public class ClassDatabase : Database`

Use `ClassDatabase.CreateCharacter` to convert a character class string into a Character object. This is the recommended method for creating a Character.

Overloaded options for CreateCharacter:

    - public Character CreateCharacter(List<CharacterClass> characterClasses, ExtensionsDatabase extensions){}
    - public Character CreateCharacter(string characterClass, ExtensionsDatabase extensions){}
    - public Character CreateCharacter(string characterClass, int level, ExtensionsDatabase extensions){}

## Constructors/Equipment

The following constructors are used to parse JSON data for weapons, ammunition, armor, and upgrades.

_Note: Applying upgrades to weapons and armor is not currently featured._

### Constructors/Equipment/Ammunition.cs
`[System.Serializable] public class Ammunition : RpgDBObject, IRpgDBEntry`

### Constructors/Equipment/AmmunitionDatabase.cs
`public class AmmunitionDatabase : Database`

### Constructors/Equipment/Armor.cs
`[System.Serializable] public class Armor : RpgDBObject, IRpgDBEntry`

### Constructors/Equipment/ArmorDatabase.cs
`public class ArmorDatabase : Database`

### Constructors/Equipment/RpgDBObject.cs
`[System.Serializable] public class RpgDBObject : IRpgDBEntry`

### Constructors/Equipment/Upgrade.cs
`[System.Serializable] public class Upgrade : RpgDBObject, IRpgDBEntry`

### Constructors/Equipment/UpgradeDatabase.cs
`public class UpgradeDatabase : Database`

### Constructors/Equipment/Weapon.cs
`[System.Serializable] public class Weapon : RpgDBObject, IRpgDBEntry`

### Constructors/Equipment/WeaponDatabase.cs
`public class WeaponDatabase : Database`


### Categories

Categories are defined at the top of the each class, and are used in `RpgDB.Database.LoadData` on `start()`. The categories are strings (or arrays of strings), which are used for the following:

1. Specify the name of the JSON file. This will be passed as the `file_path` in the method `RpgDB.Database.GetJsonFromFile(file_path)`, which will concatenate it to create the full file path: `JsonHome + file_path + ".json")`
1. Assist in the parsing of JSON. The object created by `Newtonsoft.Json.Linq.JObject.Parse` requires a top-level JSON object to hold the subsequent entries. Each JSON file will have a top level item that shares the name of the JSON file.
1. Provide a category for searching. `RpgDB.WeaponDatabase.AddWeapon` takes a parameter called `title`, which will used to populate the field `Weapon.Category`. This field is subsequently used by `RpgDB.WeaponDatabase.SearchWeaponsByCategory` in order to filter `AllWeaponsList`.

Categories examples from `WeaponsDatabase.cs`:

```
public string[] MeleeCategories = { "1h_melee", "2h_melee", "solarian_crystals" };
public string[] RangedCategories = { "small_arms", "longarms", "snipers", "heavy_weapons" };
public string ThrownCategory = "thrown";
```

If the JSON file names are modified, or if additional files are added, be sure to modify the top-level object name in the json file _and_ the appropriate categories list(s) in the associated class object. All three must explicitly match for the data to be properly loaded:

1. JSON File Name 
    - ex. **1h_melee**.json
1. Top level object name in JSON file
    - ex. { **"1h_melee"**: [...] }
1. Appropriate lists for the associated class object
    - ex. public string[] MeleeCategories = { **"1h_melee"** ... }

#### JSON

The `/json` directory is used for housing all data files. The JSON files are handled by `JSON.net`. Should you choose to modify the location of the JSON files, simply modify `RpgDB.Database.JsonHome` accordingly.

> **Debugging Tip:**
> If an error occurs due to a JSON entry, run the script
> and look at which item is the last entered to the db.
> The _next_ item on the list caused the error.


## TODO

### Searching
Search functionality will need dramatic improvement. Currently, the only reliable search function across all categories is `RpgDB.Database.GetByName()`. WeaponDatabase has starters for other search types that should be refined before being implemented elsewhere. 

Also, **the search functions are all currently case-sensitive**.

### Race
Similar to classes, races have influential modifiers for a character. This is an important next step for completing the character creation process.
