<script>
import P5 from 'p5-svelte';
import { onMount } from 'svelte';
import { Pane } from 'tweakpane';

let innerWidth = $state();
let innerHeight = $state();
let capturer = $state(null);
let capturing = $state(false);
let frameCounter = $state(0);
let animationId = $state(null);
let animationRunning = $state(true);
let p5Instance = $state(null);
let isRendering = $state(false);
let startTime = $state(false);


let maxColumns = 10
let PARAMS = {
  duration: 20,
  columns: 10,
  quality: 100,
  width: 600,
  height: 400,
  format: 'webm',
  timeline: 0,
  randomize: false,
  cols: [],
};
for (let i = 0; i < maxColumns; i++) {
  PARAMS.cols.push({
    lineCount: Math.floor(Math.random() * 100) + 1,
    speed: Math.random() * 5 + 0.5,
    thickness: Math.floor(Math.random() * 5) + 1,
    animationType: ['verticalBouncing', 'verticalContinuous', 'backAndForth'][Math.floor(Math.random() * 3)],
    direction: ['upwards', 'downwards'][Math.floor(Math.random() * 2)],  // Randomly set initial direction
  });
}

let currentLine = $state(new Array(PARAMS.columns).fill(0));
let direction = $state(new Array(PARAMS.columns).fill(1));
let increment = $state(new Array(PARAMS.columns).fill(0));

const framerate = 60;

$effect(() => {
  // $inspect(PARAMS)
});

let sketch = $state((p5) => {
  p5Instance = p5;
  p5.setup = () => {
    p5.createCanvas(PARAMS.width, PARAMS.height);
    p5.pixelDensity(2);
    p5.background(0);
    p5.frameRate(framerate);
    frameCounter = 0;
    if (animationRunning) requestAnimationFrame(() => animate(p5));
    handleCanvasResize();
  };
});

function animate(p5) {
  p5.background(0);
  const columnWidth = p5.width / PARAMS.columns;
  const duration = PARAMS.duration;
  const maxFrames = framerate * duration;
  PARAMS.timeline = (p5.millis() - startTime) / 1000
  PARAMS.cols.forEach((col, i) => {
    increment[i] = (col.lineCount / (maxFrames / 2)) * col.speed;
  });

  for (let col = 0; col < PARAMS.columns; col++) {
    const animType = PARAMS.cols[col].animationType;
    const spacing = p5.height / PARAMS.cols[col].lineCount;

    p5.stroke(255);
    p5.strokeWeight(PARAMS.cols[col].thickness).strokeCap(p5.SQUARE);

    if (animType === 'backAndForth') {
  // Set initial direction based on 'upwards' or 'downwards' parameter
  let initialDirection = PARAMS.cols[col].direction === 'upwards' ? -1 : 1;
  if (!direction[col]) direction[col] = initialDirection;

  // Update current line position with current direction
  currentLine[col] += increment[col] * direction[col];

  // Check for boundary and reverse direction if needed
  if (currentLine[col] >= PARAMS.cols[col].lineCount - 1 || currentLine[col] <= 0) {
    direction[col] *= -1; // Reverse direction
  }

  // Draw the lines up to the current line position
  for (let i = 0; i <= Math.floor(currentLine[col]); i++) {
    // Calculate yPos based on initial direction
    let yPos = initialDirection === 1 
      ? spacing * i  // Lines appear from top to bottom
      : p5.height - spacing * i;  // Lines appear from bottom to top
    p5.line(col * columnWidth, yPos, (col + 1) * columnWidth, yPos);
  }
}


    if (animType === 'verticalContinuous') {
      let speed = spacing / duration * PARAMS.cols[col].speed / 1000;
      let deltaY = ((p5.millis() - startTime) * speed) * (PARAMS.cols[col].direction === 'upwards' ? -1 : 1);

      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
        let newY = (i * spacing + deltaY) % p5.height;
        if (newY < 0) newY += p5.height;
        p5.line(col * columnWidth, newY, (col + 1) * columnWidth, newY);
      }
    }

    if (animType === 'verticalBouncing') {
      const bounceSpeed = (1 / (duration * 1000) * PARAMS.cols[col].speed); // This gives a complete oscillation in 'duration' seconds
      const deltaY = Math.abs(Math.sin((p5.millis() - startTime) * bounceSpeed * Math.PI)) * spacing; 

      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
          const newY = (i * spacing + deltaY) % p5.height;
          p5.line(col * columnWidth, newY, (col + 1) * columnWidth, newY);
      }
    }

    // if (animType === 'verticalContinuous') {
    //   const speed = spacing / duration * PARAMS.cols[col].speed / 1000;
    //   const deltaY = ((p5.millis() - startTime) * speed);

    //   for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
    //     const newY = (i * spacing + deltaY) % p5.height;
    //     p5.line(col * columnWidth, newY, (col + 1) * columnWidth, newY);
    //   }
    // }
  }

  if (capturing) {
    capturer.capture(p5.canvas);
    frameCounter++;
  }

  if (animationRunning) animationId = requestAnimationFrame(() => animate(p5));
}

