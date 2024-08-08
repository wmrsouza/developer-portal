---
title: "How to use ESP32 SoCs with PSRAM on Zephyr"
date: 2024-08-30T00:00:00-00:00
showAuthor: false
authors:
  - "marcio-ribeiro"
tags: ["ESP32", "ESP32-S2", "ESP32-S3", "Zephyr", "PSRAM", "SPIRAM"]
---

The aim of this article is guide you on how to configure Zephyr and how access external PSRAM memory whithin your application running on Espressif SoCs.

## 1. Introduction

Espressif SoCs typically feature a few hundred kilobytes of internal SRAM. However, this internal SRAM may be insufficient for certain applications. To address this limitation, some Espressif SoCs support the use of external PSRAM - Pseudostatic RAM - by mapping part of their virtual address space to this external memory. With certain restrictions, the external PSRAM can be incorporated into the memory map and used similarly to internal SRAM.

As of the time of writing, the SoCs capable of mapping external PSRAM on their virtual address space include those from the ESP32, ESP32-S2, ESP32-S3, ESP32-C61 and ESP32-P4 series, though only the first three are currently supported on Zephyr.

## 2. PSRAM from a Hardware Perspective

Each Espressif SOC series capable of mapping PSRAM into the virtual memory space includes part numbers feturing in-package PSRAM. These part numbers have an 'R' followed by a number that indicates the PSRAM capacity in megabytes. For example, the part number ESP32-S3R8 denotes that the SoC is from the ESP32-S3 series and includes 8 megabytes of in-package PSRAM.

External PSRAM can be incorporated into custom hardware designed around PSRAM-capable SoCs that do not have in-package PSRAM. PSRAM will share data I/O and clock lines with FLASH, but not chip select lines. PSRAM and FLASH have separated chip select lines. For information related on how to connect off-package PSRAM lines, the respective SoC datasheet must be consulted.

The following SPI interfaces are supported to access external RAM memories:
- Single SPI
- Dual SPI
- Quad SPI
- Octo SPI
- QPI
- OPI

After the SoC is initialized, firmware can customize the mapping of external RAM into the CPU address space. Depending on the SoC series, external RAM can be mapped into both instruction space and data space.

## 3. PSRAM from a Software Perspective

This article will show how to use PSRAM memory available on ESP32 family using Zephyr RTOS. The external PSRAM memory can be made available to apllications in several ways:
- Dynamic allocation
- Integration to the memory map
- Allowing execution in place (XIP)

### 3.1 PSRAM Dynamic Allocation

