# ExoPlayer <img src="https://img.shields.io/github/v/release/google/ExoPlayer.svg?label=latest"/>

ExoPlayer is an application level media player for Android. It provides an
alternative to Android’s MediaPlayer API for playing audio and video both
locally and over the Internet. ExoPlayer supports features not currently
supported by Android’s MediaPlayer API, including DASH and SmoothStreaming
adaptive playbacks. Unlike the MediaPlayer API, ExoPlayer is easy to customize
and extend, and can be updated through Play Store application updates.

## Documentation

* The [developer guide][] provides a wealth of information.
* The [class reference][] documents ExoPlayer classes.
* The [release notes][] document the major changes in each release.
* Follow our [developer blog][] to keep up to date with the latest ExoPlayer
  developments!

[developer guide]: https://exoplayer.dev/guide.html
[class reference]: https://exoplayer.dev/doc/reference
[release notes]: https://github.com/google/ExoPlayer/blob/release-v2/RELEASENOTES.md
[developer blog]: https://medium.com/google-exoplayer

## Using ExoPlayer

ExoPlayer modules can be obtained from [the Google Maven repository][]. It's
also possible to clone the repository and depend on the modules locally.

[the Google Maven repository]: https://developer.android.com/studio/build/dependencies#google-maven

### From the Google Maven repository

#### 1. Add ExoPlayer module dependencies

The easiest way to get started using ExoPlayer is to add it as a gradle
dependency in the `build.gradle` file of your app module. The following will add
a dependency to the full library:

```gradle
implementation 'com.google.android.exoplayer:exoplayer:2.X.X'
```

where `2.X.X` is your preferred version.

As an alternative to the full library, you can depend on only the library
modules that you actually need. For example the following will add dependencies
on the Core, DASH and UI library modules, as might be required for an app that
only plays DASH content:

```gradle
implementation 'com.google.android.exoplayer:exoplayer-core:2.X.X'
implementation 'com.google.android.exoplayer:exoplayer-dash:2.X.X'
implementation 'com.google.android.exoplayer:exoplayer-ui:2.X.X'
```

When depending on individual modules they must all be the same version.

The available library modules are listed below. Adding a dependency to the full
ExoPlayer library is equivalent to adding dependencies on all of the library
modules individually.

* `exoplayer-core`: Core functionality (required).
* `exoplayer-dash`: Support for DASH content.
* `exoplayer-hls`: Support for HLS content.
* `exoplayer-rtsp`: Support for RTSP content.
* `exoplayer-smoothstreaming`: Support for SmoothStreaming content.
* `exoplayer-transformer`: Media transformation functionality.
* `exoplayer-ui`: UI components and resources for use with ExoPlayer.

In addition to library modules, ExoPlayer has extension modules that depend on
external libraries to provide additional functionality. Some extensions are
available from the Maven repository, whereas others must be built manually.
Browse the [extensions directory][] and their individual READMEs for details.

More information on the library and extension modules that are available can be
found on the [Google Maven ExoPlayer page][].

[extensions directory]: https://github.com/google/ExoPlayer/tree/release-v2/extensions/
[Google Maven ExoPlayer page]: https://maven.google.com/web/index.html#com.google.android.exoplayer

#### 2. Turn on Java 8 support

If not enabled already, you also need to turn on Java 8 support in all
`build.gradle` files depending on ExoPlayer, by adding the following to the
`android` section:

```gradle
compileOptions {
  targetCompatibility JavaVersion.VERSION_1_8
}
```

#### 3. Enable multidex

