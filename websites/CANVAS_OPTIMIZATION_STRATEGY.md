# Canvas Graphics Optimization Strategy
## For Interactive UI Applications

### Executive Summary

The HTML5 Canvas API provides powerful capabilities for creating interactive graphics, charts, timelines, and CAD-like applications directly in the browser. However, without proper optimization techniques, Canvas-based applications can suffer from poor performance, high memory usage, and unresponsive user experiences.

This strategy document outlines production-ready patterns for building performant, interactive Canvas applications. It covers architecture patterns, rendering optimizations, event handling, and scaling strategies derived from real-world implementations in charting libraries, graph visualizations, and interactive design tools.

**Key Principles:**
- Minimize pixels redrawn per frame through dirty region tracking
- Layer canvases based on update frequency
- Use offscreen canvases for pre-rendering static or complex content
- Implement efficient hit detection for user interactions
- Batch rendering operations to reduce API calls
- Choose Canvas vs SVG vs WebGL based on data complexity and interaction requirements

---

## Architecture Layers

```
┌─────────────────────────────────────────────┐
│   Layer 4: Application/UI Layer            │
│   (User interactions, state management)     │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│   Layer 3: Interaction Layer                │
│   (Event handling, hit detection)           │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│   Layer 2: Rendering Layer                  │
│   (Drawing logic, scene graph, batching)    │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│   Layer 1: Canvas Infrastructure Layer      │
│   (Canvas elements, contexts, buffers)      │
└─────────────────────────────────────────────┘
```

---

## Layer 1: Canvas Infrastructure Layer

### Purpose
Establish the foundation for Canvas rendering with proper element setup, context configuration, and supporting canvas buffers.

### Implementation Guidelines

**1. Primary Canvas Setup**

Create a properly sized and configured main canvas element:

```javascript
// Setup with device pixel ratio support
function setupCanvas(container) {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d', {
    alpha: false,  // Disable transparency if not needed - improves performance
    desynchronized: true  // Reduces input latency for animations
  });
  
  // Get actual device pixel ratio for crisp rendering on high-DPI displays
  const dpr = window.devicePixelRatio || 1;
  
  // Set display size (css pixels)
  const displayWidth = container.clientWidth;
  const displayHeight = container.clientHeight;
  
  // Set actual canvas size in memory (scaled for device)
  canvas.width = displayWidth * dpr;
  canvas.height = displayHeight * dpr;
  
  // Set display size via CSS
  canvas.style.width = displayWidth + 'px';
  canvas.style.height = displayHeight + 'px';
  
  // Scale all drawing operations by dpr
  ctx.scale(dpr, dpr);
  
  container.appendChild(canvas);
  
  return { canvas, ctx, dpr };
}
```

**2. Offscreen Canvas Strategy**

Create dedicated offscreen canvases for different purposes:

```javascript
class CanvasManager {
  constructor(width, height) {
    // Main visible canvas
    this.mainCanvas = document.createElement('canvas');
    this.mainCtx = this.mainCanvas.getContext('2d');
    
    // Offscreen canvas for static background elements
    this.bgCanvas = document.createElement('canvas');
    this.bgCtx = this.bgCanvas.getContext('2d');
    
    // Offscreen canvas for hit detection
    this.hitCanvas = document.createElement('canvas');
    this.hitCtx = this.hitCanvas.getContext('2d', { willReadFrequently: true });
    
    // Set dimensions for all canvases
    [this.mainCanvas, this.bgCanvas, this.hitCanvas].forEach(canvas => {
      canvas.width = width;
      canvas.height = height;
    });
  }
  
  // Pre-render static content once
  renderBackground(renderFn) {
    this.bgCtx.clearRect(0, 0, this.bgCanvas.width, this.bgCanvas.height);
    renderFn(this.bgCtx);
  }
  
  // Composite background onto main canvas
  compositeBackground() {
    this.mainCtx.drawImage(this.bgCanvas, 0, 0);
  }
}
```

**3. Multi-Layer Canvas System**

For complex UIs, layer canvases based on update frequency:

```javascript
class LayeredCanvas {
  constructor(container, layerConfig) {
    this.container = container;
    this.layers = new Map();
    
    // Create layers from config: { name, zIndex, alpha }
    layerConfig.forEach(config => {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d', { alpha: config.alpha ?? true });
      
      // Position layers absolutely on top of each other
      canvas.style.position = 'absolute';
      canvas.style.zIndex = config.zIndex;
      canvas.style.pointerEvents = 'none';  // Let events pass through to main layer
      
      this.layers.set(config.name, { canvas, ctx, dirty: false });
      container.appendChild(canvas);
    });
    
    // Set main layer to capture events
    const mainLayer = this.layers.get('main');
    if (mainLayer) {
      mainLayer.canvas.style.pointerEvents = 'auto';
    }
  }
  
  getLayer(name) {
    return this.layers.get(name);
  }
  
  markDirty(name) {
    const layer = this.layers.get(name);
    if (layer) layer.dirty = true;
  }
  
  render(renderFns) {
    this.layers.forEach((layer, name) => {
      if (layer.dirty && renderFns[name]) {
        layer.ctx.clearRect(0, 0, layer.canvas.width, layer.canvas.height);
        renderFns[name](layer.ctx);
        layer.dirty = false;
      }
    });
  }
}

// Usage
const layers = new LayeredCanvas(container, [
  { name: 'background', zIndex: 1, alpha: false },
  { name: 'main', zIndex: 2, alpha: true },
  { name: 'ui', zIndex: 3, alpha: true }
]);
```

