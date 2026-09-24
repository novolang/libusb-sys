# libusb-sys

libusb is a library for talking to USB devices from an ordinary
program, without writing a kernel driver. It runs on Linux, macOS,
Windows and the BSDs, and its C API is documented at
[the libusb-1.0 API reference](https://libusb.sourceforge.io/api-1.0/).
This package declares forty-three of its entry points to novo-lang, one
declaration each.

Every function here is a declaration of a function in libusb-1.0. The
package contains no logic of its own, and it does nothing without the C
library installed. The forty-three entry points are the ones a program
needs to find a device, read what it says about itself, open it, claim
an interface and move bytes in either direction. The section "What is
not included" says what a program cannot do with them alone.

## What it is

A **device** is a thing plugged into a USB bus. It is identified while
it stays plugged in by the number of its bus and its address on that
bus, and it is identified as a model by a **vendor identifier** and a
**product identifier**, two sixteen-bit numbers the USB Implementers
Forum assigns.

A **descriptor** is a block of bytes a device reports about itself, in
a layout the USB specification lays down. The **device descriptor** is
eighteen bytes and says what the device is. A **configuration
descriptor** describes one way the device can be set up, and a device
is in one configuration at a time.

An **interface** is a function of a device, and a device may have
several. A webcam may have one for video and one for audio. A program must
**claim** an interface before it may talk to it, and the operating
system may have given it to a kernel driver already.

An **endpoint** is one direction of one channel into a device. Its
address carries the direction in its top bit. 0x81 is an endpoint the
device sends on, and 0x01 is one it receives on. Endpoint zero is the control
endpoint, which every device has and no descriptor declares.

A **transfer** moves bytes over an endpoint. A **control** transfer
carries a request in a fixed eight-byte header and is how a device is
configured. A **bulk** transfer carries a lot of data with no timing
guarantee. An **interrupt** transfer is small and polled on a schedule
the device asked for. The fourth kind, **isochronous**, is not in this
package, because libusb offers it only as an asynchronous transfer.

A **context** is one independent use of libusb inside a program, with
its own devices and settings. Every device list and every handle
belongs to one. A program that does not need its own may pass zero and
use the default context.

## Install

```
novo pkg add libusb-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package
`libusb-1.0-0-dev`:

```
sudo apt install libusb-1.0-0-dev
```

On macOS the Homebrew formula is `libusb`. On other systems the library
builds from the libusb source.

Opening a device needs permission. On Linux a device node belongs to
root by default, and a program that is not root gets
`LIBUSB_ERROR_ACCESS` from `libusb_open`. The usual answer is a udev
rule that gives the device to a group. Listing the devices and reading
their descriptors needs no permission on Linux.

## Example

Every attached device listed, with its vendor and product:

```novo ignore
use libusb

fn main() [io, ffi]
    let slot = ptr.alloc_word()
    if libusb.libusb_init(slot) != 0
        println("libusb would not start")
        return
    let ctx = ptr.read_word(slot)

    // The list is an array of addresses, one every eight bytes.
    let list_slot = ptr.alloc_word()
    let n = libusb.libusb_get_device_list(ctx, list_slot) as i32
    let devs = ptr.read_word(list_slot)
    println("${n} device(s)")

    let desc = ptr.alloc(18)
    let i = 0
    while i < n
        let dev = ptr.read_word(devs + i * 8)
        // The descriptor is cached, so this sends no request to the device.
        let _ = libusb.libusb_get_device_descriptor(dev, desc)
        // Bytes 8 and 9 are the vendor, and 10 and 11 the product.  On a
        // little-endian machine the low byte comes first.
        let vendor = ptr.load_u8(desc + 8) + ptr.load_u8(desc + 9) * 256
        let product = ptr.load_u8(desc + 10) + ptr.load_u8(desc + 11) * 256
        // A bus number and an address are one byte each.
        let bus = libusb.libusb_get_bus_number(dev) % 256
        let addr = libusb.libusb_get_device_address(dev) % 256
        println("bus ${bus} address ${addr}: ${vendor}:${product}")
        i = i + 1

    ptr.free(desc)
    libusb.libusb_free_device_list(devs, 1)
    ptr.free(list_slot)
    ptr.free(slot)
    libusb.libusb_exit(ctx)
```

The example is not compiled, because it links against libusb-1.0 and
the link fails where that library is not installed. The same calls are in `tests/libusb_tests.nv`,
where the device list and the descriptor are asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libusb` | Every entry point, in six groups: the context, the library itself, the device list, the descriptors, the open device and the synchronous transfers. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The context | 3 | Creates and destroys a context and sets its logging level. |
| The library | 5 | Reports the version and the capabilities, chooses the language and names an error. |
| The device list | 12 | Lists the attached devices and answers where each one is and how fast it runs. |
| The descriptors | 4 | Reads the device and configuration descriptors a device reports. |
| The open device | 16 | Opens a device, configures it, claims an interface and deals with the kernel driver. |
| The transfers | 3 | Moves bytes over a control, bulk or interrupt endpoint. |

## How to choose an entry point

`libusb_get_device_list` is for a program that is looking for a device.
It lists everything attached, and reading a listed device's
descriptors sends no request to the device. On Linux none of this needs
permission.

`libusb_open_device_with_vid_pid` is for a quick test program that knows
its own device. It answers zero both when no such device is attached
and when the process may not open it, and it does not say which. It
opens only the first of several matching devices. A program that has
to report why an open failed lists the devices and calls `libusb_open`.

`libusb_control_transfer` is for configuring a device and for the
standard requests every device answers. `libusb_bulk_transfer` and
`libusb_interrupt_transfer` are for the endpoints a device's own
interface declares.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned, and a null
   context selects the default context.
2. **An out-parameter is an eight-byte slot the caller owns.**
   `ptr.alloc_word` reserves one and `ptr.read_word` reads it back.
   The context, the device list and the device handle all arrive this
   way.
3. **A device list is an array of addresses.** The address of device
   `i` is at `list + i * 8`. `libusb_free_device_list` releases the
   array, and 1 for its second argument drops the reference the list
   holds on every device in it.
4. **An error is a negative number, and the answer is a C `int`.**
   Write `as i32` before comparing it with a negative number.

   | Code | Name | What it means |
   | --- | --- | --- |
   | 0 | `LIBUSB_SUCCESS` | it worked |
   | -1 | `LIBUSB_ERROR_IO` | the transfer failed |
   | -2 | `LIBUSB_ERROR_INVALID_PARAM` | an argument was wrong |
   | -3 | `LIBUSB_ERROR_ACCESS` | the process may not open the device |
   | -4 | `LIBUSB_ERROR_NO_DEVICE` | it has been unplugged |
   | -5 | `LIBUSB_ERROR_NOT_FOUND` | there is no such thing |
   | -6 | `LIBUSB_ERROR_BUSY` | something else holds it |
   | -7 | `LIBUSB_ERROR_TIMEOUT` | the timeout ran out |
   | -8 | `LIBUSB_ERROR_OVERFLOW` | the device sent more than the buffer holds |
   | -9 | `LIBUSB_ERROR_PIPE` | the endpoint halted, or a control request is not supported |
   | -10 | `LIBUSB_ERROR_INTERRUPTED` | a system call was interrupted |
   | -11 | `LIBUSB_ERROR_NO_MEM` | memory ran out |
   | -12 | `LIBUSB_ERROR_NOT_SUPPORTED` | not on this platform |
   | -99 | `LIBUSB_ERROR_OTHER` | any other failure |

5. **An entry point that answers one byte answers it in one byte, and
   the bits above it are not cleared.** The bus number, the device
   address and the port number are bytes. Write `% 256` before
   comparing one with a number.
6. **`libusb_error_name(0)` answers
   `"LIBUSB_SUCCESS / LIBUSB_TRANSFER_COMPLETED"`**, because zero is
   both of them.
7. **A device descriptor is eighteen bytes the caller reserves**, in
   the layout the USB specification gives.

   | Offset | Width | Field |
   | --- | --- | --- |
   | 0 | 1 | the length, always 18 |
   | 1 | 1 | the descriptor type, always 1 |
   | 2 | 2 | the USB version, in binary-coded decimal |
   | 4 | 1 | the device class |
   | 5 | 1 | the device subclass |
   | 6 | 1 | the device protocol |
   | 7 | 1 | the largest packet endpoint zero will carry |
   | 8 | 2 | the vendor identifier |
   | 10 | 2 | the product identifier |
   | 12 | 2 | the device version |
   | 14 | 1 | the string index of the manufacturer |
   | 15 | 1 | the string index of the product |
   | 16 | 1 | the string index of the serial number |
   | 17 | 1 | the number of configurations |

   libusb converts every two-byte field to the host's byte order. On a
   little-endian machine, which includes x86-64 and 64-bit ARM Linux,
   the low byte comes first, so
   `ptr.load_u8(desc + 8) + ptr.load_u8(desc + 9) * 256` is the vendor.

8. **The first nine bytes of a configuration descriptor are the
   specification's**, and the rest is the library's own layout, which
   this package promises nothing about.

   | Offset | Width | Field |
   | --- | --- | --- |
   | 0 | 1 | the length, always 9 |
   | 1 | 1 | the descriptor type, always 2 |
   | 2 | 2 | the total length of the configuration and everything under it |
   | 4 | 1 | the number of interfaces |
   | 5 | 1 | the value that selects this configuration |
   | 6 | 1 | the string index of its name |
   | 7 | 1 | the attributes |
   | 8 | 1 | the most power it draws, in units of 2 milliamps at high speed and below, and 8 milliamps at SuperSpeed |

9. **Reading a descriptor sends no request to the device.** The device
   descriptor is cached, and the configuration descriptors are read
   from what the operating system holds. On Linux this needs no
   permission. `libusb_open` and everything after it does.
10. **`libusb_get_max_packet_size` answers -5 for endpoint zero.** It
    reads the endpoint descriptors, and endpoint zero is declared by
    none of them. Byte 7 of the device descriptor is where its packet
    size lives.
11. **An endpoint address carries its direction in the top bit.** With
    0x80 set the device sends, and with it clear the device receives. The same bit
    is the top bit of a control transfer's request type.
12. **A bulk or interrupt transfer writes its byte count on a timeout
    too.** libusb may split a transfer into pieces, and the timeout can
    expire after some of them have moved. The count says how much got
    through.
13. **A claim must be released before the handle is closed**, and the
    handle before the session ends.
14. **`libusb_reset_device` may invalidate the handle.** It answers -5,
    `LIBUSB_ERROR_NOT_FOUND`, when the device came back different or
    did not come back. The handle is then closed, and the device must be
    found and opened again.
15. **A version structure is four two-byte numbers and two addresses.**
    The major version is at offset 0, the minor at 2, the micro at 4
    and the nano at 6. The release-candidate suffix is at 8 and the
    describe string at 16.

## What is not included

- **The asynchronous transfers.** `libusb_alloc_transfer`,
  `libusb_submit_transfer` and their relatives are driven by a
  `libusb_transfer` structure whose completion field is a C function
  pointer, and a novo-lang function is not one. The event handling
  functions that run those callbacks, `libusb_handle_events` and its
  variants, are left out with them. So is the isochronous transfer,
  which has no synchronous form.
- **Hotplug.** `libusb_hotplug_register_callback` takes a C function
  pointer.
- **The polling functions.** `libusb_get_pollfds` answers an array of
  structures and `libusb_set_pollfd_notifiers` takes two C function
  pointers.
- **The logging callback.** `libusb_set_log_cb` takes a C function
  pointer. `libusb_set_debug` is here instead.
- **`libusb_set_option`.** It is a variadic C function whose arguments
  depend on the option.
- **The bulk streams and the device memory.** `libusb_alloc_streams`,
  `libusb_dev_mem_alloc` and their pairs are used only with
  asynchronous transfers.
- **The capability descriptors.** `libusb_get_bos_descriptor` and the
  SuperSpeed companion descriptors answer structures whose layout is
  the library's rather than the wire's.
- **`libusb_init_context`.** It takes an array of option structures and
  arrived in libusb 1.0.27. `libusb_init` is in every release.

## Related packages

[usb-nv](https://novo-lang.org/packages/usb-nv) is USB 2.0 written in
novo-lang with no C library. It holds the device side of a USB
peripheral, with the standard descriptors, the control-transfer state
machine and the CDC-ACM and HID classes, and a host-side enumeration
and transfer surface beside them. It is published as an interface
release. Every function in it is declared and none has a body yet, so
a program that must talk to a USB device today uses this package.

This package is the choice for a program that must run wherever libusb
runs, which includes Linux, macOS, Windows and the BSDs. It is also the
choice for a program that needs libusb's handling of the kernel drivers
that hold an interface.

## Tests

`tests/libusb_tests.nv` holds eleven tests over the forty-three entry
points. They call the C library, so `novo test` needs libusb-1.0
installed and linkable:

```
novo test tests/libusb_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

No test needs a device and no test needs privileges. The device list
is allowed to be empty, and every assertion about a device is inside a
branch a machine with none does not take. The four tests that reach the
open ask for vendor 0x0000, which is assigned to no vendor, so the open
fails. The sequence behind it, which claims an interface, takes it from
a kernel driver and moves bytes, is written out and never run.

The suite asserts what the reference specifies. A context is created
and destroyed. The library reports version 1.0. `LIBUSB_ERROR_ACCESS`
is named and described. The device list is taken and freed. A device's
bus number, address and speed are in range. A device descriptor is
eighteen bytes of type 1 with at least one configuration. A
configuration descriptor is nine bytes of type 2.
`libusb_get_max_packet_size` answers -5 for endpoint zero.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

libusb itself is distributed under the GNU Lesser General Public
Licence version 2.1, and installing it is the reader's own step.
