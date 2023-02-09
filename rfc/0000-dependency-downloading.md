## Summary

This RFC describes how downloadable jars should be specified in a
quilt.mod.json file, and how quilt loader will download and store those
downloadable files.


## Motivation

One of the most common barriers of entry for new mod users is having to download
and install APIs and libraries in addition to the mods they want to use. Adding a
way for Quilt Loader to fetch dependencies dynamically will help ease this headache
and make Quilt a more appealing option for new mod users.

This should also help reduce the 



## Explanation

### Quilt.mod.json specification

The quilt.mod.json spec will need to be changed to add these new fields.

- [quilt_loader](https://github.com/QuiltMC/rfcs/blob/master/specification/0002-quilt.mod.json.md#the-quilt_loader-field) - The existing "quilt_loader" field.
    - [downloadable_group](#the-downloadable-group-field)
    - [downloadable_jars](#the-downloadable-jars-field) - All jar files that may be used for dependency downloading
    - [downloadable_assets](#the-downloadable_assets-fields) - All assets that should be loaded before the mod can be used.

#### The `downloadable_group` field

| Type   | Required |
|--------|----------|
| String | False    |

This indicates that this mod should only be specified as downloadable if all other mods in the same group are also specified as downloadable.

This is useful for library sub-mods which depend on an exact version of a common "core" library mods, for example QSL or LibBlockAttributes.

For a more concrete example, lets use LibBlockAttributes, which has 3 modules:

* "libblockattributes_core"
* "libblockattributes_items"
* "libblockattributes_fluids"

#### The `downloadable_jars` field

| Type   | Required |
|--------|----------|
| Array or String  | False    |

An array of Strings, where each string references a json file (contained in this mod) which either contains [Downloadable Jar Objects](#downloadable-jar-objects) directly, or an array of downloadable jar objects.

#### Downloadable Jar Objects

This describes a single jar file that may be downloaded to satisfy any dependencies declared in the [depends](https://github.com/QuiltMC/rfcs/blob/master/specification/0002-quilt.mod.json.md#the-depends-field) field. (Or any other requirements that other plugins may add).

Mod installers and packages *may* wish to pre-download all jar files specified here to the central cache folder (or specify it's own folder that quilt can read jars from). However these should not be added directly to the users mods folder - quilt loader should have control over how these get used.

* [name](#the-name-field) - The name of the jar file.
* [size](#the-size-field) - The size of that jar file, in bytes.
* [hashes](#the-hashes-field) - The hashes of that jar file, to ensure we download the correct file.
* [folder](#the-folder-field) - A folder (in this jar file) containing metadata for the file

// How do we specify any additional metadata required for downloading?
// Like, maven needs group, name, possibly others?
// I'm sure curseforge would like mods to include their project ID and file ID

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

This contains all hashes of that jar file, mapped from hash name to base64 encoded hash value. All hashes that quilt loader recognises must match the hash of the downloaded jar file. At least one of the hash types must be recognised in order to download and verify the file.

In addition, one of the hashes must be:

* Not considered broken.
* Present in the `MessageDigest Algorithms` section of the `Java Security Standard Algorithm Names` specification (JE 16's spec is here: https://docs.oracle.com/en/java/javase/16/docs/specs/security/standard-names.html#messagedigest-algorithms).

(This rules out `MD2`, `MD5`, and `SHA-1` from that document, leaving only variants of SHA 2 and SHA 3. Note that this doesn't disallow mods from including hashes for any of these algorithms, just that they aren't sufficient. They will still be checked for validity though).

It is recommended that mods use the latest hash version supported by the JVM they they depend on (so `SHA3` based algorithms for java 9 and up, `SHA2` algorithms for java 8).

It's not recommended for mods to include multiple hashes of the same type - for example including `SHA3-224` and `SHA3-256` is considered to be useless. (TODO: check this)

Including multiple different hashes is probably a good idea, but we'll need to look into whether this really matters later on.

##### The `folder` field

| Type   | Required |
|--------|----------|
| String | True     |

This points to a folder in this jar file, which contains all metadata needed by that file to perform dependency downloading. This folder must contain a `quilt.mod.json` file directly, which contains the metadata for that mod. Mods loaded by custom loader plugins, and fabric mods, are not permitted to be automatically downloaded.
It may optionally contain other files useful when downloading dependencies, such as icon files or (partial) language files.
These extra files do not need to match their real counterparts.

The `quilt.mod.json` file should not be a copy of the original file - instead some fields should be removed, and the `downloadable_jars` should be pre-processed:

All of the fields have the same meaning as declared in the main specification, and must be the same as in the real downloadable file.

* `schema_version`
* `quilt_loader`
    * `group`
    * `id`
    * `provides`
    * `version`
    * `depends`
    * `breaks`
    * `load_type`
    * `repositories` - optional.
    * `metadata` - optional. (All fields under metadata are optional).
* `minecraft`
    * `environment`

If any of the following fields are present then it is considered an error. (These fields should be stripped when copying the quilt.mod.json from the original file).

* under `quilt_loader`:
    * `entrypoints`
    * `plugins`
    * `jars`
    * `language_adapters`
    * `downloadable_jars`: Downloadable jars are expected to be collapsed down into the main array, and duplicates removed.
* `mixin`
* `access_widener`


Additional fields may be specified for plugins to use.

### Cache folders

By default quilt will download jars and assets to a single central cache folder. This will be platform-specific, but include a folder called `quilt_loader` somewhere.

The location of this can be changed via the system property `quilt.cache_folder`, or via a config file, stored directly in the minecraft folder.

Additional cache folders that quilt can read (but won't write to) can be specified with the system property `quilt.cache_read_folder`. (TODO: How can we specify multiple additional folders? Do we need multiple additional folders?)

All cache folders will use the following layout:

* `README.txt`: This describes the layout, what this is used for, and a note that any of the folders or files can be deleted safely without preventing the current mod set from being played.
* `jars`
    * `<group-id>`
        * `<mod-id>`
            * `file-name.jar`: This is identical to value in the `name` field.

### Behaviour

During each loader plugin cycle:

* For each selected tentative mod (I.E. one that isn't present in a mods folder, but is found in the `downloadable_mods` list) it will be added to a download queue.

If the download queue isn't empty (and hasn't been disabled or always-enabled) then it will be shown to the user. (This should show information about each mod to download, and the total size of what needs to be downloaded).

If downloading is disabled then the next step is skipped, and instead a "missing mods" error message is displayed to the user.

The download queue will have the following options:

* `Cancel download`: This will disallow anything else from being added to the download queue in this launch of quilt-loader, and result in the error message as mentioned above.
* `Never download`: This will change a config option in this instance preventing the download queue from being used in that instance, and the same behaviour as above.
* `Download all`: This will proceed to download everything according to the next step.
* `Always download`: proceeds to the next step, and also prevents this download queue from appearing again in this instance.

Mods will be downloaded from `https://maven.quiltmc.org`, and any repositories specified in the `repositories` field in `quilt_loader`. (At first only quiltmc.org itself and whitelisted mavens will be permitted, but later on this will be changed to a blacklist where only certain mavens will be disallowed (for example maven central).

//...no
Mods are downloaded using the following URL format: `<repository>/<group-id>/<mod-id>/<version>/<file-name>`.

After downloading mods to the cache (or if they are already present in the cache, but not found in the `<game-dir>/.quilt/downloaded_mods` folder) then they will be copied over to a local (per-instance) download cache in `<game>/.quilt/downloaded_mods`, and then loaded for the next cycle.

This `downloaded_mods` folder will use the same file layout as a normal cache folder.

----------------------

ALEXIIL: EVERYTHING ABOVE THIS LINE IS FINISHED (in theory - apart from proofreading)

----------------------


## Drawbacks

Why should we not do this?

The main drawback is that downloading just the mod file is not enough to play


## Rationale and Alternatives

Maybe assets should only be identified by their hash?

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