**Benefits:**
- Proper high-DPI display support prevents blurry graphics
- Disabling alpha channel improves rendering performance when transparency isn't needed
- Offscreen canvases enable pre-rendering expensive content
- Layered approach isolates updates to specific canvas layers

---

## Layer 2: Rendering Layer

### Purpose
Implement efficient drawing logic with dirty region tracking, batching, and scene graph management.

### Implementation Guidelines

**1. Dirty Region Tracking**

Only redraw areas that have changed:

```javascript
class DirtyRegionManager {
  constructor(canvasWidth, canvasHeight) {
    this.regions = [];
    this.fullRedraw = false;
    this.bounds = { x: 0, y: 0, width: canvasWidth, height: canvasHeight };
  }
  
  markDirty(x, y, width, height) {
    // Expand slightly to avoid edge artifacts
    const padding = 2;
    this.regions.push({
      x: Math.max(0, x - padding),
      y: Math.max(0, y - padding),
      width: Math.min(this.bounds.width, width + padding * 2),
      height: Math.min(this.bounds.height, height + padding * 2)
    });
  }
  
  markFullRedraw() {
    this.fullRedraw = true;
  }
  
  getDirtyRegions() {
    if (this.fullRedraw) {
      return [this.bounds];
    }
    
    // Merge overlapping regions to reduce draw calls
    return this.mergeRegions(this.regions);
  }
  
  clear() {
    this.regions = [];
    this.fullRedraw = false;
  }
  
  mergeRegions(regions) {
    // Simple merge - combine regions if they overlap
    if (regions.length === 0) return [];
    
    const merged = [regions[0]];
    for (let i = 1; i < regions.length; i++) {
      const region = regions[i];
      let didMerge = false;
      
      for (let j = 0; j < merged.length; j++) {
        if (this.intersects(region, merged[j])) {
          merged[j] = this.combine(region, merged[j]);
          didMerge = true;
          break;
        }
      }
      
      if (!didMerge) {
        merged.push(region);
      }
    }
    
    return merged;
  }
  
  intersects(r1, r2) {
    return !(r2.x > r1.x + r1.width ||
             r2.x + r2.width < r1.x ||
             r2.y > r1.y + r1.height ||
             r2.y + r2.height < r1.y);
  }
  
  combine(r1, r2) {
    const x = Math.min(r1.x, r2.x);
    const y = Math.min(r1.y, r2.y);
    const width = Math.max(r1.x + r1.width, r2.x + r2.width) - x;
    const height = Math.max(r1.y + r1.height, r2.y + r2.height) - y;
    return { x, y, width, height };
  }
}
```

**2. Batch Rendering**

Group rendering operations to minimize context state changes:

```javascript
class BatchRenderer {
  constructor(ctx) {
    this.ctx = ctx;
    this.batches = new Map();
  }
  
  // Add a shape to a batch based on its style
  addToBatch(shape, style) {
    const key = this.getStyleKey(style);
    
    if (!this.batches.has(key)) {
      this.batches.set(key, { style, shapes: [] });
    }
    
    this.batches.get(key).shapes.push(shape);
  }
  
  getStyleKey(style) {
    // Create a unique key from style properties
    return `${style.fill}-${style.stroke}-${style.lineWidth}`;
  }
  
  // Render all batches with minimal state changes
  render() {
    this.batches.forEach(batch => {
      this.ctx.save();
      
      // Set style once for entire batch
      if (batch.style.fill) {
        this.ctx.fillStyle = batch.style.fill;
      }
      if (batch.style.stroke) {
        this.ctx.strokeStyle = batch.style.stroke;
        this.ctx.lineWidth = batch.style.lineWidth || 1;
      }
      
      // Create single path for all shapes in batch
      this.ctx.beginPath();
      batch.shapes.forEach(shape => {
        this.addShapeToPath(shape);
      });
      
      // Single fill/stroke call for entire batch
      if (batch.style.fill) this.ctx.fill();
      if (batch.style.stroke) this.ctx.stroke();
      
      this.ctx.restore();
    });
    
    this.batches.clear();
  }
  
  addShapeToPath(shape) {
    switch (shape.type) {
      case 'circle':
        this.ctx.moveTo(shape.x + shape.radius, shape.y);
        this.ctx.arc(shape.x, shape.y, shape.radius, 0, Math.PI * 2);
        break;
      case 'rect':
        this.ctx.rect(shape.x, shape.y, shape.width, shape.height);
        break;
      case 'path':
        // Custom path points
        if (shape.points.length > 0) {
          this.ctx.moveTo(shape.points[0].x, shape.points[0].y);
          for (let i = 1; i < shape.points.length; i++) {
            this.ctx.lineTo(shape.points[i].x, shape.points[i].y);
          }
        }
        break;
    }
  }
}
```

**3. Scene Graph with Dirty Flags**

Organize objects hierarchically with change tracking:

