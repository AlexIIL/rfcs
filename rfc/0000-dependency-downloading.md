## Summary

This RFC describes how downloadable jars and assets should be specified in a
quilt.mod.json file, and how quilt loader will download and store those
downloadable files.


## Motivation

One of the most common barriers of entry for new mod users is having to download
and install APIs and libraries in addition to the mods they want to use. Adding a
way for Quilt Loader to fetch dependencies dynamically will help ease this headache
and make Quilt a more appealing option for new mod users.

Some mods contain large assets, for example video or audio files - and these files
rarely change between releases of a mod. Being able to download these separately
from a mod jar would make the mod file much smaller, and make downloading updates
much quicker than redownloading the whole file.



## Explanation

### Quilt.mod.json specification

The quilt.mod.json spec will need to be changed to add these new fields.

- [quilt_loader](https://github.com/QuiltMC/rfcs/blob/master/specification/0002-quilt.mod.json.md#the-quilt_loader-field) - The existing "quilt_loader" field.
    - [downloadable_jars](#the-downloadable-jars-field) - All jar files that may be used for dependency downloading
    - [downloadable_assets](#the-downloadable_assets-fields) - All assets that should be loaded before the mod can be used.

#### The `downloadable_jars` field

| Type   | Required |
|--------|----------|
| Array  | False    |

An array of [Downloadable Jar Objects](#downloadable-jar-objects).

#### The `downloadable_assets` field

| Type   | Required |
|--------|----------|
| Array  | False    |

An array of [Downloadable Asset Objects](#downloadable-asset-objects) describing assets that should all be downloaded before launching the mod.

#### Downloadable Jar Objects

This describes a single jar file that may be downloaded to satisfy any dependencies declared in the [depends](https://github.com/QuiltMC/rfcs/blob/master/specification/0002-quilt.mod.json.md#the-depends-field) field. (Or any other requirements that other plugins may add).

Mod installers and packages *may* wish to pre-download all jar files specified here to the central cache folder (or specify it's own folder that quilt can read jars from). However these should not be added directly to the users mods folder - quilt loader should have control over how these get used.

* [name](#the-name-field) - The name of the jar file.
* [size](#the-size-field) - The size of that jar file, in bytes.
* [hashes](#the-hashes-field) - The hashes of that jar file, to ensure we download the correct file.
* [metadata](#the-metadata-field) - A stripped down version of the quilt.mod.json file in that jar.

##### The `name` field

| Type   | Required |
|--------|----------|
| String | True     |

This is the full name of the jar file.

##### The `size` field

| Type    | Required |
|---------|----------|
| Integer | True     |

This is the raw size of the jar file, in bytes.

##### The `hashes` field

| Type   | Required |
|--------|----------|
| Object | True     |

This contains all hashes of that jar file. All hashes that quilt loader recognises must match the hash of the downloaded jar file. At least one of the hash types must be recognised in order to download and verify the file. 

In addition, one of the hashes must be:

* Considered to be a "cryptographic hash function"
* Not considered to be "broken"
* Present in the `MessageDigest Algorithms` section of the `Java Security Standard Algorithm Names` specification (JE 16's spec is here: https://docs.oracle.com/en/java/javase/16/docs/specs/security/standard-names.html#messagedigest-algorithms).

(This rules out `MD2`, `MD5`, and `SHA-1` from that document, leaving only variants of SHA 2 and SHA 3).

It is recommened that mods use the latest hash version supported by the JVM they they depend on (so `SHA3` based algorithms for java 9 and up, `SHA2` algorithms for java 8).

It's not recommened for mods to include multiple hashes of the same type - for example including `SHA3-224` and `SHA3-256` is considered to be useless.

##### The `metadata` field

| Type   | Required |
|--------|----------|
| Object | True     |

This contains a subset of the main quilt.mod.json specifiection, which is intended to be used to list all required dependencies before they are downloaded. (However omitting parts of this object isn't a problem - it just means the user may be asked to download files multiple times).

All of the fields have the same meaning as declared in the main specification. If a field is present then it must be the same as the same field in the downloadable file.

* `schema_version` - required.
* `quilt_loader` - required.
    * `group` - required
    * `id` - required.
    * `provides` - optional.
    * `version` - required.
    * `depends` - optional.
    * `breaks` - optional.
    * `repositories` - optional.
    * `metadata` - optional. (All fields under metadata are optional, except for `icon`).
* `minecraft` - optional
    * `environment` - optional.

If any of the following fields are present then it is considered an error. (These fields should be stripped when copying the quilt.mod.json from the original file).

* under `quilt_loader`:
    * `entrypoints`
    * `plugins`
    * `jars`
    * `language_adapters`
    * `load_type`
    * under `metadata`:
        * `icon`. (Since these icons would be in the missing jar file it's not possible to use this).
    * `downloadable_jars`: Downloadable jars are expected to be collapsed down into the main array, and duplicates removed.
* `mixin`
* `access_widener`


Additional fields may be specified for plugins to use.

#### Downloadable Asset Objects

Assets have very similar structure to jar files, with the exception of the metadata field (which is missing).

* [name](#the-name-field) - The name of the asset file.
* [size](#the-size-field) - The size of that asset file, in bytes.
* [hashes](#the-hashes-field) - The hashes of that asset file, to ensure we download the correct file.

### Cache folders

By default quilt will download jars and assets to a single central cache folder. This will be platform-specific, but include a folder `quilt_loader` somewhere.

The location of this can be changed via the system property "quilt.cache_folder".

Additional cache folders that quilt can read (but won't write to) can be specified with the system property "quilt."





## Drawbacks

Why should we not do this?


## Rationale and Alternatives

- Why is this the best possible design?
- What other designs are possible and why should we choose this one instead?
- What other designs have benefits over this one? Why should we choose an
  alternative instead?
- What is the impact of not doing this?


## Prior Art

If this has been done before by some other project, explain it here. This could
be positive or negative. Discuss what worked well for them and what didn't.

There may not always be prior art, and that's fine.


## Unresolved Questions

- What should be resolved before this RFC gets merged?
- What should be resolved while implementing this RFC?
- What unresolved questions do you consider out of scope for this RFC, that
  could be addressed in the future?
