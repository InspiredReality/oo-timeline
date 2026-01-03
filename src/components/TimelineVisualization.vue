<!-- TimelineVisualization.vue -->
<template>
    <div class="visualization-container">
      <!-- Camera controls -->
      <div class="camera-controls">
        <button class="reset-button" @click="resetCamera">Reset</button>
        <div class="rotation-buttons">
          <button class="rotate-button" @click="rotateCamera('x', 90)" title="Rotate up">^</button>
          <button class="rotate-button" @click="rotateCamera('x', -90)" title="Rotate down">v</button>
          <button class="rotate-button" @click="rotateCamera('y', -90)" title="Rotate right">&lt;</button>
          <button class="rotate-button" @click="rotateCamera('y', 90)" title="Rotate left">&gt;</button>
        </div>
      </div>

      <!-- Three.js canvas -->
      <div ref="threeContainer" class="three-container"></div>

      <!-- D3 Timeline overlay -->
      <div class="timeline-overlay">
        <div class="timeline-controls">
          <button class="zoom-button" @click="zoomInTimeline">+</button>
          <button class="zoom-button" @click="zoomOutTimeline">-</button>
        </div>
        <div ref="timelineContainer" class="timeline-scroll-container">
          <svg ref="timelineSvg"></svg>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, onUnmounted, defineExpose } from 'vue'
  import * as THREE from 'three'
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
  import * as d3 from 'd3'
  import gsap from 'gsap'

  // Emit event to parent
  const emit = defineEmits(['data-updated'])

  // Refs for DOM elements
  const threeContainer = ref(null)
  const timelineContainer = ref(null)
  const timelineSvg = ref(null)

  // Three.js objects
  let scene, camera, renderer, controls
  let dataPoints = []
  let resizeObserver = null

  // Default camera settings
  const DEFAULT_CAMERA_POSITION = { x: 0, y: 30, z: 80 }
  const DEFAULT_CAMERA_TARGET = { x: 0, y: 0, z: 0 }

  // Timeline zoom state
  const timelineZoom = ref(1)

  // Data will be loaded from JSON file
  const sampleData = ref([])
  const qualitativeEvents = ref([])

  // Load data from JSON file
  async function loadData() {
    try {
      const response = await fetch('/data.json')
      const jsonData = await response.json()

      // Parse timestamps as Date objects
      sampleData.value = jsonData.data.map(item => ({
        ...item,
        timestamp: new Date(item.timestamp)
      }))

      // Parse qualitative events
      qualitativeEvents.value = (jsonData.events || []).map(item => ({
        ...item,
        timestamp: new Date(item.timestamp)
      }))

      // Initialize visualizations after data is loaded
      if (sampleData.value.length > 0) {
        createDataPoints()
        initTimeline()

        // Emit data to parent component (include events)
        emit('data-updated', { data: sampleData.value, events: qualitativeEvents.value })
      }
    } catch (error) {
      console.error('Error loading data:', error)
      // Fallback to sample data if file not found
      useFallbackData()
    }
  }

  // Fallback sample data
  function useFallbackData() {
    sampleData.value = [
      {
        timestamp: new Date('2024-01-15'),
        x: -50, y: 10, z: 0,
        color: '#ff6b6b',
        count: 145,
        label: 'Server Group A'
      },
      {
        timestamp: new Date('2024-02-01'),
        x: -25, y: 25, z: 10,
        color: '#4ecdc4',
        count: 178,
        label: 'Server Group B'
      },
      {
        timestamp: new Date('2024-03-15'),
        x: 0, y: 15, z: -10,
        color: '#45b7d1',
        count: 203,
        label: 'Server Group A'
      },
      {
        timestamp: new Date('2024-04-20'),
        x: 25, y: 30, z: 5,
        color: '#f7b731',
        count: 156,
        label: 'Server Group C'
      },
      {
        timestamp: new Date('2024-05-10'),
        x: 50, y: 20, z: -5,
        color: '#5f27cd',
        count: 189,
        label: 'Server Group B'
      }
    ]

    qualitativeEvents.value = [
      {
        timestamp: new Date('2024-02-15'),
        title: 'New SLA Policy',
        description: '24hr response time for critical findings'
      },
      {
        timestamp: new Date('2024-04-01'),
        title: 'Company Acquisition',
        description: 'Merged TechCorp infrastructure'
      }
    ]

    createDataPoints()
    initTimeline()
    emit('data-updated', { data: sampleData.value, events: qualitativeEvents.value })
  }
  
  // Initialize Three.js scene
  function initThreeJS() {
    // Scene setup
    scene = new THREE.Scene()
    scene.background = new THREE.Color(0x1a1a1a)

    // Get container dimensions
    const containerWidth = threeContainer.value.clientWidth
    const containerHeight = threeContainer.value.clientHeight

    // Camera setup
    camera = new THREE.PerspectiveCamera(
      75,
      containerWidth / containerHeight,
      0.1,
      1000
    )
    camera.position.set(0, 30, 80)
    camera.lookAt(0, 0, 0)

    // Renderer setup
    renderer = new THREE.WebGLRenderer({ antialias: true })
    renderer.setSize(containerWidth, containerHeight)
    renderer.setPixelRatio(window.devicePixelRatio)
    threeContainer.value.appendChild(renderer.domElement)
    
    // Controls
    controls = new OrbitControls(camera, renderer.domElement)
    controls.enableDamping = true
    controls.dampingFactor = 0.05
    
    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
    scene.add(ambientLight)
    
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
    directionalLight.position.set(10, 50, 10)
    scene.add(directionalLight)
    
    // Grid helper for reference
    const gridHelper = new THREE.GridHelper(200, 20, 0x444444, 0x222222)
    scene.add(gridHelper)

    // Animation loop
    animate()
  }

  // Create 3D objects from data
  function createDataPoints() {
    sampleData.value.forEach(data => {
      // Get shape and size from data, with defaults
      const shape = data.shape || 'sphere'
      const size = data.size || 2

      // Create geometry based on shape
      let geometry
      switch (shape.toLowerCase()) {
        case 'box':
        case 'cube':
          geometry = new THREE.BoxGeometry(size, size, size)
          break
        case 'cone':
          geometry = new THREE.ConeGeometry(size, size * 2, 32)
          break
        case 'cylinder':
          geometry = new THREE.CylinderGeometry(size, size, size * 2, 32)
          break
        case 'torus':
          geometry = new THREE.TorusGeometry(size, size * 0.4, 16, 100)
          break
        case 'octahedron':
          geometry = new THREE.OctahedronGeometry(size)
          break
        case 'tetrahedron':
          geometry = new THREE.TetrahedronGeometry(size)
          break
        case 'dodecahedron':
          geometry = new THREE.DodecahedronGeometry(size)
          break
        case 'icosahedron':
          geometry = new THREE.IcosahedronGeometry(size)
          break
        case 'sphere':
        default:
          geometry = new THREE.SphereGeometry(size, 32, 32)
          break
      }

      const material = new THREE.MeshStandardMaterial({
        color: data.color,
        emissive: data.color,
        emissiveIntensity: 0.3
      })
      const mesh = new THREE.Mesh(geometry, material)

      // Position based on data
      mesh.position.set(data.x, data.y, data.z)

      // Store reference to data
      mesh.userData = data

      scene.add(mesh)
      dataPoints.push(mesh)

      // Add label
      createLabel(data)
    })
  }
  
  // Create text labels (simplified - you'd use sprite or CSS2DRenderer)
  function createLabel(data) {
    const canvas = document.createElement('canvas')
    const context = canvas.getContext('2d')
    canvas.width = 256
    canvas.height = 64
    
    context.fillStyle = '#ffffff'
    context.font = '24px Arial'
    context.fillText(data.label, 10, 40)
    
    const texture = new THREE.CanvasTexture(canvas)
    const spriteMaterial = new THREE.SpriteMaterial({ map: texture })
    const sprite = new THREE.Sprite(spriteMaterial)
    
    sprite.position.set(data.x, data.y + 5, data.z)
    sprite.scale.set(10, 2.5, 1)
    
    scene.add(sprite)
  }
  
  // Animation loop
  function animate() {
    requestAnimationFrame(animate)
    controls.update()
    renderer.render(scene, camera)
  }
  
  // Initialize D3 Timeline
  function initTimeline() {
    const margin = { top: 20, right: 50, bottom: 30, left: 100 }
    const containerWidth = timelineContainer.value.clientWidth - margin.left - margin.right
    const width = containerWidth * timelineZoom.value
    const height = 150 - margin.top - margin.bottom

    const svg = d3.select(timelineSvg.value)
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)

    const g = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`)

    // Time scale
    const xScale = d3.scaleTime()
      .domain(d3.extent(sampleData.value, d => d.timestamp))
      .range([0, width])

    // Y scale for quantitative data - include both positive and negative values
    const minCount = d3.min(sampleData.value, d => d.count)
    const maxCount = d3.max(sampleData.value, d => d.count)
    const yScale = d3.scaleLinear()
      .domain([Math.min(0, minCount), Math.max(0, maxCount)])
      .range([height, 0])

    // Create axis group at zero line
    const zeroY = yScale(0)
    const axisGroup = g.append('g')
      .attr('class', 'x-axis')
      .attr('transform', `translate(0,${zeroY})`)

    // Add axis line
    axisGroup.append('line')
      .attr('x1', 0)
      .attr('x2', width)
      .attr('y1', 0)
      .attr('y2', 0)
      .attr('stroke', '#666')

    // Generate month boundaries
    const [minDate, maxDate] = d3.extent(sampleData.value, d => d.timestamp)
    const monthBoundaries = d3.timeMonth.range(
      d3.timeMonth.floor(minDate),
      d3.timeMonth.offset(d3.timeMonth.ceil(maxDate), 1)
    )

    // Add tick marks at month boundaries (start of each month)
    axisGroup.selectAll('.boundary-tick')
      .data(monthBoundaries)
      .enter()
      .append('line')
      .attr('class', 'boundary-tick')
      .attr('x1', d => xScale(d))
      .attr('x2', d => xScale(d))
      .attr('y1', -5)
      .attr('y2', 5)
      .attr('stroke', '#666')

    // Generate months for labels (excluding the last boundary)
    const monthTicks = monthBoundaries.slice(0, -1)

    // Add labels at the start of each month (left-aligned with small offset)
    axisGroup.selectAll('.tick-label')
      .data(monthTicks)
      .enter()
      .append('text')
      .attr('class', 'tick-label')
      .attr('x', d => xScale(d) + 5)
      .attr('y', 9)
      .attr('dy', '0.71em')
      .attr('text-anchor', 'start')
      .style('fill', '#ffffff')
      .text(d => d3.timeFormat('%b %Y')(d))
    
    // Quantitative bars - handle both positive and negative values
    g.selectAll('.bar')
      .data(sampleData.value)
      .enter()
      .append('rect')
      .attr('class', 'bar')
      .attr('x', d => xScale(d.timestamp) - 15)
      .attr('y', d => d.count >= 0 ? yScale(d.count) : yScale(0))
      .attr('width', 30)
      .attr('height', d => Math.abs(yScale(d.count) - yScale(0)))
      .attr('fill', d => d.color)
      .attr('opacity', 0.7)
      .style('cursor', 'pointer')
      .on('click', (event, d) => {
        animateCameraToPoint(d)
      })
      .on('mouseover', function(event, d) {
        d3.select(this).attr('opacity', 1)
        showTooltip(event, d)
      })
      .on('mouseout', function() {
        d3.select(this).attr('opacity', 0.7)
        hideTooltip()
      })
    
    // Qualitative event markers - position on the timeline bar
    g.selectAll('.event-marker')
      .data(qualitativeEvents.value)
      .enter()
      .append('circle')
      .attr('class', 'event-marker')
      .attr('cx', d => xScale(d.timestamp))
      .attr('cy', yScale(0))
      .attr('r', 8)
      .attr('fill', '#ff6b6b')
      .attr('stroke', '#ffffff')
      .attr('stroke-width', 2)
      .style('cursor', 'pointer')
      .on('mouseover', function(event, d) {
        d3.select(this).attr('r', 10)
        showEventTooltip(event, d)
      })
      .on('mouseout', function() {
        d3.select(this).attr('r', 8)
        hideTooltip()
      })

    // Event labels - position above the timeline bar
    g.selectAll('.event-label')
      .data(qualitativeEvents.value)
      .enter()
      .append('text')
      .attr('class', 'event-label')
      .attr('x', d => xScale(d.timestamp))
      .attr('y', yScale(0) - 15)
      .attr('text-anchor', 'middle')
      .attr('fill', '#ffffff')
      .attr('font-size', '12px')
      .text(d => d.title)
  }
  
  // Animate Three.js camera to selected data point
  function animateCameraToPoint(data) {
    // Calculate current viewing direction (from camera to current target)
    const currentTarget = controls.target.clone()
    const currentDirection = new THREE.Vector3()
      .subVectors(camera.position, currentTarget)

    // New target is the data point
    const newTarget = new THREE.Vector3(data.x, data.y, data.z)

    // Calculate new camera position maintaining the same viewing angle
    const newCameraPosition = new THREE.Vector3()
      .addVectors(newTarget, currentDirection)

    // Disable controls during animation
    controls.enabled = false

    // Animate both camera position and controls target
    gsap.to(camera.position, {
      x: newCameraPosition.x,
      y: newCameraPosition.y,
      z: newCameraPosition.z,
      duration: 1.5,
      ease: 'power2.inOut'
    })

    gsap.to(controls.target, {
      x: newTarget.x,
      y: newTarget.y,
      z: newTarget.z,
      duration: 1.5,
      ease: 'power2.inOut',
      onComplete: () => {
        controls.enabled = true
      }
    })

    // Highlight the selected sphere
    highlightDataPoint(data)
  }

  // Zoom in timeline
  function zoomInTimeline() {
    timelineZoom.value = Math.min(timelineZoom.value + 0.5, 5)
    d3.select(timelineSvg.value).selectAll('*').remove()
    initTimeline()
  }

  // Zoom out timeline
  function zoomOutTimeline() {
    timelineZoom.value = Math.max(timelineZoom.value - 0.5, 0.2)
    d3.select(timelineSvg.value).selectAll('*').remove()
    initTimeline()
  }

  // Rotate camera around target
  function rotateCamera(axis, degrees) {
    const currentTarget = controls.target.clone()
    const currentPosition = camera.position.clone()

    // Calculate relative position from target
    const relativePosition = new THREE.Vector3().subVectors(currentPosition, currentTarget)

    // Convert to spherical coordinates for reliable rotation
    const spherical = new THREE.Spherical()
    spherical.setFromVector3(relativePosition)

    // Create rotation based on axis
    const radians = THREE.MathUtils.degToRad(degrees)

    if (axis === 'x') {
      // For X-axis rotation, modify the phi angle (vertical rotation)
      spherical.phi -= radians
      // Clamp phi to prevent flipping at poles
      spherical.phi = Math.max(0.01, Math.min(Math.PI - 0.01, spherical.phi))
    } else if (axis === 'y') {
      // For Y-axis rotation, modify the theta angle (horizontal rotation)
      spherical.theta += radians
    }

    // Convert back to Cartesian coordinates
    const newRelativePosition = new THREE.Vector3().setFromSpherical(spherical)
    const newPosition = new THREE.Vector3().addVectors(currentTarget, newRelativePosition)

    // Disable controls during animation
    controls.enabled = false

    // Animate to new position
    gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 1.0,
      ease: 'power2.inOut',
      onComplete: () => {
        controls.enabled = true
      }
    })
  }

  // Reset camera to default position and angle
  function resetCamera() {
    // Disable controls during animation
    controls.enabled = false

    // Animate camera back to default position
    gsap.to(camera.position, {
      x: DEFAULT_CAMERA_POSITION.x,
      y: DEFAULT_CAMERA_POSITION.y,
      z: DEFAULT_CAMERA_POSITION.z,
      duration: 1.5,
      ease: 'power2.inOut'
    })

    // Animate controls target back to origin
    gsap.to(controls.target, {
      x: DEFAULT_CAMERA_TARGET.x,
      y: DEFAULT_CAMERA_TARGET.y,
      z: DEFAULT_CAMERA_TARGET.z,
      duration: 1.5,
      ease: 'power2.inOut',
      onComplete: () => {
        controls.enabled = true
      }
    })

    // Reset all sphere highlights
    dataPoints.forEach(point => {
      gsap.to(point.material, {
        emissiveIntensity: 0.3,
        duration: 0.5
      })
      gsap.to(point.scale, {
        x: 1,
        y: 1,
        z: 1,
        duration: 0.5,
        ease: 'elastic.out(1, 0.3)'
      })
    })
  }

  // Highlight selected data point
  function highlightDataPoint(data) {
    // Reset all spheres
    dataPoints.forEach(point => {
      point.material.emissiveIntensity = 0.3
      point.scale.set(1, 1, 1)
    })
    
    // Highlight selected
    const selectedPoint = dataPoints.find(p => p.userData === data)
    if (selectedPoint) {
      gsap.to(selectedPoint.material, {
        emissiveIntensity: 0.8,
        duration: 0.5
      })
      gsap.to(selectedPoint.scale, {
        x: 1.5,
        y: 1.5,
        z: 1.5,
        duration: 0.5,
        ease: 'elastic.out(1, 0.3)'
      })
    }
  }
  
  // Tooltip functions
  function showTooltip(event, data) {
    d3.select('body')
      .append('div')
      .attr('class', 'tooltip')
      .style('position', 'fixed')
      .style('background', 'rgba(0, 0, 0, 0.9)')
      .style('color', '#fff')
      .style('padding', '10px')
      .style('border-radius', '5px')
      .style('pointer-events', 'none')
      .style('z-index', '9999')
      .style('max-width', '250px')
      .style('left', `${event.clientX + 10}px`)
      .style('top', `${event.clientY - 10}px`)
      .html(`
        <strong>${data.label}</strong><br/>
        Count: ${data.count}<br/>
        Date: ${data.timestamp.toLocaleDateString()}
      `)
  }

  function hideTooltip() {
    d3.selectAll('.tooltip').remove()
  }

  // Show tooltip for qualitative events
  function showEventTooltip(event, data) {
    d3.select('body')
      .append('div')
      .attr('class', 'tooltip')
      .style('position', 'fixed')
      .style('background', 'rgba(0, 0, 0, 0.9)')
      .style('color', '#fff')
      .style('padding', '10px')
      .style('border-radius', '5px')
      .style('pointer-events', 'none')
      .style('z-index', '9999')
      .style('max-width', '250px')
      .style('left', `${event.clientX + 10}px`)
      .style('top', `${event.clientY - 10}px`)
      .html(`
        <strong>${data.title}</strong><br/>
        ${data.description}<br/>
        Date: ${data.timestamp.toLocaleDateString()}
      `)
  }
  
  // Handle window resize
  function onWindowResize() {
    const containerWidth = threeContainer.value.clientWidth
    const containerHeight = threeContainer.value.clientHeight

    camera.aspect = containerWidth / containerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(containerWidth, containerHeight)

    // Redraw timeline
    d3.select(timelineSvg.value).selectAll('*').remove()
    initTimeline()
  }
  
  // Lifecycle hooks
  onMounted(() => {
    initThreeJS()

    // Load data from JSON file
    loadData()

    // Use ResizeObserver to watch for container size changes
    resizeObserver = new ResizeObserver(() => {
      onWindowResize()
    })

    if (threeContainer.value) {
      resizeObserver.observe(threeContainer.value)
    }
  })

  onUnmounted(() => {
    if (resizeObserver) {
      resizeObserver.disconnect()
    }
    renderer.dispose()
    controls.dispose()
  })

  // Expose methods to parent component
  defineExpose({
    animateCameraToPoint
  })
  </script>
  
  <style scoped>
  .visualization-container {
    position: relative;
    width: 100%;
    max-width: 100%;
    height: 65vh;
    min-height: 500px;
    overflow: hidden;
    flex-shrink: 0;
    box-sizing: border-box;
  }

  .camera-controls {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 100;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .reset-button {
    padding: 10px 20px;
    background: rgba(78, 205, 196, 0.9);
    color: #1a1a1a;
    border: none;
    border-radius: 5px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  }

  .reset-button:hover {
    background: rgba(78, 205, 196, 1);
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(78, 205, 196, 0.4);
  }

  .reset-button:active {
    transform: translateY(0);
  }

  .rotation-buttons {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 5px;
  }

  .rotate-button {
    width: 36px;
    height: 36px;
    background: rgba(78, 205, 196, 0.9);
    color: #1a1a1a;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    font-weight: 700;
    cursor: pointer;
    transition: all 0.2s ease;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    display: flex;
    align-items: center;
    justify-content: center;
    line-height: 1;
  }

  .rotate-button:hover {
    background: rgba(78, 205, 196, 1);
    transform: scale(1.1);
    box-shadow: 0 3px 6px rgba(78, 205, 196, 0.4);
  }

  .rotate-button:active {
    transform: scale(0.95);
  }

  .three-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }
  
  .timeline-overlay {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    background: rgba(26, 26, 26, 0.9);
    backdrop-filter: blur(10px);
    border-top: 2px solid #444;
    padding: 10px 0;
    z-index: 10;
    display: flex;
    align-items: center;
  }

  .timeline-controls {
    display: flex;
    flex-direction: column;
    gap: 5px;
    padding: 0 10px;
    flex-shrink: 0;
  }

  .timeline-scroll-container {
    flex: 1;
    overflow-x: auto;
    overflow-y: hidden;
  }

  .zoom-button {
    width: 32px;
    height: 32px;
    background: rgba(78, 205, 196, 0.9);
    color: #1a1a1a;
    border: none;
    border-radius: 4px;
    font-size: 18px;
    font-weight: 700;
    cursor: pointer;
    transition: all 0.2s ease;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    display: flex;
    align-items: center;
    justify-content: center;
    line-height: 1;
  }

  .zoom-button:hover {
    background: rgba(78, 205, 196, 1);
    transform: scale(1.1);
    box-shadow: 0 3px 6px rgba(78, 205, 196, 0.4);
  }

  .zoom-button:active {
    transform: scale(0.95);
  }

  .timeline-scroll-container::-webkit-scrollbar {
    height: 8px;
  }

  .timeline-scroll-container::-webkit-scrollbar-track {
    background: rgba(0, 0, 0, 0.3);
  }

  .timeline-scroll-container::-webkit-scrollbar-thumb {
    background: #4ecdc4;
    border-radius: 4px;
  }

  .timeline-scroll-container::-webkit-scrollbar-thumb:hover {
    background: #45b7d1;
  }
  
  :deep(.x-axis path),
  :deep(.x-axis line) {
    stroke: #666;
  }
  
  :deep(.bar) {
    transition: opacity 0.2s;
  }
  </style>