let startStopButton = null;
let restartButton = null;
let pane = $state()

onMount(() => {
  randomizeColumnParams();
  pane = new Pane();

  const tab = pane.addTab({
    pages: [
      { title: 'Settings' },
      { title: 'Columns' },
    ],
  });

  tab.pages[0].addBinding(PARAMS, 'duration', { min: 1, max: 120, step: 1 });
  tab.pages[0].addBinding(PARAMS, 'columns', { min: 1, max: maxColumns, step: 1 });
  tab.pages[0].addBinding(PARAMS, 'timeline', {
    readonly: true,  // Make it readonly as it's for display only
  });
  tab.pages[0].addBinding(PARAMS, 'width', { min: 100, max: 2000, step: 10 }).on('change', handleCanvasResize);
  tab.pages[0].addBinding(PARAMS, 'height', { min: 100, max: 2000, step: 10 }).on('change', handleCanvasResize);

  PARAMS.cols.forEach((col, i) => {
      const columnFolder = tab.pages[1].addFolder({ title: `Column ${i + 1}`, expanded: false,});
      columnFolder.addBinding(col, 'lineCount', { min: 1, max: 300, step: 1 });
      columnFolder.addBinding(col, 'speed', { min: 0.1, max: 5, step: 0.1 });
      columnFolder.addBinding(col, 'thickness', { min: 1, max: 5, step: 1 });
      columnFolder.addBinding(col, 'animationType', {
          options: {
              'Back and Forth': 'backAndForth',
              'Vertical Bouncing': 'verticalBouncing',
              'Vertical Continuous': 'verticalContinuous',
          }
      }).on('change', restartAnimation);
      columnFolder.addBinding(col, 'direction', {
        options: {
          'Upwards': 'upwards',
          'Downwards': 'downwards',
        }
      });
  });

  const exportFolder = pane.addFolder({ title: 'Export Controls', view: 'separator' });
  exportFolder.addBinding(PARAMS, 'format', {
    options: {
      'GIF': 'gif',
      'WebM': 'webm',
      'PNG': 'png',
      'JPG': 'jpg',
      'WebM (MediaRecorder)': 'webm-mediarecorder',
    }
  });
  exportFolder.addBinding(PARAMS, 'quality', { min: 1, max: 100, step: 1 }).on('change', updateCaptureQuality);
  exportFolder.addButton({
    title: 'Randomize',
  }).on('click', () => {
    restartAnimation();
    randomizeColumnParams();
    console.log(PARAMS.cols);
    pane.refresh();
  });
  const renderButton = exportFolder.addButton({
    title: 'Render',
    style: { color: isRendering ? '#fff' : '#000' }
  }).on('click', () => {
    if (isRendering) {
      stopCapture();
    } else {
      startCapture();
    }
    isRendering = !isRendering;
    renderButton.title = isRendering ? 'Stop Rendering' : 'Render';
    renderButton.color = isRendering ? '#fff' : '#000';
  });

  exportFolder.addButton({ title: 'Export Current Frame' }).on('click', exportCurrentFrame);

  startStopButton = exportFolder.addButton({
    title: 'Pause'
  }).on('click', toggleAnimation);

  restartButton = exportFolder.addButton({
    title: 'Restart'
  }).on('click', restartAnimation);
});

