---
title: FuseSoc options for SystemVerilog support in Yosys
date: 2022-11-06 18:30:00
tags:
    - fpga
categories:
    - tech
keywords:
    - markdown
---

If you get this error `syntax error, unexpected TOK_ID, expecting` from fusesoc invoking Yosys:

    $ fusesoc run --target=orangecrab_r0.2 shuckc:fpgachess:uci
    ...
    -- Running command `tcl edalize_yosys_template.tcl' --

    1. Executing Verilog-2005 frontend: ../src/shuckc_fpgachess_uci_0/hw/fen_decode.sv
    Lexer warning: The SystemVerilog keyword `logic' (at ../src/shuckc_fpgachess_uci_0/hw/fen_decode.sv:3) is not recognized unless read_verilog is called with -sv!
    ../src/shuckc_fpgachess_uci_0/hw/fen_decode.sv:3: ERROR: syntax error, unexpected TOK_ID, expecting ',' or '=' or ')'
    make: *** [Makefile:6: shuckc_fpgachess_uci_0.json] Error 1

This is caused by the fusesoc `.core` file specifying the `fileset` file as `{file_type : verilogSource}` rather than `{file_type : systemVerilogSource}`.

Some tools supported by edalize quietly enable SystemVerilog support from the file extension or a global flag, but Yosys requires an explicit flag on each file:


    filesets:
      rtl:
        files:
          - hw/fen_decode.sv : {file_type : systemVerilogSource}
          - hw/ascii_int_to_bin.sv : {file_type : systemVerilogSource}
          - hw/onehot_to_bin.v : {file_type : verilogSource}
