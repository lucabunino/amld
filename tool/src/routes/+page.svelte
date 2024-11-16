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
let text = $state();
let textSize = $state();
let frame = 'bottom'

const framerate = 60;
const startingColumns = 4;
const maxColumns = 10;

const PARAMS = {
  duration: 20,
  columns: 10,
  width: 1920,
  height: 1080,
  format: 'webm',
  timeline: 0,
  halvingFactor: 0,
  template: 'fullscreen',
  text: "Registration open!",
  textSize: "l",
  cols: Array.from({ length: maxColumns }, (_, index) => {
    if (index < startingColumns) {
      // For the first 4 columns
      return {
        lineCount: Math.floor(Math.random() * 100) + 1,
        speed: Math.floor(Math.random() * 9) + 1,
        width: 100/startingColumns,  // Fixed width for the first 4 columns
        positionX: index * 100/startingColumns,
        thickness: Math.floor(Math.random() * 4) + 1,
        animationType: ['verticalBouncing', 'verticalContinuous', 'backAndForth', 'edge', 'cylinder', 'halfCylinder', 'box'][Math.floor(Math.random() * 4)],
        direction: ['upwards', 'downwards'][Math.floor(Math.random() * 2)],
      };
    } else {
      // For the remaining columns
      return {
        lineCount: Math.floor(Math.random() * 100) + 1,
        speed: Math.floor(Math.random() * 9) + 1,
        width: Math.floor(Math.random() * 99) + 1,
        positionX: Math.floor(Math.random() * 99),  // Random position for other columns
        thickness: Math.floor(Math.random() * 4) + 1,
        animationType: ['verticalBouncing', 'verticalContinuous', 'backAndForth', 'edge', 'cylinder', 'halfCylinder', 'box'][Math.floor(Math.random() * 4)],
        direction: ['upwards', 'downwards'][Math.floor(Math.random() * 2)],
      };
    }
  }),
};
let currentLine = $state(new Array(PARAMS.columns).fill(0));
let direction = $state(new Array(PARAMS.columns).fill(1));

