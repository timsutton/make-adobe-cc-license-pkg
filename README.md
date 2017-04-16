# make-adobe-cc-license-pkg

## Overview

This is a command-line tool that makes it easier to deploy Adobe Creative Cloud device license files (output by the Creative Cloud Packager application) on OS X, by building them into a standard OS X package installer.

Given a path to the license file output from Creative Cloud Packager, this tool packages the `adobe_prtk` executable and adds a postinstall script to perform the activation. It can also import the package into Munki with an appropriate uninstaller script added so that the license can be deactivated simply by removing the item using Munki. If not using Munki, the uninstall script is output alongside the package for your own use.

## Why?

Here's a [blog post](http://macops.ca/adobe-creative-cloud-deployment-packaging-a-license-file).

Adobe Creative Cloud Packager offers an option "Create License Package," but does not make it obvious about how to deploy the license activation, and there's no tool for _deactivating_ the "device" license used by CC for Teams Education agreements and returning it to the license pool. The `RemoveVolumeSerial` tool output by CCP can _only_ be used for those with Enterprise agreements. (An enterprise LEID is hardcoded in the binary.) There is some very confusing documentation available [here](https://helpx.adobe.com/creative-cloud/packager/create-license-file.html), [here](https://helpx.adobe.com/creative-cloud/packager/provisioning-toolkit-enterprise.html), and more details about LEIDs [here](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html).

Why would you even want to make a license file, when there's already an option to build device licensed packages? After all, these packages will handle activating a device license, and Munki has support for running CCP's uninstaller packages to remove the application. Unfortunately, uninstalling a device-licensed package built with CCP removes that device license activation _regardless_ of whether other applications also relying on that activation are installed, making it a pain to try and manage multiple apps within a single device pool independently of one another.

