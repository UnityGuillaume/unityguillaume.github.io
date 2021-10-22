[Supporting project on
github](https://github.com/UnityGuillaume/BinarySaveTutorial)

[Link to the project zip
file](https://github.com/UnityGuillaume/BinarySaveTutorial/releases/tag/0.1)

Saving is one of the base functionalities shared by a lot of games but contrary
to moving an object or loading a mesh, there isn’t one simple accepted canonical
way to do it, as it will be dependent on specifics of the project and its
requirements.

This is why this document isn’t a step by step tutorial. Instead, it is an
explanation of the techniques used to save in the supporting sample project,
highlighting which choices were made, and why. It only shows a way to do it  and
will help as a starting point for your design choice that fits  your project and
needs.

Requirements: this is an intermediate level scripting guide, so it assume that
you are familiar with :

- Simple programming concept like functions, variables, class Know how to
- navigate code and read comment

The recommended way to follow that guide is to have the project open so you can
follow along with the explanations.

Understanding how values are encoded as binary is a plus, but the following
guide can be followed without that knowledge. A short
[primer](#primer-on-encoding-and-bytes) is included in that document to expose
the base of it, and can be skipped if you are familiar with it.

Additionally, note that the sample project is kept simple so it is easier to
parse its code and navigate it. Some shortcuts are taken in the interest of
brevity and everything is intended as a starting point for a more complete
system.

# Saving

Let’s start by defining what we mean by “saving the game” : It is capturing, in
a file, at a specific time, the values of parameters, settings or variables of
the application. For example the player position, items acquired, or skills
unlocked etc. This way it can be read back next time the application is started,
in order to return to the same state.

The first thing to think about when designing a save system is which information
we want to save. We could just take everything in memory and write it to disk to
read it all back later (sometimes referred to as a “save state”). However,
recent games can take hundreds or thousands of MB of memory when running, and a
lot of that data is only useful for the engine (like lists of objects on
screens, current textures used etc.) and is temporary data, not relevant to
gameplay.

Instead we need to pick and choose what data is useful to save. In our project
for example, the current player position is something we want to save, but the
current camera position can be reconstructed from that player position so there
is no need to save it.

The data saved in the sample project is :

- Our player position, rotation and the items they have in their inventory
- The current level opened
- The state of the world in a set of flags (see Flags section) to keep track of
  what the players have already done.

The second thing is how we will save that information in the save file. There
are 2 main ways of doing it : text and binary.

## Text

Text is just writing, in text, the information. If we want to save the position
(10.514, 15.412, 13.3) for example we can write “10.514;15.412;13.3” in the file
or “{10.514,15.412,13.3}”. This have a couple of advantage :

- It’s easy for a human that open the save file to read and understand
- It is easily exchangeable between platform, as this is just text

But come with a couple of drawback :

- It needs to be parsed, that is processed to be understood. In our position
example, we need to code that if it reads 3 number separated by comma between
two curly brackets, it is a Vector3
- This extra works make reading the save file more expensive
- As it is easily understood and modified by humans it makes cheating a lot easier.

## Binary

Binary on the other hand is writing directly the bytes encoding each value
inside the file. For example a floating point value is stored in memory using 4
bytes, so writing the position (10.514, 15.412, 13.3) means we will write the 12
bytes corresponding to the 3 numbers in the file.

The benefits are that :

- It’s a lot smaller than text. Just for our example “{10.514,15.412,13.3}” is
  19 characters, so a minimum of 19 bytes, and that will be variable based on
  how many digits we have. But in binary this will always be 12 bytes. Contrary
  to text we also don’t have all the additional characters (like comma,
  space etc.) that are just here to help parsing/reading.
- We don’t have to parse to read back. Here we can read our 12 bytes and put
  them right in memory and we have the value again, which makes it significantly
  faster to read and write.

The drawbacks though are that it makes the save file really hard to read and
understand for a human. This however makes cheating a lot harder.

In this project we illustrate using the Binary method. It is usually the
standard when it comes to save files but is a bit harder to understand, so we
choose to illustrate that method here. Note however that you can create the same
save file using text with some small modification and by swapping writing text
instead of binary in the example.

# Save Location

Where you will store your save file will depend on the platform. Our sample uses
[Application.persistentDataPath](https://docs.unity3d.com/ScriptReference/Application-persistentDataPath.html)
which will point to a folder specific to the application (and sometimes also the
current logged in user) on the current running platform (see documentation for
the actual paths used for each platform).

But some platforms (like some digital distribution services or consoles) may
have specific functions to get the currently logged in user folder, that
sometimes is also backed up on a cloud storage. You should look up on each
platform documentation which folder to use.

_Note : remember you can use [C# preprocessor directive of Unity
API](https://docs.unity3d.com/Manual/PlatformDependentCompilation.html)
to check which platform the code is running on, so you can pick the right folder
depending on the platform_

## A Note on PlayerPrefs

Unity has a class called PlayerPrefs that allows you to store some key-value
pairs that persist between runs of the application, and you may encounter some
tutorials that use it to save data for the game. This is a very easy quick way
to save some data from your game but comes with a couple of drawbacks:

- You have a single player pref per application, meaning offering multiple
  save slots requires some tricks with the key names like appending the save
  number beforehand
- As there is a single PlayerPrefs per logged user on the platform, you can’t
  offer separate save files for two different profiles using the same computer
  account.
- Depending on the OS/platform, the data is not always saved inside a file
(e.g. on Windows this will be saved in the registry). It makes it difficult to
both backup the save or transfer it (like on cloud backup server)

If your game only saves a couple of data (like the highest score) or for quick
prototyping, this may work fine, but as a general rule, you should use a file on
which you have total control to where it is saved, which is why this sample uses
a save file we write instead of PlayerPrefs.

# SaveSystem

The SaveSystem in our sample project is a static class instantiated as soon as
the application starts. Its main role is to be the entry point to any
saving/loading operation ( e.g. the Load button on the main menu will call
GetSavesFiles on the SaveSystem to get all the available SaveFile to display,
then those will in turn call Load with the chosen file )

## Flags

Our save system contains a Dictionary of flags. A flag in that context is a pair
composed of a string (the name of the flag, also called the key) and a boolean
(true if the flag is set, false otherwise.).

Flags allow us to save the state of something in our application. They will be
set by gameplay events in our game and allow us to remember if a certain event
has already happened in the game.

In our sample, a door will set a flag with the door name as key to true when it
was opened. That way, when the door is initialized when we load the scene (its
Start event), it can query the save system for the flag value. If the flag is
false or does not exist, the door is still closed and so is left unchanged and
initialized normally. But if the flag is true, then that door was already open
previously and we can set it to an open state.

This can be used for all sorts of states : was a certain area reached or a
cutscene triggered, did the player make a specific choice in a dialogue. That
way we can spawn/remove objects from our scenes based on that, or load a
different scene. Some examples :

- When the player enters their home base, if the flag “blacksmith\_freed” is set
  to true because they already saved the blacksmith during the game, we
  can spawn the blacksmith character in the home base for the player to talk
  to and interact with.
- When the player reaches a certain point in the story, a city they visited
  earlier gets destroyed. This set the flag “city\_destroyed” to true. Now when
  the player enters the city, we check the flag. If it exists and is true, we
  load our CityDestroyed scene, otherwise we load the normal City scene.

Our simplified example has one flag dictionary in the save system, and the
LevelSystem has an helper function that will append the level name in front of
the key so we can have the same keys in different levels. A more complex
application may choose to have a flags list per level, or a more complex flags
system that can store not only boolean values but also an integer or a string
value etc.

## BinaryWriter and BinaryReader

There is two class in C# that we use in that sample to write our data in the
save file : BinaryWriter and BinaryReader. Those classes act on Stream, so they
can either work on FileStream, writing/reading into a file, or a MemoryStream
for writing/reading in an array in memory.

A BinaryWriter have a Write function that is overloaded for most basic C# type
like float, int, string etc. and will take care of writing the bytes for that
type properly in the stream. So if you use Write with a float, it will write the
4 bytes in the stream. Writing a String will first write the length as an int
(so 4 byte) then each character (1 byte each) one after the other, etc.

The BinaryReader on the other hand will require the user to explicitly state
what to read from the stream. This is because when we write bytes in our file,
it is all tightly packed with just the value, no additional information of which
type was written. Writing 4 characters (1 byte each) or a float (4 bytes) will
both result in 4 bytes in the stream and there is no ulterior way to know what
was written. So the BinaryReader has functions like ReadByte, ReadFloat,
ReadString etc. that we need to explicitly call depending on what we want to
read.

## Referencing and Databases

Through the use of BinaryWriter we can write the value as binary of most
“simple” types like int, float or string.

But in an application, we will have more complex data to save : references. In
our sample, our player has a list of references to Items (which are
ScriptableObject, so assets in our projects) that are the content of its
inventory. How do we save that list in the save file?

The reference is a pointer to the address of the object in memory, and even if
we could easily write that in the file, this would be useless as when you
restart the application the object would either not exist in memory yet or be at
a different location.

We could “follow” the reference and write in the file all the simple types
values each object is made up of (and that the BinaryWriter can handle). If our
item is made up of a name and an icone (an image) we could save the list of
items as :

```
|                 name                 |
| pixels of the image (array of bytes) |
|                 name                 |
| pixels of the image (array of bytes) |
|                 name                 |
| pixels of the image (array of bytes) |
|                 ....                 |
```

But this have a couple of shortcomings :

- This will make our save file way bigger than it needs to be : if multiple
  objects use the same image, it is now saved multiple times in the save files.
- It breaks the referencing : if we update the game changing the image in our
assets, an old save file will still load the old image.

So for this we can create databases that allow us to lookup objects from a
unique name/id, which is a simple string, something a BinaryWriter can easily
write.

Let’s take the ItemDatabase as an example: it is a ScriptableObject with an
array of Items. We create an instance of it in our Project folder at edit time,
and add to it all the items we create in the editor. (see Itemdb file in the Data
folder)

Then a function called Init will be called on it when the game starts. This will
build a Dictionary to match each item id to its Item reference.

So we can just save the unique id in the save file (a simple string), and when
loading the save, we just query the database for the reference to the item from
its unique id. See ItemDatabase.cs for the code of how such a lookup database
works.

# Save Process

## Saving

When the save button is clicked on the pause menu and after a file is chosen,
the saving process is :

1.  Open a BinaryWriter on the chosen file.

2.  The first data we write in it is related to the save file itself and not the
game. This is sometimes referred to as a “header”. In our case this contains :
    1.  The date & time at which the save was made
    2.  The version of the file (used for versioning, see [Versioning](#versioning)
    just below)
    3.  A byte array which is a very small (320x180) screenshot of the game
    encoded as png
    4.  Here you can add other meta information useful to your game : platform,
    user name, offsets to specific parts of the save file we want to retrieve
    faster etc..

    Headers are useful as they are the first readily available data when you open
    and read the save file. You won’t have to seek them in the save file and can get
    their data by just opening, reading, and closing the file.   E.g. When we list
    all save files in the Load menu entry, we open all save files and read the
    header to retrieve the time at which it was saved and the screenshot.

3.  We save the list of flags. As it is a dictionary of string-bool pair, and
BinaryWriter does not support Dictionary out of the box, we “break it down” to
save it manually :
    1. First we write an int, which is the number of pair in the dictionary
    2. Then we write each pair one after the other :
        1.  The key (a string so the Write function of the BinaryWriter handle
            it by itself)
        2.  And the value, which being a bool, is also handled natively by the
            BinaryWriter.

4.  Finally the SaveSystem will call the Save function on all the Systems we
want to save the state of, giving them the BinaryWriter reference. Separating it
in the different system instead of doing everything in the save system allows us
to add more systems at a later date, and make it easier to keep track of
everything saved by breaking it in smaller chunks.
    1.  The LevelSystem will write the current open scene as an integer
    2.  The PlayerSystem will write the position of the player as 3 consecutive
    floats, then its rotation as 4 float consecutive (a Quaternion), then the
    list of all the objects unique id which are strings (cf. [Referencing and Databases](#referencing-and-databases))
    in its inventory. As List and our Item class are not supported by the BinaryWriter,
    like for the dictionary, we write first the list size then loop on it and
    just write the id of each object.

The final save file then look like this :
```
|      Date and Time (string)      |<--+
|           Version (int)          |   | Header
|   PNG of screenshot (byte array) |<--+
|         Flags Count (int)        |<--+
|          Flag id (stirng)        |   | Flag Systems
|         Flag value (bool)        |   |
| ... Repeated For Flags Count ... |<--+
|        Current Scene (int)       |<--- Level System
|         Position X (float)       |<--+
|         Position Y (float)       |   |
|         Position Z (float)       |   |
|         Rotation X (float)       |   |
|         Rotation Y (float)       |   | Player System
|         Rotation Z (float)       |   |
|         Rotation W (float)       |   |
|         Items count (int)        |   |
|          Item unique Id          |   |
| ... Repeated for Item count ...  |<--+
```

## Versioning

We write a version in the header of our save file, but what is the purpose of
it? As you can see we are writing everything back to back, with no indication of
what is saved. That means that when loading we need to read things in the same
order. Which will be a problem if we add a new thing to the save system : older
save won’t be compatible anymore.

In our sample project, when we started developing the game, we were only saving
the character positions, rotations and the opened level. Our save file content
was then :

```
|     Header    |
|  Flag System  |
| Current Scene |
|    Position   |
|    Rotation   |
```

Then we added the capacity to get items and store them in an inventory on the
player. As we now have added the need to save the inventory in the PlayerSystem,
our new save file content then became :

```
|     Header    |
|  Flag System  |
| Current Scene |
|    Position   |
|    Rotation   |
|  Item Count   |
|   Item Id     |
|   Item Id     |
|     .....     |
```

But if we try to load a save file that was made before that feature was added,
the PlayerSystem will try to read the number of items in the inventory, but we
have already reach the end of the save file as this information doesn't exist in
that old save file, leading to an error. Worst if we had saved something after
the player system, like the stats of the player, we may end up reading a stat
as the number of item own and try to read that many from the save file, messing
up everything.

Instead we need to know that the save file we are reading doesn't contain that
information.

Our SaveSytem has a constant int value, `VERSION`, that is the current version
of the SaveSystem. This version is written in the save file header and will allow
you to know which version of the SaveSystem the save file was created with.

When reading back the save, we can compare the current SaveSystem version with
the one in the save file header.

If the header version is lower than the version where the new feature was added,
you can initialise the value to a default instead of trying to read it with the
BinaryReader.

An example of this can be found in the CharacterControl script, in the function
LoadPlayerData, where we create an empty dictionary of Item instead of reading
back the items if our save file is older than version 2, which was when we added
saving the inventory.

### Manual version bump

In our sample, it is the programmer's responsibility to manually bump the save
system version by one every time they add something to the save file. This may
lead to error (forget to bump the version when adding something, or conflict
when 2 persons bump at the same time) but is also the simplest, fastest way to
handle it.

Some projects may have already in place a versioning system, maybe based on the
source code versioning system, or automated build system. In that case, you can
instead use this version to write in the save file header and test against.

One step that can be taken to help “clean” the code is to remove all version
check before the first public release (or when you “feature freeze” your
project, so no new data should be added/removed from the save file anymore) and
reset the version in the SaveSystem back to 1. All your development test save
files that weren’t updated to the latest version won’t load anymore, but that
will make the save code easier to read for future debugging or additional
features post release.

## Loading

The loading process is the same order and process but with a BinaryReader to
read back data.

1. We copy the content of the save file into a MemoryStream. As loading has
some waiting time (see point 5.2 of that process), if the game is forced to quit or
crash during that waiting time, we would leave a dangling open file. So we copy
everything into a MemoryStream so we can close the file immediately and read the
data directly from memory.
2. We open the BinaryReader on that MemoryStream.
3. We read the date & time and version of the save file and store it for easy
retrieval later. We jump over the image data as we do not need it on loading the
game, it is only used by the load menu.
4. We read back all the flags. Since the BinaryWriter/Reader doesn’t support
Dictionary, we have to rebuild it manually from the data we wrote during save :
    1.  We read an int, the number of pair we have to read
    2.  Then we loop this number of time and read
        1. One string, this is the key
        2. One bool, this is the value of the flag.
        3. And we add that pair in the dictionary
5.  We finally call load on each System with the BinaryReader
    1. The LevelSystem read the target level and trigger loading it
    2. We wait for the level to be loaded, otherwise our player data will be
    loaded before the new level.
    3. The Player System read the position (we read 3 floats and use them as
       x,y,z in a Vector3) and rotation (we read 4 float and set them as x,y,z,w
       of a Quaternion) of the player and move it to the right position, then read
       all the items unique ids, and look up the corresponding item in the Item
       Database (cf. [Referencing and Database](#referencing-and-databases)) to
       add them to the inventory.

# Additional notes on the project

## Primer on encoding and bytes

Our saving system is working directly with the byte encoding of values in memory
to save them, so if you are unfamiliar with how values are stored in a computer
memory, this is a short primer on how this works.

Computers work with binary, everything is either considered “on” (an electric
current pass, noted 1) or “off” (electric current doesn't pass, noted 0). This
is the building block on top of which everything else is built.

In order to express more complex things than just “on/off”, we can group those
values in a bundle to extend what they can express. In fact it is rare to
manipulate bits directly and instead the “practical” base building block of
every value on your computer is the byte, a group of 8 bits.

This can be used for example to express (encode) a number between 0 and 255 :

```
00000000 = 0
00000001 = 1
00000010 = 2
00000011 = 3
00000100 = 4
...
10000001 = 129
...
11111111 = 255
```

But this is just one possible encoding. With the same number of bits, we can
decide that the top (the left most) bit defines if the number is positive or
negative. Then our byte can then encode a value between -127 and 127 where for
example

```
00000001 = 1
10000001 = -1
00000011 = 3
10000011 = -3
01111111 = 127
11111111 = -127
```

_Advanced note: this is not exactly how most computer store signed byte, this is
just a simpler example to show how we can interpret the same byte differently_

Another example of how we can use those 8 bits is to associate a character with
each value. In the ASCII encoding standard, the value `01001000`, which with our
number encoding is `72`, is interpreted as capital letter `H`.

The important thing to remember is that the computer only stores those bits and
has no idea what they represent. This is us, the programmers, which decide how
we interpret (decode) them. In C# this is done through giving a type to our
variable. Giving a type to a variable will tell the computer how to interpret
the bits stored in the variable location.

For example doing :

```csharp
byte b = 72;
Debug.Log(b);
```

Will print 72 in the console, as b is a byte, this is a number and it read the
bits 01001000 as 72

But doing :

```csharp
byte b = 72;
Debug.Log((char)b);
```

Will print `H` in the console. By explicitly asking to interpret b as a char we
told the compiler “the bits `01001000` should be decoded as character” which is H.
This is called casting a variable into a different type.

_Advanced Note : in reality, in C# there can be more complex systems involved in
casting, where some conversion process will happen. But in “closer to the
machine” language like C, this cast will be exactly that : just changing how the
computer understands those 8 bits. Same memory address, same bits stored there,
just a difference of how we read them._

Following the same logic we can encode more complexe things by using more bytes:

- Bigger characters sets with lots of different grapheme can use 2 bytes (16 bits)
  or more to encode their character
- Integer types (int type in C#) will use 4 bytes (32 bits) to encode value from
  `-2,147,483,648` to `2,147,483,647`
- Floating point types (float type in C#) use a complex encoding scheme to encode
  in 4 bytes values with a decimal part.

**The main point to keep in mind for the saving system is that everything in your
computer are just bundles of bits (usually grouped in 8 as bytes)** and that the
same bits can express multiple things and it’s all a matter of how we interpret
them, which is the responsibility of us, the programmer, to tell the computer
how to interpret those through variable types.

## Entry Point script

The EntryPoint script is where we will reference everything we want to access
from anywhere in  our code. It is the “entry point” to our data.

We create a prefab that is just a GameObject with that script on it and set all
its references in the Inspector to all the assets we need (in our sample that is
our item database, our character rig prefab, the UI prefab and the Loading
fading panel prefab). We call it Main and place it in a folder named Resources.

That means we can instantiate it with a `Resource.Load` as we know its path
(Always just “Main”). In the final build we can put our EntryPoint gameobject in
the very first scene loaded and set it as `DontDestroyOnLoad`, but during
development we want to be able to start directly from any scenes to test fast.
So we can either :

- Manually add it to every scene (error prone, tedious)
- Test on access (the static Instance property in the EntryPoint.cs code) if it
  exists and if it doesn’t exist, `Resources.Load` the prefab to create it.
  As this is a test that is done on every access to the EntryPoint, this can add
  a tiny performance cost, so this is only done in the editor. Since in a build
  it will be created by the Initialization code that runs every time we start the
  game, we can be sure that the Instance will always exist.

The code in `EntryPoint.cs` in `Scripts/Gameplay` shows how this is done by using
the preprocessor directive.

```csharp
public static EntryPoint Instance
{
   get
   {
#if UNITY_EDITOR
       //we only test in editor, as this would be a check every access which is useless performance hit and in a build
       //this will be created by the startup sequence and set as DontDestroyOnLoad all through the game, so we are sure it's there
       if (s_Instance == null) Create();
#endif
       return s_Instance;
   }
}
```

# To Go Further

In this section, we will explore some alternative ways of doing some things from
that sample, or give possible areas where you can expand on the sample to adapt
it to different types or bigger applications.

## Key-value save format

Our sample uses a packed binary format to save. We just save data back to back
with no information to what each byte refers to. This makes the save small and
very fast to read back, but does require you to keep track of what you save and
in what order to read them in the same order. Adding new data and forgetting to
read them will create errors as things will be out of order.

Instead another format to use is a key-value format, where you write a key
before each value you save. Instead of reading value in order, you read the
whole save file and recreate the key-value pairing into a queryable structure
(e.g. a Dictionary), then each object can just query the right values they need.

e.g. The player will retrieve value for `player_position` and
`player_rotation`. Since the system query the data by key, adding new key-data
pair won’t break older systems.

The tradeoff is a bigger save file (as there is a key for every value), slower
loading time (you need to rebuild the lookup table entirely in memory before
loading anything) and usually memory usage (we can’t just read value from the
save file and apply them, there is usually a lookup table, and sometime the
values themselves, that need to be stored in memory temporarily)

One popular format to do this is the **JSON format**. This is a text format, meaning
it will be bigger and slower to parse than a binary format, and easier to
“cheat” as someone can just open the file and change value easily. But it does
make retrieving data easier, making it easier to support older saves in newer
versions and reading data in general.

## Automatic Unique ID/Name

For simplicity, the sample project relies on the developers to fill in the
Inspector the unique name used as key by Item in the database/save file (see for
example the FirstKey item in the Data folder).

This is of course error prone : two items could end up with the same “unique” id.
This could be fixed either by writing an editor script that would be run regularly
(or through automated testing) that will crawl through the databases/scenes to
check for duplicates, or by writing an editor script system that auto assigns
unique value for new items created.

## Automatic Serialization

In our sample, data is manually written & read from the save file.

Another more advanced solution is to use a way to tag which member to save.
Unity already has such a builtin system through its Serialization: everything
public or tagged as `SerializedField` gets serialized by the serialization system,
and there is a class called `JsonUtility` that will serialize to JSON (see [Key-value save format](#key-value-save-format)).
C# also have auto serialization through the `BinaryFormatter` class.

This however comes at the cost of longer development of the save system, a lower
control by default on how and what to save and some parts will still require
some custom code (e.g. saving items through their lookup id in a database
instead of saving the content of each item).

Note that with this solution, you usually still need to write some manual
verification functions, as deserializing without safety control of the data in a
save file can lead in rare cases to the introduction of vulnerabilities as
malicious code can be run that way from a modified save file.

## Extended Flags and additional data

Our save system has a simple flag system, which defines just whether a flag is
set or not. But you can extend on the system if your game needs more
information. You could have a dictionary of key to int to save global counts of
things, like the number of hidden collectibles the player has found.

As good practice though, this may be better to store in a CollectibleSystem that
will keep track of everything collectible related, and the save system will just
notify that CollectibleSystem that a save or a load happen to it can write
through the BinaryWriter to the save file or update its internal data
accordingly from it like our LevelSystem or PlayerSystem does it. Identically,
if your player has a storage chest where they can place items, having a
StorageSystem that handles all that.

Having everything as a separate piece also helps you handle loading/saving in a
more manageable chunk : it’s easier to check you read the same thing you write
and in the same order if the 2 functions are smaller and well contained in their
own system.

_(and as an extension of that, our flag system could of course be moved to a
WorldStateSystem or something similar that will handle a more complex world
state, this was placed in the SaveSystem in that sample for the sake of brevity
and simplicity)_

## Safe Saving Process

In our sample project, we open the file stream on the selected save file then
write our save data in it then close it at the end.

This can lead to the corruption of a save file : if the game crashes, is forced
closed or any other error making it exit early, it can lead to a left open, half
written save file. In our example the possibilities of that are slim (the save
process is extremely fast as we write only a couple of bytes) but it is not
completely nonexistent. And in a bigger game that can take a couple of seconds
to write huge amounts of data, then the risk is even more important.

One possible solution for this is to use a temporary save file. If the save
process unexpectedly exits as the file is written, then only the temporary file
is corrupted, but the save file you tried to override will be left fine with its
previous content.

Once the save file is written and closed, then you can rename and delete the
proper files to finish the save process. In short the process will be :

- User try to override `saveFile1`
- Create a temporary file `tempSave` Write your game save in it and close it.
- Rename `saveFile1` into `saveFile1_backup`
- Rename `tempSave` into `saveFile1`
- Delete `saveFile1_backup`.

If the next time the game starts (or you try to load a save) there is :

- a `tempSave` file present, that means the last saving process failed, either
display an error or silently delete that broken file.
- If a file with a `_backup` in its name exists but not the corresponding non
backup version (e.g. `save1_backup` is present but not `save1`) that means
something failed at the very end, so you can rename `save1_backup` into `save1`,
preserving at least the previous state.
