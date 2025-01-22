# The Plover HID protocol

## Background

Currently most hobbyist steno machines use either the TXBolt protocol or the
GeminiPR protocol, which are serial protocols running over USB CDC. The main
reason why many end users want to use the steno protocols instead of just
having their machines act like keyboards is that they don't want to
enable/disable plover all the time to be able to use their keyboards.

However there are certain issues with those protocols:

1. There is no serial profile for BLE, so any bluetooth low energy device would
   need a custom protocol (GATT Service).
2. GeminiPR and TXBolt only send complete chords to the host, this makes us
   lose information that could be used on the plover side to easily implement
   features such as first-up chord send or auto-repeat of chords.
3. TXBolt and GeminiPR define a very restricted amount of keys. This isn't an
   issue for English Stenotype, but makes it so that something like a velotype
   machine won't be able to use those protocols.
4. Plover handles serial disconnects pretty badly, causing the user to have to
   reconfigure their serial port in settings. This is not a big issue but has
   led to some user confusion in the plover discord server.
5. There has been talk about creating hobbyist machines supporting variable
   key-pressure/lever-pressure. Many professional machines have configurable
   per-key pressure settings, but we could also design a protocol to send the
   key pressure and handling the configuration on the plover side.

## Using HID
Arguably stenography input is Human Input, so making a steno machine a human
input device would make sense.  The pros of using HID are that it's supported
both over USB, and also over BLE.  By sending each state change instead of
whole chords we allow for having features like first up chord send and auto
repeat in plover. In the best of worlds we would have been able to just choose
a good HID page for steno, but unfortunately no such standard page exists.

So instead we choose a vendor-defined page, and choose a usage code that no one
else seems to use. It would of course be nice if this could be standardized in
the future, but that is probably out of reach for the amateur steno community.

It would also have been nice to just define the usage page and usage, and allow
the device itself to decide how its reports should be sent (how many buttons it
has, if it has pressure information, and so on), and then parse the hid
descriptor on the plover side. Unfortunately there isn't a good platform
independent way to retreive the HID descriptors from a device.

Instead we choose an approach where the Usage Page and Usage uniquely define a report
format that we call the "simple report format" consisting of a bitmap of 64 bits each
encoding if a certain key/lever is pressed or not.

## The HID Descriptor
```
{
     0x06, 0x50, 0xff,              // UsagePage (65360)
     0x0a, 0x56, 0x4c,              // Usage (19542)
     0xa1, 0x02,                    // Collection (Logical)
     0x85, 0x50,                    //     ReportID (80)
     0x25, 0x01,                    //     LogicalMaximum (1)
     0x75, 0x01,                    //     ReportSize (1)
     0x95, 0x40,                    //     ReportCount (64)
     0x05, 0x0a,                    //     UsagePage (ordinal)
     0x19, 0x00,                    //     UsageMinimum (Ordinal(0))
     0x29, 0x3f,                    //     UsageMaximum (Ordinal(63))
     0x81, 0x02,                    //     Input (Variable)
     0xc0,                          // EndCollection
}
```
Where the first byte of the Usage Page is 0xff (vendor-defined) and the second
byte is "S" and the usage bytes are "TN", together encoding the string "STN".

Due to the fact that we can't get the report descriptors in a platform-independent way
we fix the ReportID to 80 (0x50) so that we can distinguish the reports of devices sending
multiple reports on the same endpoint.

The descriptor was generated by the [HID Report Descriptor Compiler](https://github.com/nipo/hrdc) from the following python code:
```
from hrdc.usage import *
from hrdc.descriptor import *

usage = Usage('vendor-defined', 0xff504c56)
stenomachine = Collection(Collection.Logical, usage,
    Report(0x50, *(Value(Value.Input, ordinal.Ordinal(x), 1, logicalMin=0, logicalMax=1) for x in range(64))))

if __name__ == '__main__':
    compile_main(stenomachine)
```

A previous version of the protocol used the `Button` usage instead of the `Ordinal` usage. But linux HID stack also parsed those
as mouse-button pressed over BLE, while it didn't do so over USB. Even though this is probably a linux bug it's easier for us
to change the usage to one where that doesn't happen.

## The keys
For historical reasons we reuse the names of the standard steno keys, but we add extra keys so that we reach 64 keys in total.
The keys are ordered as follows:

```
   S1- S2- T- K- P- W- H- R- A- O- *1 *2 *3 *4 -E -U -F -R -P -B -L -G -T -S -D -Z #
```
and are then followed by key `X1-X37`

So the first bit/byte of the report (after the report type byte) maps to key
`S-` and the last bit/byte of the report maps to `X26`.

## Sample Firmware

Sample firmware for the georgi is included in this repository. It defines a default keymap of the form:
```
X1 S1- T- P- H- *1 *3 -F -P -L -T -D
X2 S2- K- W- R- *2 *4 -R -B -G -S -Z
          #  A   O  E  U  #
```

## Installing the plover-machine-hid plugin in Plover (Windows)

1. Get hidapi.dll. Go to https://github.com/libusb/hidapi/releases and download hidapi-win.zip. Open the zip file, navigate to the x64 folder (or x32 if you have a 32-bit OS, not common), and copy hidapi.dll to your Plover install `data` folder, usually `C:\Program Files\Open Steno Project\Plover 4.0.0rc2\data` (or, if you have the portable version of Plover, wherever you uncompressed it).
    
2. Follow the instructions from the Plover wiki (https://plover.wiki/index.php/Plugins#Plugins_not_on_PyPI) to install the plugin, which essentially tell you to install Git, open a terminal in your Plover install folder, and then run
```
.\plover_console.exe -s plover_plugins install git+https://github.com/dnaq/plover-machine-hid
```

3. Restart Plover. A new type of machine (Plover HID) should show up in the dropdown box.


## Plover HID Inspector

@jthlim has created a WebHID based tool for inspecting output from any Plover HID machine.
This tool can be used to help debug any Plover HID protocol issues if you're adding Plover HID to your own board. The tool can be found [here](https://lim.au/#/software/plover-hid) and required a web browser supporting WebHID.

## Future Work
- [x] Implement this protocol for QMK.
- [x] Implement this protocol for ZMK.
- [ ] Create a Usage Page/Usage for lever machines sending variable lever pressure.
- [ ] Look into using Output Reports for sending things to the steno machine from plover.
