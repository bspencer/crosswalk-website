# Running a Crosswalk app on Tizen

On Tizen, Crosswalk runs as a background service, only becoming active when needed (i.e. when a user session activates it). In technical terms, Crosswalk effectively runs as a daemon, exposing a D-Bus interface for managing applications.

To run an application on a Tizen target, first ensure you have set up your host for Tizen ([Windows](/documentation/tizen/windows_host_setup.html), [Linux](/documentation/linux/linux_host_setup.html)) and [set up a Tizen target](/documentation/tizen/tizen_target_setup.html). In the instructions below, we assume you are using a Tizen IVI target running under VMware, and consequently use `ssh` to push files to, and get a shell on, the target.

Next, follow these steps to get the application running:

1.  [Create a Tizen package](#Create-a-Tizen-package) (`.xpk` file) from the application code.
2.  [Push the package to the device](#Push-the-package-to-the-device).
3.  [Install the application package](#Install-the-application-package).
4.  [Run the application](#Run-the-application).

These steps are explained in detail below.

<h2 id="Create-a-Tizen-package">Create a Tizen package</h2>

A Tizen package file is a zip file with some "magic" (a special file header specific to Crosswalk) and an `.xpk` suffix. It will contain all of the files relating to your application (HTML, CSS, JavaScript, assets), as well as any metadata (`manifest.json`, icons etc.). See [the wiki](https://github.com/crosswalk-project/crosswalk-website/wiki/Crosswalk-package-management) for detailed information about the format.

To create a zip package, you will need a `bash` shell and the `zip` and `openssl` binaries installed. On Linux, these are usually available by default. On Windows, you will need to [install git SCM](/documentation/tizen/windows_host_setup.html).

Then follow the steps below to create the package.

### Add the make_xpk script

1.  Copy the following script into a file called `make_xpk.sh`:

        #!/bin/bash -e
        #
        # Purpose: Pack a CrossWalk directory into xpk format
        # Modified from http://developer.chrome.com/extensions/crx.html
        if test $# -ne 2; then
          echo "Usage: `basename $0` <unpacked dir> <pem file path>"
          exit 1
        fi

        dir=$1
        key=$2
        name=$(basename "$dir")
        xpk="$name.xpk"
        pub="$name.pub"
        sig="$name.sig"
        zip="$name.zip"
        trap 'rm -f "$pub" "$sig" "$zip"' EXIT

        [ ! -f $key ] && openssl genrsa -out $key 1024

        # zip up the xpk dir
        cwd=$(pwd -P)
        (cd "$dir" && zip -qr -9 -X "$cwd/$zip" .)

        # signature
        openssl sha1 -sha1 -binary -sign "$key" < "$zip" > "$sig"

        # public key
        openssl rsa -pubout -outform DER < "$key" > "$pub" 2>/dev/null

        byte_swap () {
          # Take "abcdefgh" and return it as "ghefcdab"
          echo "${1:6:2}${1:4:2}${1:2:2}${1:0:2}"
        }

        crmagic_hex="4372 576B" # CrWk
        pub_len_hex=$(byte_swap $(printf '%08x\n' $(ls -l "$pub" | awk '{print $5}')))
        sig_len_hex=$(byte_swap $(printf '%08x\n' $(ls -l "$sig" | awk '{print $5}')))
        (
          echo "$crmagic_hex $pub_len_hex $sig_len_hex" | xxd -r -p
          cat "$pub" "$sig" "$zip"
        ) > "$xpk"
        echo "Wrote $xpk"

2.  Make the script executable:

        > chmod +x make_xpk.sh

<h3 id="Create-the-xpk-file">Create the xpk file</h3>

1.  To create xpk packages, you will need a private key file. Use `openssl` to generate this for you:

        > openssl genrsa -out ~/mykey.pem 1024

    Note that this is a private key file, and it should not be distributed with your application.

2.  Call the shell script, passing it the path to the directory containing your application and your key file:

        > ./make_xpk.sh xwalk-simple/ mykey.pem

    This will produce a file named `xwalk-simple.xpk` in the directory where you ran the script.

<h2 id="Push-the-package-to-the-device">Push the package to the device</h2>

1.  Prepare the target (either connect it to the host or start it with VMware player/the emulator).

2.  Use `scp` to push the package to the device:

        > scp xwalk-simple.xpk root@<ip address>:/home/app/

    Note that we're using the **app** account's home directory on the target, as we'll install and run the application using this account.

<h2 id="Install-the-application-package">Install the application package</h2>

Use a terminal on the emulated device to run the following steps:

1.  Open a shell on the target as the `app` user. You can do this using the small console icon in the top-left of the screen.

2.  Install the `xwalk-simple.xpk` package using the `pkgcmd` command (still as the app user):

        app:~> pkgcmd -i -t xpk -p /home/app/xwalk-simple.xpk -q

    The output from this command will end with something like this:

        path is /home/app/xwalk-simple.xpk
        __return_cb req_id[1] pkg_type[wgt] pkgid[ihogjblnkegkonaaaddobhoncmdpbomi] key[start] val[install]
        __return_cb req_id[1] pkg_type[wgt] pkgid[ihogjblnkegkonaaaddobhoncmdpbomi] key[end] val[ok]
        spend time for pkgcmd is [2640]ms

    The long ID string (*ihogjblnkegkonaaaddobhoncmdpbomi* here) is the **application ID**. This is important, as you'll need it to launch the application in the next step.

    If you need to uninstall the application, you can do this with:

        pkgcmd -u -n <application ID> -q

    Please refer to [Tizen package management](https://wiki.tizen.org/wiki/Application_framework#Package_Management_2) for more details.

<h2 id="Run-the-application">Run the application</h2>

To start the application, you need to know the ID assigned to the application when it was installed. If you can't remember it, you can list the installed applications with:

    app:~> ail_list

Pass the ID to the `app_launcher` command, in the same shell you used to install the application:

    app:~> app_launcher -s xwalk.ihogjblnkegkonaaaddobhoncmdpbomi

The application should now start on the target. Here it is running on an emulated Tizen IVI device, on Fedora Linux:

<img src="/assets/xwalk-simple-on-tizen-ivi.png">

Please refer to [Tizen application management](https://wiki.tizen.org/wiki/Application_framework#Application_Management) for more details.

To start the application in fullscreen mode, you can [configure this in the manifest](/documentation/manifest/display.html).