function updateCaptureQuality() {
  if (capturing) {
    capturer.setQuality(PARAMS.quality);
  }
}

function startCapture() {
  restartAnimation();
  if (!capturing) {
    capturer = new CCapture({
      format: PARAMS.format,
      framerate: framerate,
      quality: PARAMS.quality,
      verbose: true,
      timeLimit: PARAMS.duration,
    });
    capturing = true;
    frameCounter = 0;
    capturer.start();
  }
}

function stopCapture() {
  if (capturing) {
    capturing = false;
    capturer.stop();
    capturer.save();
    capturer = null;
    cancelAnimationFrame(animationId);
  }
}

function exportCurrentFrame() {
  if (p5Instance) {
    const currentCanvas = p5Instance.canvas;
    const link = document.createElement('a');
    if (PARAMS.format === 'png' || PARAMS.format === 'jpg') {
      link.href = currentCanvas.toDataURL(`image/${PARAMS.format}`);
      link.download = `frame-${Date.now()}.${PARAMS.format}`;
    } else {
      alert('Current frame export only supports PNG and JPG formats.');
      return;
    }
    link.click();
  }
}

function toggleAnimation() {
  animationRunning = !animationRunning;
  if (startStopButton) {
    startStopButton.title = animationRunning ? 'Pause' : 'Play';
  }
  if (animationRunning) {
    requestAnimationFrame(() => animate(p5Instance));
  } else {
    cancelAnimationFrame(animationId);
  }
}

function randomizeColumnParams() {
  PARAMS.columns = Math.floor(Math.random() * 9) + 1
  for (let col = 0; col < PARAMS.columns; col++) {
    PARAMS.cols[col].lineCount = Math.floor(Math.random() * 100) + 1;
    PARAMS.cols[col].speed = Math.min(Math.random() * 5 + 0.1, 5);
    PARAMS.cols[col].animationType = ['verticalBouncing', 'verticalContinuous', 'backAndForth'][Math.floor(Math.random() * 3)];
    PARAMS.cols[col].direction = ['upwards', 'downwards'][Math.floor(Math.random() * 2)];
  }
}

function handleCanvasResize() {
  p5Instance.resizeCanvas(PARAMS.width, PARAMS.height);
  fitCanvasToViewport();
}

function fitCanvasToViewport() {
  const canvasAspect = PARAMS.width / PARAMS.height;
  const viewportAspect = innerWidth / innerHeight;
  let scale = 1;
  if (canvasAspect > viewportAspect) {
    scale = innerWidth / PARAMS.width;
  } else {
    scale = innerHeight / PARAMS.height;
  }
  const canvas = p5Instance.canvas;
  canvas.style.transform = `scale(${scale})`;
  canvas.style.left = `${(innerWidth - PARAMS.width * scale) / 2}px`;
  canvas.style.top = `${(innerHeight - PARAMS.height * scale) / 2}px`;
}

function restartAnimation() {
  currentLine = new Array(PARAMS.columns).fill(0);
  direction = new Array(PARAMS.columns).fill(1);
  increment = new Array(PARAMS.columns).fill(0);
  frameCounter = 0;
  PARAMS.timeline = 0;
  startTime = p5Instance.millis();
}
</script>

<svelte:window bind:innerWidth bind:innerHeight onresize={handleCanvasResize}/>

<P5 {sketch} />

<style>
  :global(body) {
    overflow: hidden;
    margin: 0;
    background-color: #333;
  }
  :global(canvas) {
    position: absolute;
    left: 0;
    top: 0;
    transform-origin: left top;
  }
</style>