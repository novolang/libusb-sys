# libusb-sys

libusb is a library for talking to USB devices from an ordinary
program, without writing a kernel driver. It runs on Linux, macOS,
Windows and the BSDs, and its C API is documented at
[libusb.info](https://libusb.sourceforge.io/api-1.0/). This package
declares forty-three of that library's entry points to novo-lang, one
declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libusb-1.0. The package contains no logic
of its own, and it does nothing without the C library installed. The
forty-three entry points are the ones a program needs to find a device,
read what it says about itself, open it, claim an interface and move
bytes in either direction; the section "What is not included" says what
a program still cannot do with them alone.

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
several: a webcam has one for video and one for audio. A program must
**claim** an interface before it may talk to it, and the operating
system may have given it to a kernel driver already.

An **endpoint** is one direction of one channel into a device. Its
address carries the direction in its top bit: 0x81 is an endpoint the
device sends on, 0x01 one it receives on. Endpoint zero is the control
endpoint, which every device has and no descriptor declares.

A **transfer** moves bytes over an endpoint. A **control** transfer
carries a request in a fixed eight-byte header and is how a device is
configured. A **bulk** transfer carries a lot of data with no timing
guarantee. An **interrupt** transfer is small and polled on a schedule
the device asked for. The fourth kind, **isochronous**, is not in this
package: it needs the asynchronous interface.

A **session** is what libusb calls a context. Every device list and
every handle belongs to one, and a program that does not want one may
pass zero and use the default session.

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

**Opening a device needs permission.** On Linux a device node belongs
to root by default, and a program that is not root gets
`LIBUSB_ERROR_ACCESS`. The usual answer is a udev rule that gives the
device to a group. Listing the devices and reading their descriptors
needs no permission at all.

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
        // The descriptor is cached, so this reaches no hardware.
        let _ = libusb.libusb_get_device_descriptor(dev, desc)
        // Bytes 8 and 9 are the vendor, 10 and 11 the product, each
        // written with the low byte first.
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

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in `tests/libusb_tests.nv`,
where the device list and the descriptor are asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libusb` | Every entry point, in six groups: the session, the library itself, the device list, the descriptors, the open device and the synchronous transfers. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The session | 3 | Starts and ends a session and sets its logging level. |
| The library | 5 | Reports the version and the capabilities, chooses the language and names an error. |
| The device list | 12 | Lists the attached devices and answers where each one is and how fast it runs. |
| The descriptors | 4 | Reads the device and configuration descriptors a device reports. |
| The open device | 16 | Opens a device, configures it, claims an interface and deals with the kernel driver. |
| The transfers | 3 | Moves bytes over a control, bulk or interrupt endpoint. |

## How to choose an entry point

`libusb_get_device_list` is for a program that is looking: it lists
everything attached and asks the operating system for nothing it has
not already cached, so it needs no permission.

`libusb_open_device_with_vid_pid` is for a program that knows its own
device. It answers zero both when no such device is attached and when
the process may not open it, and it does not say which, so it is the
wrong call for a program that has to report why.

`libusb_control_transfer` is for configuring a device and for the
standard requests every device answers. `libusb_bulk_transfer` and
`libusb_interrupt_transfer` are for the endpoints a device's own
interface declares.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned, and a null
   context selects the default session.
2. **An out-parameter is an eight-byte slot the caller owns.**
   `ptr.alloc_word` reserves one and `ptr.read_word` reads it back.
   The session handle, the device list and the device handle all
   arrive this way.
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
   | -9 | `LIBUSB_ERROR_PIPE` | the endpoint halted |
   | -12 | `LIBUSB_ERROR_NOT_SUPPORTED` | not on this system |

5. **An entry point that answers one byte answers it in one byte, and
   the bits above it are not cleared.** The bus number, the device
   address and the port number are bytes: write `% 256` before
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

   Every two-byte field is written with its low byte first, so
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
   | 8 | 1 | the power it draws, in units of 2 milliamps |

9. **The device descriptor is cached and the configuration descriptors
   are not fetched from the device either.** Reading them reaches no
   hardware and needs no permission. Everything from `libusb_open`
   onwards does.
10. **`libusb_get_max_packet_size` answers -5 for endpoint zero.** It
    reads the endpoint descriptors, and endpoint zero is declared by
    none of them. Byte 7 of the device descriptor is where its packet
    size lives.
11. **An endpoint address carries its direction in the top bit.** 0x80
    set means the device sends, clear means it receives. The same bit
    is the top bit of a control transfer's request type.
12. **A bulk or interrupt transfer writes its byte count even when it
    fails.** That is how a program finds what got through before a
    timeout.
13. **A claim must be released before the handle is closed**, and the
    handle before the session ends.
14. **`libusb_reset_device` may invalidate the handle.** It answers -4
    when the device came back different or did not come back, and the
    device must then be found and opened again.
15. **A version structure is four two-byte numbers and two addresses**:
    major at 0, minor at 2, micro at 4, nano at 6, then the
    release-candidate suffix at 8 and the describe string at 16.

## What is not included

- **The asynchronous interface.** `libusb_alloc_transfer`,
  `libusb_submit_transfer` and their relatives are driven by a
  `libusb_transfer` structure whose completion field is a C function
  pointer, and a novo-lang function is not one. The event loop that
  runs those callbacks — `libusb_handle_events` and its five variants —
  goes with them, and so does the isochronous transfer, which has no
  synchronous form.
- **Hotplug.** `libusb_hotplug_register_callback` takes a C function
  pointer.
- **The polling interface.** `libusb_get_pollfds` answers an array of
  structures and `libusb_set_pollfd_notifiers` takes two C function
  pointers.
- **The logging callback.** `libusb_set_log_cb` takes a C function
  pointer. `libusb_set_debug` is here instead.
- **`libusb_set_option`.** It is a variadic C function whose arguments
  depend on the option.
- **The bulk streams and the device memory.** `libusb_alloc_streams`
  and `libusb_dev_mem_alloc` and their pairs only matter to the
  asynchronous interface.
- **The capability descriptors.** `libusb_get_bos_descriptor` and the
  SuperSpeed companion descriptors answer structures whose layout is
  the library's rather than the wire's.
- **`libusb_init_context`.** It takes an array of option structures and
  arrived in libusb 1.0.27; `libusb_init` is in every release.

## Related packages

`usb-nv` is the host-side USB stack written in novo-lang, speaking to
the operating system directly with no C library. It is the package to
reach for on a system it supports. `usb-nv` is planned and not
published yet.

Choose this package when the program must run wherever libusb runs,
which is every desktop operating system, or when it needs libusb's own
handling of the kernel drivers that hold an interface.

## Tests

`tests/libusb_tests.nv` holds eleven tests written against the
signatures. They call the C library, so `novo test` needs libusb-1.0
installed and linkable:

```
novo test tests/libusb_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

**No test needs a device and no test needs privileges.** The device
list is allowed to be empty, and every assertion about a device is
inside a branch a machine with none does not take. The three tests that
reach the open ask for vendor 0x0000, which the USB specification
reserves and no device carries, so the open fails and the sequence
behind it — claiming an interface, taking it from a kernel driver,
moving bytes — is written out and never run. The suite asserts that a
session starts and ends, that the library reports version 1.0, that
`LIBUSB_ERROR_ACCESS` is named and described, that the device list is
taken and freed, that a device's bus number, address and speed are in
range, that a device descriptor is eighteen bytes of type 1 with at
least one configuration, that a configuration descriptor is nine bytes
of type 2, and that `libusb_get_max_packet_size` answers -5 for
endpoint zero.

**`novo test` exits 23 on this suite even when every assertion passes.**
`ptr.read_str` copies a string the C library owns, and the default leak
check counts that copy as an object the test leaked: the suite makes
four such copies and the report names four objects. The exit code is
the leak check's, not an assertion's; the output above it says how many
assertions passed. Running with `--no-leak-check` exits 0.

## Implementation status

| Group | State |
| --- | --- |
| The session | Complete for `libusb_init`. |
| The library | Complete. |
| The device list | Complete. |
| The descriptors | Complete for the device and configuration descriptors. |
| The open device | Complete. |
| The transfers | Complete for the control, bulk and interrupt kinds. |
| Asynchronous transfers | Absent. The transfer structure carries a C function pointer. |
| Isochronous transfers | Absent. They have no synchronous form. |
| Hotplug | Absent. It takes a C function pointer. |
| The polling interface | Absent. It belongs to the event loop. |
| `libusb_set_option` | Absent. It is variadic. |
| Capability descriptors | Absent. Their layout is the library's own. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

libusb itself is distributed under the GNU Lesser General Public
Licence version 2.1, and installing it is the reader's own step.
