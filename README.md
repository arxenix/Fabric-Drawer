# Fabric Drawer
![CurseForge](https://cf.way2muchnoise.eu/fabric-drawer.svg)
![Discord](https://img.shields.io/discord/219787567262859264?color=blue&label=Discord)
![Bintray](https://api.bintray.com/packages/natanfudge/libs/fabric-drawer/images/download.svg) 
![Latest Commit](https://img.shields.io/github/last-commit/natanfudge/fabric-drawer)

Drawer is a Fabric library mod for Kotlin mods that allows you to easily save data to and load from NBT and PacketByteBuf using kotlinx.serialization.

## Gradle

Add `jcenter()` to repositories if you haven't yet:
```groovy
repositories {
    // [...]
    jcenter()
}
```
And add to dependencies:
```groovy
dependencies {
    // [...]
    modImplementation("com.lettuce.fudge:fabric-drawer:1.0.27")
}
```
Add the kotlinx.serialization gradle plugin:
```groovy
plugins {
    // [...]
    id("kotlinx-serialization")
}
```
Since "the serialization plugin is not published to Gradle plugin portal yet" , you'll need to add plugin resolution rules to your settings.gradle:
```groovy
pluginManagement {
    resolutionStrategy {
        eachPlugin {
            if (requested.id.id == "kotlinx-serialization") {
                useModule("org.jetbrains.kotlin:kotlin-serialization:${kotlin_version}") // set kotlin_version to 1.3.40 for example in gradle.properties
            }
        }
    }
}
```

It's recommended you update the IDEA Kotlin plugin to 1.3.50 by going to `Tools -> Kotlin -> Configure Kotlin Plugin Updates`
 because it provides special syntax highlighting for kotlinx.serialization.

## Usage

Annotate any class with `@Serializable` to make it serializable. **Make sure that every property has a usable default value when storing data for a block entity.** More information on this farther down.
```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class BlockInfo(var timesClicked : Int = 0, val placementTime : Long = 0, val firstToClick : String? = null)
```

Then you can serialize it back and forth.
#### In a block entity
```kotlin
fun fillData(){
    myInfo = BlockInfo(timesClicked = 7, placementTime = 1337, firstToClick = "fudge")
}
// Or make myInfo lateinit if initializing it at first placement is guaranteed
var myInfo : BlockInfo = BlockInfo()
    private set

override fun toTag(tag: CompoundTag): CompoundTag {
    // Serialize
    BlockInfo.serializer().put(myInfo, inTag = tag)
    return super.toTag(tag)
}

override fun fromTag(tag: CompoundTag) {
    super.fromTag(tag)
    // Deserialize
    myInfo = BlockInfo.serializer().getFrom(tag)
}
```

#### In a packet

```kotlin
val data = BlockInfo(timesClicked = 0, placementTime = 420, firstToClick = null)

val packetData = PacketByteBuf(Unpooled.buffer())
// Serialize
BlockInfo.serializer().write(data, toBuf = packetData)

    
for (player in PlayerStream.all(world.server)) {
    ServerSidePacketRegistry.INSTANCE.sendToPlayer(player, "packetId", packetData)
}
```

```kotlin
ClientSidePacketRegistry.INSTANCE.register(Identifier("modId", "packetId")){ context, buf ->
    // Deserialize
    val data = BlockInfo.serializer().readFrom(buf)
}
```

An example mod can be seen [here](https://github.com/natanfudge/fabric-drawer-example).

#### Putting two objects of the same type in one CompoundTag
 If you are putting two objects of the same type in one CompoundTag you need to specify a unique key for each one. (Note: You don't need to do this with a `PacketByteBuf`.)
 For example:
```kotlin
val myInfo1 = BlockInfo(timesClicked = 7, placementTime = 1337, firstToClick = "fudge")
val myInfo2 = BlockInfo(timesClicked = 3, placementTime = 9999, firstToClick = "you")
override fun toTag(tag: CompoundTag) : CompoundTag {
    BlockInfo.serializer().put(myInfo1, inTag = tag, key = "myInfo1")
    BlockInfo.serializer().put(myInfo1, inTag = tag, key = "myInfo2")
}

override fun fromTag(tag : CompoundTag){
    myInfo1 = BlockInfo.serializer.getFrom(tag, key = "myInfo1")
    myInfo2 = BlockInfo.serializer.getFrom(tag, key = "myInfo2")
}
```
 
This is only true for when YOU are putting 2 instances of the same type. If a class has multiple of the same type that's OK.
```kotlin
// No need for a key
data class MyData(val int1: Int, val int2: Int)
fun toTag(tag : CompoundTag){
    MyData.serializer().put(MyData(1,2))
}
```

```kotlin
// Need a key
data class MyData(val int1: Int, val int2: Int)
fun toTag(tag : CompoundTag){
    MyData.serializer().put(MyData(1,2), key = "first")
    MyData.serializer().put(MyData(3,4), key = "second")
}
```

### Serializing Java and Mojang objects
You can serialize any primitive, and any list of primitives, and any class of your own that is annotated with `@Serializable`, without any extra modification:
```kotlin
// OK
@Serializable
data class MyData(val str : String, val list : List<Double>)
@Serializable
data class Nested(val myData : MyData, val c : Char)
```
However, if you try to put in a `UUID` or a `BlockPos`, for example:
```kotlin
// Error!
@Serializable
data class MyPlayer(val id : UUID)
```

To fix this, put at the very top of the file:
```kotlin
@file:UseSerializers(ForUuid::class, ForBlockPos::class)
```

Or, [because of a bug you should thumbs up the issue on](https://github.com/Kotlin/kotlinx.serialization/issues/533), specifically when using a `DefaultedList<>` you need to annotate every property that uses `DefaultedList<>` like this:
```kotlin
@Serializable
data class MyPlayerInventory(
    @Serializable(with = ForDefaultedList::class) val list1 : DefaultedList<ItemStack>,
    @Serializable(with = ForDefaultedList::class) val list2 : DefaultedList<Ingredient>
)
```

Serializers for the following classes are available:
- UUID
- BlockPos
- Identifier
- All NBT classes
- ItemStack (note: requires being in a Minecraft context as it accesses the registry)
- Ingredient (note: requires being in a Minecraft context as it accesses the registry)
- DefaultedList<> (note: bug requires special syntax, see above)

Appropriate extension methods of the form `CompoundTag#putFoo` / `CompoundTag#getFoo`, `PacketByteBuf#writeFoo` / `PacketByteBuf#readFoo` are also available for the mentioned classes when they are missing from the vanilla API.

If I've missed anything you need please [open an issue](https://github.com/natanfudge/Fabric-Drawer/issues/new).

You can also add your own serializers and more using the kotlinx.serialization API. For more information, [see the README](https://github.com/Kotlin/kotlinx.serialization/blob/master/README.md). 

### Why does every property need to have a default value when storing data for a block entity?
There are 2 main reasons:
1. Nbt data is volatile. It can change at any time, via modifying the save file, or by using the `/data` command.
This means you can never trust the information provided to by the NBT to be valid, or the server might crash endlessly on startup trying to deserialize non-existent nbt data.
Having a default value avoids this problem by simply using those default values when the data is invalid.
2. Sometimes you only want to store data on the server, so you don't use `BlockEntityClientSerializable`.
Your data will (usually, see point 1.) be restored on the server just fine.  
However, Minecraft will also call `fromTag` on the client, in an attempt to sync the data to him as well.
You don't send him any of the nbt data required to load your `@Serializable` classes, so if there are no default values, it will simply crash.

Make sure that the default values are **usable**, meaning trying to use them in your mod will never crash!

### Polymorphic serialization
- Read [this](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/polymorphism.md) first. 
- In order to do this in drawer you need to add the `SerialModule` instance whenever you serialize / deserialize using that module. 
If this is cumbersome a simple extension method on `KSerialize<T>` can be used that automatically inserts your module.

### Tips
- To avoid boilerplate it's recommended to add a `putIn()` / `writeTo()` function to your serializable classes, for example:
```kotlin
@Serializable
data class MyData(val x :Int, val y : String){
    fun putIn(tag : CompoundTag) = MyData.serializer().put(this,tag)
}
//Usage:
fun toTag(tag :CompoundTag){
    val data = MyData(1,"hello")
    tag.putIn(tag) // Instead of MyData.serializer().put(data,tag)
}
```

Please thumbs-up [this issue](https://github.com/Kotlin/kotlinx.serialization/issues/329) so we can have this syntax built-in to the library for all serializable classes! Having a common interface for serializable classes would also enable avoiding boilerplate in other places.

- Serializable classes are also serializable to [Json](https://github.com/Kotlin/kotlinx.serialization/blob/master/README.md), and any other format that kotlinx.serialization and its addons support. 

### Troubleshooting
- It's saying `serializer()` is undefined!
Refresh gradle (ReImport all Gradle projects button)

### Closing notes
You are looking at the first revision of this library and its readme. 
As things usually go, problems will likely arise, in which case I urge you to open an issue or contact me via [discord](https://discord.gg/CFaCu97).
