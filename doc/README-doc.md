# DOC Software Building Block

## Overview

This Software Building Block (SBB) demonstrates typical use of the Renesas Data Operation Circuit (DOC) HAL APIs. The DOC operation is selected in the FSP configurator, and the reference application writes predefined data that is intended to trigger the selected DOC event.

This document covers only:

- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

All four boards are explicitly listed as supported in the supplied `readme.txt`.

The supported demonstration events are:

- Comparison Match
- Comparison Mismatch
- Addition Overflow
- Subtraction Underflow

When the configured event occurs, the DOC callback sets an event flag. The reference application reports success on the serial terminal and turns on the first BSP-defined LED. If the event is not detected, it reports failure and leaves the LED off.

## What use case does this SBB demonstrate?

The SBB demonstrates how DOC can detect a configured comparison or arithmetic condition in hardware and notify the application through a callback.

The reference flow is:

1. Configure one DOC event in the FSP configurator.
2. Initialize the UART console.
3. Open the generated DOC instance.
4. Inspect `g_doc_cfg.event` to identify the configured event.
5. Write predefined test data to the DOC data-input register.
6. Allow the configured DOC operation to evaluate the data.
7. Set `g_doc_event_flag` from the DOC callback when the selected condition occurs.
8. Print the result to the terminal.
9. Turn on the first BSP-defined LED when the callback event is detected.
10. Reset the MCU to run the demonstration again.

The application executes the configured operation once and then remains in an infinite loop.

## Hardware requirements

- 1 x FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1
- 1 x appropriate USB cable for programming and debugging

Connect the board USB debug port to the host PC.

### FPB-RA2E3

Close **E25, E28, E46, and E47** to use J-Link On-Board VCOM for the UART terminal.

### FPB-RA4E2

The supplied `readme.txt` lists FPB-RA4E2 as supported but does not specify additional board-specific hardware configuration.

### FPB-RA6E2

The supplied `readme.txt` lists FPB-RA6E2 as supported but does not specify additional board-specific hardware configuration.

### FPB-RA8E1

The supplied `readme.txt` lists FPB-RA8E1 as supported but does not specify additional board-specific hardware configuration.

## What does the SBB core code do?

The reusable SBB layer is implemented by:

- `doc_ep.c`
- `doc_ep.h`

The core provides:

- DOC initialization through the generated FSP configuration.
- A wrapper for writing data to the DOC data-input register.
- DOC cleanup.
- A DOC callback that records event occurrence in `g_doc_event_flag`.
- Predefined input values for the four demonstrated conditions.
- The LED active-level definition used by the reference application.

The core does not select the DOC event at runtime. Event selection is taken from the generated `g_doc_cfg` configuration.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

`doc_ep.c` and `doc_ep.h` form the reusable DOC layer. They wrap `R_DOC_Open()`, `R_DOC_Write()`, and `R_DOC_Close()`, expose the callback event flag, and define the demonstration operands.

### Demo/reference application code

`doc_example_app.c` and `doc_example_app.h` form the one-shot reference application. They are responsible for:

- UART-console initialization.
- FSP version retrieval and presentation.
- Displaying SBB information.
- Inspecting `g_doc_cfg.event`.
- Selecting which predefined values are written.
- Printing the selected event.
- Checking `g_doc_event_flag`.
- Displaying success or failure.
- Turning on the first BSP-defined LED after success.
- Printing the reset instruction.
- Handling errors and entering a breakpoint trap.

The application entry point is:

```c
void doc_example_app_run(void);
```

The source identifies it as the single function called from `hal_entry.c`.

## Which public functions should I look at first?

### `doc_init()`

```c
fsp_err_t doc_init(void);
```

Opens the generated DOC instance using `g_doc_ctrl` and `g_doc_cfg`.

### `doc_write()`

```c
fsp_err_t doc_write(uint32_t data);
```

Writes data to the DOC data-input register through `R_DOC_Write()`. Comparison operations use one call. Addition and subtraction demonstrations use two calls.

### `doc_callback()`