let lastPreset;
const presets = {
  1: {
    lineCount: { min: 5, max: 10 },
    speed: { min: 1, max: 1 },
    thickness: { min: 1, max: 1 },
    orderedColumns: true,
    animationTypes: ['verticalContinuous', 'verticalBouncing'],
  },
  2: {
    lineCount: { min: 5, max: 40 },
    speed: { min: 1, max: 3 },
    thickness: { min: 1, max: 1 },
    orderedColumns: true,
    animationTypes: ['verticalContinuous', 'verticalBouncing'],
  },
  3: {
    lineCount: { min: 20, max: 200 },
    speed: { min: 1, max: 1 },
    thickness: { min: 1, max: 1 },
    orderedColumns: true,
    animationTypes: ['backAndForth'],
  },
  4: {
    lineCount: { min: 2, max: 10 },
    speed: { min: 1, max: 4 },
    thickness: { min: 5, max: 10 },
    orderedColumns: true,
    animationTypes: ['backAndForth'],
  },
  5: {
    lineCount: { min: 30, max: 30 },
    speed: { min: 5, max: 10 },
    thickness: { min: 1, max: 1 },
    orderedColumns: true,
    animationTypes: ['verticalContinuous', 'verticalBouncing'],
  },
  6: {
    lineCount: { min: 2, max: 30 },
    speed: { min: 5, max: 10 },
    thickness: { min: 10, max: 30 },
    orderedColumns: true,
    animationTypes: ['verticalContinuous', 'verticalBouncing'],
  },
  7: {
    lineCount: { min: 20, max: 300 },
    speed: { min: 1, max: 8 },
    thickness: { min: 1, max: 1 },
    orderedColumns: true,
    animationTypes: ['backAndForth', 'verticalContinuous',],
  },
  8: {
    lineCount: { min: 5, max: 40 },
    speed: { min: 1, max: 6 },
    thickness: { min: 1, max: 1 },
    orderedColumns: true,
    animationTypes: ['edge', 'verticalBouncing'],
  },
  9: {
    lineCount: { min: 6, max: 20 },
    speed: { min: 1, max: 2 },
    thickness: { min: 1, max: 10 },
    orderedColumns: true,
    animationTypes: ['cylinder', 'halfCylinder'],
  },
  10: {
    lineCount: { min: 5, max: 50 },
    speed: { min: 1, max: 2 },
    thickness: { min: 1, max: 10 },
    orderedColumns: false,
    width: { min: 10, max: 50 },
    positionX: { min: 0, max: 75 },
    animationTypes: ['verticalContinuous', 'verticalBouncing', 'backAndForth'],
  },
};
function getRandom(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

function setPreset(preset) {
  lastPreset = preset;
  const config = presets[preset];
  if (!config) {
    console.error(`Preset "${preset}" not found`);
    return;
  }

  if (config.orderedColumns) {
    let widths = [];
    let baseWidth = 100 / PARAMS.columns;
    let factorAdjustment = Math.pow(2, Math.abs(PARAMS.halvingFactor));

    for (let i = 0; i < PARAMS.columns; i++) {
      let widthMultiplier = (PARAMS.halvingFactor >= 0)
        ? Math.pow(factorAdjustment, i / (PARAMS.columns))
        : Math.pow(factorAdjustment, (PARAMS.columns - i) / (PARAMS.columns));

      widths.push(baseWidth * widthMultiplier);
    }

    // Normalize and round widths to integers
    const totalWidth = widths.reduce((a, b) => a + b, 0);
    widths = widths.map(width => Math.round((width / totalWidth) * 100));

    // Adjust rounding to ensure widths sum to exactly 100
    let widthSum = widths.reduce((a, b) => a + b, 0);
    let adjustment = 100 - widthSum;
    
    // Distribute adjustment across widths to reach exact 100
    for (let i = 0; adjustment !== 0; i = (i + 1) % widths.length) {
      if ((adjustment > 0 && widths[i] < 100) || (adjustment < 0 && widths[i] > 1)) {
        widths[i] += Math.sign(adjustment);
        adjustment -= Math.sign(adjustment);
      }
    }

    // Assign rounded widths and positions
    let positionX = 0;
    PARAMS.cols.forEach((col, index) => {
      col.lineCount = getRandom(config.lineCount.min, config.lineCount.max);
      col.speed = getRandom(config.speed.min, config.speed.max);
      col.thickness = getRandom(config.thickness.min, config.thickness.max);
      col.animationType = config.animationTypes[Math.floor(Math.random() * config.animationTypes.length)];
      col.direction = ['upwards', 'downwards'][Math.floor(Math.random() * 2)];
      col.width = widths[index];
      col.positionX = positionX;
      positionX += widths[index];
    });
  } else {
    // Handle non-ordered columns
    PARAMS.cols.forEach(col => {
      col.lineCount = getRandom(config.lineCount.min, config.lineCount.max);
      col.speed = getRandom(config.speed.min, config.speed.max);
      col.width = getRandom(config.width.min, config.width.max);
      col.positionX = getRandom(config.positionX.min, config.positionX.max);
      col.thickness = getRandom(config.thickness.min, config.thickness.max);
      col.animationType = config.animationTypes[Math.floor(Math.random() * config.animationTypes.length)];
      col.direction = ['upwards', 'downwards'][Math.floor(Math.random() * 2)];
    });
  }
}

let svgImage;
let font;
let sketch = $state((p5) => {
  
  p5.preload = () => {
    svgImage = p5.loadImage('/logo.svg');  // Replace with the correct path to your SVG file
    font = p5.loadFont('/fonts/SuisseIntl-SemiBold.otf');
  }

  p5Instance = p5;
  p5.setup = () => {
    p5.createCanvas(PARAMS.width, PARAMS.height, p5.WEBGL);
    p5.ortho(0, PARAMS.width, -PARAMS.height, 0);
    p5.textFont(font);
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
  PARAMS.timeline = (p5.millis() - startTime) / 1000;
  const duration = PARAMS.duration;
  const maxFrames = framerate * PARAMS.duration;

  let sideFrameThickness;
  let topFrameThickness;
  let bottomFrameThickness;

  if (PARAMS.template === 'fullscreen') {
    sideFrameThickness = 0;
    topFrameThickness = 0;
    bottomFrameThickness = 0;
  } else if (PARAMS.template === 'horizontalStandard' || PARAMS.template === 'horizontalCustom') {
    sideFrameThickness = p5.width / 100 * 2.5;
    topFrameThickness = p5.width / 100 * 2.5;
    bottomFrameThickness = p5.width / 100 * 10;
  } else if (PARAMS.template === 'other') {
    sideFrameThickness = p5.width / 100 * 2.5;
    topFrameThickness = p5.width / 100 * 2.5;
    bottomFrameThickness = p5.width / 100 * 2.5;
  }

  const usableWidth = p5.width - sideFrameThickness * 2; // Adjusted width
  const usableHeight = p5.height - topFrameThickness - bottomFrameThickness; // Adjusted height, excluding bottom frame

  for (let col = 0; col < PARAMS.columns; col++) {
    const columnWidth = usableWidth / 100 * PARAMS.cols[col].width;
    const columnPositionX = sideFrameThickness + (usableWidth / 100 * PARAMS.cols[col].positionX);
    const increment = (PARAMS.cols[col].lineCount / (maxFrames / 2)) * PARAMS.cols[col].speed;
    const animType = PARAMS.cols[col].animationType;
    const spacing = usableHeight / PARAMS.cols[col].lineCount;

    p5.stroke(255);
    p5.strokeWeight(PARAMS.cols[col].thickness).strokeCap(p5.SQUARE);

    if (animType === 'backAndForth') {
      // Set initial direction based on 'upwards' or 'downwards' parameter
      let initialDirection = PARAMS.cols[col].direction === 'upwards' ? -1 : 1;
      if (!direction[col]) direction[col] = initialDirection;

      // Update current line position with current direction
      currentLine[col] += increment * direction[col];

      // Adjust to prevent the animation from skipping the last frame
      if (currentLine[col] >= PARAMS.cols[col].lineCount) {
          currentLine[col] = PARAMS.cols[col].lineCount - 1; // Ensure the line doesn't overshoot the boundary
          direction[col] *= -1; // Reverse direction
      } else if (currentLine[col] <= -1) {
          currentLine[col] = 0; // Ensure it doesn't undershoot the starting point
          direction[col] *= -1; // Reverse direction
      }

      // Calculate usable width, considering the frame boundaries
      const columnStartX = columnPositionX;
      const columnEndX = columnPositionX + columnWidth;

      // Draw the lines up to the current line position
      for (let i = 0; i <= Math.floor(currentLine[col]); i++) {
          // Calculate yPos based on initial direction and the usable area
          let yPos = initialDirection === 1
              ? topFrameThickness + spacing * i // Lines appear from top to bottom within the usable height
              : topFrameThickness + (usableHeight - spacing * i); // Lines appear from bottom to top within the usable height

          // Ensure lines stay within the frame boundaries
          if (yPos >= topFrameThickness && yPos <= p5.height - bottomFrameThickness) {
              p5.line(columnStartX, yPos, columnEndX, yPos);
          }
      }
    }




    if (animType === 'verticalContinuous') {
      let speed = spacing / duration / 1000 * PARAMS.cols[col].speed;   
      let deltaY = ((p5.millis() - startTime) * speed) * (PARAMS.cols[col].direction === 'upwards' ? -1 : 1);

      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
        let newY = (i * spacing + deltaY) % usableHeight;
        if (newY < 0) newY += usableHeight;
        newY += topFrameThickness;

        if (newY >= topFrameThickness && newY <= p5.height - bottomFrameThickness) {
            p5.line(columnPositionX, newY, columnPositionX + columnWidth, newY);
        }
      }
    }


    if (animType === 'verticalBouncing') {
      const bounceSpeed = (1 / (duration * 1000) * PARAMS.cols[col].speed);
      const deltaY = Math.pow(Math.sin((p5.millis() - startTime) * bounceSpeed * Math.PI), 2) * spacing;

      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
        let newY = (i * spacing + deltaY) % usableHeight;
        if (newY < 0) newY += usableHeight; // Adjust to avoid negative Y values
        newY += topFrameThickness;

        if (newY >= topFrameThickness && newY <= p5.height - bottomFrameThickness) {
            p5.line(columnPositionX, newY, columnPositionX + columnWidth, newY);
        }
      }
    }

    if (animType === 'edge') {
      const durationMillis = PARAMS.duration * 1000 / PARAMS.cols[col].speed;
      const elapsed = ((p5.millis() - startTime) % durationMillis) / durationMillis;

      const spacingFactorTop = Math.pow(Math.sin(elapsed * Math.PI), 2);
      const spacingFactorBottom = 1 - spacingFactorTop;
      const baseSpacing = usableHeight / PARAMS.cols[col].lineCount; // Adjust for usableHeight
      const dynamicSpacingTop = baseSpacing * spacingFactorTop;
      const dynamicSpacingBottom = baseSpacing * spacingFactorBottom;

      // Draw lines top-to-bottom (first set)
      let currentYTop = topFrameThickness; // Start from the top within usable area
      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
        p5.line(columnPositionX, currentYTop, columnPositionX + columnWidth, currentYTop);
        currentYTop += dynamicSpacingTop;
        if (currentYTop > p5.height - bottomFrameThickness) break; // Stop when reaching the bottom frame boundary
      }

      // Draw lines bottom-to-top (second set)
      let currentYBottom = p5.height - bottomFrameThickness; // Start from the bottom within usable area
      for (let i = 0; i < PARAMS.cols[col].lineCount; i++) {
        p5.line(columnPositionX, currentYBottom, columnPositionX + columnWidth, currentYBottom);
        currentYBottom -= dynamicSpacingBottom;
        if (currentYBottom < topFrameThickness) break; // Stop when reaching the top frame boundary
      }
    }

    if (animType === 'cylinder') {
      const numLines = PARAMS.cols[col].lineCount;
      const columnWidth = usableWidth / 100 * PARAMS.cols[col].width; 
      const startX = -columnWidth / 2, endX = columnWidth / 2;

      p5.push();
      p5.translate(columnPositionX + columnWidth / 2, usableHeight / 2 + topFrameThickness, -p5.height);

      const anglePerLine = p5.TWO_PI / numLines; // Angle per line (360° / numLines)
      
      const durationMillis = PARAMS.duration * 1000 / PARAMS.cols[col].speed; // Time to rotate by one line
      const elapsed = ((p5.millis() - startTime) % durationMillis) / durationMillis;

      const rotationAmount = elapsed * anglePerLine; // The rotation amount for each line

      p5.rotateX(rotationAmount); // Apply linear rotation (no easing)

      p5.stroke(255);
      p5.noFill();

      const radius = usableHeight / 2;
      const angleOffset = anglePerLine / 2; // Offset by half the angle per line to center the lines
      for (let i = 0; i < numLines; i++) {
        const angle = p5.map(i, 0, numLines, 0, p5.TWO_PI) + angleOffset;

        const rotatedAngle = angle + rotationAmount; // Apply the linear rotation directly
        const y1 = p5.cos(rotatedAngle) * radius;
        const z1 = p5.sin(rotatedAngle) * radius;

        if (Math.abs(y1) <= p5.height / 2 && Math.abs(z1) <= p5.height / 2) {
            p5.line(startX, y1, z1, endX, y1, z1); // Line along the column width
        }
      }

      p5.pop();
    }




    if (animType === 'halfCylinder') {
        const numLines = PARAMS.cols[col].lineCount;

        // Use the provided usableWidth
        const columnWidth = usableWidth / 100 * PARAMS.cols[col].width;
        const startX = -columnWidth / 2, endX = columnWidth / 2;

        p5.push();
        p5.translate(columnPositionX + columnWidth / 2, usableHeight / 2 + topFrameThickness, -p5.height);

        // Calculate rotation based on elapsed time and speed
        const durationMillis = PARAMS.duration * 1000 / PARAMS.cols[col].speed; // Convert duration from seconds to milliseconds
        const elapsed = ((p5.millis() - startTime) % durationMillis) / durationMillis;
        p5.rotateX(elapsed * p5.TWO_PI);  // Rotate for 360° over the duration

        p5.stroke(255);
        p5.noFill();

        // Adjust the radius to account for frame thickness
        const radius = usableHeight / 2;  // Use half the height for the vertical radius

        for (let i = 0; i < numLines; i++) {
            const angle = p5.map(i, 0, numLines, 0, p5.TWO_PI);
            const y1 = p5.cos(angle) * radius;
            const z1 = p5.sin(angle) * radius;

            // Only draw lines where Z position is positive (in the visible half-circle area)
            // And ensure lines are within the usable height (taking frame thickness into account)
            if (z1 >= topFrameThickness && z1 <= p5.height - bottomFrameThickness) {
                p5.line(startX, y1, z1, endX, y1, z1); // Line along the column width
            }
        }

        p5.pop();
    }

    
    if (animType === 'box') {
      const columnHeight = usableHeight / Math.SQRT2;
      const columnWidth = usableWidth / 100 * PARAMS.cols[col].width;
      const numLines = PARAMS.cols[col].lineCount; // Number of lines based on column's lineCount

      p5.push();
      p5.translate(columnPositionX + columnWidth / 2, usableHeight / 2 + topFrameThickness, -p5.height);

      // Calculate the rotation angle based on elapsed time
      const durationMillis = PARAMS.duration * 1000 / PARAMS.cols[col].speed; // Convert duration to milliseconds
      const elapsed = ((p5.millis() - startTime) % durationMillis) / durationMillis;

      // Rotate the box on the X-axis based on elapsed time
      p5.rotateX(p5.TWO_PI * elapsed);  // Rotate on the X-axis over time

      p5.stroke(255);  // White outline for the lines
      p5.noFill(); // No fill color for the lines
      
      const lineSpacing = columnHeight / (numLines - 1);
      for (let i = 0; i < numLines; i++) {
        const yPos = -columnHeight / 2 + i * lineSpacing; // Y position of the current line
        p5.line(-columnWidth / 2, yPos, -columnHeight / 2, columnWidth / 2, yPos, -columnHeight / 2); // Front side lines
      }

      for (let i = 0; i < numLines; i++) {
        const yPos = -columnHeight / 2 + i * lineSpacing; // Y position of the current line
        p5.line(-columnWidth / 2, yPos, columnHeight / 2, columnWidth / 2, yPos, columnHeight / 2); // Back side lines
      }

      for (let i = 0; i < numLines; i++) {
        const yPos = -columnHeight / 2 + i * lineSpacing; // Y position of the current line
        p5.line(-columnWidth / 2, -columnHeight / 2, yPos, columnWidth / 2, -columnHeight / 2, yPos); // Bottom side lines
      }

      for (let i = 0; i < numLines; i++) {
        const yPos = -columnHeight / 2 + i * lineSpacing; // Y position of the current line
        p5.line(-columnWidth / 2, columnHeight / 2, yPos, columnWidth / 2, columnHeight / 2, yPos); // Top side lines
      }

      p5.pop();
    }
  }

  if (PARAMS.template !== 'fullscreen') {
    p5.fill(0, 0, 0);
    p5.rectMode(p5.CORNER);
    p5.noStroke();
    // p5.noStroke();
    p5.rect(0, 0, sideFrameThickness, p5.height); // Left frame
    p5.rect(p5.width - sideFrameThickness, 0, sideFrameThickness, p5.height); // Right frame
    p5.rect(0, 0, p5.width, topFrameThickness); // Top frame
    p5.rect(0, p5.height - bottomFrameThickness, p5.width, bottomFrameThickness); // Bottom frame

    const svgAspectRatio = svgImage.width / svgImage.height; // Aspect ratio of the SVG
    const svgHeight = bottomFrameThickness / 2;
    const svgWidth = svgHeight * svgAspectRatio;
    const svgX = sideFrameThickness;
    const svgY = p5.height - bottomFrameThickness + (bottomFrameThickness - svgHeight) / 2;
    
    // Draw the SVG image
    p5.image(svgImage, svgX, svgY, svgWidth, svgHeight);

    if (PARAMS.textSize === 'm') {
      p5.textSize(p5.width * 0.021875);
    } else if (PARAMS.textSize === 'l') {
      p5.textSize(p5.width * 0.041875);
    }
  }
  if (PARAMS.template === 'horizontalCustom') {
    // Text properties
    const textBaselineY = p5.height - bottomFrameThickness / 4; // Baseline position
    const textX = p5.width - sideFrameThickness; // Padding from the right edge

    // Set text styles
    p5.textAlign(p5.RIGHT, p5.BASELINE); // Align right and use alphabetic baseline
    p5.fill(255);
    p5.text(PARAMS.text, textX, textBaselineY);
  } else if (PARAMS.template === 'horizontalStandard') {
    // Text properties
    const lineHeight = p5.textSize() * 1; // Line height
    p5.textLeading(lineHeight); // Set line height for multiline text
    
    // Set base Y position for text (bottom of frame)
    const baseY = p5.height - bottomFrameThickness / 4;
    
    // Set text styles
    p5.textAlign(p5.LEFT, p5.BASELINE);
    p5.fill(255);

    // Set the text size (you can adjust this based on your preference)
    if (PARAMS.textSize === 'm') {
      p5.textSize(p5.width * 0.021875);
    } else if (PARAMS.textSize === 'l') {
      p5.textSize(p5.width * 0.041875);
    }

    const xPositions = [
      (p5.width - sideFrameThickness * 2) / 4 * 1 + sideFrameThickness,
      (p5.width - sideFrameThickness * 2) / 4 * 2 + sideFrameThickness,
      (p5.width - sideFrameThickness * 2) / 4 * 3 + sideFrameThickness,
    ];

    // Text for the three columns
    const multilineTexts = [
      "Shaping the Future\nwith AI",
      "February 11-14\n2025",
      "Lausanne\nSwitzerland"
    ];

    // Loop through each multiline text and render at different X positions
    multilineTexts.forEach((line, index) => {
      // Split the text into lines to get the total number of lines
      const lines = line.split("\n");

      // Render each multiline text at its specific X position
      p5.text(line, xPositions[index], p5.height - bottomFrameThickness/4 - lines.length*lineHeight + lineHeight);
    });
  }

  function textHeight(text, maxWidth) {
    var words = text.split(' ');
    var line = '';
    var h = this._textLeading;

    for (var i = 0; i < words.length; i++) {
        var testLine = line + words[i] + ' ';
        var testWidth = drawingContext.measureText(testLine).width;

        if (testWidth > maxWidth && i > 0) {
            line = words[i] + ' ';
            h += this._textLeading;
        } else {
            line = testLine;
        }
    }
    return h;
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
  PARAMS.columns = startingColumns
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
      { title: 'Presets' },
      { title: 'Columns' },
    ],
  });

  tab.pages[0].addBinding(PARAMS, 'duration', { min: 1, max: 120, step: 1 });
  tab.pages[0].addBinding(PARAMS, 'template', {
    options: {
      'Fullscreen': 'fullscreen',
      'Horizontal custom': 'horizontalCustom',
      'Horizontal standard': 'horizontalStandard',
    }
  }).on('change', () => {setTemplate(PARAMS.template, text, textSize); pane.refresh(); console.log(PARAMS.template);});
  text = tab.pages[0].addBinding(PARAMS, 'text', {hidden: true});
  textSize = tab.pages[0].addBinding(PARAMS, 'textSize', {
    hidden: true,
    options: {
      'M': 'm',
      'L': 'l',
    }
  });
  tab.pages[0].addBinding(PARAMS, 'columns', { min: 1, max: maxColumns, step: 1 }).on('change', () => {if (presets[lastPreset]?.orderedColumns) {setPreset(lastPreset);pane.refresh();}});
  tab.pages[0].addBinding(PARAMS, 'width', { min: 100, max: 2000, step: 10 }).on('change', handleCanvasResize);
  tab.pages[0].addBinding(PARAMS, 'height', { min: 100, max: 2000, step: 10 }).on('change', handleCanvasResize);

  const orderedPresets = tab.pages[1].addFolder({ title: `Ordered`});
  orderedPresets.addBinding(PARAMS, 'halvingFactor', {min: -5, max: 5, step: 0.1,}).on('change', () => {if (presets[lastPreset]?.orderedColumns) {setPreset(lastPreset);pane.refresh();}});
  orderedPresets.addButton({title: 'Preset 1',}).on('click', () => {restartAnimation();setPreset(1);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 2',}).on('click', () => {restartAnimation();setPreset(2);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 3',}).on('click', () => {restartAnimation();setPreset(3);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 4',}).on('click', () => {restartAnimation();setPreset(4);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 5',}).on('click', () => {restartAnimation();setPreset(5);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 6',}).on('click', () => {restartAnimation();setPreset(6);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 7',}).on('click', () => {restartAnimation();setPreset(7);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 8',}).on('click', () => {restartAnimation();setPreset(8);pane.refresh();});
  orderedPresets.addButton({title: 'Preset 9',}).on('click', () => {restartAnimation();setPreset(9);pane.refresh();});

  const chaoticPresets = tab.pages[1].addFolder({ title: `Chaotic`});

  tab.pages[1].addBlade({
    view: 'separator',
  });
  const exportImport = tab.pages[1].addFolder({ title: `Export/import presets`});
  exportImport.addButton({
    title: 'Export preset',
  }).on('click', () => {
    exportPreset(pane.exportPreset())
  });
  exportImport.addButton({
    title: 'Import preset',
  }).on('click', () => {
    importPreset(pane, (pane, importedState) => {
      pane.importPreset(importedState);
    });
  });

  PARAMS.cols.forEach((col, i) => {
    const columnFolder = tab.pages[2].addFolder({ title: `Column ${i + 1}`, expanded: false });
    columnFolder.addBinding(col, 'lineCount', { min: 1, max: 300, step: 1 });
    columnFolder.addBinding(col, 'width', { min: 1, max: 100, step: 1 });
    columnFolder.addBinding(col, 'positionX', { min: 0, max: 99, step: 1 });
    columnFolder.addBinding(col, 'speed', { min: 1, max: 10, step: 1 });
    columnFolder.addBinding(col, 'thickness', { min: 1, max: 30, step: 1 });
    columnFolder.addBinding(col, 'animationType', {
      options: {
        'Back and Forth': 'backAndForth',
        'Vertical Bouncing': 'verticalBouncing',
        'Vertical Continuous': 'verticalContinuous',
        'Edge': 'edge',
        '3D Cylinder': 'cylinder',
        '3D Cylinder (half)': 'halfCylinder',
        'Box': 'box',
      }
    }).on('change', restartAnimation);
    columnFolder.addBinding(col, 'direction', {
      options: {
        'Upwards': 'upwards',
        'Downwards': 'downwards',
      }
    });
  });
  pane.addBlade({
    view: 'separator',
  });
  const playerFolder = pane.addFolder({ title: 'Player'});
  playerFolder.addBinding(PARAMS, 'timeline', {
    readonly: true,
  });
  startStopButton = playerFolder.addButton({
    title: 'Pause'
  }).on('click', () => toggleAnimation());
  restartButton = playerFolder.addButton({
    title: 'Restart'
  }).on('click', () => restartAnimation());
  pane.addBlade({
    view: 'separator',
  });
  const exportFolder = pane.addFolder({ title: 'Export'});
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
});