```javascript
class SceneNode {
  constructor() {
    this.children = [];
    this.dirty = true;
    this.visible = true;
    this.transform = { x: 0, y: 0, scaleX: 1, scaleY: 1, rotation: 0 };
  }
  
  addChild(node) {
    this.children.push(node);
    node.parent = this;
    this.markDirty();
  }
  
  removeChild(node) {
    const index = this.children.indexOf(node);
    if (index !== -1) {
      this.children.splice(index, 1);
      node.parent = null;
      this.markDirty();
    }
  }
  
  markDirty() {
    this.dirty = true;
    // Propagate dirty flag up the tree
    if (this.parent) {
      this.parent.markDirty();
    }
  }
  
  render(ctx, dirtyManager) {
    if (!this.visible) return;
    
    ctx.save();
    this.applyTransform(ctx);
    
    // Only render if dirty
    if (this.dirty) {
      this.draw(ctx);
      
      // Track dirty region
      const bounds = this.getBounds();
      dirtyManager.markDirty(bounds.x, bounds.y, bounds.width, bounds.height);
      
      this.dirty = false;
    }
    
    // Render children
    this.children.forEach(child => child.render(ctx, dirtyManager));
    
    ctx.restore();
  }
  
  applyTransform(ctx) {
    ctx.translate(this.transform.x, this.transform.y);
    ctx.scale(this.transform.scaleX, this.transform.scaleY);
    ctx.rotate(this.transform.rotation);
  }
  
  draw(ctx) {
    // Override in subclasses
  }
  
  getBounds() {
    // Override in subclasses to return { x, y, width, height }
    return { x: 0, y: 0, width: 0, height: 0 };
  }
}
```

**4. Animation Loop with requestAnimationFrame**

Implement efficient rendering loop:

```javascript
class RenderLoop {
  constructor(canvas, ctx, scene) {
    this.canvas = canvas;
    this.ctx = ctx;
    this.scene = scene;
    this.running = false;
    this.lastTime = 0;
    this.fps = 0;
    this.frameCount = 0;
    this.fpsUpdateTime = 0;
    this.dirtyManager = new DirtyRegionManager(canvas.width, canvas.height);
  }
  
  start() {
    this.running = true;
    this.lastTime = performance.now();
    this.loop();
  }
  
  stop() {
    this.running = false;
  }
  
  loop = (currentTime) => {
    if (!this.running) return;
    
    const deltaTime = currentTime - this.lastTime;
    this.lastTime = currentTime;
    
    // Update FPS counter
    this.updateFPS(currentTime);
    
    // Update scene
    this.scene.update(deltaTime);
    
    // Render only dirty regions
    this.render();
    
    requestAnimationFrame(this.loop);
  }
  
  render() {
    const dirtyRegions = this.dirtyManager.getDirtyRegions();
    
    if (dirtyRegions.length === 0) return;  // Nothing to redraw
    
    dirtyRegions.forEach(region => {
      // Clear only the dirty region
      this.ctx.clearRect(region.x, region.y, region.width, region.height);
      
      // Save and clip to dirty region
      this.ctx.save();
      this.ctx.beginPath();
      this.ctx.rect(region.x, region.y, region.width, region.height);
      this.ctx.clip();
      
      // Render scene
      this.scene.render(this.ctx, this.dirtyManager);
      
      this.ctx.restore();
    });
    
    this.dirtyManager.clear();
  }
  
  updateFPS(currentTime) {
    this.frameCount++;
    const elapsed = currentTime - this.fpsUpdateTime;
    
    if (elapsed >= 1000) {  // Update FPS every second
      this.fps = Math.round((this.frameCount * 1000) / elapsed);
      this.frameCount = 0;
      this.fpsUpdateTime = currentTime;
    }
  }
}
```

**Benefits:**
- Dirty region tracking reduces pixels processed per frame by 70-90%
- Batch rendering minimizes expensive canvas API calls
- Scene graph provides natural organization and change propagation
- requestAnimationFrame ensures smooth, browser-optimized updates

---

## Layer 3: Interaction Layer

### Purpose
Handle user interactions efficiently with hit detection, event management, and input handling.

### Implementation Guidelines

**1. Mathematical Hit Detection**

For simple shapes, use mathematical calculations:

```javascript
class HitDetector {
  // Check if point is inside rectangle
  static hitTestRect(point, rect) {
    return point.x >= rect.x &&
           point.x <= rect.x + rect.width &&
           point.y >= rect.y &&
           point.y <= rect.y + rect.height;
  }
  
  // Check if point is inside circle
  static hitTestCircle(point, circle) {
    const dx = point.x - circle.x;
    const dy = point.y - circle.y;
    return (dx * dx + dy * dy) <= (circle.radius * circle.radius);
  }
  
  // Check if point is near a line (with tolerance)
  static hitTestLine(point, line, tolerance = 5) {
    const { x1, y1, x2, y2 } = line;
    
    // Distance from point to line segment
    const A = point.x - x1;
    const B = point.y - y1;
    const C = x2 - x1;
    const D = y2 - y1;
    
    const dot = A * C + B * D;
    const lenSq = C * C + D * D;
    let param = -1;
    
    if (lenSq !== 0) {
      param = dot / lenSq;
    }
    
    let xx, yy;
    
    if (param < 0) {
      xx = x1;
      yy = y1;
    } else if (param > 1) {
      xx = x2;
      yy = y2;
    } else {
      xx = x1 + param * C;
      yy = y1 + param * D;
    }
    
    const dx = point.x - xx;
    const dy = point.y - yy;
    
    return Math.sqrt(dx * dx + dy * dy) <= tolerance;
  }
  
  // Check if point is inside polygon
  static hitTestPolygon(point, polygon) {
    let inside = false;
    const points = polygon.points;
    
    for (let i = 0, j = points.length - 1; i < points.length; j = i++) {
      const xi = points[i].x, yi = points[i].y;
      const xj = points[j].x, yj = points[j].y;
      
      const intersect = ((yi > point.y) !== (yj > point.y)) &&
        (point.x < (xj - xi) * (point.y - yi) / (yj - yi) + xi);
      
      if (intersect) inside = !inside;
    }
    
    return inside;
  }
}
```

**2. Color-Based Hit Detection (for complex shapes)**

