# Changelog

All notable changes to libusb-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-16

The first release: forty-three entry points of the libusb-1.0 C API,
one `@ffi` declaration each, and no logic.

### Added

- `libusb` — the whole synchronous surface, in six groups.
  - The session: `libusb_init`, `libusb_exit` and
    `libusb_set_debug`.
  - The library itself: `libusb_get_version`,
    `libusb_has_capability`, `libusb_setlocale`,
    `libusb_error_name` and `libusb_strerror`.
  - The device list and what a device reports:
    `libusb_get_device_list`, `libusb_free_device_list`,
    `libusb_ref_device`, `libusb_unref_device`, the bus number, the
    address, the port number, the port chain, the parent, the speed
    and the two maximum packet sizes.
  - The descriptors: `libusb_get_device_descriptor`,
    `libusb_get_active_config_descriptor`,
    `libusb_get_config_descriptor` and
    `libusb_free_config_descriptor`.
  - The open device: `libusb_open`, `libusb_close`,
    `libusb_open_device_with_vid_pid`, `libusb_get_device`, the two
    configuration calls, the three interface calls,
    `libusb_clear_halt`, `libusb_reset_device`, the four kernel
    driver calls and `libusb_get_string_descriptor_ascii`.
  - The synchronous transfers: `libusb_control_transfer`,
    `libusb_bulk_transfer` and `libusb_interrupt_transfer`.
- `tests/libusb_tests.nv` — eleven tests over the signatures. They
  call the C library, so they need libusb-1.0 installed. **No test
  needs a device and no test needs privileges**: the device list may
  be empty, every assertion about a device is inside a branch a
  machine with none does not take, and the tests that reach the open
  ask for vendor 0x0000, which the USB specification reserves and no
  device carries.

### `wraps = "libusb-1.0"` and not `libusb`

The stem the loader resolves carries the major version:
`libusb-1.0.so.0` is the file, and the `-1.0` is part of the name and
not a version suffix on it. A `libusb.so` on the same system is the
0.1 library, which is a different interface with different symbols.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**The asynchronous interface.** `libusb_alloc_transfer`,
`libusb_submit_transfer`, `libusb_cancel_transfer` and
`libusb_free_transfer` are driven by a `libusb_transfer` structure the
caller fills in, and one of its fields is a C FUNCTION POINTER the
library calls when the transfer completes. A novo-lang function is not
one, so the whole interface goes, and with it the event loop —
`libusb_handle_events` and its five relatives — which exists to run
those callbacks.

**Hotplug.** `libusb_hotplug_register_callback` takes a C function
pointer, for the same reason.

**The polling interface.** `libusb_get_pollfds` answers an array of
`libusb_pollfd` structures and `libusb_set_pollfd_notifiers` takes two
C function pointers. Both belong to the event loop above.

**The logging callback.** `libusb_set_log_cb` takes a C function
pointer. `libusb_set_debug` is here instead, and it writes to the
standard error stream.

**`libusb_set_option`.** It is a VARIADIC C function whose argument
list depends on the option it is given. `libusb_set_debug` covers the
only option this release needs.

**The bulk streams and the device memory.** `libusb_alloc_streams`,
`libusb_free_streams`, `libusb_dev_mem_alloc` and
`libusb_dev_mem_free` are USB 3.0 and Linux-specific facilities that
only matter to the asynchronous interface.

**The capability descriptors.** `libusb_get_bos_descriptor`,
`libusb_get_ss_endpoint_companion_descriptor` and their relatives
answer structures whose layout is the library's own rather than the
wire's, and this release promises no offsets for them. The device
descriptor and the first nine bytes of the configuration descriptor,
which the USB specification lays down, are the two it does.

**`libusb_init_context`.** It takes an array of option structures and
arrived in libusb 1.0.27. `libusb_init` is in every release of the
library.
