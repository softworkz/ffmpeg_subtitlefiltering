# AVSubtitleFlowMode


enum `AVSubtitleFlowMode`{
    `AV_SUBTITLE_FLOW_UNKNOWN` = -1, 
    `AV_SUBTITLE_FLOW_NORMAL `= 0, 
    `AV_SUBTITLE_FLOW_KILLPREVIOUS `= 1, 
    `AV_SUBTITLE_FLOW_HEARTBEAT `= 2, 
    `AV_SUBTITLE_FLOW_SCATTERED `= 3, 
};


## `AV_SUBTITLE_FLOW_NORMAL` 

### Behavior

There's a 1-to-1 relationship between frames and events. 
It's the most typical case for what is output by subtitle decoders.

If there are multiple events having the same start-time which are in separate frames, then the frame-time is slightly increased per frame to achieve a strictly monotonical progression of frame pts times. The `sub-start-time` values are always precise, though.

### Fields

- **frame-pts:** around start of display
- **frame-duration:** unknown/empty
- **sub-start-time:** Precise time of subtitle display start
- **sub-duration:** Precise duration of subtitle display

### Emitted by

- Text Subtitle Decoders
- Bitmap Sub Decoders with durations



<br />


## `AV_SUBTITLE_FLOW_KILLPREVIOUS `

### Behavior

This mode applies to cases where durations are unknown in the moment of the begin of the event and the end-time is controlled by the start-time of the subsequent event only.
This happens for example with DVB subtitles (bitmap) or in case of OCR conversion and closed captions. It always requires to wait until something gets hidden, as that is not known up-front.

### Fields

- **frame-pts:** Precise start of event (on or off)
- **frame-duration:** unknown/empty
- **sub-start-time:** Precise start of event (on or off)
- **sub-duration:** NO_PTS

### Emitted by

- Bitmap Subtitle Decoders where no duration is available
- graphicsub2text Filter (OCR)
- sub_cc Filter (closed caption splitter filter)


<br />


## `AV_SUBTITLE_FLOW_HEARTBEAT`

### Behavior

This flow mode is created by the subfeed filter which is inserted into a filter graph when synchronization is needed between subtitles and video or audio - like for overlaying subtitles onto a video ("burn-in").
It repeats subtitle frames at a (configurable) frequency to achieve a constant flow in a filtergraph.

### Fields

- **frame-pts:** between `sub-start-time` and `sub-start-time + sub-duration` (unless empty event)
- **frame-duration:** reciprocal heartbeat frequency
- **sub-start-time:** Precise time of subtitle display start
- **sub-duration:** Precise duration of subtitle display

### Emitted by

- subfeed filter

<br />

## `AV_SUBTITLE_FLOW_SCATTERED `

### Behavior

This flow has a (largely) constant frequency of frames as well, but it doesn't repeat any frames (except the empty frame).

It is meant for cases where the origin has a flow type of `AV_SUBTITLE_FLOW_KILLPREVIOUS` and other cases where the display-end time is not known at the time of display-start.

When the flow mode is `AV_SUBTITLE_FLOW_KILLPREVIOUS` it is often not possible to wait for the next event that defines the end of the current event, especially in realtime situations like TV or streaming or burn-in with a short pre-roll time of subtitles.

In this mode, subtitle events are scattered (or sliced) into short subsequent events, all with the same content. The scatter interval can be something like 100ms or 500ms and it also determines the max delay that a user might experience. Assuming the interval it 500ms and a subtitle display start is shortly after an interval border, it will take 500ms until the event scatter/slice gets emitted (earlier only when there's another change before the interval ends.

This method allows to handle all the cases with upfront-unknown durations. 

At the end, an encoder would also be able to stich those scatters back together, all or just partial scatter sequences. That also allows to re-segment subtitle events for cases like HLS streaming with WebVTT subtitles.


### Fields

- **frame-pts:** follows scatter frequency raster
- **frame-duration:** reciprocal scatter frequency
- **sub-start-time:** same as frame-ptds
- **sub-duration:** frame-duration or shorter

### Emitted by

- subfeed filter (optional)
- sub_cc (optional)
- graphicsub2text filter (optional)