Use a dedicated hit canvas with color-coded shapes:

```javascript
class ColorHitDetector {
  constructor(width, height) {
    this.hitCanvas = document.createElement('canvas');
    this.hitCanvas.width = width;
    this.hitCanvas.height = height;
    this.hitCtx = this.hitCanvas.getContext('2d', { willReadFrequently: true });
    
    this.colorToObject = new Map();
    this.currentColorIndex = 1;  // 0 reserved for no hit
  }
  
  registerObject(obj) {
    // Generate unique color for this object
    const color = this.indexToColor(this.currentColorIndex);
    this.colorToObject.set(color, obj);
    obj._hitColor = color;
    this.currentColorIndex++;
    
    return color;
  }
  
  indexToColor(index) {
    // Convert index to RGB color
    const r = (index & 0xFF0000) >> 16;
    const g = (index & 0x00FF00) >> 8;
    const b = (index & 0x0000FF);
    return `rgb(${r},${g},${b})`;
  }
  
  colorToIndex(r, g, b) {
    return (r << 16) | (g << 8) | b;
  }
  
  drawObject(obj, drawFn) {
    this.hitCtx.fillStyle = obj._hitColor;
    this.hitCtx.strokeStyle = obj._hitColor;
    drawFn(this.hitCtx);
  }
  
  getObjectAt(x, y) {
    const pixel = this.hitCtx.getImageData(x, y, 1, 1).data;
    const color = `rgb(${pixel[0]},${pixel[1]},${pixel[2]})`;
    return this.colorToObject.get(color) || null;
  }
  
  clear() {
    this.hitCtx.clearRect(0, 0, this.hitCanvas.width, this.hitCanvas.height);
  }
}
```

**3. Hybrid Hit Detection Strategy**

Combine both approaches for optimal performance:

```javascript
class HybridHitDetector {
  constructor(width, height) {
    this.colorDetector = new ColorHitDetector(width, height);
    this.spatialIndex = new Map();  // Rough spatial partitioning
  }
  
  registerObject(obj) {
    // Register with color detector for complex shapes
    if (obj.complex) {
      this.colorDetector.registerObject(obj);
    }
    
    // Add to spatial index for quick filtering
    const bounds = obj.getBounds();
    const cellKey = this.getCellKey(bounds.x, bounds.y);
    
    if (!this.spatialIndex.has(cellKey)) {
      this.spatialIndex.set(cellKey, []);
    }
    this.spatialIndex.get(cellKey).push(obj);
  }
  
  getCellKey(x, y) {
    const cellSize = 100;  // Adjust based on typical object size
    const cellX = Math.floor(x / cellSize);
    const cellY = Math.floor(y / cellSize);
    return `${cellX},${cellY}`;
  }
  
  hitTest(x, y) {
    // First, check which cell the point is in
    const cellKey = this.getCellKey(x, y);
    const candidates = this.spatialIndex.get(cellKey) || [];
    
    // Filter by bounding box
    const boundsCandidates = candidates.filter(obj => {
      const bounds = obj.getBounds();
      return HitDetector.hitTestRect({ x, y }, bounds);
    });
    
    if (boundsCandidates.length === 0) return null;
    
    // Test simple shapes with math
    for (const obj of boundsCandidates) {
      if (!obj.complex && this.testSimpleShape(obj, { x, y })) {
        return obj;
      }
    }
    
    // For complex shapes, use color detection
    return this.colorDetector.getObjectAt(x, y);
  }
  
  testSimpleShape(obj, point) {
    switch (obj.type) {
      case 'rect':
        return HitDetector.hitTestRect(point, obj);
      case 'circle':
        return HitDetector.hitTestCircle(point, obj);
      case 'line':
        return HitDetector.hitTestLine(point, obj, obj.hitTolerance || 5);
      default:
        return false;
    }
  }
}
```

**4. Event Handling System**

Implement canvas event handling with coordinate translation:

