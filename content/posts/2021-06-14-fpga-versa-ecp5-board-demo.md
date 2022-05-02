---
title: Bringing up ECP5 Versa with open tools on Windows
date: 2021-06-14 17:00:00
tags:
    - fpga
categories:
    - tech
keywords:
    - markdown
    - code block
---

Specifically the Lattice ECP5 FPGA Versa Dev Kit with part number `LFE5UM-45F-VERSA-EVN` which is ever so slightly different from the prjtrellis example [versa5g design](https://github.com/YosysHQ/prjtrellis/tree/master/examples/versa5g) as it has a previous iteration of the FPGA.

I used a windows machine for this walkthrough, so most of this was setting up the host USB stack ready for OpenOCD.

FPGA Toolchain
----

I used the [YoWASP binary builds by Whitequark](http://yowasp.org/) which are distributed as... _python packages_ and installed using pip. `Yosys` is needed for synthesis and `nextpnr-ecp5` for place and route. Using a recent 3.6+ python virtual environment:

    $ python -m venv venv
    $ ./venv/Scripts/activate
    $ echo requirements.txt
    yowasp-nextpnr-ecp5-all
    yowasp-yosys
    $ pip install -r requirements.txt
    ...
    $ yowasp-yosys --version
	Yosys 0.9+4052 (git sha1 d8c5d681, ccache clang 11.0.0-2~ubuntu20.04.1 -Os -flto -flto)


Now grab the sample code and tweak build to use `yowasp-*` tools.
Set the part to be `--um-45k` not `--um5g-45k` - the package is the same, as is the chip pin out.

    $ git clone git@github.com:YosysHQ/prjtrellis.git
    $ cd prjtrellis/examples/versa5g
    $ vi build.sh
    $ cat build.sh
    yowasp-yosys -p "synth_ecp5 -json demo.json" demo.v
    yowasp-nextpnr-ecp5 --json demo.json --lpf versa.lpf --textcfg demo_out.config --um-45k --package CABGA381
    yowasp-ecppack --svf-rowsize 100000 --svf demo.svf demo_out.config demo.bit
    $ ./build.sh
    ...
    Info: Device utilisation:
	Info:          TRELLIS_SLICE:   102/21924     0%
	Info:             TRELLIS_IO:    23/  245     9%
	Info:                   DCCA:     1/   56     1%
	Info:                 DP16KD:     0/  108     0%
	Info:             MULT18X18D:     0/   72     0%
	Info:                 ALU54B:     0/   36     0%
	...
	$

This should produce a huge `demo.svf` programming file ready to send to the device.

Supply the dev board with 12V using the supplied AC adapter and hook up the USB cable.

Now for the fun part - Windows USB & JTAG flashing.

WinUSB Driver setup
----

If any FTDI generic drivers are installed, remove them using the ["CDM Uninstaller" tool](http://www.ftdichip.com/Support/Utilities/CDM_Uninst_GUI_Readme.html) and the well known USB VID 0403 PID 6010 combination.

![FTDI CDM uninstaller](/images/ftdi-cdm-uninstall-0403-6010-devices.png)

Now use [Zadig 2.5](https://zadig.akeo.ie/) to install the WinUSB driver. You need to choose Options and make sure the option to show Composite Devices is enabled. Click the "Lattice ECP5_5G VERSA Board" *composite* device and switch the desired driver to "WinUSB (v6.1.7600.16385)", click to apply the change. Let windows refresh it's device list etc etc.

![Zadig show composite devices](/images/fpga-zadig-show-composite-devices.png)

Grab OpenOCD from [github releases](https://github.com/ntfreak/openocd/releases). There is a windows binary package built for each tag however it is just a tar.gz so looks like a source release - I used openocd-v0.11.0-i686-w64-mingw32.tar.gz - download and unpack


Double Check dev board USB name!
----
The dev kit I recieved had the FTDI chip programmed to show USB device name `ECP5_5G`, but it is only an `ECP5` chip populated on the board. The silkscreen label has also been modified with a marker pen to mask `5G`. Perhaps delays with the 5G parts meant Lattice had to fit older parts to an early run of the boards.

You can see the mismatch next when we enumerate the JTAG chain - so I had to set `ftdi_device_desc "Lattice ECP5_5G VERSA Board"` in the OpenOCD config file for OpenOCD to find the JTAG programmer. You will need to check that the IDCODE of the chip comes back as expected.

Resulting diff against prjtrellis upstream was:

	$ git diff
	diff --git a/examples/versa5g/Makefile b/examples/versa5g/Makefile
	index 862aa8a..9f12ae2 100644
	--- a/examples/versa5g/Makefile
	+++ b/examples/versa5g/Makefile
	@@ -11,7 +11,7 @@ pattern.vh: make_14seg.py text.in
	        yosys -p "synth_ecp5 -json $@" $<

	 %_out.config: %.json
	-       nextpnr-ecp5 --json $< --lpf ${CONSTR} --textcfg $@ --um5g-45k --package CABGA381
	+       nextpnr-ecp5 --json $< --lpf ${CONSTR} --textcfg $@ --um-45k --package CABGA381

	 %.bit: %_out.config
	        ecppack --svf-rowsize 100000 --svf ${PROJ}.svf $< $@
	diff --git a/misc/openocd/ecp5-versa.cfg b/misc/openocd/ecp5-versa.cfg
	index 4958627..219090c 100644
	--- a/misc/openocd/ecp5-versa.cfg
	+++ b/misc/openocd/ecp5-versa.cfg
	@@ -1,7 +1,7 @@
	 # this supports ECP5 Versa board

	 interface ftdi
	-ftdi_device_desc "Lattice ECP5 Versa Board"
	+ftdi_device_desc "Lattice ECP5_5G VERSA Board"
	 ftdi_vid_pid 0x0403 0x6010
	 # channel 1 does not have any functionality
	 ftdi_channel 0



Link out JTAG Scan chain for ispCLOCK chip
---
The prjtrellis Versa board sample configuration suggests linking the Lattice ispCLOCK chip out of the JTAG chain. Supposidly the chips JTAG engine is unreliable when in the chain.

This is configured by J50 on the board - link `1-2` and `3-5`, leaving `4-6` open circuit per page 6 of the [ECP5-5G Versa Development Board User Guide](https://www.latticesemi.com/-/media/LatticeSemi/Documents/UserManuals/EI2/FPGA-EB-02021-2-3-ECP5-Versa-Development-Board.ashx?document_id=50996).


![Versa JTAG chain jumper settings](/images/fpga-versa-jtag-patch-jumper-settings.png)

If you do not patch out the ispCLOCK chip, [uncomment Line 16 in `prjtrellis/misc/openocd/ecp5-versa.cfg`]( https://github.com/YosysHQ/prjtrellis/blob/master/misc/openocd/ecp5-versa.cfg#L16) to include the ispClock device.

Push bitstream over JTAG
----

    $ vi ../../misc/openocd/ecp5-versa.cfg
    ...
    $ winpty ../../../openocd-v0.11.0-i686-w64-mingw32/bin/openocd.exe -f ../../misc/openocd/ecp5-versa.cfg -c "transport select jtag; init; svf demo.svf; exit"
	Open On-Chip Debugger 0.11.0 (2021-03-07-12:52)
	Licensed under GNU GPL v2
	For bug reports, read
	        http://openocd.org/doc/doxygen/bugs.html
	Info : auto-selecting first available session transport "jtag". To override use 'transport select <transport>'.
	Warn : Transport "jtag" was already selected
	Info : ftdi: if you experience problems at higher adapter clocks, try the command "ftdi_tdo_sample_edge falling"
	Info : clock speed 25000 kHz
	Info : JTAG tap: ecp5.tap tap/device found: 0x01112043 (mfg: 0x021 (Lattice Semi.), part: 0x1112, ver: 0x0)
	Warn : gdb services need one or more targets defined
	svf processing file: "demo.svf"
	HDR     0;
	HIR     0;
	...


If the JTAG upload works you will see the text message scroll through the 13-seg display.
