---
layout: post
title: Bringing up ECP5 Versa Dev Kit with Yosys, nextpnr
---

Specifically the Lattice Dev Kit with part number LFE5UM-45F-VERSA-EVN
Using the example design from https://github.com/YosysHQ/prjtrellis/tree/master/examples/versa5g which is for a slightly different board.

I used a windows machine for this walkthrough, so part of this was to setup the host USB stack ready for OpenOCD.

FPGA Toolchain
----

Install fpga toolchain, I used the binary build by Whitequark called "yowasp" which is distributed as ... python packages installed using pip. Using a recent 3.6+ python:

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

This should produce a huge `demo.svf` programming file ready to send to the device

Now for the fun part - Windows USB & JTAG flashing.

WinUSB Driver setup
----
Supply the dev board with 12V using the supplied AC adapter and hook up the USB cable.

If any FTDI generic drivers are installed, remove them using the "CDM Uninstaller" tool http://www.ftdichip.com/Support/Utilities/CDM_Uninst_GUI_Readme.html and the well known USB VID 0403 PID 6010 combination.

Now use Zadig 2.5 https://zadig.akeo.ie/ to install the WinUSB driver. You need to choose Options and make sure the option to show Composite Devices is enabled. Click the "Lattice ECP5_5G VERSA Board" *composite* device and switch the desired driver to "WinUSB (v6.1.7600.16385)", click to apply the change. Let windows refresh it's device list etc etc.

Grab OpenOCD from github releases https://github.com/ntfreak/openocd/releases. There is a windows binary package built for each tag however it is just a tar.gz so looks like a source release - I used openocd-v0.11.0-i686-w64-mingw32.tar.gz - download and unpack


JTAG flashing
----
Note that the dev kit I had has a FTDI chip flashed with the USB device name ECP5_5G, but it is only an ECP5 chip on the board. You can see the mismatch next when we enumerate the JTAG chain - so I had to set `ftdi_device_desc "Lattice ECP5_5G VERSA Board"` for OpenOCD to match the JTAG programmer.

Additionally the versa setup scripts suggest linking the Lattice ispCLOCK chip out of the JTAG chain. This is configured by J50 on the board - link `1-2` and `3-5`, leaving `4-6` open circuit per page 6 of the [ECP5-5G Versa Development Board User Guide](https://www.latticesemi.com/-/media/LatticeSemi/Documents/UserManuals/EI2/FPGA-EB-02021-2-3-ECP5-Versa-Development-Board.ashx?document_id=50996).

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


It works and scrolls the message. Hopefully the next post will use something more interesting than Verilog for the frontend.

Final diff against prjtrellis upstream was:

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



