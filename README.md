# make-adobe-cc-licenser-pkg

## Overview

This is a command-line tool that makes it easier to deploy Adobe Creative Cloud device license files output by the Creative Cloud Packager application.

Given a `prov.xml` output file from Creative Cloud Packager, this tool packages the `adobe_prtk` executable and adds a postinstall script to perform the activation. It can also import the package into Munki with an appropriate uninstaller script added so that the license can be deactivated simply by removing the item using Munki. If not using Munki, the uninstall script is output alongside the package for your own use.

## Why?

Adobe Creative Cloud Packager offers an option "Create License File," but does not make it obvious about how to deploy the license activation, and there's no tool for deactivating the license with a CC For Teams license. The `RemoveVolumeSerial` tool output by CCP can only be used for those with Enterprise agreements. (An enterprise LEID is hardcoded in the binary.) There is some very confusing documentation available [here](https://helpx.adobe.com/creative-cloud/packager/create-license-file.html), [here](https://helpx.adobe.com/creative-cloud/packager/provisioning-toolkit-enterprise.html), and more details about LEIDs [here](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html).

Why would you even want to make a license file, when there's already an option to build device licensed packages? After all, these packages will handle activating a device license, and Munki has support for running CCP's uninstaller packages to remove the application. Unfortunately, uninstalling a device-licensed package built with CCP removes that device license activation _regardless_ of whether other applications also relying on that activation are installed, making it a pain to try and manage multiple apps within a single device pool independently of one another. This behaviour is a bug scheduled to be fixed by the Adobe IT Toolkit team.

So instead, it seems possible to install a Named License and "convert" it to a device license using a license file and Adobe's activation tools. This might be a more manageable deployment scenario for your environment, regardless of the bug described above, because it allows you to build a single Named installer and then optionally apply a device license to that machine (and remove it later to restore it to a Named license).

With Munki, you could then assign this license pkg to a manifest directly, or potentially add it as an `update_for` multiple CC products with Named licenses. This way, Munki will keep the license around for as long as at least one CC app (for which this is an `update_for` is installed on the system). Munki will deactivate the license if the last dependent CC app is removed (using Munki).

## Requirements

1. A copy of the adobe_prtk executable, which is installed as part of Adobe's Creative Cloud Packager. It can be provided as a command option or if omitted, the CCP installation path will be searched in the following location: `/Applications/Utilities/Adobe Application Manager/CCP/utilities/APTEE`
1. A prov.xml license activation file, output from the "Create License File" task in CCP.

## Usage

Run the command with a single argument: path to a prov.xml file, and at least the required options `--name` and `--reverse-domain`:

```
./make-adobe-cc-license-pkg \
    --name AdobeCCCompletePool \
    --reverse-domain diner.r.double \
    path/to/my/prov.xml
```

Run the command with the `-h` (or `--help`) option to print out the full usage.

If you've also used the `--munki` option to import the package into Munki, you will likely want to manually modify other pkginfo keys such as `display_name`, `description`, etc. This script isn't meant to be an all-encompassing Munki importer tool that passes any other options through to `munkiimport`.

## What's it actually do?

1. `adobe_prtk` is scanned to extract a version number so it can be installed in a path that does not conflict with other versions in the future.
1. `prov.xml` is read to extract the [LEID](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html), which is needed if performing an uninstallation (deactivation) later.
1. Stages your copies of adobe_prtk and prov.xml to be installed to `/usr/local/bin/adobe_prtk_$version/adobe_prtk` and `/private/tmp/prov.xml`, respectively.
1. A Python-based postinstall script is written for the package, which executes `adobe_prtk` with the required options. Additionally, if the command emits an error code, the error code along with an explanation (taken from Adobe's documentation) is printed to the install log (!).
1. Similarly, the uninstaller script is written to disk, which executes the appropriate `adobe_prtk` options, using the LEID that was extracted from the `prov.xml` earlier. It also forgets the package receipt and removes the copy of `adobe_prtk` that was installed.
1. If the `--munki` option is given, the pkg will be imported into Munki, with the uninstall script set as the `uninstall_script`.
1. The resultant pkg and uninstall script will be written to the directory specified by `--output` or if omitted, the current working directory.
