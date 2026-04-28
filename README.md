The Windows ARM HEVC Wall: Why Command-Line Tools Can't Use Your Snapdragon GPU

If you are trying to use hardware-accelerated HEVC (H.265) encoding on a Windows
ARM laptop (Snapdragon X Elite / Plus) using FFmpeg, HandBrake, or custom
scripts, you have likely hit a brick wall.

You probably noticed that H.264 hardware encoding (h264_mf) works flawlessly out
of the box, pushing your Adreno VPU to encode video at blazing speeds. But the
moment you switch to HEVC (hevc_mf), everything breaks.

This repository documents the complete investigation into the Windows ARM Media
Foundation stack, revealing how Microsoft and Qualcomm have artificially locked
HEVC hardware encoding behind Universal Windows Platform (UWP) gatekeeping.

Phase 1: The Missing MFT (Media Foundation Transform)

The first error you will encounter in FFmpeg when calling hevc_mf on a fresh
Windows ARM machine is:

could not find any MFT for the given media type

The Cause: To avoid paying HEVC licensing fees for every Windows PC, Microsoft
stripped the HEVC codec from the base installation of Windows 10/11. The
Qualcomm Snapdragon Adreno GPU physically has an HEVC encoder, but the OS-level
bridge is missing.

The Fix: You must install the HEVC Video Extensions from the Microsoft Store
(often $0.99, though a free OEM version exists on Microsoft's update servers).
Once installed, FFmpeg recognizes HEVCVideoExtensionEncoder.

But the nightmare is just beginning.

Phase 2: The Direct3D 11 Surface Trap

Now that the encoder is present, FFmpeg attempts to use it—and immediately fails
with:

failed processing input: 80004005 (E_FAIL)

The Cause: FFmpeg naturally decodes video into CPU system memory (usually as
yuv420p). The H.264 Media Foundation Transform (MFT) silently accepts this,
converts it, and encodes it. The HEVC MFT, however, strictly refuses standard
CPU memory buffers. It demands Direct3D 11 (D3D11) hardware texture surfaces.

The Attempted Fix: We can force FFmpeg to maintain the video entirely in GPU
memory by initializing a hardware context and converting the pixel format to
nv12 (the strict format required by Windows D3D11 hardware encoders):

ffmpeg -hwaccel d3d11va -hwaccel_output_format d3d11 -i input.mp4 -c:v hevc_mf output.mp4

The Result: The pipeline connects, but the HEVC encoder throws a new, fatal
error:

[hevc_mf] Failed to set D3D manager: 80004001 (E_NOTIMPL)

The encoder explicitly replies: "Not Implemented." It refuses to share the D3D11
hardware context with a standard desktop application.

Phase 3: The Native WinRT / Package Identity Block

Assuming FFmpeg's wrapper was the problem, we built a native C# Windows Runtime
(WinRT) desktop application using the Windows.Media.Transcoding namespace to
bypass FFmpeg entirely.

When feeding the native C# app an input file and requesting HEVC via
MediaEncodingProfile.CreateHevc(), the OS threw:

Transcode failed. Reason: Unknown

The Smoking Gun: We changed exactly one word in the C# code—switching
CreateHevc() to CreateMp4() to request H.264. The application instantly fired up
the Snapdragon Adreno GPU and successfully hardware-encoded the video.

The Truth: The code is perfect, and the hardware is capable. Microsoft has
intentionally crippled the Microsoft Store HEVC Extension. It checks the calling
process for a Windows Store Package Identity and AppContainer Isolation. If your
app is a standard Win32 .exe (like FFmpeg, HandBrake, or a custom Python/C#
script), the DRM silences the HEVC encoder and drops the connection.

Phase 4: Will MSIX / Desktop Bridge Save You? (No.)

A common theory is that you can package your Win32/Console app as an MSIX (using
the Desktop Bridge) to give it the required Package Identity.

This does not work for HEVC. While MSIX gives you a Package Family Name (passing
the billing check), the HEVC Media Foundation Transform demands true UWP sandbox
isolation (AppContainer) and routing through modern MediaCapture APIs. Because a
Desktop Bridge app runs as "Full Trust" (standard desktop permissions), Windows
intentionally breaks the COM activation linking your app to the GPU driver's
secure frame routing.

To use the Microsoft-provided API, you must build a true UWP/WinUI 3 sandboxed
app, which completely ruins the ability to use standard command-line tools or
raw file paths.

Conclusion & Current Workarounds

If you are doing video engineering on Snapdragon/Windows ARM, HEVC hardware
encoding is practically impossible for command-line tools.

Here are the only viable paths forward:

1. Fallback to Hardware H.264 (Recommended for CLI)

The H.264 MFT on Snapdragon is unrestricted. It accepts system memory, D3D11
surfaces, and Win32 calls. To maximize GPU usage in FFmpeg, simply use:

ffmpeg -hwaccel d3d11va -i input.mp4 -c:v h264_mf -b:v 4M output.mp4

2. Fallback to CPU Software HEVC

If file size is paramount, abandon the GPU. The Snapdragon X Elite/Plus CPUs are
incredibly powerful at multi-threaded workloads. Software encoding (libx265)
will max out your CPU but yield perfectly compressed HEVC files:

ffmpeg -i input.mp4 -c:v libx265 -crf 28 -preset fast output.mp4

3. Use Commercial Software (Bypass Media Foundation)

Why does DaVinci Resolve 19 hardware encode HEVC flawlessly on Windows ARM?
Because Blackmagic partnered with Qualcomm and uses the MainConcept ARM SDK.
This proprietary SDK talks directly to the Adreno GPU drivers, completely
ignoring Microsoft's crippled Media Foundation APIs. Open-source developers do
not have access to this SDK due to massive licensing fees.

4. Hijack First-Party Apps

If you just need to convert a few files natively, use Microsoft's built-in
Clipchamp or the legacy Windows Photos app. Because they are first-party UWP
apps, they hold the magic token to unlock the Adreno HEVC hardware encoder.

This investigation confirms the exact DRM/API limitations currently frustrating
open-source developers on Windows on ARM. Until Microsoft removes the UWP
AppContainer restriction from their HEVC Extension, command-line video engineers
are strictly bound to H.264 for hardware acceleration.
# windows-arm-media-foundation-block
