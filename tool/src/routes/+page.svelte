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
let startStopButton = null;
let restartButton = null;
let renderButton = $state();

const framerate = 60;

// PARAMS
let maxColumns = 10
let PARAMS = {
  duration: 20,
  columns: 10,
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
    speed: Math.floor(Math.random() * 9) + 1,
    width: Math.floor(Math.random() * 99) + 1,
    positionX: Math.floor(Math.random() * 99),
    thickness: Math.floor(Math.random() * 4) + 1,
    animationType: ['verticalBouncing', 'verticalContinuous', 'backAndForth'][Math.floor(Math.random() * 3)],
    direction: ['upwards', 'downwards'][Math.floor(Math.random() * 2)],  // Randomly set initial direction
  });
}
let currentLine = $state(new Array(PARAMS.columns).fill(0));
let direction = $state(new Array(PARAMS.columns).fill(1));
function randomizeColumnParams() {
  for (let col = 0; col < PARAMS.columns; col++) {
    PARAMS.cols[col].lineCount = Math.floor(Math.random() * 100) + 1;
    PARAMS.cols[col].speed = Math.floor(Math.random() * 9) + 1,
    PARAMS.cols[col].width = Math.floor(Math.random() * 99),
    PARAMS.cols[col].positionX = Math.floor(Math.random() * 99) + 1,
    PARAMS.cols[col].thickness = Math.floor(Math.random() * 4) + 1,
    PARAMS.cols[col].animationType = ['verticalBouncing', 'verticalContinuous', 'backAndForth'][Math.floor(Math.random() * 3)];
    PARAMS.cols[col].direction = ['upwards', 'downwards'][Math.floor(Math.random() * 2)];
  }
}

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
  PARAMS.timeline = (p5.millis() - startTime) / 1000
  const duration = PARAMS.duration;
  const maxFrames = framerate * duration;

  for (let col = 0; col < PARAMS.columns; col++) {
    const columnWidth = p5.width / 100 * PARAMS.cols[col].width;
    const columnPositionX = p5.width / 100 * PARAMS.cols[col].positionX;
    const increment = (PARAMS.cols[col].lineCount / (maxFrames / 2)) * PARAMS.cols[col].speed;
    const animType = PARAMS.cols[col].animationType;
    const spacing = p5.height / PARAMS.cols[col].lineCount;

    p5.stroke(255);
    p5.strokeWeight(PARAMS.cols[col].thickness).strokeCap(p5.SQUARE);

    if (animType === 'backAndForth') {
      // Set initial direction based on 'upwards' or 'downwards' parameter
      let initialDirection = PARAMS.cols[col].direction === 'upwards' ? -1 : 1;
      if (!direction[col]) direction[col] = initialDirection;

      // Update current line position with current direction
      currentLine[col] += increment * direction[col];

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
        p5.line(columnPositionX, yPos, columnPositionX + columnWidth, yPos);
      }
    }


    if (animType === 'verticalContinuous') {
      let speed = spacing / duration / 1000 * PARAMS.cols[col].speed;
      // console.log(duration * PARAMS.cols[col].speed);
      
      let deltaY = ((p5.millis() - startTime) * speed) * (PARAMS.cols[col].direction === 'upwards' ? -1 : 1);

      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
        let newY = (i * spacing + deltaY) % p5.height;
        if (newY < 0) newY += p5.height;
        p5.line(columnPositionX, newY, columnPositionX + columnWidth, newY);
      }
    }

    if (animType === 'verticalBouncing') {
        const bounceSpeed = (1 / (duration * 1000) * PARAMS.cols[col].speed); // Complete oscillation in 'duration' seconds        
        const deltaY = Math.pow(Math.sin((p5.millis() - startTime) * bounceSpeed * Math.PI), 2) * spacing; 

        for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
            const newY = (i * spacing + deltaY) % p5.height;
            p5.line(columnPositionX, newY, columnPositionX + columnWidth, newY);
        }
    }
  }
  
  // Capture and frame counting logic
  if (capturing) {
    capturer.capture(p5.canvas);
    frameCounter++;
    if (frameCounter >= maxFrames) {
      isRendering = false
      capturing = false
    }
  }
  if (animationRunning) animationId = requestAnimationFrame(() => animate(p5));
}

$effect(() => {
  if (renderButton && isRendering) {
    renderButton.title = 'Stop Rendering';
  }
  if (renderButton && !isRendering) {
    renderButton.title = 'Render';
  }
})