// Set format
function setTemplate(template, text, textSize) {
  text.hidden = true;
  textSize.hidden = true;
  if (template === 'horizontalCustom') {
      text.hidden = false;
      textSize.hidden = false;
      PARAMS.width = 1920;
      PARAMS.height = 1080;
  } else if (template === 'horizontalStandard') {
      PARAMS.width = 1920;
      PARAMS.height = 1080;
      PARAMS.text = 'Layout da completare';
      PARAMS.textSize = 'm';
  }
}

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
function exportPreset(state) {
  const stateJSON = JSON.stringify(state, null, 2);
  const blob = new Blob([stateJSON], { type: 'application/json' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  const datetime = new Date().toISOString().replace(/[-T:.]/g, '_');
  link.download = `amld-state-${datetime}.json`;
  link.click();
}
function importPreset(pane, callback) {
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
  p5Instance.ortho(0, PARAMS.width, -PARAMS.height, 0);
  fitCanvasToViewport();
}
function fitCanvasToViewport() {
  const canvasAspect = PARAMS.width / PARAMS.height;
  const viewportAspect = innerWidth / innerHeight;
  let scale = 1;
  let screenFactor = innerWidth > 600 ? .8 : .9;
  if (canvasAspect > viewportAspect) {
    scale = innerWidth*screenFactor / PARAMS.width;
  } else {
    scale = innerHeight*screenFactor / PARAMS.height;
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