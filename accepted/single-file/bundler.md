# The Single-file Bundler

## Requirements

Ideally, the bundler should be:

* Able to embed any file required to run a .NET Core app -- managed assemblies, native binaries, configuration files, data files, etc.
* Able to generate cross-platform bundles (ex: publish a single-file for Linux target from Windows)
* Deterministic (generate the exact same single-file on multiple runs)
* Amenable to post-processing (ex: signing tools)

## Bundle Layout

| Bundle Layout (ver 2.0) |
| ------------------------------------------------------------ |
| **AppHost**<br />`(Bundle-Marker)`                               |
| **Embedded Files**<br />`app.dll` <br />`app.deps.json`<br />`app.runtimeconfig.json`<br />`dependency.managed.dll`<br />`...`<br /> |
| **Bundle Header** <br />`2.0` (Version #)<br /> `#Number of Embedded Files`<br />`Bundle-ID` <br />`Needs Extraction?`<br />`deps.json file location (offset, size)`, if any.<br />`runtimeconfig.json file location (size,offset)`, if any. |
| **Bundle Manifest**<br />For each bundled file:<br />   `Location (Offset, Size)`<br />   `Type:  IL, ReadyToRun, deps.json, runtimeconfig.json`, or `other` (extract) |

### Bundle Marker

Every `AppHost` has a static variable that identifies the location of the single-file header (if any).  By default, this value is zero, which indicates that the `AppHost` is not a bundle. The bundler tool rewrites this value with the location of the bundle header. 

Using a special marker for recognizing bundle-header (instead of simply writing the header at the end of the file) enables compatibility with other post-processing tools (such as `signtool`) which require their own content to be at the end of the file.

### Bundle Identifier

The bundle identifier is a *path-compatible* cryptographically strong name. 

* This identifier is used to distinguish bundles for different versions of the same app.
* This identifier is used as part of the bundle extraction mechanism as described in [this document](extract.md).
* A new bundle identifier is generated for each bundle transformation.  

## Bundle Transformation

Given a .NET Core app published with respect to a specific runtime, the bundler transforms the `AppHost` binary to a single-file bundle through the following actions:

* Append the managed app and its dependencies as a binary-blob at the end of the `AppHost` executable. This includes all files to be embedded located within the publish directory and any sub-directories. The path (relative to the bundle-root) of published files are noted in the bundle meta-data.
* Write meta-data headers and manifest that help locate the contents of the bundle. 
* Set the bundle-indicator in the `AppHost` to the offset of the bundle-header.

The bundler should generate the correct format of single-file bundles based on the target framework.

## Implementation

The bundling tool is implemented as a library in the [Microsoft.NET.HostModel](https://www.nuget.org/packages/Microsoft.NET.HostModel/) package in the [runtime](https://github.com/dotnet/runtime) repo. 

* The bundler is located alongside the host components, since their implementations are closely related.
* Separating the bundler (and other AppHost transformers in HostModel) implementation from SDK repo aligns with code ownership, and facilitates maintenance of the library. 
* The build targets/tasks that use the HostModel library are in the SDK repo. This facilitates the MSBuild tasks to be multi-targeted. It also helps generate localized error messages, since SDK repo has the localization infrastructure. 

## Limitations

* Currently, a new bundle identifier is generated for each bundle transformation.  In future, the bundle identifiers should be generated by hashing the contents of the bundle -- so that bundle transformation is deterministic.
*  `Codesign` tool on mac-os performs strict consistency checks, and cannot tolerate the additional files appended at the end of the `AppHost` executable. [Further work](https://github.com/dotnet/core-setup/issues/7065) is necessary to update the binary headers to make the bundle file compatible for signing. 
