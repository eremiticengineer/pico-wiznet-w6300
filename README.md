# Pico WIZnet W6300-EVB-Pico2

Small Pico SDK C++ wrapper/example for the WIZnet W6300-EVB-Pico2 using FreeRTOS, WIZnet `ioLibrary_Driver`, the WIZnet Pico QSPI/PIO port, DHCP/DNS, and Mbed TLS.

## Cloning the project

```bash
git clone https://github.com/eremiticengineer/pico-wiznet-w6300

cd pico-wiznet-w6300

git submodule update --init lib/FreeRTOS-Kernel lib/ioLibrary_Driver lib/WIZnet-PICO-FREERTOS-C

cp examples/config/secrets.example.hpp examples/config/secrets.hpp
```

## Hardware

W6300-EVB-Pico2 uses the board-wired QSPI interface driven by the RP2350 PIO hardware.

The W6300 interface uses:

* GPIO15 INT
* GPIO16 CSn
* GPIO17 SCLK
* GPIO18 IO0
* GPIO19 IO1
* GPIO20 IO2
* GPIO21 IO3
* GPIO22 RESET

Those pins are unavailable to other peripherals while Ethernet is active.

The project uses the W6300 in quad QSPI mode through the WIZnet Pico PIO driver.

## Secrets

Edit:

```text
examples/config/secrets.hpp
```

and replace the placeholder certificate in:

```text
examples/config/WebServerCertificate.hpp
```

## Build

```bash
./build_project
```

## Flash

```bash
./deploy_usb
```

## Notes

A conceptual overview of the data journey from the Pico 2, through the W6300 to the server and back is provided in [PROTOCOL.md](PROTOCOL.md).

The RP2350 handles application logic, HTTP, TLS, certificate validation, and cryptography.

The W6300 handles the lower network layers, including TCP, IPv4/IPv6, Ethernet framing, MAC, and PHY operation.

Communication between the RP2350 and W6300 is performed over QSPI using RP2350 PIO.

## Creating from new

```bash
git submodule add https://github.com/raspberrypi/FreeRTOS-Kernel.git lib/FreeRTOS-Kernel

git submodule add https://github.com/Wiznet/ioLibrary_Driver.git lib/ioLibrary_Driver

git submodule add https://github.com/WIZnet-ioNIC/WIZnet-PICO-FREERTOS-C.git lib/WIZnet-PICO-FREERTOS-C
```

Initialise the submodules with:

```bash
git submodule update --init
```

## Acknowledgments

chatgpt was consulted with extensively and was used to create the code rather than use
an AI Agent such as codex. Using chatgpt allows understanding the codebase and the decisions
made on the architecture which lets the developer guide the implementation rather than
issuing commands and flashing the compiled binary without fully understanding the architecture
and decisions that led to it.

[PROTOCOL.md](PROTOCOL.md) was created after lengthy discussions and learning using chatgpt.

## References

* [W6300-EVB-Pico2](https://wiznet.io/products/evaluation-boards/w6300-evb-pico2)
* [W6300 documentation](https://docs.wiznet.io/Product/iEthernet/W6300/w6300-evb-pico2)
* [WIZnet ioLibrary_Driver](https://github.com/Wiznet/ioLibrary_Driver)
* [WIZnet Pico FreeRTOS port](https://github.com/WIZnet-ioNIC/WIZnet-PICO-FREERTOS-C)
* [FreeRTOS Kernel](https://github.com/raspberrypi/FreeRTOS-Kernel)
* [Mbed TLS](https://mbed-tls.readthedocs.io/)
* [Embedded C Coding Standard](https://barrgroup.com/embedded-c-coding-standard)
