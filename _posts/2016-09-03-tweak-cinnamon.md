---
layout: post
title:  "Tweaking Cinnamon in Arch Linux"
date:   2016-09-03 09:55:05 +0200
categories: linux
author: giulio
---

One of the best GNU/Linux distributions I used is Linux Mint Cinnamon edition. It's quick, stable and good-looking. But some time ago, I discovered Arch Linux, and I immediately started using it. Thanks to its great possibilities to customize every bit of the system, I actually managed to recreate the Linux Mint look and feeling, but still having in the background the power of Arch Linux. Sadly some minor features that are present in Mint disappeared (quite obviously) in Arch, such as the right click "Uninstall" button in the main application menu.

{% include image.html url="/assets/images/cinnarch.png" %}

Furthermore, recently I managed to get my nVidia card working on Linux, using bumblebee, and I discovered that there were no easy and quick means to use bumblebee to run a program from the desktop manager which didn't involve opening a terminal.

So I thought: let's try and fix it... In this post I'll explain what I did and provide the links to download my modifications.

### The "Uninstall" option

To try to reimplement the "Uninstall" feature in Arch I first needed to understand how it worked, and to do it, I had to explore a bit the Linux Mint repositories
Enabling the option

#### Enabling the option

I discovered a few interesting things in the [menu applet source file](https://github.com/linuxmint/Cinnamon/blob/86e14ef1000adcf6b21b98e3addf279c6dfe159c/files/usr/share/cinnamon/applets/menu%40cinnamon.org/applet.js):

<ol>
<li>In the activate function (called when clicking an option of the right click menu) of the ApplicationContextMenuItem class (by the name it's what we're looking for) there's this possible action:

{% highlight javascript linenos %}
case "uninstall":
    Util.spawnCommandLine("gksu -m '" + _("Please provide your password to uninstall this application") + "' /usr/bin/cinnamon-remove-application '" + this._appButton.app.get_app_info().get_filename() + "'");
    this._appButton.appsMenuButton.menu.close();
    break;
{% endhighlight %}

This means that when the uninstall action is triggered it will launch the program cinnamon-remove-application passing a single parameter, which in this case, after some tests in Lookin Glass (Melange Cinnamon debugger), I discovered to be the path to the .desktop file located in /usr/share/applications;
</li>
<li>In the toggleMenu function (called when right-clicking the application menu item) in the GenericApplicationButton class, a base class to build application buttons, there's this bit of code:

{% highlight javascript linenos %}
if (this.appsMenuButton._canUninstallApps) {
    menuItem = new ApplicationContextMenuItem(this, _("Uninstall"), "uninstall");
    this.menu.addMenuItem(menuItem);
}
{% endhighlight %}

This basically means that when it's building the right click menu it checks for the _canUninstallApps variable: if it's true it's going to build the "Uninstall" menu item and add it to the right-click menu. So the key to enabling the "Uninstall" option is finding that variable and making sure Cinnamon sets it to true when loading the menu applet.
</li>
<li>The _canUninstallApps variable is assigned towards the middle of the file, in the _init function of the applet:

{% highlight javascript linenos %}
    this._canUninstallApps = GLib.file_test("/usr/bin/cinnamon-remove-application", GLib.FileTest.EXISTS);
{% endhighlight %}

From this single line we finally understand that the key to enable our "Uninstall" option is providing the /usr/bin/cinnamon-remove-application script... Quite obvious, I must admit.
</li>
</ol>

#### The cinnamon-remove-application script

At that time the simplest and quickest thing to do seemed to find the cinnamon-remove-application script and to copy it in the needed location, but sadly I didn't remember that Linux Mint is based on Debian, and that its package manager, apt-get, is not compatible with pacman, Arch's one.

So I googled "cinnamon-remove-application", and I discovered that it is located in the mint-common repository, where is reported to be a symbolic link to [/usr/lib/linuxmint/common/mint-remove-application.py](https://github.com/linuxmint/mint-common/blob/master/usr/lib/linuxmint/common/mint-remove-application.py).

Basically what this script does is that:

1. finds to which package the provided .desktop file belongs;
2. if it doesn't belong to any, it asks you to remove it;
3. if it belongs to a package, it finds which packages depend on it;
4. shows a dialog listing the packages to be removed and asking for a confirmation;
5. If you confirm, it uninstalls the packages.

Obviously, because of the incompatibility between the two distros' package managers, which I explained before, the script had to be adapted, by replacing the apt-get commands with pacman ones:

1. to find which package the .desktop file belongs to, we can use "pacman -Qoq" followed by the file path;
2. the command to list those packages depending on the one that is going to be removed is "pacman -Rpc" followed by the package name;
3. the last thing is to effectively remove that packages, and to do this, it's enough to remove the "-p" option from the previous command, and it's done!

#### Links

The code for this script can be found in the GitHub [repo](https://github.com/rapgenic/cinnamon-remove-application), and it can be installed in any Arch system with the AUR package [cinnamon-remove-application](https://aur.archlinux.org/packages/cinnamon-remove-application/).

### The "nVidia" option (main menu applet)

This is something completely different from what I described before, a completely new option, that, at the moment of writing isn't available, but that, since the Cinnamon maintainers have merged my pull request, I hope will be in one of the next versions of Cinnamon.

Let's start from the beginning: a few days ago I decided to give up on Nouveau drivers, because I have a fairly powerful graphic card, and not using it disappointed me... So I decided to install the official closed-source drivers, but since I had heard that the dedicated graphic card eats a lot more power than the internal one, I set up bumblebee, which basically enables you to switch between the two cards on your system, by running "optirun" followed by the program name in a command line.

Soon I realized that it was very frustrating to open the terminal each time I wanted to run Blender or some other software, and so I decided, using what I'd discovered while implementing the "Uninstall" option, to create a completely new one: "Run with nVidia GPU".

All my changes are in a single file, the [same](https://github.com/linuxmint/mint-common/blob/master/usr/lib/linuxmint/common/mint-remove-application.py) we examined before, and that's what I did:

<ol>
<li>I created a new _canRunBumblebee variable in the _init function of the applet:

{% highlight javascript linenos %}
    this._isBumblebeeInstalled = GLib.file_test("/usr/bin/optirun", GLib.FileTest.EXISTS);
{% endhighlight %}

that is assigned true when the optirun executable is found;
</li>
<li>
then in the toggleMenu function I added this bit of code to build a new context button, "Run with nVidia GPU":

{% highlight javascript linenos %}
    if (this.appsMenuButton._isBumblebeeInstalled) {
    menuItem = new ApplicationContextMenuItem(this, _("Run with nVidia GPU"), "run_with_nvidia_gpu");
    this.menu.addMenuItem(menuItem);
}
{% endhighlight %}
</li>
<li>
finally, in the activate function, I added this bit of code to handle the click on the new button:

{% highlight javascript linenos %}
    case "run_with_nvidia_gpu":
    Util.spawnCommandLine("optirun gtk-launch " + this._appButton.app.get_id());
    this._appButton.appsMenuButton.menu.close();
break;
{% endhighlight %}

What this does is to run optirun which in turn runs gtk-launch, which when given the .desktop file path as the parameter, launches the corresponding program.
</li>
</ol>

### The "nVidia" option (Nemo action)

With the previous solution you can easily start applications using the dedicated nVidia card, but what if you have a generic program? Usually you run it from the file manager, so we should implement an option to run binaries with optirun.

The Cinnamon file manager is Nemo (a fork of Nautilus), which currently has three possibilities of implementing new options:

1. extensions;
2. scripts;
3. actions;

Extensions and scripts are too complex for a simple task such as launching a program, but an action is perfect for our purpose.

Nemo actions are small files, with a ".nemo_action" extension, which can be placed in:

- /usr/share/nemo/actions/
- ~/.local/share/nemo/actions/

They define a new action that the user can choose in the right-click context menu of a file or directory.

Let's have a look at the action file I wrote to run binaries with optirun:

{% highlight ini linenos %}
[Nemo Action]
Name=Run with nVidia GPU
Comment=Run %f with dedicated nVidia graphic card
Exec=optirun %F
Icon-Name=nvidia-settings
Selection=s
Mimetypes=application/x-executable;application/x-shellscript;text/x-script.phyton;application/x-bytecode.python;
Dependencies=optirun;
EscapeSpaces=true
{% endhighlight %}

The "Name" and "Comment" strings are fairly self-explanatory; the most important thing to note is the possibility to use some tokens to customize the content of the strings; of them %f and %F are the two most used ones: the first represents the name of the file, and can be used in the "Comment" string to explain more clearly what that action does, the second represents the full path to the file, and is very useful to be passed as an argument to another program (which is exactly our case).

The "Exec" string is the most important one, it's compulsory and defines exactly what the action does: when the action is clicked, Nemo executes that line. In our case we are going to run optirun followed by the path of the file (%F).

The "Icon" strings defines which icon is going to be displayed near the action name in the right-click menu.

The "Selection" string is quite important, too: it defines how many files have to be selected to have Nemo show up that action in the right-click menu. In our case, since we don't want to run many programs at once, we assigned it to "s", which means single.

The "Mimetypes" array defines which mime types of file, when right-clicked will open a context menu with our option shown. In our case we chose the main program mime types.

The "Dependencies" options defines which programs this action depends on; Nemo will look for them in the path, and if it doesn't find them, it's not going tho show the action.

The "EscapeSpaces" option, finally, defines whether Nemo is going tho escape spaces with "\" when replacing tokens with their actual content, and, if you're not using quotes, this should be set to true.

This is a basic example of an action file, that doesn't cover all the available options. To read more about the other options just have a look at /usr/share/nemo/actions/sample.nemo_action which explains with a lot of detail what every bit of code does.

The "Run with nVidia GPU" action can be easily downloaded from the [github](https://github.com/rapgenic/nemo-run-with-nvidia) repository, or, if you're using Arch Linux, it can be directly installed from the [AUR](https://aur.archlinux.org/packages/nemo-run-with-nvidia/).