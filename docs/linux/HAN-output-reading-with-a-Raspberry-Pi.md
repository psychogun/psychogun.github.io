---
layout: default
title: HAN output reading with a Raspberry Pi
parent: Linux
nav_order: 5 
---

# How to use a Raspberry Pi to read the HAN output from your smart meter
{: .no_toc }
I wanted to use a Raspberry Pi 3 Model B to have output reading from a smart meter.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Getting started
This is how I used a Raspberry Pi to have a output reading from my smart meter, using the HAN port.

### Prerequisites
* Aidon HAN Adapter with an RJ45 connector
* Raspberry Pi 3 Model B

## HAN interface
The Aidon RF2 System Modules have a physical HAN interface that can be taken into use by external devices according to the M-Bus standard (EN 13757-2). The RJ45 connector on the System Modules is either integrated or can be wired outside the Aidon ESD with an HAN adapter.

On the RJ45 connector, 2 pins are used for HAN:
* RJ45 PIN1: +24V M-bus TX
* RJ45 PIN2: GND

The interface supplies power to a connected HAN device up to 700 mW. The interface is protected against short circuits. The Aidon System Module software can turn the power off from the interface in case of fault current.

https://www.rapidtables.com/calc/electric/watt-volt-amp-calculator.html
230 Volts * 4 Amps = 920 W
24 Volts * X Amps = 0.7W
24 Volts * 0.02916666666667 Amps

5 V * 2.5 Amps = 12,5W.

https://www.raspberrypi.org/documentation/hardware/raspberrypi/README.md
https://www.raspberrypi.org/products/raspberry-pi-zero/


## Authors
Mr. Johnson

## Acknowledgments
* [https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-power]https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-power
* [https://github.com/turbokongen/hass-AMS](https://github.com/turbokongen/hass-AMS)
* [https://aliexpress.ru/item/32719562958.html?spm=a2g0s.9042311.0.0.c8314c4dpbv1pv](https://aliexpress.ru/item/32719562958.html?spm=a2g0s.9042311.0.0.c8314c4dpbv1pv)
* [https://www.nek.no/wp-content/uploads/2018/11/Aidon-HAN-Interface-Description-v10A-ID-34331.pdf](https://www.nek.no/wp-content/uploads/2018/11/Aidon-HAN-Interface-Description-v10A-ID-34331.pdf)
* [https://www.nek.no/wp-content/uploads/2018/10/Kamstrup-HAN-NVE-interface-description_rev_3_1.pdf](https://www.nek.no/wp-content/uploads/2018/10/Kamstrup-HAN-NVE-interface-description_rev_3_1.pdf)
* [https://norgesnett.no/wp-content/uploads/2019/01/Datablad-HAN-modul.pdf](https://norgesnett.no/wp-content/uploads/2019/01/Datablad-HAN-modul.pdf)
* [https://byggebolig.no/imageoriginals/88b3d1774ecb41e6a3fe067ae9e6a893.pdf](https://byggebolig.no/imageoriginals/88b3d1774ecb41e6a3fe067ae9e6a893.pdf)