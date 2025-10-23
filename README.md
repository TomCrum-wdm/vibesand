# VibeSand

A high-performance GPU-powered voxel physics simulator running on WebGPU. Simulates 2 million voxels updating at 60fps with real-time physics and dynamic lighting.

## Code Structure

### Architecture Overview

VibeSand is a **single-file application** (`index.html`) with no build system or external dependencies. The entire application is self-contained and runs directly in WebGPU-enabled browsers.

**Tech Stack:**
- **Pure WebGPU API** - No wrapper libraries or frameworks (no Three.js, Babylon.js, etc.)
- **WGSL Shaders** - All GPU compute and rendering shaders written in WebGPU Shading Language
- **Vanilla JavaScript** - DOM manipulation and application logic in plain JS
- **HTML5 Canvas** - Rendering target for the WebGPU context

### Core Components

The application is organized into several logical sections within `index.html`:

#### 1. **UI Layer** (HTML/CSS)
- Material design-inspired interface with controls panel
- Real-time FPS counter and performance metrics
- Tool selection and parameter sliders
- Camera mode toggle and instructions

#### 2. **Shader Pipeline** (WGSL)

The GPU pipeline consists of multiple compute and render shaders:

**Common Shader Code:**
- Voxel data packing/unpacking (type, variant, physics flags, RGB lighting in single `u32`)
- Coordinate system utilities
- Hash functions for procedural generation

**Compute Shaders:**
- `generationShaderCode` - World generation (terrain/flat modes)
- `simulationShaderCode` - Physics simulation (gravity, sliding, tumbling, fluid dynamics)
- `resetFlagsShaderCode` - Clears per-frame physics flags
- `lightingShaderCode` - Dynamic point light calculations
- `paintShaderCode` - Tool application (sand, stone, water, destroy)
- `clearLooseVoxelsShaderCode` - Removes non-solid voxels
- `targetingShaderCode` - GPU raycasting for cursor 3D position
- `managePlayerShaderCode` - Player voxel creation/maintenance (first-person mode)
- `findPlayerShaderCode` - Player voxel position tracking (GPU scan)

**Render Shader:**
- `renderShaderCode` - Single-pass DDA raymarching fragment shader
  - Renders voxels with lighting
  - **Enhanced procedural sky system** with:
    - Atmospheric gradient (horizon to zenith)
    - Animated sun disk with corona and glow effects
    - Volumetric procedural clouds using Fractal Brownian Motion
    - Dynamic time-of-day lighting (sunrise/sunset coloring)
  - Water surface shimmer effects
  - Tool preview sphere visualization

#### 3. **Camera System**

VibeSand implements **two distinct camera modes**:

##### **Orbital Camera Mode** (Default)
- **Architecture**: Traditional 3D viewport camera orbiting around a target point
- **Implementation**: Spherical coordinates (alpha, beta, radius) converted to Cartesian
- **Controls**:
  - Right-click + drag: Orbit camera around target
  - Middle-mouse / Space + Left-click + drag: Pan camera
  - Mouse wheel: Zoom in/out
  - Left-click: Use selected tool
- **Camera State**:
  ```javascript
  camera = {
    alpha: -0.5,        // Horizontal angle
    beta: 0.3,          // Vertical angle  
    radius: 230.4,      // Distance from target
    target: [64, 30, 64] // Look-at point
  }
  ```
- **Position Calculation**:
  ```javascript
  x = target[0] + radius * cos(beta) * sin(alpha)
  y = target[1] + radius * sin(beta)
  z = target[2] + radius * cos(beta) * cos(alpha)
  ```

##### **First Person Camera Mode**
- **Architecture**: FPS-style camera with WASD movement, mouse look, and physics-based player voxel
- **Implementation**: Bi-directional binding between camera and player voxel with collision detection
- **Controls**:
  - WASD: Move forward/left/backward/right (applies force to player voxel)
  - Space: Jump (upward movement)
  - Mouse: Look around (pointer lock API)
  - Left-click: Use tool at crosshair
