---
title: "Self-Made Karaoke"
date: 2019-11-03T23:53:04+01:00
---

I am definitely not the singer type.
Never went to any karaoke and probably will never do.
But for some reason I was intrigued to see, from a technical standpoint, how to create a karaoke song.
How does one remove the vocals from a song and make the lyrics appear at the right time?
I will show you the poor man's approach of making your own karaoke songs and get them to play on a website.

As my main system runs Linux and I'm a terminal guy, I will only use open source command line tools.
Most of the tools are probably also available for your platform of choice.


# Getting A Song

The example song we will use is ["Play Crack the Sky" by Brand New][yt], mainly because of the awesome lyrics.
Most songs nowadays are available on YouTube, and this one is no exception.
So, let's just download it from there.
The best tool for this job is `youtube-dl`.

`youtuble-dl` supports a gazillion video platforms, not just YouTube.
If you ever wanted to download a file from a streaming website, this tool should have you covered.

Let us first have a look at the available formats we can choose from with the option `-F`.
```
$ youtube-dl -F 'https://www.youtube.com/watch?v=--EeaSYoH04'
[youtube] --EeaSYoH04: Downloading webpage
[youtube] --EeaSYoH04: Downloading video info webpage
[youtube] --EeaSYoH04: Downloading js player vflO1GesB
[youtube] --EeaSYoH04: Downloading js player vflO1GesB
[info] Available formats for --EeaSYoH04:
format code  extension  resolution note
249          webm       audio only tiny   62k , opus @ 50k (48000Hz), 2.12MiB
250          webm       audio only tiny   80k , opus @ 70k (48000Hz), 2.79MiB
140          m4a        audio only tiny  129k , m4a_dash container, mp4a.40.2@128k (44100Hz), 5.03MiB
251          webm       audio only tiny  157k , opus @160k (48000Hz), 5.48MiB
160          mp4        256x144    144p   86k , avc1.4d400c, 15fps, video only, 2.11MiB
134          mp4        640x360    360p  139k , avc1.4d401e, 30fps, video only, 1.97MiB
133          mp4        426x240    240p  180k , avc1.4d4015, 30fps, video only, 4.30MiB
135          mp4        854x480    480p  300k , avc1.4d401f, 30fps, video only, 3.69MiB
```

Four audio and four video formats are available, in various codecs and qualities.
We are just interested in audio.
Let us pick the best audio quality and store it to the file _song.webm_ by using the option `-f` followed by the _format code_.
```
$ youtube-dl -f 251 'https://www.youtube.com/watch?v=--EeaSYoH04' -o song.webm
```


# Removing Vocals in the Center

A tiny tool we can use is [SoX][sox], "the Swiss Army knife of audio manipulation", as they call it themselves.
`sox` supports a lot of audio effects, and one of them, called _oops_, does exactly what we want.
It remixes a stereo audio file to a file with two mono channels where each mono channel contains the difference between the two channels in the stereo file.
If the vocals are in the center, they get extinguished as they are the same on the left and right channel, no difference.

There are a lot of limitations with this approach.
Not every song has the vocals exactly in the center, and even more important: everything else in the center gets removed as well.
This can lead to awful sounding results.
Our example song works quite well.
The only instrument, a guitar, stays unchanged and the vocals are nearly gone, only a faint echo remains.

`sox` does not support every audio codec and container format on the planet.
We have to convert the file first with yet another Swiss Army knife, `ffmpeg`.
It is my favorite tool when it comes to transcoding of media files.

We convert the song with `ffmpeg` to the wav format which `sox` understands.
```
$ ffmpeg -i song.webm song.wav
```

Next, we remove the vocals by applying the _oops_ effect of `sox` and store the result to _sound.wav_.
```
$ sox song.wav sound.wav oops
```

Finally, we transcode the resulting wav file back to a webm file, using the opus codec with a bit rate of 48 kbps.
You can choose a higher bit rate if you like.
```
$ ffmpeg -i sound.wav -c:a libopus -b:a 48k sound.webm
```

The last step is optional, but it reduces the file size considerably.

Have a listen to the file!
Any decent media player should be able to play it.
Otherwise, drag & drop it into an empty browser window to have a listen.


# Splitting Vocals from Instruments

A different approach using artificial neural networks is used by [Spleeter][spleeter].
It comes with pre-trained models for TensorFlow and is quite easy to use.
We will use it to separate the vocals from everything else and therefore get the music without the vocals.

`spleeter` converts the input file internally with `ffmpeg`.
We do not have to do it ourselves beforehand.
The following command separates the vocals from the music and produces two wav files in the directory _output_.
It downloads the model to a subfolder of the current directory when it runs the first time.
Try to stay in this directory as it otherwise downloads the model every time.
```
$ spleeter separate -i song.webm -p spleeter:2stems -o output
```

