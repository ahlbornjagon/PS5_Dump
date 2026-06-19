# PS5_Dump
Not really sure what im trying to do atm, but heres the PS5 firmware I pulled :) 

## My setup
- [Tigard Interface Tool]((https://www.mouser.com/ProductDetail/Securing-Hardware/TIGARD-V1?qs=aP1CjGhiNiFnjSEE%2FnXyEw%3D%3D&srsltid=AfmBOooH5jOMCDTLE95DhnocCY4HxXkiPbUnKPJEHd9VonJOJryouzSg))
- PS5 Board with intact flash
- USB Digital Microscope (Optional, used for pictures and part numbers)
- 8 pin SPI clip with SOP16/8 to DIP8 Breakout board
- Breadboard
- Jumper cables (M2M, F2F, M2F)
- MacOS Tahoe with:
    - Flashrom: `brew install flashrom`
    - Libftdi (if not installed): `brew install libftdi`
- Basic binary analysis tools `strings`, `xxd`, `dd`, `binwalk`
- Ghidra for deeper binary analysis (TBD)

## PS5_nor process
1) Identified PS_Nor flash chip as `winbond 25Q16JVNIM`, link to data sheet [here](https://www.digikey.com/en/products/detail/winbond-electronics/W25Q16JVUXIQ-TR/15182017?gclsrc=aw.ds&gad_source=1&gad_campaignid=120565755&gbraid=0AAAAADrbLliHSu34b5Nj9GE-Z5bIe5o__&gclid=Cj0KCQjwrs7RBhDuARIsAIVfBD3IoxU_W4c1-ybE17Pcejw0p3QaBs6kXyCxctW2vPSR23RBtM_ljVoaAtIVEALw_wcB)
2) Identified Pins on Winbond as shown in next section
3) Connected SOIC8 Clip onto flash chip lining up pin 1
4) Run the following command to pull the firmware: 
    - **NOTE**: This part may require you to adjust the clip to ensure a secure connection. 
    - I noticed that with correct wiring but poor clip placement, it would attempt to read but fail on the ID check. 
    - If it fails to find your chip, i.e `No EEPROM/flash device found` your connections are most likely incorrect.
    - I specicially needed the `divisor=8` option as capacitance from the clip distorts the signal at high frequencies. This option essentially slows the clock to allow for more accurate capture. 

``` 
sudo flashrom -p ft2232_spi:type=2232H,port=B,divisor=8 -r dump.bin
```

5) You should see something similar if it was successful:
```
flashrom v1.7.0 on Darwin 25.3.0 (arm64)
flashrom is free software, get the source code at https://flashrom.org

Found Winbond flash chip "W25Q16JV_M" (2048 kB, SPI) on ft2232_spi.
===
... 

Reading flash... done.
```
7) You should now have dump.bin available for whatever activities you have planned :)

**See `/images` for pictures of setup and NOR Chip**

------------------------------------------------------------------------------------------------
### PS5_nor Connection diagram


```
SOIC8 CLIP (8-pin)          SOP16/8 to DIP8 Breakout
┌─────────────┐             ┌─────────────────┐
│ 1  CS   ────┼─────────────┼── CS   (pin 1)  │
│ 2  DO   ────┼─────────────┼── DO   (pin 2)  │
│ 3  WP   ────┼─────────────┼── WP   (pin 3)  │
│ 4  GND  ────┼─────────────┼── GND  (pin 4)  │
│ 5  DI   ────┼─────────────┼── DI   (pin 5)  │
│ 6  CLK  ────┼─────────────┼── CLK  (pin 6)  │
│ 7  HOLD ────┼─────────────┼── HOLD (pin 7)  │
│ 8  VCC  ────┼─────────────┼── VCC  (pin 8)  │
└─────────────┘             └────────┬────────┘
                                     │
                            ┌────────┴────────────────────────┐
                            │         BREADBOARD              │
                            │                                 │
                            │  VTG ──┬── VCC  (pin 8)        │
                            │        ├── HOLD (pin 7)        │
                            │        └── WP   (pin 3)        │
                            │                                 │
                            └────────┬────────────────────────┘
                                     │
                            ┌────────┴────────┐
                            │  TIGARD (SPI)   │
                            │                 │
                            │  COPI ── DI     │
                            │  CIPO ── DO     │
                            │  SCK  ── CLK    │
                            │  CS   ── CS     │
                            │  GND  ── GND    │
                            │  VTG  ──────────┼── Breadboard VTG rail
                            └─────────────────┘
                            
```

### RE Work
- Strings on `PS5_nor.bin` produced almost 2 million entries.
- Opening up in a hexeditor I noticed
```
SLB2        -> 16385 *Something* Load booter? 
C0000001    -> 16432 *???
C0008001    -> 16480 *???
```
- Checking for all instances of SLB with `strings -o PS5_nor.bin | grep -i "SLB"`
```
  16384 SLB2    -> 0x4000
 165521 slbv    -> 0x28691
 532480 SLB2    -> 0x82000
 681617 slbv    -> 0xA6691
1048576 SLB2    -> 0x100000
1114112 SLB2    -> 0x110000
```

- Lets check whats at all SLB2's
    - 0x4000: Nothing, just C0000001 & C0008001 (*Maybe address?*)
    - 0x82000: Same as above
    - 0x100000 & 0x110000: C0050100 & C0058100, two new values (address?)

xxd -s 0x110000 -l 112 PS5_nor.bin
00100000: 534c 4232 0300 0000 0000 0000 0200 0000  SLB2............
00100010: 3e00 0000 0000 0000 0000 0000 0000 0000  >...............
00100020: 0100 0000 a077 0000 0000 0000 0000 0000  .....w..........
00100030: 4330 3035 3031 3030 0000 0000 0000 0000  C0050100........
00100040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00100050: 3d00 0000 0c00 0000 0000 0000 0000 0000  =...............
00100060: 4330 3035 3831 3030 0000 0000 0000 0000  C0058100........
00110000: 534c 4232 0300 0000 0000 0000 0200 0000  SLB2............
00110010: 3e00 0000 0000 0000 0000 0000 0000 0000  >...............
00110020: 0100 0000 a077 0000 0000 0000 0000 0000  .....w..........
00110030: 4330 3035 3031 3030 0000 0000 0000 0000  C0050100........
00110040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00110050: 3d00 0000 0c00 0000 0000 0000 0000 0000  =...............
00110060: 4330 3035 3831 3030 0000 0000 0000 0000  C0058100........

// Next step: Need to figure out what the SLB2 is, looks like theres a header at the beginning so we need to try and guess what the size is so we can get a clean extract. Multiple SLB2 headers may hint at a blob of boot stages that we need to unpack