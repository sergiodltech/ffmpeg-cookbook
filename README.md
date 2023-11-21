# ffmpeg-cookbook
A collection of useful ffmpeg commands I have used

## Animations

### High quality GIF's

```shell-script
    1) ffmpeg -i "$input_file" -vf fps="$desired_fps",scale="$desired_downscale":flags=lanczos,palettegen "$palette_file"
    2) ffmpeg -i "$input_file" -i "$palette_file" -filter_complex "fps=$desired_fps,scale=$desired_downscale:flags=lanczos [out1]; [out1][1:v] paletteuse" "$output_file"
```

1. We first create a color palette of the input animation. There is a limited amount of colors GIF's can use. Instead of using a generic color limited palette, we generate a specific one for the original file.
  - $desired\_fps: 12 to 18 fps can greatly reduce the file size and still demonstrate movement perception. High FPS animations also load heavily whatever presenter software we might be using.
  - $desired\_scale: in the format of `width:height`. Same reasoning as before for downscaling our animation. A `-1` in one of the dimensions can be used to automatically calculate such dimension according to the aspect ratio.
  - lanczos: I like the results of this scaling algorithm.

2. In this command we use both the animation file and the palette to generate our new GIF.

### "Boomerang" GIF's

An animation where the second part is the first part but reversed. On looping animated GIFs, this avoids the abrupt timeskip when the animation period is restarting.
Based on the previous commands for "High Quality GIF's".

```shell-script
    1) ffmpeg -i "$input_file" -vf fps="$desired_fps",scale="$desired_downscale":flags=lanczos,palettegen "$palette_file"
    2) # Setting up variables to improve readability
    export INPUT_BRANCH="scale=$desired_scale:flags=lanczos[out1];[out1][1:v]paletteuse, split [main][tmp]"
    export REVERSE_BRANCH="[tmp] reverse [reversed]"
    export OUTPUT_BRANCH="[main][reversed] concat"
    ffmpeg -i "$input_file" -i "$palette_file" -filter_complex "$INPUT_BRANCH; $REVERSE_BRANCH; $OUTPUT_BRANCH" "$output_file"
```

The second command splits the output of the output of "paletteuse" into two video streams. "tmp" is reversed and renamed to "reversed" and finally, both streams are concatenated.

### Pulsating GIF from an Image

Having a still image, fade it in and out once. This commands applies chromakey to include transparency outside of the image contour.

Even if the zooming part of the animation has no transparency (only the countour), this animation may be overlayed over other presentations by applying it a lower opacity level. This can be used for a pulsating highlight over the background image.

```shell-script
    1) ffmpeg -f lavfi -i color=color=white -loop 1 -i "$input_image" -filter_complex "[1] format=rgba,fade=in:st=0:d=1.5:alpha=1,fade=out:st=1.5:d=1.5:alpha=1 [overlay];[0][overlay]scale2ref[bg][overlay]; [bg][overlay] overlay=(main_w-overlay_w)/2:(main_h-overlay_h)/2" -t 3 -c:v libx264 -pix_fmt yuv420p "$tmp_video_file"
    2) ffmpeg -i "$tmp_video_file" -filter_complex "[0:v]chromakey=white,split[v0][v1];[v0]palettegen[p];[v1][p]paletteuse" "$output_gif"
```

1. We first create a temporal video file of our animation. I haven't been able to combine both commands in just one filter graph.
  - `[1] format=rgba` new stream from the second input source, with pixel format rgba
  - `fade=in:st=0:d=1.5:alpha=1` fade in filter, starting from 0s, for 1.5s, considering alpha channel
  - `fade=out:st=1.5:d=1.5:alpha=1 [overlay]` fade out filter, starting from 1.5s, for 1.5s, considering alpha channel, this is renamed to "overlay"
  - `[0][overlay] scale2ref [bg][overlay]` the first input source is taken and scaled to the size of the overlay stream. This new stream is renamed to "bg" (background).
  - `[bg][overlay] overlay=(main_w-overlay_w)/2:(main_h-overlay_h)/2` both streams are overlayed and overlay is positioned in the middle.
  - `-t 3 -c:v libx264 -pix_fmt yuv420p $tmp_video_file` three seconds of the stream are taken and encoded into a video file using h264
2. GIF creating, similar to past examples of palette generation. The (pure) white color is used a the key color to consider it transparent.