- **Camera State**:
  ```javascript
  firstPersonCamera = {
    position: [64, 35, 64], // 3D position (follows player voxel)
    yaw: -π/2,              // Horizontal rotation
    pitch: 0,                // Vertical rotation
    speed: 20.0              // Movement speed
  }
  playerVoxelPosition = [64, 35, 64];  // Actual player voxel position from GPU
  playerMoveIntent = [0, 0, 0];        // Movement intent from WASD input
  ```
- **Bi-Directional Binding System**:
  1. **Input → Player Voxel**: WASD keys calculate movement intent, applied to player voxel on GPU
  2. **Player Voxel → Camera**: Player position read back from GPU, camera follows with eye height offset
  3. **Physics Integration**: Player voxel affected by gravity, collision, and terrain
- **View Direction Calculation**:
  ```javascript
  forward = [
    cos(pitch) * cos(yaw),
    sin(pitch),
    cos(pitch) * sin(yaw)
  ]
  ```
- **Features**:
  - Crosshair HUD overlay
  - Pointer lock for seamless mouse look
  - **Physics-based FPS movement**:
    - Bright green player voxel represents player body
    - Camera position = player voxel position + 1.6 voxel units (eye height)
    - Player voxel affected by gravity (falls when no ground)
    - Collision detection: blocked by stone/sand, passes through air/water
    - Falls, slides diagonally down slopes, swaps with water
    - GPU-tracked with async CPU readback (1-frame latency)
    - Automatically respawns if destroyed or falls off world
    - True FPS game behavior: camera follows physics-simulated player

**Camera Shared Systems:**
- Both modes calculate `currentRight`, `currentUp`, `currentForward` vectors
- Ray direction computation for both rendering and targeting
- Unified frustum and aspect ratio handling

#### 4. **Physics Engine** (GPU-based)

All physics runs on the GPU using compute shaders:

**Single-Buffer Architecture:**
- One storage buffer holds all 2M voxels
- Each voxel packed into single `u32` (4 bytes)
- "Updated" flag prevents double-processing per frame

**Physics Priority System:**
1. Direct vertical fall (gravity)
2. Diagonal fall/tumbling 
3. Horizontal tumbling over edges
4. Water horizontal spread (2-cell pressure)

**Particle Types:**
- Stone: Static solid, never moves
- Sand: Falls, slides diagonally, tumbles, swaps with water
- Water: Falls, flows, spreads horizontally with pressure
- **Player**: Physics-simulated voxel controlled by player input in first-person mode
  - Bright green color (RGB: 0.2, 0.8, 0.2) for high visibility
  - Spawned automatically in first-person mode
  - Responds to WASD movement intent with collision detection
  - Affected by gravity (falls, slides) like sand
  - Can move through air/water, blocked by stone/sand
  - GPU position tracked with async CPU readback (1-frame latency)
  - Camera position bound to player voxel position + eye height offset
  - Protected from deletion at y=0 (respawns if lost)
- Air: Empty space

#### 5. **Rendering Pipeline**

**Multi-Pass GPU Architecture:**
1. **Targeting Pass** - GPU raycast finds cursor 3D position
1.5. **Player Management Pass** - Apply player movement intent with collision detection (first-person mode only)
2. **Physics Passes** - Multiple simulation steps per frame (player voxel affected by gravity, slides, falls)
2.5. **Player Tracking Pass** - GPU scan to locate player voxel after physics (first-person mode only)
3. **Tool Application** - Modify voxels based on user input
4. **Lighting Pass** - Calculate light contributions from 32 dynamic point lights
5. **Render Pass** - Single-pass DDA raymarching renders the scene
6. **Position Readback** - Async GPU→CPU transfer of player voxel position (first-person mode only)

**DDA Raymarching:**
- Optimized voxel traversal algorithm
- Configurable max ray steps (64-384)
- Ray start bias prevents visual artifacts
- Integrated lighting, water shimmer, and tool preview in one shader

#### 6. **Memory Layout**

