# rpi-i2s
Using the ICS43432 MEMS microphone on a Raspberry Pi with i2s

There has been poor documentation online about using i2s on a RaspberryPi and in particular connecting a MEMs microphone.  There is plenty of discussion but no clear tutorial and/or explaination.  These notes are meant to be a comprehensive way of connecting a microphone to an RPi over i2s.

## Hardware Setup

The following documentation used the ICS43432 MEMs microphone with a breakout board on an RPi 3.  Mirophone documentation can be found [here](https://www.embeddedmasters.com/datasheets/embedded/EMMIC-ICS43432-DS.pdf).  Header pins were soldered to the breakout board.  Unfortunately the breakout board was poorly designed and in order to properly install the header pins, the pin labels were covered.  Regardless, the connection uses Pulse Code Modulation which requires four GPIO pins from the RPi.  The PCM setup can be found [here](https://pinout.xyz/pinout/pcm).  The connection is as follows:

- VCC - 3.3v
- G - Gnd
- L/R - Gnd (this is used for channel selection. Connect to 3.3 or GND)
- SCK - BCM 18 (pin 12)
- SD - BCM 19 (pin 35)
- WS - BCM 20 (pin 38)

## Software Requirements

The following is taken from [Paul Creaser's writeup](https://paulcreaser.wordpress.com/2015/11/01/mems-mic-module/).  I've added a bit more how-to description as well as fixed a few typo's in Paul's execution.  They're simple fiexes but for someone who's never compiled a kernal driver, debugging a typo can be annoying.

### i2s Configuration
Several files need to be modified.  

```
$ uname -a
Linux raspberrypi 4.4.14-v7+ #896 SMP Sat Jul 2 15:09:43 BST 2016 armv7l GNU/Linux

$ sudo nano /boot/config.txt    # uncomment dtparam=i2s=on
$ sudo nano /etc/modules        # add line snd-bcm2835

$ sudo reboot                   # reboot RPi

$ lsmod | grep snd              # confirm modules are loaded
snd_soc_simple_card     6790  0 
snd_soc_bcm2835_i2s     6354  2 
snd_soc_core          125885  2 snd_soc_bcm2835_i2s,snd_soc_simple_card
snd_pcm_dmaengine       3391  1 snd_soc_core
snd_bcm2835            20511  3 
snd_pcm                75698  3 snd_bcm2835,snd_soc_core,snd_pcm_dmaengine
snd_timer              19160  1 snd_pcm
snd                    51844  10 snd_bcm2835,snd_soc_core,snd_timer,snd_pcm
```
### Kernel Compiling

Paul provides some explaination about installing the proper gcc compiler.  I didnt have any propblems with this.

First update and install necessary dependencies:

```
$ sudo apt-get update
$ sudo rpi-update

$ gcc --version
gcc (Raspbian 4.9.2-10) 4.9.2

# mis dependencies that needed to be installed
$ sudo apt-get install bc
$ sudo apt-get install libncurses5-dev

# just for good measure
$ sudo apt-get update
$ sudo rpi-update
```
Get the kernel source and compile.  This takes a very long time on an RPi.  I believe there are ways of doing this on a local machine but I didnt try.  I would recommend installing the application '''sudo apt-get install screen'''.  

```
$ sudo whet https://raw.githubusercontent.com/rpi-source/master/rpi-source -O /usr/bin/rpi-source
$ sudo chmod +x /usr/bin/rpi-source
$ rpi-source
```

### Compile the i2S module

```
$ sudo mount -t debugfs debugs /sys/kernel/debug
$ sudo cat /sys/kernel/debug/asoc/platforms
3f203000.i2s
snd-soc-dummy
```
```
$ git clone https://github.com/PaulCreaser/rpi-i2s-audio
$ cd rpi-i2s-audio
$ make -C /lib/modules/$(uname -r )/build M=$(pwd) modules
$ sudo insmod my_loader.ko
```

Verify the module was loaded:

```
$ lsmod | grep my_loader
my_loader               1789  0 

$ dmesg | tail
[    9.777145] Bluetooth: HCI UART protocol H4 registered
[    9.777150] Bluetooth: HCI UART protocol Three-wire (H5) registered
[    9.777251] Bluetooth: HCI UART protocol BCM registered
[    9.946690] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[    9.946701] Bluetooth: BNEP filters: protocol multicast
[    9.946716] Bluetooth: BNEP socket layer initialized
[36934.903189] request module load 'bcm2708-dmaengine': 0
[36934.903444] register platform device 'asoc-simple-card': 0
[36934.903455] Hello World :)
[36934.921322] asoc-simple-card asoc-simple-card.0: snd-soc-dummy-dai <-> 3f203000.i2s mapping ok
```
