# ZMK PAT9125 Driver

This driver enables the use of the PIXART PAT9125 miniature optical navigation sensor with the ZMK(v0.3) framework.

## Overview

The PAT9125EL is a low-power laser optical tracking chip with wide DOF, glossy surface tracking, and no calibration.
This driver communicates with the PAT9125 sensor via I2C interface.

**This module has just been transplanted from Zephyr4.1.**
Utilization only with ZMK v0.3 is recommended.

## Acknowledgments

Currently, it requires [badjeff](https://github.com/badjeff)'s [zmk-input-processor-report-rate-limit](https://github.com/badjeff/zmk-input-processor-report-rate-limit) to work smoothly.
This porting is highly inspired by [sekigon-gonnoc](https://github.com/sekigon-gonnoc)'s [zmk-driver-paw3222](https://github.com/sekigon-gonnoc/zmk-driver-paw3222).

## Installation

1. Add as a ZMK module in your west.yml:

```
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    ...
    # START #####
    - name: badjeff
      url-base: https://github.com/badjeff
    - name: BP
      url-base: https://github.com/taichan1113
    # END #####
    ...
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3
      import: app/west.yml
    ...
    # START #####
    - name: zmk-input-processor-report-rate-limit
      remote: badjeff
      revision: main
    - name: zmk-driver-pat9125
      remote: BP
      revision: main
    # END #####
    ...
  self:
    path: config
```

## Device Tree Configuration

Configure in your shield or board config file (`.overlay` or `.dtsi`):

```dts
&pinctrl {
	i2c1_default: i2c1_default {
		group1 {
			psels = <NRF_PSEL(TWIM_SDA, 0, 4)>,
				<NRF_PSEL(TWIM_SCL, 1, 13)>;
		};
	};

	i2c1_sleep: i2c1_sleep {
		group1 {
			psels = <NRF_PSEL(TWIM_SDA, 0, 4)>,
				<NRF_PSEL(TWIM_SCL, 1, 13)>;
			low-power-enable;
		};
	};
};

#include <zephyr/dt-bindings/input/input-event-codes.h>
#include <input/processors.dtsi>
#include <input/processors/report_rate_limit.dtsi>

&i2c1 {
	status = "okay";
	compatible = "nordic,nrf-twim";
	pinctrl-0 = <&i2c1_default>;
	pinctrl-1 = <&i2c1_sleep>;
	pinctrl-names = "default", "sleep";

	trackball: trackball@79 {
		status = "okay";
		compatible = "pixart,pat9125";
		reg = <0x79>; // HIGH:0x73, LOW:0x75, NC:0x79
		motion-gpios = <&gpio0 5 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
		res-x-cpi = <300>;
		res-y-cpi = <300>;
		zephyr,axis-x = <INPUT_REL_X>;
		zephyr,axis-y = <INPUT_REL_Y>;
		// invert-x;
		invert-y;
		sleep1-enable;
		sleep2-enable;
	};
};

// example for using it on peripheral side
/{
  split_inputs {
    #address-cells = <1>;
    #size-cells = <0>;
    
    split_peripheral: split_peripheral@0 {
      status = "okay";
      compatible = "zmk,input-split";
      reg = <0>;
      device = <&trackball>;
      input-processors = <&zip_report_rate_limit 16>;
    };
  };
};
```

## Enable the module in your keyboard's Kconfig file

Add the following to your keyboard's `.conf`:

```conf
CONFIG_I2C=y
CONFIG_PINCTRL=y
CONFIG_INPUT=y
CONFIG_ZMK_POINTING=y
CONFIG_PAT9125=y
```

## Properties

- `motion-gpios`: GPIO connected to the motion pin (required)
- `res-res-x-cpi`, `res-y-cpi`: CPI resolution for the sensor (optional)