This behaviour where uninstalling a licensed application package also uninstalls the license (even if other applications may require it) was classified by Adobe as a bug and scheduled to be fixed. The bug was first [acknowledged in April 2015](https://twitter.com/Adobe_ITToolkit/status/591361032905433088), and quite a while after this tool was written it was eventually [fixed](https://helpx.adobe.com/creative-cloud/kb/products-package-get-unserialized-unsubscribed.html) by the Adobe Enterprise tools team in the fall of 2016.

It is possible install a Named License and "convert" it to a device license using a license file and Adobe's activation tools. This might be a more manageable deployment scenario for your environment, regardless of the bug described above, because it allows you to build a single Named installer and then optionally apply a device license to that machine (and remove it later to restore it to a Named license).

With Munki, you could then assign this license pkg to a manifest directly, or potentially add it as an `update_for` multiple CC products with Named licenses. This way, Munki will keep the license around for as long as at least one CC app (for which this is an `update_for` is installed on the system). Munki will deactivate the license if the last dependent CC app is removed (using Munki).

## Requirements

This tool only requires the output from the Creative Cloud Packager's "Create License Package" workflow.

## Usage

Run the command with a single argument: the path to a directory containing the output of a "Create License Package" workflow from CCP:

```
./make-adobe-cc-license-pkg path/to/ccp/license/files/dir
```

Run the command with the `-h` (or `--help`) option to print out the full usage. There are multiple customization options for package parameters.

If you've also used the `--munki` option to import the package into Munki, you will likely want to modify some other pkginfo keys in the resultant pkginfo file, such as `display_name`, `description`, etc. This script isn't meant to be an all-encompassing Munki importer tool that passes any other options through to `munkiimport`, just enough to import a functional item into Munki with the appropriate `uninstall_*` keys.

## What's it actually do?

1. `helper.bin` (which is really the `adobe_prtk` tool) is scanned to extract a version number so it can be installed in a path that does not conflict with other versions in the future.
1. `prov.xml` is read to extract the [LEID](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html), which is needed if performing an uninstallation (deactivation) later.
1. Stages your copies of adobe_prtk and prov.xml to be installed to `/usr/local/bin/adobe_prtk_$version/adobe_prtk` and `/private/tmp/prov.xml`, respectively.
1. A Python-based postinstall script is written for the package, which executes `adobe_prtk` with the required options, and removes the prov.xml file. Additionally, if the command emits an error code, the error code along with an explanation (taken from Adobe's documentation) is printed to the install log (!).
1. Similarly, the uninstaller script is written to disk, which executes the appropriate `adobe_prtk` options, using the LEID that was extracted from the `prov.xml` earlier. It also forgets the package receipt and removes the copy of `adobe_prtk` that was installed.
1. If the `--munki` option is given, the pkg will be imported into Munki, with the uninstall script set as the `uninstall_script`.
1. The resultant pkg and uninstall script will be written to the directory specified by `--output` or if omitted, the current working directory.

## Additional tool for June 2016 and newer CC applications

New application versions released from June 2016 onwards use a [newer installer framework](http://blogs.adobe.com/deployment/2016/06/creative-cloud-package-1-9-5-is-live-redesigned-installer-technology-plus-much-more.html) known as "HyperDrive" (or "HD" for short). One frustrating regression of this installer format is that if a Named Licensed app package is installed _after_ a license is installed, that application will show a "Sign In to Creative Cloud" dialog box, which is confusing to both users and administrators - it suggests that a device or serialized license isn't correctly applied.

This regression occurred because the Adobe installer tools team did not consider the use case of wanting to install applications after a license had already been installed to the machine, and while it's possible this may be addressed in a future update, it is not resolved as of April 2017.

The only officially-supported workaround for resolving this issue is to 1) package all applications with licensing included, or 2) re-install a license file after each Named License application installation, which among other things will iterate over all installed products and insert a specific element to packaging metadata XML files which is read by the licensing framework in these applications at launch. Needing to re-install the license file after every application installation, while possible, is awkward to orchestrate via deployment tools. I also am concerned with needing to make modifications to the licensing system more frequently than is necessary, because Adobe has a history of issues with licensing subsystem failures.

As a solution for this issue, when the `make-adobe-cc-license-pkg` tool builds an installer package, it will install an additional script to `/usr/local/bin/adobe_cc_license_pkg_support/suppress_cc_hd_signin`. This script, when run as root, does the very simple operation of adding this same XML element added as part of the license installation, and nothing else. It does not perform any licensing operations or call out to any Adobe deployment tools, and all updates to XML files are idempotent.

To use this tool, _simply call this executable (as root)_ immediately after any Named License package installation. Any metadata files which have already had the modification performed will not be modified again, so this script is idempotent.

This script is not executed for you at any time, it is provided only as an additional tool on the system which you may use if you choose to. You may also choose to take this script out and package it up or run it outside of the context of this repo. This script doesn't require anything else from this repo, it just made sense to me for it to be included here because it's directly related to deploying Adobe apps in conjunction with a license file.

This script also functions on Windows systems which have Python installed. Both Python 2.7 and 3.x versions are supported, but only 64-bit versions of Windows have been tested. This script is not supported by Adobe.

For a more visual explanation of this issue, see my [MacADUK 2017 talk on Mac software deployment](https://youtu.be/pD6Pze1zQ4c?t=1579) where I talked about this issue and the workaround.

## More documentation

I've written a [series of blog posts](https://macops.ca/tag/creative-cloud) on deploying Creative Cloud using Munki, which includes considerations on deploying licenses.

At [MacSysAdmin 2015](http://macsysadmin.se/2015/Home.html), I gave a [recorded video session](http://docs.macsysadmin.se/2015/video/Day1Session4.mp4) on Mac Admin tools that included an explanation of deploying Adobe CC using Munki, aamporter and make-adobe-cc-license-pkg (see 12:00-37:30 in the video).

## Thanks

Thanks to [James Stewart](https://github.com/jgstew) for pointing out that the `helper.bin` file is identical to `adobe_prtk`. Previously this tool used to look for the undocumented location where `adobe_prtk` is installed along with CCP, or require you to include your own copy of the tool. The script is now much more portable.

Thanks to [Patrick Fergus](https://foigus.wordpress.com) for testing and feedback with Enterprise licenses.

## You're welcome

Adobe, you could have made this so much easier. Please use this utility as an example of how you can make things at least a little bit easier.