```javascript
class CanvasEventHandler {
  constructor(canvas, scene, hitDetector) {
    this.canvas = canvas;
    this.scene = scene;
    this.hitDetector = hitDetector;
    this.hoveredObject = null;
    this.draggedObject = null;
    this.dragOffset = { x: 0, y: 0 };
    
    this.setupEventListeners();
  }
  
  setupEventListeners() {
    this.canvas.addEventListener('mousemove', this.handleMouseMove.bind(this));
    this.canvas.addEventListener('mousedown', this.handleMouseDown.bind(this));
    this.canvas.addEventListener('mouseup', this.handleMouseUp.bind(this));
    this.canvas.addEventListener('click', this.handleClick.bind(this));
    this.canvas.addEventListener('wheel', this.handleWheel.bind(this));
    
    // Prevent context menu on right-click
    this.canvas.addEventListener('contextmenu', (e) => e.preventDefault());
  }
  
  getCanvasCoordinates(e) {
    const rect = this.canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;
    
    return {
      x: (e.clientX - rect.left) * (this.canvas.width / rect.width) / dpr,
      y: (e.clientY - rect.top) * (this.canvas.height / rect.height) / dpr
    };
  }
  
  handleMouseMove(e) {
    const pos = this.getCanvasCoordinates(e);
    
    // Handle dragging
    if (this.draggedObject) {
      this.draggedObject.x = pos.x - this.dragOffset.x;
      this.draggedObject.y = pos.y - this.dragOffset.y;
      this.draggedObject.markDirty();
      return;
    }
    
    // Handle hover
    const hitObject = this.hitDetector.hitTest(pos.x, pos.y);
    
    if (hitObject !== this.hoveredObject) {
      // Mouse out
      if (this.hoveredObject && this.hoveredObject.onMouseOut) {
        this.hoveredObject.onMouseOut();
      }
      
      // Mouse over
      if (hitObject && hitObject.onMouseOver) {
        hitObject.onMouseOver();
      }
      
      this.hoveredObject = hitObject;
      this.canvas.style.cursor = hitObject ? 'pointer' : 'default';
    }
    
    // Mouse move event
    if (hitObject && hitObject.onMouseMove) {
      hitObject.onMouseMove(pos);
    }
  }
  
  handleMouseDown(e) {
    const pos = this.getCanvasCoordinates(e);
    const hitObject = this.hitDetector.hitTest(pos.x, pos.y);
    
    if (hitObject) {
      if (hitObject.draggable) {
        this.draggedObject = hitObject;
        this.dragOffset = {
          x: pos.x - hitObject.x,
          y: pos.y - hitObject.y
        };
      }
      
      if (hitObject.onMouseDown) {
        hitObject.onMouseDown(pos);
      }
    }
  }
  
  handleMouseUp(e) {
    const pos = this.getCanvasCoordinates(e);
    
    if (this.draggedObject) {
      if (this.draggedObject.onMouseUp) {
        this.draggedObject.onMouseUp(pos);
      }
      this.draggedObject = null;
    }
  }
  
  handleClick(e) {
    const pos = this.getCanvasCoordinates(e);
    const hitObject = this.hitDetector.hitTest(pos.x, pos.y);
    
    if (hitObject && hitObject.onClick) {
      hitObject.onClick(pos);
    }
  }
  
  handleWheel(e) {
    e.preventDefault();
    const pos = this.getCanvasCoordinates(e);
    const delta = e.deltaY;
    
    // Zoom at mouse position
    if (this.scene.onZoom) {
      this.scene.onZoom(pos, delta);
    }
  }
}
```

**Benefits:**
- Mathematical hit detection is fastest for simple shapes
- Color-based detection handles complex shapes accurately
- Hybrid approach provides optimal performance across all scenarios
- Proper coordinate translation ensures accurate event handling on scaled canvases

---

## Layer 4: Application/UI Layer

### Purpose
Provide high-level APIs for building specific interactive applications (timelines, graphs, CAD tools).

### Implementation Guidelines

**1. Interactive Timeline Component**

```javascript
class Timeline {
  constructor(canvas, data) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.data = data;
    
    this.viewport = {
      startTime: data.minTime,
      endTime: data.maxTime,
      zoom: 1
    };
    
    this.eventHandler = new CanvasEventHandler(canvas, this, new HybridHitDetector(canvas.width, canvas.height));
    this.items = this.createTimelineItems();
  }
  
  createTimelineItems() {
    return this.data.events.map(event => ({
      time: event.timestamp,
      label: event.label,
      type: 'event',
      x: this.timeToX(event.timestamp),
      y: 50,
      width: 10,
      height: 20,
      color: event.color || '#3b82f6',
      onClick: () => this.onEventClick(event)
    }));
  }
  
  timeToX(timestamp) {
    const range = this.viewport.endTime - this.viewport.startTime;
    const ratio = (timestamp - this.viewport.startTime) / range;
    return ratio * (this.canvas.width - 100) + 50;
  }
  
  xToTime(x) {
    const range = this.viewport.endTime - this.viewport.startTime;
    const ratio = (x - 50) / (this.canvas.width - 100);
    return this.viewport.startTime + ratio * range;
  }
  
  render() {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    
    // Draw timeline axis
    this.drawAxis();
    
    // Draw events
    this.items.forEach(item => this.drawEvent(item));
  }
  
  drawAxis() {
    const y = 60;
    this.ctx.beginPath();
    this.ctx.moveTo(50, y);
    this.ctx.lineTo(this.canvas.width - 50, y);
    this.ctx.strokeStyle = '#cbd5e1';
    this.ctx.lineWidth = 2;
    this.ctx.stroke();
    
    // Draw time labels
    this.drawTimeLabels();
  }
  
  drawTimeLabels() {
    const numLabels = 10;
    const step = (this.viewport.endTime - this.viewport.startTime) / numLabels;
    
    this.ctx.fillStyle = '#64748b';
    this.ctx.font = '12px sans-serif';
    this.ctx.textAlign = 'center';
    
    for (let i = 0; i <= numLabels; i++) {
      const time = this.viewport.startTime + step * i;
      const x = this.timeToX(time);
      const label = new Date(time).toLocaleDateString();
      this.ctx.fillText(label, x, 85);
    }
  }
  
  drawEvent(item) {
    this.ctx.fillStyle = item.color;
    this.ctx.fillRect(item.x - item.width / 2, item.y, item.width, item.height);
    
    // Draw label
    this.ctx.fillStyle = '#1e293b';
    this.ctx.font = '11px sans-serif';
    this.ctx.textAlign = 'center';
    this.ctx.fillText(item.label, item.x, item.y - 5);
  }
  
  onZoom(pos, delta) {
    const zoomFactor = delta > 0 ? 0.9 : 1.1;
    const mouseTime = this.xToTime(pos.x);
    
    const range = this.viewport.endTime - this.viewport.startTime;
    const newRange = range * zoomFactor;
    
    const ratio = (mouseTime - this.viewport.startTime) / range;
    this.viewport.startTime = mouseTime - newRange * ratio;
    this.viewport.endTime = mouseTime + newRange * (1 - ratio);
    
    // Update item positions
    this.items.forEach(item => {
      item.x = this.timeToX(item.time);
    });
    
    this.render();
  }
  
  onEventClick(event) {
    console.log('Event clicked:', event);
    // Dispatch custom event or callback
  }
}
```

