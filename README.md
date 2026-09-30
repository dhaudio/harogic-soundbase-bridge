# HAROGIC SAN-60 and SoundBase: a bridge for live spectrum monitoring

**Darío Hernández Martínez · RF Solutions**  
**Project status: September 2026**

As an RF coordinator, I need to monitor the spectrum and manage frequencies within my coordination software. I wanted to use my HAROGIC SAN-60 as a data source for SoundBase, with control over the sweep configuration and access to live traces.

To achieve this, I developed an HTTP bridge with AI assistance. The setup I have tested uses the HAROGIC connected over USB to Ubuntu running in Parallels, with SoundBase Desktop running on macOS.

I have also prepared a version for 64-bit Windows in Boot Camp. This version still needs validation with the physical analyzer and SoundBase on Windows.

## The problem I wanted to solve

My goal was to receive SAN-60 sweeps directly in SoundBase, without exporting and importing a CSV every time I needed to update the spectrum.

The bridge provides a configuration and trace interface for SoundBase, while HAROGIC's HTRA SDK handles communication with the analyzer. This separates spectrum acquisition from the interface used to deliver the data to SoundBase.

## How it works

SoundBase connects to the service as a **Spectrum Analyzer Bridge**. The service receives configuration requests, translates them into HTRA SDK calls, and publishes the latest acquired trace.

In the tested setup:

1. The SAN-60 USB device is assigned to the Ubuntu virtual machine.
2. The bridge runs in Ubuntu and listens on port **8088**.
3. SoundBase on macOS connects to Ubuntu's IP address.
4. The analyzer performs the requested sweeps, and the resulting samples are returned to SoundBase.

In my tests, the virtual machine's address was `10.211.55.4`. This address is specific to my installation and may differ on another computer.

The Windows version is designed to run SoundBase and the bridge on the same system. In that setup, the address is `127.0.0.1`, and the port remains `8088`.

## The bridge API

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/info` | Analyzer identification and advertised capabilities. |
| GET | `/configuration` | Read the current configuration. |
| POST | `/configuration` | Change the frequency range, RBW, VBW, step size, and reference level. |
| POST | `/sweep/start` | Start continuous acquisition. |
| POST | `/sweep/stop` | Request acquisition to stop. |
| GET | `/trace` | Read the latest available trace. |

Each trace includes its start and stop frequencies, sample spacing, point count, amplitudes in dBm, acquisition timestamp, and a sweep identifier: `sweepId`.

The identifier helps confirm that new sweeps are arriving. A successful `/info` response confirms part of the connection; checking acquisition also requires verifying that `/trace` returns data and that `sweepId` keeps increasing.

## Resolution: RBW and sample spacing

An important part of the development was distinguishing **RBW** from **step size**.

RBW is the resolution bandwidth used in the analysis. Step size describes the frequency spacing between samples in the published trace. These are different parameters: a 10 kHz RBW does not mean that the trace contains a sample every 10 kHz.

In an initial test, with a requested range of **470–900 MHz**, I received **3,988 points**, with an actual spacing of approximately **107.88 kHz** and RBW and VBW configured to 10 kHz.

I then reduced the requested range to **470–800 MHz** and increased the requested point count to a maximum of **16,000**. One verified acquisition produced:

| Parameter | Result |
|---|---:|
| Trace start frequency | 469,982,649.10 Hz |
| Trace stop frequency | 800,013,421.20 Hz |
| Received points | 15,995 |
| Actual sample spacing | 20,634.66 Hz |
| Configured RBW and VBW | 10,000 Hz |

The trace endpoints differ slightly from the requested limits because the SDK returns its own frequency grid. The bridge publishes those actual values.

This result describes a particular acquisition. It does not guarantee a fixed point count for every configuration. SoundBase can change the settings, and the SDK determines the resulting trace.

## Preserving analyzer samples

The bridge uses a complete sweep in **SWP** mode and the SDK's spectrum cropping function to obtain the requested frequency range. It publishes the resulting samples without interpolation or point-count reduction.

Because the trace format represents the frequency axis using a start frequency, step size, and point count, the bridge checks that the axis is approximately uniform. It rejects an irregular axis rather than assigning samples to an invented frequency grid.

Non-finite amplitude values are represented as **−160 dBm**, preserving their positions. This is a placeholder for invalid data, not a measurement of the SAN-60's noise floor.

The version described here does not add averaging in the bridge. The published trace mode is `clear-write`.

## Starting and stopping without the terminal

To simplify operation, I prepared a desktop control that can start the service, stop it, and display its log.

I have tested the Linux control. For Windows, I prepared a window with **Activar** (Start), **Desactivar** (Stop), and **Ver registro** (View log) buttons.

Starting the service does not immediately start a sweep: acquisition begins when SoundBase requests it. Stopping the bridge releases the analyzer so another application can use it.

## Installation and version status

| Environment | Status |
|---|---|
| Ubuntu ARM64 in Parallels + SoundBase on macOS | Connection, configuration, and sweep reception tested with the SAN-60. |
| Linux desktop control | Installed and tested. |
| Windows x64 in Boot Camp + SoundBase on the same Windows system | Package prepared; physical hardware validation pending. |

The Windows package includes the **HTRA 0.55.100** SDK, bridge code, desktop control, and instructions. The installer requires an Internet connection to set up Python and its dependencies. SoundBase and the official USB driver are installed separately.

Windows checks include Python syntax validation, SDK structure sizes and field offsets, and application logic tested with a simulated analyzer. These checks do not constitute execution of the DLLs, installer, and SAN-60 on Windows.

## What still needs testing

The next step is to validate the Windows version with the actual hardware and assess stability during long sessions. I also want to record sweep times for different combinations of frequency range, RBW, and point count.

On macOS, I observed that SoundBase could become blank after a period without interaction. The cause still needs investigation. I have not established that the installer fixes this behavior or that the bridge causes it.

## Sharing the project

This project grew out of a practical need in my work as an RF coordinator: using the SAN-60 within my SoundBase workflow and checking exactly which data reaches the application.

If you test the bridge, a useful report includes the operating system, analyzer model and firmware, SDK version, sweep configuration, received point count, and any relevant error logs.

**An independent RF Solutions project. HAROGIC and SoundBase belong to their respective owners; this bridge is not an official integration from either manufacturer.**
