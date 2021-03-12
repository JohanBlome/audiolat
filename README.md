# latencycheck
Measure ear-to-mouth (output->input) delay in android systems.


# 1. Introduction

This app calculates the "audio latency" of an android device by performing an ear-to-mouth (e2m) experiment.


# 2. Discussion

The exact operation of the experiment is as follows.

* (1) we open 2 audio streams, one that records audio from the mic (uplink/input direction), and one that plays audio into the speaker (downlink/output direction). Each of the audio streams requires a callback function, which will be called every time the mic has new data ("record" stream), and every time the speaker needs new data ("playout" stream). The callback function includes as parameters:
  * (1.1) the data being captured, with its length (in number of frames) for the record stream
  * (1.2) an empty buffer for the app to provide the data being requested, with the required length (in number of frames), for the playout stream.
We also open a file stream where we store audio samples.

* (2) in the common operation, whenever we get a record stream callback, we just copy the data (the mic input) to the file stream. Whenever we get a playout stream callback, we provide silence into the output data (the speaker output).

* (3) we run a periodic latency experiment (every 2 seconds more or less). The experiment is started by a record stream callback. Instead of copying the full input, we replace the last few samples with that of a (short) well-known signal called "begin". This is effectively marking in the output file the moment where we started calculating an audio latency sample. Once we have inserted the full "begin" signal, we set a static variable in the callback to `true`.

When we get the next playout callback, we detect that we are in the middle of an audio latency experiment. In this case, we replace the silence with a second well-known signal called "end". This causes the "end" signal to be played at the speaker. At some point it gets captured back by the mic, and dumped into the file stream (see bullet (2)).

* (4) once the experiment is finished, we analyze the file stream, looking for poccurrences of both the begin and the end signals. We assume that the begin signal was injected with zero latency (as we are copying it directly to the output stream), but that the end signal went through the speaker, then the mic, and then the file. We assume the e2m latency of the system to be `timestamp_end - timestamp_begin`.

![Operation Details](doc/audio.latency_checker.png)

Figure 1 shows the exact experiment operation.

* The top image shows the common operation: Data from the record (uplink) callback is just copied to the output file (we use light blue to mean just background noise). Calls from the playout (downlink) callback are just filled with silence (white box).

* The middle image shows the beginning of an experiment. It starts with a record (uplink) callback, where the app replaces the last tidbit of the incoming signal with a begin signal (red box). In the next playout callback, the app will start playing the end signal instead of silence (pink sin).

* The bottom image shows the moment where the mic captures the signal it just played. This is common operation again, but in this case the mic signal we are copying into the file stream contains the end signal.

In the post-experiment analysis, we look for pairs of begin and end signals in the file stream, and calculate the `audio_latency` as the distance between both.


# 3. Operation

## 3.1. Prerequisites

* android sdk setup and environment variables set
* android ndk
* adb connection to the device being tested.
* ffmpeg with decoding support for the codecs to be tested


## 3.2. Operation

(1.a) Install the debug build apk located at:

```
$ adb install ./app/build/outputs/apk/debug/app-debug.apk
```

Note that you can uninstall the app at any moment by running:

```
$ adb uninstall com.facebook.latencycheck
```


(1.b) As an alternative, build and install your own apk.

First set up the android SDK and NDK in the `local.properties` file. Create
a `local.properties` file with valid entries for the `ndk.dir` and `sdk.dir`
variables.

```
$ cat local.properties
ndk.dir: /opt/android_ndk/android-ndk-r19/
sdk.dir: /opt/android_sdk/
```

Note that this file should not be added to the repo.

Second build the encapp app:

```
$ ./gradlew clean
$ ./gradlew build
$ ./gradlew installDebug
```

(2) Run the app for the very first time to get permissions

```
$ adb shell am start -n com.facebook.latencycheck/.MainActivity
```

The very first time you run the app, you will receive 2 requests to give permissions to the app. The app needs:

* permission to access to photos, media, and files on your device in order to store the mic capture which will be used to measure the e2m latency, and
* permission to record audio, for obvious reasons.


(3) Run an experiment

Compared to other solutions, latencycheck is not very sensitive to background noise. However, results are better if the volume of the playout in the DUT is high.

The syntax of the experiment is:

```
$ adb shell am force-stop com.facebook.latencycheck
$ adb shell am start -n com.facebook.latencycheck/.MainActivity [parameters]
```

where the possible parameters are:

* `-e sr <SAMPLE_RATE>`: set the sample rate (in samples per second). Supports 48000, 16000 (default and recommended), and 8000.
* `-e t <TEST_LENGTH_SECS>`: duration of the length (in seconds). Default value is 15.
* `-e rbs <REC_BUFFER_SIZE_SAMPLES>`: size of the recording (mic) buffer (in frames). Default is 32.
* `-e pbs <PLAY_BUFFER_SIZE_SAMPLES>`: size of the playout (speaker) buffer (in frames). Default is 32.

For example, to use 512 frames as the size of the playout buffer

```
$ adb shell am start -n com.facebook.latencycheck/.MainActivity -e pbs 512
```

Run the command. You should hear some chirps (a signal of continuously increasing frequency). Wait until you hear no more chirps (around the test length, or 15 seconds by default).


(4) Analyze result

The recorded file can be found in `/sdcard/lc_capture*.raw`. First pull it
and convert it to pcm s16 wav.

```
$ adb pull /sdcard/lc_capture_chirp2_16k_300ms.raw .
$ ffmpeg -f s16le -acodec pcm_s16le -ac 1 -ar 16000 -i lc_capture_chirp2_16k_300ms.raw lc_capture_chirp2_16k_300ms.raw.wav
```

Then, run the analysis in the wav file:

```
$ ./scripts/find_pulses.py lc_capture_chirp2_16k_300ms.raw.wav CHIRP_SAMPLERATE -i RECORDER_FILE -sr SAMPLE_RATE -t 20
'-t 20' will trigger on a badly damaged signal.

or

* run the bash script:
> $ run_test.sh SAMPLERATE TEST_LENGTH_SECS REC_BUFFER_SIZE_SAMPLES PLAY_BUFFER_SIZE_SAMPLES

Finally
* Run the output csv file from previous step:
> $ python3 calc_delay.py RECORDER_FILE.CSV