**2. Interactive Graph Visualization**

```javascript
class GraphCanvas {
  constructor(canvas, graphData) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.nodes = graphData.nodes.map(n => new GraphNode(n));
    this.edges = graphData.edges.map(e => new GraphEdge(e, this.nodes));
    
    this.camera = { x: 0, y: 0, zoom: 1 };
    this.selectedNode = null;
    
    this.hitDetector = new HybridHitDetector(canvas.width, canvas.height);
    this.nodes.forEach(node => this.hitDetector.registerObject(node));
    
    this.setupInteractions();
    this.renderLoop = new RenderLoop(canvas, this.ctx, this);
    this.renderLoop.start();
  }
  
  setupInteractions() {
    let isDragging = false;
    let dragStart = { x: 0, y: 0 };
    
    this.canvas.addEventListener('mousedown', (e) => {
      const pos = this.screenToWorld(e.clientX, e.clientY);
      const node = this.hitDetector.hitTest(pos.x, pos.y);
      
      if (node) {
        this.selectedNode = node;
        node.isDragging = true;
      } else {
        isDragging = true;
        dragStart = { x: e.clientX, y: e.clientY };
      }
    });
    
    this.canvas.addEventListener('mousemove', (e) => {
      if (this.selectedNode && this.selectedNode.isDragging) {
        const pos = this.screenToWorld(e.clientX, e.clientY);
        this.selectedNode.x = pos.x;
        this.selectedNode.y = pos.y;
      } else if (isDragging) {
        const dx = e.clientX - dragStart.x;
        const dy = e.clientY - dragStart.y;
        this.camera.x += dx / this.camera.zoom;
        this.camera.y += dy / this.camera.zoom;
        dragStart = { x: e.clientX, y: e.clientY };
      }
    });
    
    this.canvas.addEventListener('mouseup', () => {
      if (this.selectedNode) {
        this.selectedNode.isDragging = false;
      }
      isDragging = false;
    });
    
    this.canvas.addEventListener('wheel', (e) => {
      e.preventDefault();
      const zoom = e.deltaY > 0 ? 0.9 : 1.1;
      this.camera.zoom *= zoom;
      this.camera.zoom = Math.max(0.1, Math.min(5, this.camera.zoom));
    });
  }
  
  screenToWorld(screenX, screenY) {
    const rect = this.canvas.getBoundingClientRect();
    const x = (screenX - rect.left - this.canvas.width / 2) / this.camera.zoom - this.camera.x;
    const y = (screenY - rect.top - this.canvas.height / 2) / this.camera.zoom - this.camera.y;
    return { x, y };
  }
  
  update(deltaTime) {
    // Apply force-directed layout or other physics
    this.applyForces();
  }
  
  render(ctx) {
    ctx.save();
    
    // Apply camera transform
    ctx.translate(this.canvas.width / 2, this.canvas.height / 2);
    ctx.scale(this.camera.zoom, this.camera.zoom);
    ctx.translate(this.camera.x, this.camera.y);
    
    // Render edges first
    this.edges.forEach(edge => edge.render(ctx));
    
    // Render nodes
    this.nodes.forEach(node => node.render(ctx));
    
    ctx.restore();
  }
  
  applyForces() {
    // Simple repulsion force between nodes
    for (let i = 0; i < this.nodes.length; i++) {
      for (let j = i + 1; j < this.nodes.length; j++) {
        const n1 = this.nodes[i];
        const n2 = this.nodes[j];
        
        const dx = n2.x - n1.x;
        const dy = n2.y - n1.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        
        if (dist > 0 && dist < 200) {
          const force = (200 - dist) / dist * 0.1;
          n1.vx -= dx * force;
          n1.vy -= dy * force;
          n2.vx += dx * force;
          n2.vy += dy * force;
        }
      }
    }
    
    // Apply velocity and damping
    this.nodes.forEach(node => {
      if (!node.isDragging) {
        node.x += node.vx;
        node.y += node.vy;
        node.vx *= 0.85;  // Damping
        node.vy *= 0.85;
      }
    });
  }
}

class GraphNode {
  constructor(data) {
    this.id = data.id;
    this.label = data.label;
    this.x = data.x || Math.random() * 800 - 400;
    this.y = data.y || Math.random() * 600 - 300;
    this.vx = 0;
    this.vy = 0;
    this.radius = 20;
    this.color = data.color || '#3b82f6';
    this.isDragging = false;
  }
  
  render(ctx) {
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
    ctx.fillStyle = this.color;
    ctx.fill();
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 2;
    ctx.stroke();
    
    ctx.fillStyle = '#1e293b';
    ctx.font = '12px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(this.label, this.x, this.y + this.radius + 15);
  }
  
  getBounds() {
    return {
      x: this.x - this.radius,
      y: this.y - this.radius,
      width: this.radius * 2,
      height: this.radius * 2
    };
  }
}

class GraphEdge {
  constructor(data, nodes) {
    this.source = nodes.find(n => n.id === data.source);
    this.target = nodes.find(n => n.id === data.target);
  }
  
  render(ctx) {
    ctx.beginPath();
    ctx.moveTo(this.source.x, this.source.y);
    ctx.lineTo(this.target.x, this.target.y);
    ctx.strokeStyle = '#cbd5e1';
    ctx.lineWidth = 1;
    ctx.stroke();
  }
}
```

**Benefits:**
- High-level components abstract complexity for specific use cases
- Reusable patterns for common interactive visualizations
- Consistent interaction models across different visualization types

