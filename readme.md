Possible ways to transcode or at least partially decode in the browser...


Audio decode with Web Audio
===========================

decodeAudioData and OfflineAudioContext
---------------------------------------

This sample http://mdn.github.io/offline-audio-context/ renders an audio file into a OfflineAudioContext via AudioContext.decodeAudioData, then plays the resulting PCM audio buffer on demand. Current versions of Firefox, Chrome, Safari, and Edge all support this with an MP4 input file, reading the AAC audio track and ignoring the video.

There are some limitations using this technique for large files however:
* decodeAudioData() takes a single ArrayBuffer as input, requiring the entire file to be preloaded before beginning
 * for local filesystem sources this could still take a little while, but more worryingly it could eat TONS of memory
* OfflineAudioContext returns only a single final output buffer containing the entire lifetime of the output at the end. no progress info, and no way to upload during decode.
 * workaround is apparently to hook a scriptProcessorNode on the output of the OfflineAudioContext to read its output out as it progresses? (untested)
 * that would however still leave the giant buffer in memory. on hour-length conference videos this would be >1gb of data!

A possible alternative to OfflineAudioContext's giant buffer:
* play/capture in real time and suppress the speaker output, capturing via ScriptProcessorNode instead of letting the offline buffer accumulate
 * if we can't keep up with the upload we're still going to accumulate, just not quite as much.
  * could pause/resume, but sample accuracy may suffer
 * if we also do Vorbis or Opus compression in JS, we could reduce that usage a lot since no need to store uncompressed.

A possible alternative to decodeAudioData's preloaded input buffer requirement:
* use an <audio> element source node with createMediaElementSource
 * hey that should totally work

Video decode with canvas
========================

Untested. Can paint a <video> to a <canvas> at any time, and could then read data from canvas and recompress in JS.

Likely problems:
* probably won't keep up in real time :)
* stopping/starting may have frame-accuracy problems
* stepping through frame-by-frame likely to also have frame-accuracy problems, and perf sounds awful
* encoder perf probably awful!

Transcode with MediaRecorder
============================

Untested

May only be supported reliably in Chrome 49?

May be supported in Firefox?

https://developers.google.com/web/updates/2016/01/mediarecorder isn't clear on whether an existing media element can be used to provide a MediaSource suitable for recording. (looks like... probably not)

An AudioNode can be used as a source, which should allow for audio transcoding at least.

In theory a MediaSource could be created and have file data buffered into it for this usage as well as for more traditional playback usage.
No idea if this actually works in practice.

https://w3c.github.io/mediacapture-fromelement/ indicates possible API for pulling from a canvas or video but don't know implementation status. Also may fail if can't record in real-time.

Per https://github.com/Fyrd/caniuse/issues/2250 seems to be supported in both Firefox and Chrome now on canvas, unknown if on video.
* ok Firefox has captureStream for canvas but not video... but does have mozCaptureStream on video \o/
* Chrome does not have it in 49
 * does have it on canvas in 51, but not on video

So, for Firefox could possibly transcode straight from a <video> source, and on Chrome could play via a canvas.... nasty but maybe workable. Might complicate things if we have to connect up audio as well... though we could take the output and remux it in a pinch.

For any of these if decoding gets ahead of uploading we may have to buffer into memory, which could be expensive on very long files.
