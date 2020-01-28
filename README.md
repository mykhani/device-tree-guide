# Devicetree Guide
A devicetree is a tree based data structure containing nodes, which desribe the overall system and the physical devices, present on a computing platform (Beaglebone, RapberryPi etc).

Both Linux and uboot follows the same devictree format to specify the hardware.

## Format
Here's the basic format of the device tree file.
```
/dts-v1/;

/ {   
      label: node_name@node_address {
          reg = <node_address>;
          property-1 = "xyz";
          property-2 = <2 1>;
          property-3 = 1;
      };

      // rest of the devices
};
~
```
### Root node
The root node is the starting point of the devicetree's tree structure. It is denoted by ```/ {}```. Each devicetree should have a single root node, which contains all other nodes.

### Device node
A device node is of the following form:
```
label: node_name@node_address {
    /* properties */
};
```
#### Label
The label of the devicetree nodes serve its purpose during devicetree compilation. Depending on the context, a label could be used to:

1. **generate the full devicetree node path** - when a label is referenced with ```&``` in ```aliases``` node or some other device node without ```<>```.
2. **generate the devicetree phandle** - when a label is referenced with ```&``` inside ```<>```.

See the below example to see both path and phandle generation from the label.
```
ykhan@ykhan:~/uboot$ cat label.dts
/dts-v1/;

/ {
	device_a: device_a {
		property-1 = "xyz";
	};

	device_b {
		device-parent = <&device_a>; // label to phandle
		device-parent-path =  &device_a; // label to fullpath
	};
};

ykhan@ykhan:~/uboot$ # Output a pre-processed dts
ykhan@ykhan:~/uboot$ dtc -O dts label.dts
/dts-v1/;

/ {

	device_a: device_a {
		property-1 = "xyz";
		phandle = <0x1>;
	};

	device_b {
		device-parent = <0x1>;
		device-parent-path = "/device_a";
	};
};
```
#### Node name
The node name serves as an unique identifier for the specfic devicetree node.

#### Node address
The node address is appended to the node name after ```&```. Usually it is equal to the base address of the device registers e.g. ```uart base reg address```.

**The main purpose of the node address is for debugging and other devicetree utilties. For devices without base register, these addresses could be set to any unique numerical values.**

#### Phandle
The phandle property of a devicetree node is a numerical identifier, which can be used to reference this node from other devicetree nodes.

**Note: the phandle property is not specified inside the devicetree sources, instead, it is generated automatically by the devicetree compiler during compilation.**

See the above example in sub-section "Label", where ```device_b``` is referring to its parent ```device_a``` via ```device-parent = <0x1>```, where ```<0x1>``` is the phandle of ```device_a```.

#### Properties
A device node can have single or array of numerical and string properties. Except few standard properties like ```compatible``` and ```status```, the meaning of the rest of the properties depends on a given devicetree binding.

#### Device tree binding
A devicetree binding is a standardized device tree node for a specific device. For example, below is the devicetree binding for the *STMicroelectronics USART*.
```
* STMicroelectronics STM32 USART

Required properties:
- compatible: can be either:
  - "st,stm32-uart",
  - "st,stm32f7-uart",
  - "st,stm32h7-uart".
  depending is compatible with stm32(f4), stm32f7 or stm32h7.
- reg: The address and length of the peripheral registers space
- interrupts:
  - The interrupt line for the USART instance,
  - An optional wake-up interrupt.
- interrupt-names: Contains "event" for the USART interrupt line.
- clocks: The input clock of the USART instance

Optional properties:
- resets: Must contain the phandle to the reset controller.
- pinctrl-names: Set to "default". An additional "sleep" state can be defined
  to set pins in sleep state when in low power. In case the device is used as
  a wakeup source, "idle" state is defined in order to keep RX pin active.
  For a console device, an optional state "no_console_suspend" can be defined
  to enable console messages during suspend. Typically, "no_console_suspend" and
  "default" states can refer to the same pin configuration.
- pinctrl-n: Phandle(s) pointing to pin configuration nodes.
  For Pinctrl properties see ../pinctrl/pinctrl-bindings.txt
- st,hw-flow-ctrl: bool flag to enable hardware flow control.
- rs485-rts-delay, rs485-rx-during-tx, rs485-rts-active-low,
  linux,rs485-enabled-at-boot-time: see rs485.txt.
- dmas: phandle(s) to DMA controller node(s). Refer to stm32-dma.txt
- dma-names: "rx" and/or "tx"
- wakeup-source: bool flag to indicate this device has wakeup capabilities
- interrupt-names : Should contain "wakeup" if optional wake-up interrupt is
  used.
```
In short, whenever a standardized device tree node is specified for a new device, it is called a device tree binding and all of the properties and their meaning should be documented.

