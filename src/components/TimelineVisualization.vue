<!-- TimelineVisualization.vue -->
<template>
    <div class="visualization-container">
      <!-- Three.js canvas -->
      <div ref="threeContainer" class="three-container"></div>
      
      <!-- D3 Timeline overlay -->
      <div ref="timelineContainer" class="timeline-overlay">
        <svg ref="timelineSvg"></svg>
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
  
  // Sample data - your FastAPI would provide this
  const sampleData = [
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
  
  const qualitativeEvents = [
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
    
    // Create data points
    createDataPoints()
    
    // Animation loop
    animate()
  }
  
  // Create 3D objects from data
  function createDataPoints() {
    sampleData.forEach(data => {
      // Sphere geometry
      const geometry = new THREE.SphereGeometry(2, 32, 32)
      const material = new THREE.MeshStandardMaterial({
        color: data.color,
        emissive: data.color,
        emissiveIntensity: 0.3
      })
      const sphere = new THREE.Mesh(geometry, material)
      
      // Position based on data
      sphere.position.set(data.x, data.y, data.z)
      
      // Store reference to data
      sphere.userData = data
      
      scene.add(sphere)
      dataPoints.push(sphere)
      
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
    const margin = { top: 20, right: 50, bottom: 30, left: 50 }
    // Make timeline wider for scrolling - use a minimum width that's larger than viewport
    const minTimelineWidth = 1500
    const viewportWidth = window.innerWidth - margin.left - margin.right
    const width = Math.max(minTimelineWidth, viewportWidth)
    const height = 150 - margin.top - margin.bottom

    const svg = d3.select(timelineSvg.value)
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)

    const g = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`)

    // Time scale
    const xScale = d3.scaleTime()
      .domain(d3.extent(sampleData, d => d.timestamp))
      .range([0, width])

    // Y scale for quantitative data
    const yScale = d3.scaleLinear()
      .domain([0, d3.max(sampleData, d => d.count)])
      .range([height, 0])

    // Axis - show more ticks since we have more space
    const xAxis = d3.axisBottom(xScale)
      .ticks(10)
      .tickFormat(d3.timeFormat('%b %Y'))
    
    g.append('g')
      .attr('class', 'x-axis')
      .attr('transform', `translate(0,${height})`)
      .call(xAxis)
      .selectAll('text')
      .style('fill', '#ffffff')
    
    // Quantitative bars
    g.selectAll('.bar')
      .data(sampleData)
      .enter()
      .append('rect')
      .attr('class', 'bar')
      .attr('x', d => xScale(d.timestamp) - 15)
      .attr('y', d => yScale(d.count))
      .attr('width', 30)
      .attr('height', d => height - yScale(d.count))
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
    
    // Qualitative event markers
    g.selectAll('.event-marker')
      .data(qualitativeEvents)
      .enter()
      .append('circle')
      .attr('class', 'event-marker')
      .attr('cx', d => xScale(d.timestamp))
      .attr('cy', -10)
      .attr('r', 8)
      .attr('fill', '#ff6b6b')
      .attr('stroke', '#ffffff')
      .attr('stroke-width', 2)
      .style('cursor', 'pointer')
      .on('click', (event, d) => {
        showEventModal(d)
      })
    
    // Event labels
    g.selectAll('.event-label')
      .data(qualitativeEvents)
      .enter()
      .append('text')
      .attr('class', 'event-label')
      .attr('x', d => xScale(d.timestamp))
      .attr('y', -20)
      .attr('text-anchor', 'middle')
      .attr('fill', '#ffffff')
      .attr('font-size', '12px')
      .text(d => d.title)
  }
  
  // Animate Three.js camera to selected data point
  function animateCameraToPoint(data) {
    // Target position (slightly offset from point for better view)
    const targetPosition = {
      x: data.x + 20,
      y: data.y + 15,
      z: data.z + 30
    }
    
    // Target to look at (the actual data point)
    const lookAtTarget = new THREE.Vector3(data.x, data.y, data.z)
    
    // Disable controls during animation
    controls.enabled = false
    
    // Animate camera position
    gsap.to(camera.position, {
      x: targetPosition.x,
      y: targetPosition.y,
      z: targetPosition.z,
      duration: 1.5,
      ease: 'power2.inOut',
      onUpdate: () => {
        camera.lookAt(lookAtTarget)
      },
      onComplete: () => {
        controls.target.copy(lookAtTarget)
        controls.enabled = true
      }
    })
    
    // Highlight the selected sphere
    highlightDataPoint(data)
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
    const tooltip = d3.select('body')
      .append('div')
      .attr('class', 'tooltip')
      .style('position', 'absolute')
      .style('background', 'rgba(0, 0, 0, 0.8)')
      .style('color', '#fff')
      .style('padding', '10px')
      .style('border-radius', '5px')
      .style('pointer-events', 'none')
      .style('left', `${event.pageX + 10}px`)
      .style('top', `${event.pageY - 10}px`)
      .html(`
        <strong>${data.label}</strong><br/>
        Count: ${data.count}<br/>
        Date: ${data.timestamp.toLocaleDateString()}
      `)
  }
  
  function hideTooltip() {
    d3.selectAll('.tooltip').remove()
  }
  
  function showEventModal(event) {
    alert(`${event.title}\n\n${event.description}`)
    // You'd use a proper Vue modal component here
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
    initTimeline()
    window.addEventListener('resize', onWindowResize)

    // Emit data to parent component
    emit('data-updated', sampleData)
  })

  onUnmounted(() => {
    window.removeEventListener('resize', onWindowResize)
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
    height: 65vh;
    min-height: 500px;
    overflow: hidden;
    flex-shrink: 0;
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
    overflow-x: auto;
    overflow-y: hidden;
  }

  .timeline-overlay::-webkit-scrollbar {
    height: 8px;
  }

  .timeline-overlay::-webkit-scrollbar-track {
    background: rgba(0, 0, 0, 0.3);
  }

  .timeline-overlay::-webkit-scrollbar-thumb {
    background: #4ecdc4;
    border-radius: 4px;
  }

  .timeline-overlay::-webkit-scrollbar-thumb:hover {
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