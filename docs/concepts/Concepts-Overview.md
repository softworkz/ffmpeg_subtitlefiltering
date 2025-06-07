# Concepts: Introduction


The changes that are required for enabling filtering capabilities for subtitles in FFmpeg are complex and affect many areas in FFmpeg.

Let's start with some basics which are important to understand the 
implications of the subject.


## Semantic Mismatch between S and A/V

There are good reasons why we have different representations in code for
these at the moment: AVFrame for audio and video and AVSubtitle for
subtitles. Fundamental differences exist in semantics and the
requirements for any kind of processing logic:

- Subtitle events are sparse while audio and video frames are contiguous.
  This means that in the latter case, once the duration of one frame has
  elapsed, there will be another frame (unless EOF). I.e., the duration
  of a frame (if available) indicates the start of the following frame,
  or at least it can be expected that there's a next frame that will
  arrive at some time to replace the previous one (sure, there are tons
  of special cases). For subtitles, there's usually a single event with
  a duration that is fully independent from any subsequent event
  (exceptions exist here as well, of course).

- Subtitle events are non-exclusive in the time dimension. For videos,
  only one frame can be shown at a time, and for audio, only one sound
  can be played at a time, but in the case of subtitles there can be 10,
  20 or even more events with the identical start time (e.g. ASS
  animations). Each one of those can have a different duration, and of
  course there can also be events that start while previous events
  haven't ended yet.

In these regards (and some other details), subtitle events are strictly
incompatible with audio and video frames.

That's the very reason for the separation between AVFrame and AVSubtitle.


The wide-spread Misconception
-----------------------------

Some people have commented and demanded that when we start using AVFrame
for subtitles (like my patchset does), the timing fields and possibly
other details of an AVFrame should be the same as in AVSubtitle, i.e. a
single start time and a single duration.

But they are not considering an important "detail" about AVFrame:

The entire FFmpeg code base -avcodec, avfilter, ffmpeg, ffprobe, ffplay -
is full of code that handles AVFrames, and hat code expects AVFrames to 
have the semantics of audio and video frames. This code is not
able to process AVFrame data when it has the semantics of AVSubtitle -
that's the very reason why AVSubtitle exists.

We cannot change all the code to introduce different handling for two
kinds of AVFrames: S and A/V - and even then, why should we do that at
all? There's nothing won by doing so. At this point, we need to go back 
and answer the following question:


Why would we actually want to use AVFrame for Subtitles?
--------------------------------------------------------

As the title says "Subtitle Filtering" - we want to enable filtering for
subtitles. What does it mean exactly? There are two ways of
interpretation:

1. Adding a filtering feature for subtitles that is similar to filtering
   for audio and video but exclusive to subtitles.

2. Extending the existing filtering feature for audio and video in a way
   that subtitle data can be included and handled in the same way as
   audio and video.

For (1), there wouldn't be a need for using AVFrame for subtitle data. 
This could be implemented using AVSubtitle alone.
But (1) wouldn't allow any interaction between subtitles and audio 
or video.

The goal of my subtitle-filtering effort is clearly (2) and has always
been. The existing filtering code is built upon AVFrame, and as we have
learned, AVFrame is a world with its own rules that differ from
AVSubtitle logic.

For handline subtitles according to their actual semantics, it would 
require a complete rewrite of filtering so that it can work with
AVSubtitle data for subtitles and handle subtitles according to their 
own logic. Filtering is a crucial and complex part of FFmpeg, and even 
a minimal change to the base implementation of filtering can easily
cause severe regressions. Anybody who is familiar with the subject 
and tries to tell that this is what should be done, knows very
well that this is a suicidal task.


The Approach
------------

I chose an approach that is actually feasible, involves no risk for 
existing functionality, enables a wide range of use cases and even 
consolidates existing code in some areas.

So how does it work?

If we do not want to change the filtering implementation and that
implementation is based on AVFrames, there's just one way:

We have to play in the "AVFrame world".

This cannot be done by simply using the AVFrame struct instead of the
AVSubtitle struct - this alone cannot work and would fail
(incompatible semantics). 

What we need to do to make this work is to adapt the subtitle
events so they're ready to play in the "AVFrame world", allowing them
to be treated in the same way as normal AVFrames.

This is probably what some reviewers hadn't understood in all
consequences involved, so:

> [!NOTE]
> It might be easier to picture this, when thinking of it as if 
> the AVSubtitle struct still existed and was merely wrapped 
> inside an AVFrame.

When thinking about it in this way, the timing fields of the AVFrame are
the values of the wrapper frame, and the subtitle-timing extra fields are
the timing values of the wrapped AVSubtitle data itself.

The AVFrames are _logically_ just wrappers around the actual subtitle 
data, in order to allow the subtitle data to take part and play in the 
"AVFrame World", specifically in filtering.

This also consolidates code in many places where subtitles can now be 
treated like audio and video frames, but it is important to understand
that an AVFrame is not always a 1:1 projection of what AVSubtitle 
is currently. There can be multiple AVFrames which wrap the same 
AVSubtitle data and there can also be (subtitle) AVFrames which have
no AVSubtitle data to carry and are just empty.

This is clearly a PRAGMATIC approach - it's not how one would
implement it when starting blank. But that train has departed long ago.
Millions of users all over the world are relying on FFmpeg functionality
and expecting it to continue working as is in all detail.
Re-implementing a fundamental part of FFmpeg like filtering is an
approach with low chances to succeed and be accepted.

