# Scroll Animation Skill

## When to use
Use this technique whenever the user wants a video to play in sync with scrolling (scroll-scrubbed video). This replaces the naive `video.currentTime` seeking approach, which is inherently choppy due to keyframe decoding delays.

## Technique: Canvas Frame Extraction

The approach is to pre-extract every frame from the video into an `ImageBitmap` array at page load, then draw the correct frame to a `<canvas>` element based on scroll progress. This gives instant, perfectly smooth frame display with zero decode latency during scrolling.

### HTML Structure

Place a hidden `<video>` (used only for frame extraction) and a visible `<canvas>` inside a sticky container within a tall scroll section:

```html
<section class="video-section" id="videoSection" style="height: 500vh;">
    <div class="video-container" style="position: sticky; top: 0; width: 100%; height: 100vh; overflow: hidden;">
        <video id="myVideo" muted playsinline preload="auto" style="display: none;">
            <source src="video.mp4" type="video/mp4">
        </video>
        <canvas id="myCanvas" style="width: 100%; height: 100%; object-fit: cover;"></canvas>
    </div>
</section>
```

### CSS

```css
.video-container video { display: none; }
.video-container canvas {
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: center center;
}
```

### JavaScript (requires GSAP + ScrollTrigger)

```javascript
// 1. Extract all frames from a video into ImageBitmap array
function extractFrames(videoEl, numFrames) {
    return new Promise((resolve) => {
        const offscreen = document.createElement('canvas');
        offscreen.width = videoEl.videoWidth;
        offscreen.height = videoEl.videoHeight;
        const ctx = offscreen.getContext('2d');
        const frames = [];
        let idx = 0;
        const duration = videoEl.duration;

        function next() {
            if (idx >= numFrames) { resolve(frames); return; }
            videoEl.currentTime = (idx / (numFrames - 1)) * duration;
        }

        videoEl.onseeked = () => {
            ctx.drawImage(videoEl, 0, 0);
            createImageBitmap(offscreen).then((bmp) => {
                frames.push(bmp);
                idx++;
                next();
            });
        };
        next();
    });
}

// 2. Bind canvas drawing to scroll progress
function setupCanvasScrub(canvasEl, frames, triggerSelector) {
    const ctx = canvasEl.getContext('2d');
    canvasEl.width = frames[0].width;
    canvasEl.height = frames[0].height;
    let lastFrame = -1;
    ctx.drawImage(frames[0], 0, 0);

    ScrollTrigger.create({
        trigger: triggerSelector,
        start: 'top top',
        end: 'bottom bottom',
        scrub: 0,
        onUpdate: (self) => {
            const frameIdx = Math.round(self.progress * (frames.length - 1));
            if (frameIdx !== lastFrame) {
                lastFrame = frameIdx;
                ctx.drawImage(frames[frameIdx], 0, 0);
            }
        }
    });
}

// 3. Init: wait for video metadata, extract frames, bind to scroll
function initVideoCanvas(videoEl, canvasEl, triggerSelector) {
    function go() {
        const totalFrames = Math.round(videoEl.duration * 24); // adjust fps as needed
        extractFrames(videoEl, totalFrames).then((frames) => {
            setupCanvasScrub(canvasEl, frames, triggerSelector);
        });
    }
    if (videoEl.readyState >= 2) { go(); }
    else { videoEl.addEventListener('loadeddata', go, { once: true }); }
}
```

## Key details

- **Frame count**: Use `video.duration * fps` to get total frames. 24fps is standard. For longer videos or memory-constrained situations, you can reduce to 12-15fps and it will still look smooth.
- **Memory**: Each frame is an `ImageBitmap`. A 752x416 video at 241 frames uses roughly 300MB. For very long or high-res videos, consider extracting fewer frames.
- **Load time**: Frame extraction happens sequentially on page load. The page will take a few seconds to fully prepare. Consider showing a loading indicator for long videos.
- **Scroll section height**: Controls scrub speed. 500vh = the video plays over 5 viewport-heights of scrolling. Increase for slower playback, decrease for faster.
- **object-fit: cover on canvas**: Works in modern browsers and handles aspect ratio mismatches between the video and the viewport.

## Why NOT to use video.currentTime seeking

Setting `video.currentTime` on scroll is choppy because:
1. Browsers seek to the nearest keyframe, then decode forward to the target frame
2. With typical h264 encoding (GOP of 24-250 frames), this means significant decode work per seek
3. Even with lerp/smoothing (rAF interpolation), the underlying seeks are still slow
4. The result is visible stuttering, dropped frames, and videos that appear to stop partway through