To find the documentation of a certain devicetree binding, ```grep``` the ```compatible``` string property in the ```doc``` directory of the sources.

*example*
```
ykhan@ykhan:~/uboot$ grep -snr "st,stm32h7-uart" doc/
doc/device-tree-bindings/serial/st,stm32-usart.txt:7:  - "st,stm32h7-uart".
```

## Devicetree files hierarchy
### .dtsi files
By convention, the devicetree source follows a certain hierarchy of devicetree include (.dtsi) and devicetree source (.dts) files. The dtsi files contains generic/ common data, that could be shared among many dts files. Each dts file include one or many dtsi files and make changes specific to that dts file. A bare minimum example of this heirarchy is the one consisting of a SoC based dtsi file and actual board based board.dts file.

*soc.dtsi*
```
/*
  Define the nodes for all the available devices on this chipset.
  Mark them disabled by default as not all of them could be supported
  by the actual board, based on this chipset.
*/
```

*board.dts*
```
#include <soc.dtsi>

/*
  Board specific customization to the soc.dtsi i.e.
  1. enable all the devices actually supported on this board
  2. set the board specific parameteres e.g. clock rate, available RAM,
  console uart etc.
*/
```
A dts file could also ```#include``` another dts file, if the board is similar to an existing board. For e.g. consider the below example of ```board_revB```, including the dts file for the previous revision ```board_revA```.
```
/*
    file board_revB.dts
*/

#include <board_revA.dts>

// Changes specific to the board revision B
```

### Device tree include
When a dtsi file is included in a dts file, the entire content of dtsi file is copied in the dts file. Based on the requirements, the dts file can add, remove modify or overwrite the nodes defined in the dtsi files.

See the below image for an example of dtsi being included in the board specific dts file.

![Overlay example](/images/dt_overlay.jpg)
[Image source](http://wiki.dreamrunner.org/public_html/Embedded-System/Files/DT-inclusion-example.jpg)

**As a general rule of thumb - latter definitions overwrite the earlier ones**

## Compiling a Devicetree
To compile a devicetree file (.dts), a dtc compiler is used which generates the output devicetree blob file (.dtb).

To install the dtc compiler on ubuntu 18.04.
```
$ apt-get install device-tree-compiler
```

Below is the command to compile an input dts file into an out dtb file.
```
$ dtc -I dts -O dtb -p out.dtb input.dts
```
Similarly, a dtb file can be decompiled into the source dts file via below command. This step can be useful for debugging purposes i.e. to see the final form of the input devicetree file after all the processing and substitutions, before it is compiled.
```
$ dtc -I dtb -O dts -o out.dts input.dtb
```

## Mapping a dts node and its driver
The mapping between a devicetree node and it's corresponding driver is done via compatible string. A devicetree node specifies a ```compatible``` string property, which should also be defined in any driver that handles this device. An easy way to find the drivers for a devicetree device is by simply doing a ```grep``` with the ```compatible``` string inside the source code. It will identify all the drivers/source files, which handle the corrsponding device. For example, consider the below uart dts node.
```
usart2: serial@4000e000 {
    compatible = "st,stm32h7-uart";
    reg = <0x4000e000 0x400>;
    ...
};
```
To find the uart driver handling this device, run the below grep command inside uboot source directory.
```
ykhan@ykhan:~/uboot$ grep -snr "st,stm32h7-uart" drivers/
drivers/serial/serial_stm32.c:221:	{ .compatible = "st,stm32h7-uart", .data = (ulong)&stm32h7_info},
```
The output of the above command shows that the driver source file is drivers/serial/serial_stm32.c.

## Enabling/disabling a device
To enable/disable a device, the ```status``` property inside a devicetree node is used.

* To enable a device, it should be set as ```status = "okay"```
* To disable a device, it should be set as ```status = "disabled"```

During parsing stage, the devices with ```status``` property set as ```"disabled"``` are skipped and only device with ```status``` set as ```"okay"``` are parsed for further processing.

Moreoever, by default, all the devices present on a SoC are disabled in a SoC dtsi file and each device present on a board based on that SoC should be explicity enabled in the board dts file.