---

## Framework-Specific Adaptations

### Vanilla JavaScript

Pure JavaScript approach as shown in examples above. Best for:
- Maximum control and performance
- Small to medium applications
- Learning and understanding fundamentals

### React Integration

```javascript
import { useRef, useEffect } from 'react';

function CanvasComponent({ width, height, data }) {
  const canvasRef = useRef(null);
  const rendererRef = useRef(null);
  
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    
    // Initialize renderer once
    if (!rendererRef.current) {
      rendererRef.current = new Timeline(canvas, data);
    }
    
    // Update data
    rendererRef.current.updateData(data);
    rendererRef.current.render();
    
    return () => {
      // Cleanup
      if (rendererRef.current) {
        rendererRef.current.destroy();
      }
    };
  }, [data]);
  
  return <canvas ref={canvasRef} width={width} height={height} />;
}
```

### Vue Integration

```javascript
<template>
  <canvas ref="canvas" :width="width" :height="height"></canvas>
</template>

<script>
export default {
  props: ['width', 'height', 'data'],
  data() {
    return {
      renderer: null
    };
  },
  mounted() {
    this.renderer = new Timeline(this.$refs.canvas, this.data);
    this.renderer.render();
  },
  watch: {
    data: {
      deep: true,
      handler(newData) {
        if (this.renderer) {
          this.renderer.updateData(newData);
          this.renderer.render();
        }
      }
    }
  },
  beforeUnmount() {
    if (this.renderer) {
      this.renderer.destroy();
    }
  }
};
</script>
```

---

## Component Guidelines

### When to Use Canvas vs SVG vs WebGL

**Use Canvas when:**
- Rendering thousands of simple shapes (>1000 data points)
- Real-time animations with frequent full redraws
- Pixel-perfect control needed
- Creating custom charts, graphs, or visualizations
- Interactive drawing or painting applications

**Use SVG when:**
- Small to medium number of elements (<1000)
- Elements need individual event listeners
- DOM manipulation and CSS styling important
- Accessibility is critical
- Crisp scaling at all zoom levels required

**Use WebGL when:**
- Extremely large datasets (100,000+ points)
- 3D visualizations
- Advanced visual effects (shaders, particles)
- Maximum performance is critical
- Willing to accept higher complexity

### Performance Thresholds

| Data Points | Recommended Technology |
|-------------|----------------------|
| < 100 | SVG (simplest) |
| 100 - 1,000 | SVG or Canvas |
| 1,000 - 10,000 | Canvas |
| 10,000 - 100,000 | Canvas with optimizations |
| > 100,000 | WebGL or Canvas with virtualization |

### Optimization Checklist

**Always Do:**
- ✓ Support high-DPI displays with proper scaling
- ✓ Use `requestAnimationFrame` for animations
- ✓ Implement dirty region tracking
- ✓ Batch similar drawing operations
- ✓ Cache static content in offscreen canvases
- ✓ Use `willReadFrequently` hint when reading pixels often

**Never Do:**
- ✗ Call `clearRect` and redraw entire canvas every frame (unless necessary)
- ✗ Create new objects in animation loops
- ✗ Use `setInterval` or `setTimeout` for animations
- ✗ Ignore device pixel ratio
- ✗ Draw invisible content outside viewport

**Consider:**
- Layering canvases by update frequency
- Object pooling for frequently created/destroyed objects
- Web Workers for heavy computations
- OffscreenCanvas for background rendering (where supported)

---

## File Organization

### Recommended Project Structure

```
src/
├── core/
│   ├── CanvasManager.js          # Canvas setup and management
│   ├── RenderLoop.js              # Animation loop
│   └── DirtyRegionManager.js      # Dirty tracking
├── rendering/
│   ├── BatchRenderer.js           # Batch rendering
│   ├── SceneNode.js               # Scene graph base
│   └── LayerManager.js            # Multi-layer management
├── interaction/
│   ├── HitDetector.js             # Mathematical hit detection
│   ├── ColorHitDetector.js        # Color-based detection
│   ├── HybridHitDetector.js       # Combined approach
│   └── EventHandler.js            # Event management
├── components/
│   ├── Timeline.js                # Timeline component
│   ├── Graph.js                   # Graph visualization
│   └── CADCanvas.js               # CAD-style editor
├── utils/
│   ├── geometry.js                # Geometric calculations
│   ├── colors.js                  # Color utilities
│   └── transforms.js              # Coordinate transforms
└── index.js                       # Public API
```

---

## Migration Examples

### From Basic Canvas to Optimized

**Before:**
```javascript
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  
  // Draw all objects every frame
  objects.forEach(obj => {
    ctx.fillStyle = obj.color;
    ctx.fillRect(obj.x, obj.y, obj.width, obj.height);
  });
  
  requestAnimationFrame(render);
}
```

**After:**
```javascript
const dirtyManager = new DirtyRegionManager(canvas.width, canvas.height);
const batchRenderer = new BatchRenderer(ctx);

function render() {
  const dirtyRegions = dirtyManager.getDirtyRegions();
  
  if (dirtyRegions.length === 0) {
    requestAnimationFrame(render);
    return;
  }
  
  dirtyRegions.forEach(region => {
    ctx.clearRect(region.x, region.y, region.width, region.height);
    
    // Only process objects in dirty region
    const objectsInRegion = objects.filter(obj => 
      intersects(obj.getBounds(), region)
    );
    
    // Batch by style
    objectsInRegion.forEach(obj => {
      batchRenderer.addToBatch(obj, obj.style);
    });
  });
  
  batchRenderer.render();
  dirtyManager.clear();
  
  requestAnimationFrame(render);
}
```