You can optionally encode the wav file with `ffmpeg` as before.

This approach works for most of the songs I tested it with, but sometimes the resulting file contains quite some awful sounding artifacts.
For our example song, I actually prefer the output of `sox` as it leaves the background vocals unchanged, and the faint echo of the main voice is also nice.
In the end, it depends heavily on the song.
`spleeter` works with a much larger variety of songs and does an amazing job for such a difficult task.
Hats off!

There are similar tools out there like [Open-Unmix][open-unmix] which I have not tried yet.


# Timed Lyrics

That was all quite easy, wasn't it?
Just invoking some commands.
Well, now comes the tedious part.

The easiest way to get text displayed in a timely fashion is by using subtitles.
There are multiple formats available.
It all depends which media platform we want to target.
The "cool kids" are on the web, aren't they?
So, let's target HTML5.

The subtitle format for HTML5 is [WebVTT][webvtt].
The specification is still just a draft and not done yet.
Even more problematic, browser support is lacking a lot of the more interesting features like proper time-tag support.
Styling with CSS is also hit and miss.
It might work with some browsers but not with others.
Therefore, I will focus only on the basic functionality which has support in all modern browsers.

Like all standard web formats, WebVTT is a text format and can be created with any text editor.
Here is a basic example.

{{< highlight webvtt >}}
WEBVTT

00:01.000 --> 00:10.000
Hello, World!

00:10.000 --> 00:15.000
This is a WebVTT file.
{{< / highlight >}}

Every WebVTT file has to begin with the string "WEBVTT" followed by a blank line.
The main part of the file consists of a sequence of _cues_.
Each _cue_ is active for a certain timespan specified by a start time and an end time.
During this timespan, it displays a text segment which can span multiple lines.
_Cues_ can overlap which means that multiple text segments are displayed at the same time.
A blank line separates two _cues_ from each other.

Well then, that's all there is.
Get the lyrics from the web, listen through the song and format the lyrics with timestamps accordingly.
All you need is a text editor and some stamina to get through the tedious work.

I did it for the example song.
You can download the file [here](/karaoke/lyrics.vtt).


# Playback in HTML5

Unfortunately, there seems to be no way to get the audio-tag of HTML5 and WebVTT playing along nicely.
It did not work in any of the browsers I tried.
The only work around I found working consistently in all browsers was to add a video track to the audio file, resulting in a video file which works fine with the video-tag and WebVTT.

The following command creates a black image file which we will use as video image.
You can use any other image.
```
$ convert -size 640x480 xc:black black.png
```

Next, we use `ffmpeg` to create a video from the image and our audio file without the vocals.
```
$ ffmpeg -loop 1 -i black.png -i sound.webm -c:v libvpx-vp9 -c:a copy -shortest karaoke.webm
```
We loop the image forever with `-loop 1`, the two input files follow.
Then, we specify the video codec to be VP9 by using the encoder library _libvpx-vp9_.
The audio should be just copied into the video file which we do with the option `-c:a copy`.
Finally, we specify with `-shortest` that we want to stop encoding when one of the inputs ends.
This option is required as the image loops forever.

Now, we can put it all together on a website with following HTML-code.
{{< highlight html >}}
<video controls src="karaoke.webm">
	<track default src="lyrics.vtt">
</video>
{{< / highlight >}}

You can try out the video file below, if you have JavaScript enabled in your browser.
The lyrics file is added automatically.

{{< plain >}}
<hr/>
<p>Choose the video webm file:</p>
<input type="file" id="files" name="files[]" accept="video/webm" />

<video id="player" controls>
  <track default src="/karaoke/lyrics.vtt">
</video>

<script>
	document.getElementById('files').addEventListener('change', function(event){
		var file = event.target.files[0];
		document.getElementById('player').src = URL.createObjectURL(file);
	}, false);
</script>

<hr/>
{{< / plain >}}


# Conclusion

It was interesting fiddling around with the different tools and formats.
I learned a bunch of new things.
I hope that tools like `spleeter` improve over time with more and better training data.
The same goes for the WebVTT support in the browsers.
Only the basic functionality is usable, but it would be nice to highlight the currently sung word on a line.
The specification of WebVTT supports it, but no browser does.

Let us hope for a brighter future where we can sing along our favorite songs not caring what our neighbors might think about it.


# References

- [SoX][sox]
- [Spleeter][spleeter]
- [Open-Unmix][open-unmix]
- [WebVTT specification][webvtt]


[yt]: https://www.youtube.com/watch?v=--EeaSYoH04
[sox]: http://sox.sourceforge.net/
[spleeter]: https://github.com/deezer/spleeter
[open-unmix]: https://sigsep.github.io/open-unmix/
[webvtt]: https://w3c.github.io/webvtt/
