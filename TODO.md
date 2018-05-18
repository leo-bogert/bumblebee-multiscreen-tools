## High priority
- The Noveau drivers provide [much more simple configuration of Optimus](https://nouveau.freedesktop.org/wiki/Optimus/) than Bumblebee. They should also be a lot faster because they probably don't copy around data in memory as much as intel-virtual-output does. Create a copy / branch of this repository which uses Noveau. If it works nicely put a big fat warning into the README.md to use Noveau instead (notice that it is already mentioned there, replace that text).

- Update wikis to mention bumblebee-multiscreen-tools once development is sufficiently finished (which it is not yet!):
  - https://github.com/Bumblebee-Project/Bumblebee/wiki/Multi-monitor-setup
  - https://wiki.ubuntuusers.de/Hybrid-Grafikkarten/Bumblebee/
    - "Multimonitorbetrieb" section says that multiscreen mode doesn't work, but in fact with bumblebee-multiscreen-tools it is possible

## Low priority
- Renice Xorg, perhaps helps with vsync issues

- Split this up into multiple repositories:
	- thinkpad-dock-tools:
		- dock-handler script. Will call all scripts in /etc/thinkpad-dock-tools/(un)dock.d
		- /etc/acpi/events/thinkpad-series3dock_(un)docked to trigger the execution of the dock-handler script.
	- bumblebee-multiscreen-tools:
		- switch-screen script to switch the screen.
		- /etc/thinkpad-dock-tools/(un)dock.d/choose-screen script as hook to call switch-screen when dock state changes.
	- thinkpad-pm-tools:
		- /etc/pm/sleep.d/thinkpad script to handle hibernation/suspend and wakeup. Shall trigger the execution of the dock-handler script and thereby /etc/thinkpad-dock-tools/(un)dock.d - which ensures screen switching happens before and after suspend/hibernate. Also shall set the CPU governor before hibernation and after wakeup.
		- /etc/pm/power.d/thinkpad script to handle switching between AC and battery power. Shall also call the dock handler and deal with the CPU governor.
		
		See /usr/share/doc/pm-utils/HOWTO.hooks.gz for how to do the above using /etc/pm/