### Adding Interaction

**Before:**
```javascript
canvas.addEventListener('click', (e) => {
  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  
  // Check every object
  objects.forEach(obj => {
    if (x >= obj.x && x <= obj.x + obj.width &&
        y >= obj.y && y <= obj.y + obj.height) {
      obj.onClick();
    }
  });
});
```

**After:**
```javascript
const hitDetector = new HybridHitDetector(canvas.width, canvas.height);
const eventHandler = new CanvasEventHandler(canvas, scene, hitDetector);

// Register objects
objects.forEach(obj => {
  hitDetector.registerObject(obj);
  obj.onClick = () => console.log('Clicked:', obj.id);
});

// Event handler automatically manages clicks, hovers, drags
```

---

## Additional Considerations

### Handling Large Datasets

**Virtualization:**
```javascript
class VirtualizedRenderer {
  constructor(canvas, ctx, allItems) {
    this.canvas = canvas;
    this.ctx = ctx;
    this.allItems = allItems;
    this.viewport = { x: 0, y: 0, width: canvas.width, height: canvas.height };
  }
  
  render() {
    // Only render items visible in viewport
    const visibleItems = this.allItems.filter(item => 
      this.isInViewport(item)
    );
    
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    visibleItems.forEach(item => this.drawItem(item));
  }
  
  isInViewport(item) {
    const bounds = item.getBounds();
    return !(bounds.x + bounds.width < this.viewport.x ||
             bounds.x > this.viewport.x + this.viewport.width ||
             bounds.y + bounds.height < this.viewport.y ||
             bounds.y > this.viewport.y + this.viewport.height);
  }
}
```

### Caching and Memoization

```javascript
class CachedRenderer {
  constructor() {
    this.cache = new Map();
  }
  
  renderWithCache(ctx, obj) {
    const key = obj.getCacheKey();
    
    if (!this.cache.has(key) || obj.dirty) {
      // Create offscreen canvas for this object
      const offscreen = document.createElement('canvas');
      offscreen.width = obj.width;
      offscreen.height = obj.height;
      const offCtx = offscreen.getContext('2d');
      
      obj.render(offCtx);
      this.cache.set(key, offscreen);
      obj.dirty = false;
    }
    
    // Draw cached version
    ctx.drawImage(this.cache.get(key), obj.x, obj.y);
  }
}
```

### Error Handling

```javascript
class SafeCanvasRenderer {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = null;
    this.errorHandler = null;
  }
  
  initialize() {
    try {
      this.ctx = this.canvas.getContext('2d');
      if (!this.ctx) {
        throw new Error('Canvas context not available');
      }
      return true;
    } catch (error) {
      this.handleError('Initialization failed', error);
      return false;
    }
  }
  
  safeRender(renderFn) {
    try {
      this.ctx.save();
      renderFn(this.ctx);
      this.ctx.restore();
    } catch (error) {
      this.handleError('Render failed', error);
      this.ctx.restore();  // Ensure state is restored
    }
  }
  
  handleError(message, error) {
    console.error(message, error);
    if (this.errorHandler) {
      this.errorHandler(message, error);
    }
  }
  
  setErrorHandler(handler) {
    this.errorHandler = handler;
  }
}
```

### Accessibility Considerations

```javascript
class AccessibleCanvas {
  constructor(canvas, description) {
    this.canvas = canvas;
    this.description = description;
    
    // Add ARIA attributes
    canvas.setAttribute('role', 'img');
    canvas.setAttribute('aria-label', description);
    
    // Create fallback content
    this.fallbackContainer = document.createElement('div');
    this.fallbackContainer.className = 'canvas-fallback';
    this.fallbackContainer.setAttribute('aria-hidden', 'true');
    canvas.parentNode.insertBefore(this.fallbackContainer, canvas.nextSibling);
  }
  
  updateFallback(data) {
    // Update screen reader accessible description
    this.fallbackContainer.innerHTML = this.generateDescription(data);
  }
  
  generateDescription(data) {
    // Generate text description of visual content
    // For a graph: list nodes and connections
    // For a timeline: list events chronologically
    return data.map(item => item.description).join(', ');
  }
}
```

---

## Conclusion

Optimized Canvas-based interactive graphics require careful attention to rendering performance, event handling, and architectural patterns. By implementing the strategies outlined in this document, you can build responsive, performant applications that handle complex visualizations and large datasets.

### Key Takeaways

1. **Layer your architecture** - Separate concerns from infrastructure through rendering to application logic
2. **Track dirty regions** - Only redraw what changes to minimize pixel processing
3. **Batch rendering operations** - Group similar drawing calls to reduce API overhead
4. **Choose the right hit detection** - Use math for simple shapes, color-based for complex ones
5. **Scale appropriately** - Use Canvas for medium to large datasets, SVG for small, WebGL for massive
6. **Optimize continuously** - Profile, measure, and iterate based on your specific use case

### Further Resources

- [MDN Canvas API Reference](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)
- [HTML5 Canvas Performance](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)
- [AG Charts Rendering Techniques](https://blog.ag-grid.com/optimising-html5-canvas-rendering-best-practices-and-techniques/)
- [Apache ECharts](https://echarts.apache.org/) - Production-ready Canvas charting library
- [Konva.js](https://konvajs.org/) - Canvas framework with built-in interaction support

The patterns and techniques in this document represent current best practices as of 2024-2025, synthesized from production implementations in charting libraries, graph visualizations, and interactive design tools.
