# adobe-cc-licenser-pkg

Deploying Adobe Creative Cloud to managed workstations _sucks_.

This is a work-in-progress tool for building a CC device license installer, given a `prov.xml` output file from Creative Cloud Packager, packaging the `adobe_prtk` executable and adding a postinstall script to perform the activation. It's currently a Bash script, though if this deployment technique proves workable, it will be rewritten in Python. It also imports the package into Munki with an appropriate uninstaller script so that the license can be deactivated simply by removing the item using Munki.

Adobe Creative Cloud Packager offers an option "Create License File," but does not make it obvious about how to deploy the license activation, and there's no tool for deactivating the license with a CC For Teams license. The `RemoveVolumeSerial` tool output by CCP can only be used for those with Enterprise agreements. (Why? For one, an enterprise LEID is hardcoded in the binary.) There is some very confusing documentation available [here](https://helpx.adobe.com/creative-cloud/packager/create-license-file.html), [here](https://helpx.adobe.com/creative-cloud/packager/provisioning-toolkit-enterprise.html), and more details about LEIDs [here](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html).

Why would you even want to make a license file, when there's already an option to build device licensed packages? After all, these packages will handle activating a device license, and Munki has support for running CCP's uninstaller packages to remove the application. Unfortunately, uninstalling a device-licensed package built with CCP removes that device license activation _regardless_ of whether other applications also relying on that activation are installed. This makes it a pain to try and manage multiple apps within a single device pool independently of one another.

So instead, it seems possible to install a Named License and "convert" it to a device license using a license file and Adobe's activation tools. I'm not sure anyone at Adobe has actually confirmed whether this approach is a good idea, but they also have not provided any details about the bug beyond confirming it as a bug.


The resultant Munki item you could then assign to a manifest directly, or potentially add as an `update_for` multiple CC products with Named licenses. This way, Munki will keep the license around for as long as at least one CC app (for which this is an `update_for` is installed on the system). Munki will deactivate the license if the last dependent CC app is removed (using Munki).

## Requirements:

1. A copy of the adobe_prtk executable, which is installed as part of Adobe's Creative Cloud Packager. adobe_prtk will be searched in this directory, or if not found, its installation path (alongside CCP) will be searched: /Applications/Utilities/Adobe Application Manager/CCP/utilities/APTEE/adobe_prtk
1. A prov.xml file, output from the "Create License File" task in CCP.

## Usage

Run the command with a single argument: path to a prov.xml file, and several configuration
variables given as environment variables: `PKGNAME`, `MUNKI_REPO_SUBDIR`, `REVERSE_DOMAIN`

One additional optional variable: `VERSION`, which if not set will be: YYYY.MM.DD

```
PKGNAME=MyAdobeLicensePkgName \
MUNKI_REPO_SUBDIR=apps/AdobeCC \
REVERSE_DOMAIN=org.my \
./build.sh path/to/prov.xml
```