```c
void doc_callback(doc_callback_args_t * p_args);
```

Sets `g_doc_event_flag` to `true` when the configured DOC callback is invoked. The supplied callback does not inspect the callback arguments.

### `doc_deinit()`

```c
void doc_deinit(void);
```

Closes the DOC instance if it is open.

## What configuration assumptions were made for the reference boards?

### Generated DOC instance

The SBB expects these generated objects:

```text
g_doc_ctrl
g_doc_cfg
```

The selected operation is read from:

```c
g_doc_cfg.event
```

The reference application supports these configured event values:

- `DOC_EVENT_COMPARISON_MISMATCH`
- `DOC_EVENT_COMPARISON_MATCH`
- `DOC_EVENT_ADDITION`
- `DOC_EVENT_SUBTRACTION`

### Predefined operation data

The supplied header defines these demonstration values:

| Demonstration | Value or values written |
|---|---|
| Comparison Match | `0xA5A5` |
| Comparison Mismatch | `0xB5B5` |
| Addition Overflow | `0x5A5A`, followed by `0x0001` |
| Subtraction Underflow | `0xA5A4`, followed by `0x0002` |

For comparison operations, the comparison reference or other required DOC configuration is supplied by the generated `g_doc_cfg`; it is not established by the hand-written wrapper files.

### Callback configuration

The application treats callback invocation as the indication that the selected condition occurred. The generated DOC configuration must therefore be associated with `doc_callback()` for the demonstrated success path.

### LED configuration

For FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, and FPB-RA8E1, none of the active-low board conditions in `doc_ep.h` apply. The source therefore defines:

```c
#define LED_ON (BSP_IO_LEVEL_HIGH)
```

The application writes this level to:

```c
g_bsp_leds.p_leds[0]
```

using `R_IOPORT_PinWrite()`.

### One-run execution model

The application performs the configured operation once. It then prints a request to reset the MCU and remains in an infinite loop.

## What must I change for my board, pins, clocks, timing, memory, task priorities, or middleware setup?

### DOC event selection

Select the required event in the FSP-generated DOC configuration. The hand-written application does not change the DOC event at runtime.

If another event value is configured, the current `switch` statement reaches its default branch and performs no demonstration write.

### DOC comparison/reference configuration

For comparison operations, keep the generated comparison configuration consistent with the predefined match/mismatch values or change the demonstration values to suit the configured comparison reference.

### Callback

Ensure the generated DOC instance uses `doc_callback()`. Without callback invocation, `g_doc_event_flag` remains false and the reference application reports failure.

### LED and pins

The reference application uses the first LED in `g_bsp_leds` and assumes the `LED_ON` level selected by `doc_ep.h`. If the target LED has a different active level or the product does not use a BSP LED, update the indication logic accordingly.

No external DOC signal pin is used by the supplied demonstration. The only board-specific hardware setup stated for the requested boards is the FPB-RA2E3 VCOM configuration.

### Clocks and timing

The supplied files do not describe DOC clock configuration, operation latency, or a required software delay. The application checks the callback flag immediately after the write sequence. Any changed execution model or DOC integration must preserve reliable event completion before the result is consumed.

### Memory

The hand-written SBB uses a Boolean event flag and predefined constants. The supplied files do not provide a complete RAM/ROM footprint for the SBB or generated FSP driver.

### Task priorities and synchronization

The supplied application does not use an RTOS or define task priorities. The callback and foreground application communicate through the volatile `g_doc_event_flag`. An RTOS-based integration must define the appropriate synchronization and execution context for consuming the DOC event.

### Middleware and console

The UART console and standard-output messages belong to the reference application. The reusable DOC wrapper can be integrated without the terminal interface.

For advanced DOC use with other modules, the supplied README refers to Renesas document **R30AN0384**.

## Dependencies

### Renesas FSP DOC driver

The reusable SBB directly uses:

- `R_DOC_Open()`
- `R_DOC_Write()`
- `R_DOC_Close()`

### Generated FSP configuration

The source includes `hal_data.h` and depends on:

- `g_doc_ctrl`
- `g_doc_cfg`
- the generated DOC callback configuration
- `g_ioport_ctrl`
- `g_bsp_leds`

### FSP/BSP services

The reference application uses:

- `R_FSP_VersionGet()`
- `R_IOPORT_PinWrite()`
- `__BKPT()` for its error trap

### Console/reference application

The demo uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_flush()`
- `printf()`

## Serial terminal configuration

The supplied `readme.txt` specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied SBB and reference application demonstrate:

- Opening a DOC instance.
- Selecting behavior from `g_doc_cfg.event`.
- Writing one value for comparison operations.
- Writing two values for arithmetic operations.
- Detecting a DOC callback through a volatile flag.
- Reporting operation success or failure on the terminal.
- Turning on the first BSP-defined LED after a detected event.
- Requiring an MCU reset before another run.

### Demonstrated event cases

The predefined application paths are:

- Comparison Match with `0xA5A5`.
- Comparison Mismatch with `0xB5B5`.
- Addition Overflow with `0x5A5A` then `0x0001`.
- Subtraction Underflow with `0xA5A4` then `0x0002`.

### Performance measurements

The supplied files do **not** report DOC operation latency, event-detection latency, interrupt latency, CPU utilization, throughput, or memory footprint.

The predefined values demonstrate functional event generation; they are not performance measurements. This SBB should therefore be treated as a functional DOC reference rather than a performance benchmark.

## What files are safe to modify and what files are generated, preset, or demo-only?

| File | Classification | Guidance |
|---|---|---|
| `doc_ep.c` | Reusable SBB implementation | Modify when intentionally changing DOC initialization, write, callback, event-flag, or cleanup behavior. |
| `doc_ep.h` | Reusable SBB public interface and demonstration presets | Modify carefully when changing public APIs, predefined operation values, LED polarity rules, or exposed state. |
| `doc_example_app.c` | Demo/reference application | Appropriate place for changing the selected-value sequencing, output, LED indication, reset behavior, and error handling. |
| `doc_example_app.h` | Demo/reference application interface | Contains the application entry point, version, banner, and descriptive text. |
| `readme.txt` | Existing example documentation | Source for the supported-board list, FPB-RA2E3 VCOM setup, DOC event description, terminal settings, and R30AN0384 reference. |

The SBB includes `hal_data.h` and relies on generated FSP configuration. The supplied files do not enumerate individual generated files, so generated configuration should be maintained separately from the hand-written SBB and reference-application files.

## What happens if I modify the source and later adopt a newer SBB release?

Local changes to `doc_ep.c` or `doc_ep.h` become modifications to the reusable SBB layer and can overlap changes in a future release.

To reduce upgrade conflicts:

- Keep terminal messages, result presentation, and one-run demo sequencing in `doc_example_app.*`.
- Keep DOC event selection, comparison configuration, and callback configuration in the generated FSP project when those are configuration concerns.
- Keep reusable-source changes focused and record why they are needed.
- Keep product-specific LED behavior separate from generic DOC operations where practical.
- Keep generated FSP configuration separate from hand-maintained SBB source.
- When adopting a newer release, compare the new `doc_ep.c` and `doc_ep.h` with the locally modified versions before carrying customizations forward.
- Revalidate configured event selection, comparison/arithmetic operands, callback behavior, LED polarity, and application synchronization after integration.

The supplied files do not define an automated migration or compatibility mechanism for later SBB releases. Compatibility must therefore be validated by the integrating project.

## Recommended reading order

1. **`doc_ep.h`** to understand the public interface, predefined values, callback flag, and LED-level selection.
2. **`doc_ep.c`** to see the DOC wrappers and callback implementation.
3. **`doc_example_app.c`** to understand how `g_doc_cfg.event` selects the demonstration flow and how success is reported.
4. **`doc_example_app.h`** to review the application entry point and SBB description.
5. **`readme.txt`** for the validated board list, FPB-RA2E3 setup, terminal settings, and advanced-use reference.