onMount(() => {
  const pane = new Pane({
    container: document.getElementById('settings'),
  });

  // Main "Settings" folder
  const settingsFolder = pane.addFolder({
    title: 'Settings',
  });

  // General settings under "Settings"
  const tab = settingsFolder.addTab({
    pages: [
      { title: 'General' },
      { title: 'Columns' },
    ],
  });

  tab.pages[0].addBinding(PARAMS, 'duration', { min: 1, max: 120, step: 1 });
  tab.pages[0].addBinding(PARAMS, 'columns', { min: 1, max: maxColumns, step: 1 });
  tab.pages[0].addBinding(PARAMS, 'width', { min: 100, max: 2000, step: 10 }).on('change', handleCanvasResize);
  tab.pages[0].addBinding(PARAMS, 'height', { min: 100, max: 2000, step: 10 }).on('change', handleCanvasResize);

  PARAMS.cols.forEach((col, i) => {
    const columnFolder = tab.pages[1].addFolder({ title: `Column ${i + 1}`, expanded: false });
    columnFolder.addBinding(col, 'lineCount', { min: 1, max: 300, step: 1 });
    columnFolder.addBinding(col, 'width', { min: 1, max: 100, step: 1 });
    columnFolder.addBinding(col, 'positionX', { min: 0, max: 99, step: 1 });
    columnFolder.addBinding(col, 'speed', { min: 1, max: 10, step: 1 });
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

  const playerFolder = settingsFolder.addFolder({ title: 'Player'});
  playerFolder.addBinding(PARAMS, 'timeline', {
    readonly: true,
  });
  playerFolder.addButton({
    title: 'Randomize',
  }).on('click', () => {
    restartAnimation();
    randomizeColumnParams();
    pane.refresh();
  });
  startStopButton = playerFolder.addButton({
    title: 'Pause'
  }).on('click', () => toggleAnimation());
  restartButton = playerFolder.addButton({
    title: 'Restart'
  }).on('click', () => restartAnimation());

  const exportFolder = settingsFolder.addFolder({ title: 'Export'});
  exportFolder.addBinding(PARAMS, 'format', {
    options: {
      'WebM (Quality)': 'webm',
      'WebM (Fast)': 'webm-mediarecorder',
      'PNG': 'png',
      'JPG': 'jpg',
    }
  });
  renderButton = exportFolder.addButton({
    title: 'Render',
  }).on('click', () => {
    if (isRendering) {
      stopCapture();
    } else {
      startCapture();
    }
  });
  exportFolder.addButton({ title: 'Export Current Frame' }).on('click', exportCurrentFrame);
  exportFolder.addButton({
    title: 'Export state',
  }).on('click', () => {
    exportState(pane.exportState())
  });
  exportFolder.addButton({
    title: 'Import state',
  }).on('click', () => {
    importState(pane, (pane, importedState) => {
      pane.importState(importedState);
    });
  });
});

// Player and export
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
function restartAnimation() {
  currentLine = new Array(PARAMS.columns).fill(0);
  direction = new Array(PARAMS.columns).fill(1);
  frameCounter = 0;
  PARAMS.timeline = 0;
  startTime = p5Instance.millis();
}
function startCapture() {
  isRendering = true;
  if (!capturing) {
    capturer = new CCapture({
      format: PARAMS.format,
      framerate: framerate,
      quality: 100,
      verbose: true,
      timeLimit: PARAMS.duration,
    });
    capturing = true;
    restartAnimation();
    capturer.start();
  }
}
function stopCapture() {
  isRendering = false;
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

// Export/import Pane
function exportState(state) {
  const stateJSON = JSON.stringify(state, null, 2);
  const blob = new Blob([stateJSON], { type: 'application/json' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  const datetime = new Date().toISOString().replace(/[-T:.]/g, '_');
  link.download = `amld-state-${datetime}.json`;
  link.click();
}
function importState(pane, callback) {
  const importInput = document.createElement('input');
  importInput.type = 'file';
  importInput.accept = '.json';
  importInput.style.display = 'none';

  importInput.onchange = (event) => {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = () => {
        try {
          const importedState = JSON.parse(reader.result);
          callback(pane, importedState); // Pass pane along with imported state
        } catch (error) {
          alert('Failed to import state: Invalid JSON file');
        }
      };
      reader.readAsText(file);
    }
  };
  importInput.click();
}

// Canvas resize
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
</script>

<svelte:window bind:innerWidth bind:innerHeight onresize={handleCanvasResize}/>

<P5 {sketch} />
<div id="settings"></div>

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
  :global(:root) {
    --tp-blade-value-width: 240px;
  }
  #settings {
    position: fixed;
    top: 10px;
    right: 10px;
    z-index: 99;
    max-width: 400px;
    width: calc(100vw - 18px);
    background-color: black;
    overflow: scroll;
    max-height: calc(100vh - 18px);
    max-height: calc(100svh - 18px);
    border-radius: 6px;
  }
  @media only screen and (max-width: 600px) {
    #settings {
      max-width: unset;
    }
  }
</style>