PSRAM memory blocks can bem made available to applications through the Zephyr shared multi-heap library. The siihared multi-heap memory pool manager uses the multi-heap allocator to manage a set of reserved memory regions with different capabilities and attributes. In the case of PSRAM, enabling the `ESP_SPIRAM` and `SHARED_MULTI_HEAP` config parameters, (during Zephyr's early boot stage) the external RAM is mapped to data virtual space, and the shared multi-heap framework is initialized, adding this memory region to the pool.

When the application runs and needs memory allocated from PSRAM, it can call the function `shared_multi_heap_alloc()` (or the aligned version `shared_multi_heap_aligned_alloc()`) whith the attribute `SMH_REG_ATTR_EXTERNAL` that will return an address pointing to a block of memory from the PSRAM. Whith the ownership of this memory block the application is granted to read and write to its positions. Once the memory block is no longer necessary, it can be returned to the pool from which it was allocated by calling the `shared_multi_heap_free()` function.

To install the Zephyr RTOS and the necessary tools, follow the instructions found in the Zephyr [Getting Started Guide](https://docs.zephyrproject.org/latest/develop/getting_started/index.html). At the end of the process you will have set up a command-line Zephyr development environment ready to build your application.

Additionally, it is necessary to execute the following command to have your environment prepared to build applications to Espressif SoCs:

```sh
west blobs fetch hal_espressif
```
While building applications that make use o PSRAM memory it is necessary to inform to `west` the mode used to access the PSRAM attached to the SoC. `west` is the meta-tool responsible for the build process in Zephyr.

Lets build a Zephyr sample who tests dynamic allocation of PSRAM memory using shared multi-heap api targeted to a `ESP32-S3-DevKitC` board that includes a Octo SPI PSRAM:

```sh
west build -b esp32s3_devkitc/esp32s3/procpu zephyr/samples/boards/esp32/spiram_test/ -DCONFIG_SPIRAM_MODE_OCT=y --pristine
```

This command will create a directory called `build`, which will contain the binary file of the test application, along with other intermediate files produced during the build process.

To flash the application binary onto the `ESP32-S3-DevKitC` and see the messaged printed by the application on tbe console, type the following commands:

```sh
west flash
west espressif monitor
```

The resulting messages are the following:

```sh
SPIRAM mem test pass
Internal mem test pass
ESP-ROM:esp32s3-20210327
Build:Mar 27 2021
rst:0x1 (POWERON),boot:0x8 (SPI_FAST_FLASH_BOOT)
SPIWP:0xee
mode:DIO, clock div:2
load:0x3fc8dd94,len:0x1ac8
load:0x40374000,len:0x9d74
SHA-256 comparison failed:
Calculated: 03b83f1c5042e02e19d1e05f9102a4e9a81d681ac452e9b3066d50128096373d
Expected: 0000000080470000000000000000000000000000000000000000000000000000
Attempting to boot anyway...
entry 0x40377af4
I (69) boot: ESP Simple boot
I (69) boot: compile time Sep  5 2024 23:29:24
W (70) boot: Unicore bootloader
I (70) spi_flash: detected chip: gd
I (71) spi_flash: flash io: dio
W (74) spi_flash: Detected size(8192k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (86) boot: chip revision: v0.2
I (89) boot.esp32s3: Boot SPI Speed : 40MHz
I (93) boot.esp32s3: SPI Mode       : DIO
I (96) boot.esp32s3: SPI Flash Size : 8MB
I (100) boot: Enabling RNG early entropy source...
I (105) boot: DRAM: lma 0x00000020 vma 0x3fc8dd94 len 0x1ac8   (6856)
I (111) boot: IRAM: lma 0x00001af0 vma 0x40374000 len 0x9d74   (40308)
I (117) boot: padd: lma 0x0000b878 vma 0x00000000 len 0x4780   (18304)
I (123) boot: IMAP: lma 0x00010000 vma 0x42000000 len 0x3ef0   (16112)
I (130) boot: padd: lma 0x00013ef8 vma 0x00000000 len 0xc100   (49408)
I (136) boot: DMAP: lma 0x00020000 vma 0x3c010000 len 0x1328   (4904)
I (142) boot: Image with 6 segments
I (145) boot: DROM segment: paddr=00020000h, vaddr=3c010000h, size=01330h (  4912) map
I (153) boot: IROM segment: paddr=00010000h, vaddr=42000000h, size=03EEEh ( 16110) map

I (172) octal_psram: vendor id    : 0x0d (AP)
I (172) octal_psram: dev id       : 0x02 (generation 3)
I (173) octal_psram: density      : 0x03 (64 Mbit)
I (175) octal_psram: good-die     : 0x01 (Pass)
I (179) octal_psram: Latency      : 0x01 (Fixed)
I (183) octal_psram: VCC          : 0x01 (3V)
I (187) octal_psram: SRF          : 0x01 (Fast Refresh)
I (192) octal_psram: BurstType    : 0x01 (Hybrid Wrap)
I (197) octal_psram: BurstLen     : 0x01 (32 Byte)
I (202) octal_psram: Readlatency  : 0x02 (10 cycles@Fixed)
I (207) octal_psram: DriveStrength: 0x00 (1/1)
I (211) esp_psram: Found 8MB PSRAM device
I (215) esp_psram: Speed: 40MHz
I (948) esp_psram: SPI SRAM memory test OK
*** Booting Zephyr OS build v3.7.0-2052-g0237d375de5f ***
SPIRAM mem test pass
Internal mem test pass
```

To build the same sample targeted to a `ESP32-DevKitC-WROVER` board that includes a Quad SPI PSRAM:

```sh
west build -b esp32_devkitc_wrover/esp32/procpu zephyr/samples/boards/esp32/spiram_test/ -DCONFIG_SPIRAM_MODE_QUAD=y --pristine
```

prj.conf:

```sh
CONFIG_ESP_SPIRAM=y
CONFIG_SHARED_MULTI_HEAP=y
```

main.c:
```c
#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <soc/soc_memory_layout.h>
#include <zephyr/multi_heap/shared_multi_heap.h>

int main(void)
{
  uint32_t *p_mem;

  p_mem = shared_multi_heap_aligned_alloc(SMH_REG_ATTR_EXTERNAL, 32, 1024*sizeof(uint32_t));

  if (p_mem == NULL) {
    prink("Error: allocation PSRAM memory.");
    retunr -ENOMEM;
  }

  for (int i=0; i<1024; i++) {
    p_mem[i] = i;
  }

  for (int i=0; i<1024; i++) {
    if (p_mem[i] != i) {
      prink("Error: checking memory contents.");
      break;
    }
  }

	shared_multi_heap_free(m_ext);

	return 0;
}
```

### _.2. As heap for memory allocation of WiFi and NET stack

This will increase RAM memory space available for application stack.

`CONFIG_ESP32_WIFI_NET_ALLOC_SPIRAM=y`

dd
### _.3. As memory section for variable static allocation

Implies changes on linker script for section definition

https://medium.com/@mkklyci/an-introduction-to-linker-files-crafting-your-own-for-embedded-projects-60ad17193229


### _.4. To execute external applications

It is possible to execute code from SPIRAM

### _.4.1 How to build application to run from SPIRAM

- PIC -> Position independent code

- Linker script with code and bss segments located located in SPIRAM memory region

https://docs.zephyrproject.org/latest/kernel/code-relocation.html


## _. Conclusion

## _	. References

https://www.espressif.com/en

https://www.zephyrproject.org/

https://docs.zephyrproject.org/latest/index.html

https://docs.zephyrproject.org/latest/develop/getting_started/index.html

https://docs.zephyrproject.org/latest/develop/beyond-GSG.html

https://docs.zephyrproject.org/latest/boards/espressif/esp32s3_devkitc/doc/index.html

https://docs.zephyrproject.org/latest/kernel/memory_management/shared_multi_heap.html