If your Gradle `minSdkVersion` is 20 or lower, you should
[enable multidex](https://developer.android.com/studio/build/multidex) in order
to prevent build errors.

### Locally

Cloning the repository and depending on the modules locally is required when
using some ExoPlayer extension modules. It's also a suitable approach if you
want to make local changes to ExoPlayer, or if you want to use a development
branch.

First, clone the repository into a local directory and checkout the desired
branch:

```sh
git clone https://github.com/google/ExoPlayer.git
cd ExoPlayer
git checkout release-v2
```

Next, add the following to your project's `settings.gradle` file, replacing
`path/to/exoplayer` with the path to your local copy:

```gradle
gradle.ext.exoplayerModulePrefix = 'exoplayer-'
apply from: file("path/to/exoplayer/core_settings.gradle")
```

You should now see the ExoPlayer modules appear as part of your project. You can
depend on them as you would on any other local module, for example:

```gradle
implementation project(':exoplayer-library-core')
implementation project(':exoplayer-library-dash')
implementation project(':exoplayer-library-ui')
```

## Developing ExoPlayer

#### Project branches

* Development work happens on the `dev-v2` branch. Pull requests should
  normally be made to this branch.
* The `release-v2` branch holds the most recent release.

#### Using Android Studio

To develop ExoPlayer using Android Studio, simply open the ExoPlayer project in
the root directory of the repository.


## RTSP Issues and Work-around

### Non-compliant CCTVs

ExoPlayer performs several assertions following the RTSP standard (https://www.ietf.org/rfc/rfc2326.txt).
However, some CCTV manufacturers, particularly BOSCH, don't follow the standard and would cause
ExoPlayer to crash or not be able to play the video stream.

* Missing FMTP Parameter  
  ExoPlayer asserts that the FMTP parameter should be present.  But, the BOSCH camera we used for
  testing does not send this value.  This would cause ExoPlayer to crash.

  As a work-around, we *inject* the FMTP values.  However, we use a hard-coded value for the
  `sprop-parameter-sets` property since we don't know how to generate/calculate this.  We set it to
  `"Z00AHpY1QWAk03AQEBAg,aO4xsg=="`, which we copied from a COMELIT camera.  This work-around is
  implemented in `RtspMediaTrack.generatePayloadFormat`.

  The table below shows the paramters sent by the BOSCH camera vs the COMELIT camera  
  | BOSCH | COMELIT |
  |-|-|
  | v=0 | v=0 |
  | o=- 0 0 IN IP4 127.0.0.1 | o=- 0 0 IN IP4 127.0.0.1 |
  | s=Stream | s=Stream |
  | c=IN IP4 0.0.0.0 | c=IN IP4 0.0.0.0 |
  | t=0 0 | t=0 0 |
  | m=video 0 RTP/AVP 35 | m=video 0 RTP/AVP 96 |
  | | b=AS:5000 |
  | a=rtpmap:35 H264/90000 | a=rtpmap:96 H264/90000 |
  | | **a=fmtp:96 profile-level-id=4d001e; sprop-parameter-sets=Z00AHpY1QWAk03AQEBAg,aO4xsg==; packetization-mode=1** |
  | a=control:trackID=0 | a=control:trackID=0 |

* Non-dynamic `Payload Type`  
  ExoPlayer also asserts that the `Payload Type` should be set to *dynamic* (i.e. >= 96) as defined
  in **RFC3551 Section 3**.  However, the BOSCH camera we have uses `Payload Type` **`35`**.  As a
  work-around, we removed this assertion (located in `RtspMediaTrack.generatePayloadFormat`).

* Missing `RTP-info` Header  
  The BOSCH camera we test also does not send the `RTP-info` response header, which ExoPlayer
  asserts that it should be present.  As a work-around, we removed the assertion in
  `RtspMediaPeriod.onPlaybackStarted`.

  Note also that, because of the missing track info, pausing and resuming RTSP streams would not work.

### Slow Loading Times

From our testing, we noticed that ExoPlayer takes about 7-10 seconds on average to load and play a
video stream.  This is quite slow compared to VLC's ~3 second loading times.  We tried to play
around with buffer durations through `LoadControl`.  However, even after setting the buffer to as
low as **`500ms`**, we still couldn't see a noticeable difference; the loading time is still quite
long (~9 seconds).
