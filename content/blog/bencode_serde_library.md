+++
title = "Kotlin learning with a Bencode serde library"
date = 2026-07-26
taxonomies.tags = [
  "bencode", 
  "kotlin"
]
+++

I will showcase a simple serde library for **bencode** I did in order to finally pick up on Kotlin and hopefully learn some of it.

Bencode is a binary encoding format used in the BitTorrent protocol, it's the format used for .torrent files and also the 
encoding used on top of HTTP trackers. You can make sense of the entire encoding in 10 minutes by reading its Wikipedia [page](https://en.wikipedia.org/wiki/Bencode).

The API for the library ended up like this, in this example showcasing how to deserialize a torrent file:
```kotlin
package models

import serde.annotations.Bencode
import serde.deserialize

data class Torrent(
    var announce: String,
    var info: TorrentMetaInfo,
    @Bencode(name = "created by") var createdBy: String?,
    @Bencode(name = "creation date") var creationDate: Long?,
    var comment: String?,
    @Bencode(name = "announce-list") var announceList: List<List<String>>?
) {
    data class TorrentMetaInfo(
        val name: String,
        val length: Long,
        @Bencode(name = "piece length") val pieceLength: Long,
        val pieces: ByteArray,
   )
    
    companion object {
        fun fromFile(path: String): Torrent {
            return File(path).inputStream().use { input ->
                deserialize<Torrent>(PushbackInputStream(input))
            }
        }
    }
}
```
First personal note on Kotlin is the expressiveness of the `?` operator. A simple character that lets the reader very
quickly know what's going on and solves the billion dollar mistake we still have to contend with in Java, Python and Go.
Additionally, the other related operators `.?, :?, !!, !` and functional features make this language a breeze to work with. It feels very similar to Rust,
without the borrow checking hassle.

---

# Decoding

Decoding bencode is very straightforward, you read a stream of bytes and emit a value, it resembles a
recursive parser. I chose to map the bencode value types to Kotlin's `Long`, `ByteArray`, `List` and `Map`.

```kotlin
// Decoder.kt
internal fun decode(reader: PushbackInputStream): Any {
    val readValue = reader.read()
    return when (readValue) {
        'i'.code -> { decodeInteger(reader) }
        'd'.code -> { decodeDict(reader) }
        'l'.code -> { decodeList(reader)}
        in '0'.code..'9'.code -> {
            reader.unread(readValue)
            decodeString(reader)
        }
        else -> {
            error("invalid bencode")
        }
    }
}

private fun decodeInteger(reader: PushbackInputStream): Long {
    var sum: Long = 0
    var isNegative = false

    var readValue = reader.read()
    if (readValue == '-'.code) {
        isNegative = true
        readValue = reader.read()
    }

    while (readValue != 'e'.code) {
        if (readValue  !in '0'.code..'9'.code) {
            error("invalid bencode")
        }
        sum = sum * 10 + readValue - '0'.code
        readValue = reader.read()
    }
    return if (isNegative) -sum else sum
}

private fun decodeString(reader: PushbackInputStream): ByteArray {
    var length: Long = 0;
    var readValue = reader.read()
    while (readValue != ':'.code) {
        if (readValue  !in '0'.code..'9'.code) {
            error("invalid bencode")
        }
        length = length * 10 + readValue - '0'.code
        readValue = reader.read()
    }
    val bytes = ByteArray(length.toInt())
    reader.read(bytes)
    return bytes
}

private fun decodeList(reader: PushbackInputStream): List<Any> {
    val list = mutableListOf<Any>()
    var readValue = reader.read()
    while (readValue != 'e'.code) {
        reader.unread(readValue)
        list.addLast(decode(reader))
        readValue = reader.read()
    }
    return list
}

private fun decodeDict(reader: PushbackInputStream): Map<String, Any> {
    val map = mutableMapOf<String, Any>()
    var readValue = reader.read()
    while (readValue != 'e'.code) {
        reader.unread(readValue)
        val key = String(decodeString(reader), Charsets.UTF_8)
        val value = decode(reader)
        map[key] = value
        readValue = reader.read()
    }
    return map
}
```
The decoder is not doing any checks on the return values of `read`. We need to check for EOF or the program might crash or produce wrong values!
We will make use of another very convenient Kotlin feature called extensions:
```kotlin
// Decoder.kt
private fun PushbackInputStream.readOrThrow(): Int {
    val b = this.read()
    return if (b != -1) b else throw EOFException("unexpected end of stream")
}

private fun PushbackInputStream.readOrThrow(bytes: ByteArray): Int {
    val b = this.read(bytes)
    return if (b != -1) b else throw EOFException("unexpected end of stream")
}
```
We just need to swap our `read` for `readOrThrow` calls. Even though this is no different from a utility function it feels nicer to work with.

---

# Deserializing

Now we need to deserialize bencode dictionaries into Kotlin objects. Bencode dictionary keys allow special characters 
(like whitespaces), so we cannot infer our dictionary keys from class field names or parameters in all situations. Users need to be 
able to "talk" to the library on how it should serialize and deserialize their instances.

For this simple library I introduced an annotation that lets users specify which key will hold the field's value. Additionally 
it offers an `hide` flag, so users can compose their classes with more attributes that the library will just ignore (used in serialization). 
```kotlin
// Annotations.kt
package serde.annotations

@Target(AnnotationTarget.VALUE_PARAMETER, AnnotationTarget.FIELD, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Bencode(val name: String = "", val hide: Boolean = false)
```

The deserializer will use reflection to inspect the user defined class and start building an instance from the decoded 
dictionary:
```kotlin
// Deserializer.kt
inline fun <reified T: Any> deserialize(inputStream: InputStream): T {
    return deserialize(inputStream, T::class)
}

@PublishedApi
internal fun <T: Any> deserialize(inputStream: InputStream, klass: KClass<T>): T {
    return deserialize(decode(PushbackInputStream(inputStream)) as Map<String, *>, klass)
}
```
By inlining the deserialize function we can reify the generic user class type (because JVM erases generics type information at compile time) 
and we are allowed to use reflection on the generic class type in our library. 
The practical effect of this is that we can offer a smoother API for users, e.g, `deserialize<MyClass>(inputStream)` _vs_ 
 `deserialize<MyClass>(inputStream, MyClass::class)`. The `@PublishedApi` annotation is meant to hack around the fact that calling an internal function from an inlined function "should" not be legal. 

Finally, the juice:
```kotlin
// Deserializer.kt
private fun <T : Any> deserialize(obj: Map<String, *>, klass: KClass<T>) : T {
    val ctor = klass.primaryConstructor ?: error("No primary constructor")
    val args = mutableMapOf<KParameter, Any?>()
    for (param in ctor.parameters) {
        val key = keyFor(param)
        val raw = obj[key]
        when {
            raw != null -> args[param] = convert(raw, param.type)
            param.type.isMarkedNullable -> args[param] = null
            param.isOptional -> Unit
            else -> error("Missing required field \"$key\"")
        }
    }
    return ctor.callBy(args)
}

private fun keyFor(param: KParameter): String {
    val annotation = param.findAnnotation<Bencode>()
    return annotation?.name?.takeIf { it.isNotBlank() } ?: param.name!!
}
```
We will build the instance by inspecting the user class constructor parameters. We iterate each parameter, extract its key
(either from the annotation or parameter name) and lookup that value from the decoded dictionary. If the key is not found in the dictionary 
we analyze whether that parameter is nullable or optional, if it is we simply pass in `null` or `Unit` as arguments.

We only emit 4 types from our decoder, however, it would be non-sensical to throw an error when a user expects a 
String and we find a ByteArray. Our `convert` function will take care of binding our intermediate bencode types 
(Long, ByteArray, Map, List) into more Kotlin types!

A little snippet with some types conversion.
```kotlin
// Deserializer.kt
private fun convert(src: Any, target: KType): Any {
    val classifier = target.classifier as KClass<*>

    return when {
        classifier == String::class -> when (src) {
            is ByteArray -> String(src, Charsets.UTF_8)
            is String -> src // this is used for dictionary keys, not referenced in post
            else -> error("Unsupported type ${src::class}")
        }
        classifier == Int::class -> when (src) {
            is Long -> src.toInt()
            else -> error("Unsupported type ${src::class}")
        }
        classifier == Long::class -> when (src) {
            is Long -> src
            else -> error("Unsupported type ${src::class}")
        }
        ...
    }
}
```
We allow a parameter declared as String to be populated from a ByteArray (a bencode byte string). 
For Long (a bencode int) we allow Int and Long declared fields.

Now the interesting part... This also allows us to **compose objects**! We can deserialize an emmited Map into 
another object! If you remember the first snippet this is something very crucial to the library:
```kotlin
// Deserializer.kt
private fun convert(src: Any, target: KType): Any {
    val classifier = target.classifier as KClass<*>
    
    return when {
        ...
        // last branch 
        classifier.primaryConstructor != null -> when (src) {
            is Map<*, *> -> {
                deserializeErased(src, classifier)
            }
            else -> error("Unsupported type ${src::class}")
        }
        else -> error("Unsupported target")
    }
}
```

We can finally work with torrent files in Kotlin:
```kotlin
import models.Torrent

suspend fun main() {
    val torrent = Torrent.fromFile("~/ubuntu-25.10-desktop-amd64.iso.torrent")
    println(torrent.announce)
    println(torrent.info.name)
}
```

There is still one more interesting feature we can add. For tracker responses, we might get 
a failure, where the tracker sends only an error message on the bencode payload, whether in success it sends a list of
BitTorrent peers. We would like to use a sealed class for this (analogous to Enum variants in Rust):
```kotlin
sealed class TrackerResponse {

    data class Failure(
        val failureReason: String
    ) : TrackerResponse()

    data class Success(
        val peers: ByteArray,
        val interval: Int,
        val complete: Int,
        val incomplete: Int,
    ) : TrackerResponse()
    
    companion object {
        suspend fun fromHttpResponse(response: HttpResponse): TrackerResponse {
            return withContext(Dispatchers.IO) {
                deserialize<TrackerResponse>(response.content.toInputStream())
            }
        }
    }
}
```
With some adjustments we can support this in our deserializer:
```kotlin
private fun <T : Any> deserialize(obj: Map<String, *>, klass: KClass<T>) : T {
    // new deserialize logic to resolve the new target class type 
    val target = if (klass.isSealed) resolvedSealedSubclass(obj, klass) else klass
    val ctor = target.primaryConstructor ?: error("No primary constructor")
    ...
}
    
private fun <T: Any> resolvedSealedSubclass(obj: Map<String, *>, klass: KClass<T>) : KClass<out T> {
    val candidates = klass.sealedSubclasses.filter { sub ->
        val ctor = sub.primaryConstructor ?: return@filter false
        ctor.parameters.all { param ->
            param.type.isMarkedNullable || 
            param.isOptional || 
            obj.containsKey(keyFor(param))
        }
    }
    return when (candidates.size) {
        1 -> candidates.first()
        0 -> error("Matching subclass not found")
        else -> error("Multiple subclasses found")
    }
}
```
It's a a frail approach, it won't work if subclasses share a parameter name in their constructor declarations, 
but it suffices for our simple use case where both types have mutually exclusive parameter sets.

Serialization was left out of this article, but given it is implemented we are now ready to start torrenting!
```kotlin
import io.ktor.client.*
import io.ktor.client.engine.cio.CIO
import io.ktor.client.request.*

import models.Torrent
import models.TrackerResponse
import serde.serialize

suspend fun main() {
    val torrent = Torrent.fromFile("~/ubuntu-25.10-desktop-amd64.iso.torrent")
    // Not included in the post, but this will serialize the torrent info dictionary,
    // compute its SHA1 hash and URL encode it
    val infoHash = torrent.info.urlSafeEncodedHash()
    
    val httpClient = HttpClient(engineFactory=CIO)
    val rsp = httpClient.get(urlString = torrent.announce) {
        url {
            encodedParameters.append("info_hash", infoHash)
            parameters.append("peer_id", "12345678901234567890")
            parameters.append("port", "6881")
            parameters.append("downloaded", "0")
            parameters.append("uploaded", "0")
            parameters.append("left", torrent.info.length.toString())
            parameters.append("compact", "1")
            parameters.append("event", "started")
            parameters.append("numwant", "50")
        }
    }
    val trackerResponse = TrackerResponse.fromHttpResponse(rsp)
    when (trackerResponse) {
        is TrackerResponse.Success -> println(trackerResponse.peers)
        is TrackerResponse.Failure -> println(trackerResponse.failureReason)
    }
```
Last personal note, the way we can build objects in Kotlin is very aesthetic, look at that HTTP request. 

**Disclaimer**: This library is part of a toy project I am doing to learn Kotlin, it's full of shortcuts, ignores many
problems and is not intended for production use!