**Voxel Bit Packing** (32-bit unsigned integer):
```
Bits 0-2:   Voxel type (3 bits = 8 types)
Bits 3-5:   Color variant (3 bits = 8 variants)
Bit 6:      Physics updated flag
Bit 7:      (Reserved)
Bits 8-15:  Red light channel (8 bits)
Bits 16-23: Green light channel (8 bits)
Bits 24-31: Blue light channel (8 bits)
```

**GPU Buffers:**
- `voxelBuffer`: 8,388,608 bytes (128³ * 4 bytes)
- `lightsUniformBuffer`: Point light data (32 lights * 32 bytes)
- `renderPassUniformsBuffer`: Camera and sun data
- `computePassUniformsBuffer`: Tool and simulation params
- `targetingOutputBuffer`: Cursor raycast results
- `playerPosBuffer`: Player voxel position tracking (16 bytes: x, y, z, found_flag)

### Key Features

- **Bi-Directional Camera-Player Binding**: First-person camera bound to physics-simulated player voxel
  - Player input (WASD) applies movement intent to player voxel on GPU
  - Player voxel responds to physics (gravity, collision, terrain)
  - Camera position follows player voxel with eye height offset
  - Async GPU→CPU position readback with 1-frame latency
  - True FPS game physics: falls, jumps, collides with terrain
- **GPU-First Architecture**: Minimal CPU involvement, all simulation and rendering on GPU
- **Atomic Operations**: Lock-free parallel physics using atomics
- **Player Voxel Physics**: GPU-managed player with collision detection and gravity
- **Dynamic Lighting**: 32 falling colored point lights ("light rain")
- **Interactive Tools**: Brush-based painting system with preview
- **Resolution Scaling**: Adjustable render resolution for performance
- **Continuous Simulation**: User-configurable physics step count
- **Enhanced Sky System**: Procedural skybox with sun and volumetric clouds (see below)

### Sky Rendering System

The sky system uses **procedural generation** for maximum performance and visual quality:

**Technical Approach:**
- **Skybox-based rendering** (not voxel-based) for optimal performance
- Computed per-pixel in fragment shader during ray marching
- No additional textures or GPU memory required
- Fully dynamic and synchronized with sun position

**Sky Components:**

1. **Atmospheric Gradient**
   - Smooth transition from horizon (light blue) to zenith (deep blue)
   - Uses power function for realistic atmospheric scattering appearance
   - Sunset/sunrise coloring near horizon based on sun elevation

2. **Sun Rendering**
   - **Sun disk**: Bright core using high-power falloff (pow 128)
   - **Sun glow**: Medium falloff halo (pow 8) 
   - **Sun corona**: Wide soft glow (pow 2)
   - Warm yellow-white color (1.0, 0.95, 0.8)
   - Automatically positioned using existing sun direction vector

3. **Procedural Clouds**
   - **4-octave Fractal Brownian Motion** for realistic cloud shapes
   - **3D value noise** with smooth interpolation
   - **Animated**: Clouds drift slowly over time
   - **Sun-lit**: Cloud brightness varies based on sun position
   - **Performance optimization**: Only rendered in upper hemisphere (y > 0.05)
   - **View-dependent fading**: Clouds fade when looking straight up or down
   - Uses domain warping for natural, billowy appearance

**Performance Characteristics:**
- Minimal performance impact (~1-2ms per frame on modern GPUs)
- Clouds use early exit for rays pointing down
- Noise functions use hardware-accelerated math operations
- No texture fetches or additional memory bandwidth

## Running the Project

1. Open `index.html` in a WebGPU-compatible browser (Chrome 113+, Edge 113+)
2. Ensure your GPU drivers are up to date
3. No build step or package installation required

## Browser Compatibility

- **Requires WebGPU support** (Chrome/Edge 113+, Safari 17.4+)
- Will display error message if WebGPU is unavailable
- Check compatibility at [WebGPUReport.org](https://webgpureport.org)

## Performance

- 2,097,152 voxels (128³ grid)
- Target: 60 FPS on modern GPUs
- Adjustable quality settings:
  - Render scale (25%-100%)
  - Max ray steps (64-384)
  - Simulation steps per frame (0-25)
