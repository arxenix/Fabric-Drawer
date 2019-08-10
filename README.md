# Fabric Drawer

Drawer is a Fabric library mod for Kotlin mods that allows you to easily save data to NBT and PacketByteBuf using kotlinx.serialization.

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
    modImplementation("com.lettuce.fudge:fabric-drawer:1.0.7")
}
```
Add the kotlinx.serialization gradle plugin:
```groovy
plugins {
    // [...]
    id("kotlinx.serialization")
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

It's recommended you update the IDEA Kotlin plugin to 1.50 by going to `Tools -> Kotlin -> Configure Kotlin Plugin Updates`
 because it provides special syntax highlighting for kotlinx.serialization.

## Usage

Annotate any class with `@Serializable` to make it serializable:
```kotlin
@Serializable
data class BlockInfo(var timesClicked : Int, val timeOfPlacement : Long, val nameOfFirstPersonClicked : String?)
```

Then you can serialize it back and forth.
#### In a block entity
```kotlin
fun fillData(){
    myInfo = BlockInfo(timesClicked = 7, timeOfPlacement = 1337, nameOfFirstPersonClicked = "fudge")
}
// Or make myInfo nullable without lateinit if initializing it at first placement is not guaranteed
lateinit var myInfo : BlockInfo
    private set
fun toTag(tag: CompoundTag){
    // Serialize
    BlockInfo.serializer().put(myInfo, inTag = tag)
}

fun fromTag(tag : CompoundTag){
    // Deserialize
    myInfo = BlockInfo.serializer.getFrom(tag)
}
```

#### In a packet

```kotlin
val data = BlockInfo(timesClicked = 0, timeOfPlacement = 420, nameOfFirstPersonClicked = null)

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

#### Putting two objects of the same type in one CompoundTag
 If you are putting two objects of the same type in one CompoundTag you need to specify a unique key for each one. (Note: You don't need to do this with a `PacketByteBuf`.)
 For example:
```kotlin
val myInfo1 = BlockInfo(timesClicked = 7, timeOfPlacement = 1337, nameOfFirstPersonClicked = "fudge")
val myInfo2 = BlockInfo(timesClicked = 3, timeOfPlacement = 9999, nameOfFirstPersonClicked = "you")
fun toTag(tag: CompoundTag){
    BlockInfo.serializer().put(myInfo1, inTag = tag, key = "myInfo1")
    BlockInfo.serializer().put(myInfo1, inTag = tag, key = "myInfo2")
}

fun fromTag(tag : CompoundTag){
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
data class myPlayer(val id : UUID)
```

To fix this, put at the very top of the file:
```kotlin
@file:UseSerializers(Serializers.Uuid::class, Serializers.BlockPos::class)
```

Serializers for the following classes are available:
- UUID
- BlockPos
- Text
- ItemStack
- DefaultedList

If I've missed anything you need please [open an issue](https://github.com/natanfudge/Fabric-Drawer/issues/new).

### Closing notes
You are looking at the first revision of this library and its readme. 
As things usually go, problems will likely arise, in which case I urge you to open an issue or contact me via [discord](https://discord.gg/CFaCu97).