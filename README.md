[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]

# SX1255_HAT-hw
Raspberry Pi Zero shield with the SX1255 RF transceiver chip.

Revision B. Work in progress. Do not order.

## Revision B

Revision B is for series production. Changes from revision A:

- All parts come from the shared [M17 KiCad library](https://github.com/M17-Project/M17_kicad_library)
  (git submodule `lib/`). Each part has its manufacturer, MPN, Mouser and LCSC numbers.
- 4 layer board, JLCPCB stackup `JLC04101H-7628` (1.0 mm). The RF lines are 50 Ω
  grounded coplanar waveguides on the top layer over a ground plane.
- Passive parts are 0402. The RF inductors and capacitors are Murata high Q parts.
- ID EEPROM (AT24C32D, address 0x50) for the Raspberry Pi HAT format.
  The ID_SD and ID_SC pull-ups are 3.9 kΩ and the WP pull-up is 1 kΩ, as the
  Raspberry Pi HAT design guide specifies. Connect test point TP4 (WP) to TP5 (GND) to write the EEPROM.
- The RF supply (+3V3_RF) comes from a low noise LDO (TPS7A2033) on the 5 V pins of the GPIO header.
- A 100 pF capacitor connects the TCXO output to the SX1255 XTB pin, as in the Semtech reference design.
- Low cost Adam Tech RF2-04A-T-00-50-G SMA connectors, as in revision A.

The GPIO header pinout is the same as in revision A.

| Signal | GPIO header pin | Raspberry Pi function |
|---|---|---|
| SX1255 SCK, MOSI, MISO | 23, 19, 21 | SPI0 |
| SX1255 NSS | 24 | SPI0 CE0 (GPIO8) |
| SX1255 RESET | 22 | GPIO25 |
| SX1255 CK_OUT | 12 | PCM CLK (GPIO18) |
| SX1255 DIO2 | 35 | PCM FS (GPIO19) |
| SX1255 I_OUT | 38 | PCM DIN (GPIO20) |
| SX1255 I_IN | 40 | PCM DOUT (GPIO21) |
| ID EEPROM | 27, 28 | ID_SD, ID_SC |

> [!NOTE]
> In case the TCXO used in the project is not available, `TG2016SMN 32.0000M-MCGNNM0` is a valid replacement (0.5ppm frequency stability).
> Alternatively, `ECS-TXO-20CSMV-320-EY` can be used (2.5ppm frequency stability).

## Basic specifications
| Parameter                       | Value                                       |
|---------------------------------|---------------------------------------------|
| Tuning range                    | 400 to 510MHz (356 to 562MHz out-of-spec)   |
| RF output power                 | +5dBm typ. (1dB compression point)          |
| Baseband interface              | I2S (master)                                |
| Frequency synthesizer step size | ~30.5Hz                                     |
| RX gain range                   | 70dB                                        |

See the [SX1255 datasheet](https://www.semtech.com/products/wireless-rf/lora-core/sx1255) for more details.

## Get the files

Clone the repository with the library submodule:

```sh
git clone --recursive https://github.com/M17-Project/SX1255_HAT-hw.git
```

If you cloned without `--recursive`, get the library:

```sh
git submodule update --init
```

## Build

Install KiCad 10 and KiKit. Then run:

```sh
make
```

The build checks the design first. Any ERC or DRC violation stops the build.
Each part on the BOM must have a Mouser number. The outputs are in `build/`.

To use the same tools as the CI, run the build in Docker:

```sh
make docker
```

To update the library, run `make lib-update`.

## Usage
### RPi sample config code
Sample Raspberry Pi code for SX1255 control can be found [here](https://github.com/M17-Project/LinHT-utils/tree/main/sx1255).<br>
[I2S slave overlay](https://github.com/AkiyukiOkayasu/RaspberryPi_I2S_Slave) is required. Sample rate for both RX and TX can be set to 125, 250, or 500kHz.

RX/TX frequency setting (500kHz sample rate):
```
./sx1255-spi -s 500 -r 435000000 -t 438000000
```

### Example RX with ZeroMQ
```
arecord -t raw -f S32_LE -r 192000 -c 2 --device="hw:0,1" | ./int_to_short | ./zmq-pub
```

In which `int_to_short` is a simple `int` to `short` format converter:
```C
while(1)
{
  ioctl(STDIN_FILENO, FIONREAD, &rcvd);
  if(rcvd>0)
  {
    if(read(STDIN_FILENO, inp, rcvd)==rcvd)
    {
      for(int i=0; i<rcvd; i+=8)
      {
        memcpy(&out[i/2  ], &inp[i  +2], 2);
        memcpy(&out[i/2+2], &inp[i+4+2], 2);
      }

      write(STDOUT_FILENO, out, rcvd/2);
    }
  }
}
```

### Example TX with ZeroMQ
```
./zmq-sub 192.168.50.243:17018 | aplay -t raw -f S32_LE -r 192000 -c 2 --device="hw:0,0"
```

## Example GNU Radio flowgraphs
There are 2 demo flowgraphs available:
- [*cari_sx1255_rx_demo.grc*](https://github.com/M17-Project/CARI/blob/main/gr_demos/cari_sx1255_rx_demo.grc)
- [*cari_sx1255_multicarrier_tx_demo.grc*](https://github.com/M17-Project/CARI/blob/main/gr_demos/cari_sx1255_multicarrier_tx_demo.grc)

The RX demo also provides a set of sinc compensating FIR filter taps through the `Decimating FIR Filter` block.

## License
This work is licensed under a
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa]

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg
