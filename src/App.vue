<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'

const canvas = ref<HTMLCanvasElement | null>(null)
const ready = ref(false)
const failed = ref(false)
const activeMenuIndex = ref<number | null>(null)
const nightTheme = ref(false)

type LetterMenuEntry = { label: string; href: string; external?: boolean }
type LetterMenu = { letter: string; title: string; accent: 'orange' | 'blue'; entries: LetterMenuEntry[] }

const letterMenus: LetterMenu[] = [
  { letter: 'L', title: '定位', accent: 'orange', entries: [{ label: '博客', href: '/blog/' }] },
  { letter: 'O', title: '观察', accent: 'orange', entries: [] },
  { letter: 'O', title: '整理', accent: 'blue', entries: [] },
  { letter: 'L', title: '连接', accent: 'blue', entries: [{ label: 'GitHub', href: 'https://github.com/afterinit', external: true }] },
  { letter: 'C', title: '创作', accent: 'orange', entries: [] },
]

const activeLetterMenu = computed(() => (
  activeMenuIndex.value === null ? null : letterMenus[activeMenuIndex.value] ?? null
))

const PAPER = '#F2EFE7'
const INK = '#171715'
const MUTED = '#777166'
const ORANGE = '#F05032'
const BLUE = '#3155C6'
const SCALE = 0.024
const ART_WIDTH = 1200 * SCALE
const ART_HEIGHT = 620 * SCALE
const FRONT_Z = 0.48
const renderPixelRatio = () => Math.min(Math.max(window.devicePixelRatio || 1, 1.75), 2.25)

const sliceBoxes = [
  { x: 46, y: 138, w: 194, h: 286 },
  { x: 204, y: 138, w: 302, h: 286 },
  { x: 474, y: 138, w: 252, h: 286 },
  { x: 728, y: 138, w: 190, h: 286 },
  { x: 894, y: 138, w: 264, h: 286 },
]

const letterCenters = [149, 355, 600, 825, 1029]
const letterDepthOffsets = [0, 0.05, 0.65, 0.16, 0.24]
const letterBaseRotations = [
  { x: -0.035, y: -0.035 },
  { x: -0.04, y: -0.055 },
  { x: -0.04, y: -0.055 },
  { x: -0.035, y: -0.025 },
  { x: -0.025, y: -0.065 },
]
const xFromSvg = (x: number) => (x - 600) * SCALE
const yFromSvg = (y: number) => (286 - y) * SCALE
const resources: Array<{ dispose: () => void }> = []
const blobUrls: string[] = []
const rotors: Array<{ object: THREE.Object3D; speed: number }> = []
const pulses: Array<{ mesh: THREE.Mesh; material: THREE.MeshBasicMaterial; phase: number }> = []
const runners: Array<{
  mesh: THREE.Mesh
  curve: THREE.Curve<THREE.Vector3>
  speed: number
  phase: number
  orient?: boolean
  depthCue?: { material: THREE.MeshBasicMaterial; range: number }
}> = []
const tracers: Array<{ geometry: THREE.BufferGeometry; curve: THREE.Curve<THREE.Vector3>; speed: number; phase: number; length: number; count: number }> = []
const flowDashes: Array<{ geometry: THREE.BufferGeometry; curve: THREE.Curve<THREE.Vector3>; phase: number; speed: number; length: number }> = []
const headerFlights: Array<{
  mesh: THREE.Mesh
  keyframes: Array<{ t: number; x: number; y: number; rotation: number; scale: number }>
  exhaust: THREE.Line[]
}> = []
const letterAssemblies: THREE.Group[] = []
const letterHitTargets: THREE.Mesh[] = []
const floatingLabels: Array<{ mesh: THREE.Mesh; baseY: number; baseZ: number; phase: number }> = []
const nightOverlayMaterials: THREE.MeshBasicMaterial[] = []
const raycaster = new THREE.Raycaster()
const screenPointer = new THREE.Vector2()

let renderer: THREE.WebGLRenderer | null = null
let scene: THREE.Scene | null = null
let camera: THREE.PerspectiveCamera | null = null
let artwork: THREE.Group | null = null
let backdrop: THREE.Group | null = null
let frame = 0
let observer: ResizeObserver | null = null
let fitScale = 1
let scrollTarget = 0
let scrollPosition = 0
let lastWheelAt = 0
let navigationTween: { from: number; to: number; startedAt: number; duration: number } | null = null
let touchPointerId: number | null = null
let touchStartX = 0
let touchStartY = 0
let touchStartScroll = 0
let touchStartTarget = 0
let touchStartStage = 0
let touchAxis: 'x' | 'y' | null = null
let touchDragging = false
let suppressClickUntil = 0
let locateScan: THREE.Mesh | null = null
let ambientLight: THREE.HemisphereLight | null = null
let keyLight: THREE.DirectionalLight | null = null
let backdropShadowMaterial: THREE.ShadowMaterial | null = null
let lightingMinute = -1

const damp = (value: number, target: number, speed: number, dt: number) => value + (target - value) * (1 - Math.exp(-speed * dt))

function line(points: THREE.Vector3[], color: string, opacity = 1) {
  const geometry = new THREE.BufferGeometry().setFromPoints(points)
  const material = new THREE.LineBasicMaterial({ color, transparent: opacity < 1, opacity, toneMapped: false })
  resources.push(geometry, material)
  return new THREE.Line(geometry, material)
}

function makeLShape() {
  const shape = new THREE.Shape()
  shape.moveTo(-81 * SCALE, 122 * SCALE)
  shape.lineTo(-23 * SCALE, 122 * SCALE)
  shape.lineTo(-23 * SCALE, -66 * SCALE)
  shape.lineTo(81 * SCALE, -66 * SCALE)
  shape.lineTo(81 * SCALE, -124 * SCALE)
  shape.lineTo(-81 * SCALE, -124 * SCALE)
  shape.closePath()
  return shape
}

function makeOShape() {
  const shape = new THREE.Shape()
  shape.absarc(0, 0, 122 * SCALE, 0, Math.PI * 2, false)
  const hole = new THREE.Path()
  hole.absarc(0, 0, 67 * SCALE, 0, Math.PI * 2, true)
  shape.holes.push(hole)
  return shape
}

function makeCShape() {
  const outer = 116 * SCALE
  const inner = 64 * SCALE
  const outerCenterX = 1114 - Math.sqrt(116 ** 2 - 79 ** 2)
  const innerCenterX = 1075 - Math.sqrt(64 ** 2 - 41 ** 2)
  const outerOffset = (outerCenterX - letterCenters[4]!) * SCALE
  const innerOffset = (innerCenterX - letterCenters[4]!) * SCALE
  const outerStart = Math.atan2(79, 1114 - outerCenterX)
  const outerEnd = Math.PI * 2 - outerStart
  const innerStart = Math.atan2(41, 1075 - innerCenterX)
  const innerEnd = Math.PI * 2 - innerStart
  const shape = new THREE.Shape()
  shape.moveTo(outerOffset + Math.cos(outerStart) * outer, Math.sin(outerStart) * outer)
  shape.absarc(outerOffset, 0, outer, outerStart, outerEnd, false)
  shape.lineTo(innerOffset + Math.cos(innerEnd) * inner, Math.sin(innerEnd) * inner)
  shape.absarc(innerOffset, 0, inner, innerEnd, innerStart, true)
  shape.closePath()
  return shape
}

function addLetter(shape: THREE.Shape, x: number) {
  if (!artwork) return
  const assembly = new THREE.Group()
  const letterIndex = letterAssemblies.length
  const baseRotation = letterBaseRotations[letterIndex] ?? { x: 0, y: 0 }
  assembly.position.x = x
  assembly.position.z = letterDepthOffsets[letterIndex] ?? 0
  assembly.rotation.set(baseRotation.x, baseRotation.y, 0)
  assembly.userData.baseZ = assembly.position.z
  assembly.userData.baseRotationX = baseRotation.x
  assembly.userData.baseRotationY = baseRotation.y
  artwork.add(assembly)
  letterAssemblies.push(assembly)
  const geometry = new THREE.ExtrudeGeometry(shape, {
    depth: 0.72,
    steps: 1,
    bevelEnabled: true,
    bevelSegments: 6,
    bevelSize: 0.035,
    bevelThickness: 0.045,
    curveSegments: 128,
  })
  geometry.center()
  const face = new THREE.MeshPhysicalMaterial({ color: PAPER, roughness: 0.9, metalness: 0, clearcoat: 0.04, clearcoatRoughness: 0.9 })
  const side = new THREE.MeshPhysicalMaterial({ color: '#817A70', roughness: 0.72, metalness: 0.03 })
  const mesh = new THREE.Mesh(geometry, [face, side])
  mesh.userData.letterIndex = letterAssemblies.length - 1
  mesh.castShadow = true
  mesh.receiveShadow = true
  assembly.add(mesh)
  letterHitTargets.push(mesh)

  const edgeGeometry = new THREE.EdgesGeometry(geometry, 18)
  const edgeMaterial = new THREE.LineBasicMaterial({ color: INK, transparent: true, opacity: 0.96, toneMapped: false })
  const edges = new THREE.LineSegments(edgeGeometry, edgeMaterial)
  assembly.add(edges)
  resources.push(geometry, face, side, edgeGeometry, edgeMaterial)
}

function addMarker(letterIndex: number, x: number, y: number, color: string, radius = 0.12) {
  const assembly = letterAssemblies[letterIndex]
  if (!assembly) throw new Error('Letter unavailable')
  const geometry = new THREE.SphereGeometry(radius, 24, 24)
  const material = new THREE.MeshBasicMaterial({ color, toneMapped: false })
  const mesh = new THREE.Mesh(geometry, material)
  mesh.position.set((x - letterCenters[letterIndex]!) * SCALE, yFromSvg(y), FRONT_Z + 0.09)
  assembly.add(mesh)
  resources.push(geometry, material)
  return mesh
}

function addDynamicLinework() {
  if (!artwork) return
  const firstL = letterAssemblies[0]!
  const observeLetter = letterAssemblies[1]!
  const organizeLetter = letterAssemblies[2]!
  const createLetter = letterAssemblies[4]!

  const scanGeometry = new THREE.BoxGeometry(40 * SCALE, 0.032, 0.035)
  const scanMaterial = new THREE.MeshBasicMaterial({ color: BLUE, transparent: true, opacity: 0.92, toneMapped: false })
  const scan = new THREE.Mesh(scanGeometry, scanMaterial)
  scan.position.set((97 - letterCenters[0]!) * SCALE, yFromSvg(190), FRONT_Z + 0.06)
  firstL.add(scan)
  locateScan = scan
  resources.push(scanGeometry, scanMaterial)

  const locate = addMarker(0, 97, 164, ORANGE, 0.13)
  const locateRingGeometry = new THREE.TorusGeometry(13 * SCALE, 0.018, 8, 64)
  const locateRingMaterial = new THREE.MeshBasicMaterial({ color: ORANGE, transparent: true, opacity: 0.55, toneMapped: false })
  const locateRing = new THREE.Mesh(locateRingGeometry, locateRingMaterial)
  locateRing.position.copy(locate.position)
  firstL.add(locateRing)
  pulses.push({ mesh: locateRing, material: locateRingMaterial, phase: 0 })
  resources.push(locateRingGeometry, locateRingMaterial)

  const orbitGroup = new THREE.Group()
  orbitGroup.position.set(0, 0, FRONT_Z + 0.055)
  const orbitTilt = THREE.MathUtils.degToRad(18)
  const orbitCenter = new THREE.Vector3((430 - 355) * SCALE, yFromSvg(306), 0)
  const orbitMajor = new THREE.Vector3(
    115 * SCALE * Math.cos(orbitTilt),
    115 * SCALE * Math.sin(orbitTilt),
    0,
  )
  const orbitMinor = new THREE.Vector3(
    44 * SCALE * Math.sin(orbitTilt),
    -44 * SCALE * Math.cos(orbitTilt),
    0.95,
  )
  const orbitCurve = new class extends THREE.Curve<THREE.Vector3> {
    constructor() {
      super()
    }

    override getPoint(t: number, target = new THREE.Vector3()) {
      const angle = Math.PI - t * Math.PI * 2
      return target
        .copy(orbitCenter)
        .addScaledVector(orbitMajor, Math.cos(angle))
        .addScaledVector(orbitMinor, Math.sin(angle))
    }
  }()
  const orbitPoints = orbitCurve.getPoints(240)
  const orbitFront = orbitPoints.slice(0, 121)
  const orbitBack = orbitPoints.slice(120)
  orbitGroup.add(line(orbitBack, '#AFA89B', 0.34))
  orbitGroup.add(line(orbitFront, MUTED, 0.86))
  const orbitDot = addMarker(1, 355, 286, ORANGE, 0.14)
  const orbitDotMaterial = orbitDot.material as THREE.MeshBasicMaterial
  orbitDotMaterial.transparent = true
  observeLetter.remove(orbitDot)
  orbitDot.position.set(0, 0, 0)
  orbitGroup.add(orbitDot)
  runners.push({
    mesh: orbitDot,
    curve: orbitCurve,
    speed: 0.000083,
    phase: 0,
    depthCue: { material: orbitDotMaterial, range: 0.95 },
  })
  observeLetter.add(orbitGroup)

  const organize = new THREE.Group()
  organize.position.set(0, 0, FRONT_Z + 0.07)
  const outerRingGeometry = new THREE.TorusGeometry(122 * SCALE, 0.022, 6, 180, Math.PI * 1.91)
  const outerRingMaterial = new THREE.MeshBasicMaterial({ color: INK, toneMapped: false })
  organize.add(new THREE.Mesh(outerRingGeometry, outerRingMaterial))
  resources.push(outerRingGeometry, outerRingMaterial)
  organizeLetter.add(organize)
  rotors.push({ object: organize, speed: 0.42 })
  const organizeInner = new THREE.Group()
  organizeInner.position.set(0, 0, FRONT_Z + 0.075)
  const innerRingMaterial = new THREE.MeshBasicMaterial({ color: BLUE, toneMapped: false })
  resources.push(innerRingMaterial)
  for (let segment = 0; segment < 12; segment++) {
    const arcGeometry = new THREE.TorusGeometry(67 * SCALE, 0.018, 6, 16, 0.28)
    const arc = new THREE.Mesh(arcGeometry, innerRingMaterial)
    arc.rotation.z = segment / 12 * Math.PI * 2
    organizeInner.add(arc)
    resources.push(arcGeometry)
  }
  organizeLetter.add(organizeInner)
  rotors.push({ object: organizeInner, speed: -0.62 })
  const organizeCore = addMarker(2, 600, 286, BLUE, 0.14)
  const coreRingGeometry = new THREE.TorusGeometry(18 * SCALE, 0.014, 8, 64)
  const coreRingMaterial = new THREE.MeshBasicMaterial({ color: BLUE, transparent: true, opacity: 0.42, toneMapped: false })
  const coreRing = new THREE.Mesh(coreRingGeometry, coreRingMaterial)
  coreRing.position.copy(organizeCore.position)
  organizeLetter.add(coreRing)
  pulses.push({ mesh: coreRing, material: coreRingMaterial, phase: 0.45 })
  resources.push(coreRingGeometry, coreRingMaterial)

  const linkStart = new THREE.Vector3((773 - letterCenters[3]!) * SCALE, yFromSvg(184), FRONT_Z + 0.08)
  const linkCorner = new THREE.Vector3((773 - letterCenters[3]!) * SCALE, yFromSvg(381), FRONT_Z + 0.08)
  const linkEnd = new THREE.Vector3((886 - letterCenters[3]!) * SCALE, yFromSvg(381), FRONT_Z + 0.08)
  const linkCurve = new THREE.CatmullRomCurve3([linkStart, linkCorner, linkEnd], false, 'catmullrom', 0.03)
  const linkRunner = addMarker(3, 773, 184, BLUE, 0.085)
  runners.push({ mesh: linkRunner, curve: linkCurve, speed: 0.0001, phase: 0 })

  const cCenter = new THREE.Vector3(0, 0, FRONT_Z + 0.08)
  const cPoints: THREE.Vector3[] = []
  for (let i = 0; i < 120; i++) {
    const t = i / 119
    const angle = 0.75 + t * (Math.PI * 2 - 1.5)
    cPoints.push(new THREE.Vector3(cCenter.x + Math.cos(angle) * 116 * SCALE, Math.sin(angle) * 116 * SCALE, cCenter.z))
  }
  const createCurve = new THREE.CatmullRomCurve3(cPoints, false)
  const tracerCount = 16
  const traceGeometry = new THREE.BufferGeometry()
  const tracePositions = new THREE.BufferAttribute(new Float32Array(tracerCount * 3), 3)
  tracePositions.setUsage(THREE.DynamicDrawUsage)
  traceGeometry.setAttribute('position', tracePositions)
  const traceMaterial = new THREE.LineBasicMaterial({ color: ORANGE, transparent: true, opacity: 0.96, toneMapped: false })
  const createTrace = new THREE.Line(traceGeometry, traceMaterial)
  createTrace.frustumCulled = false
  createLetter.add(createTrace)
  tracers.push({ geometry: traceGeometry, curve: createCurve, speed: 0.000075, phase: 0, length: 0.105, count: tracerCount })
  resources.push(traceGeometry, traceMaterial)

  const spark = new THREE.Group()
  spark.position.set((1025 - letterCenters[4]!) * SCALE, 0, FRONT_Z + 0.09)
  spark.add(line([new THREE.Vector3(0, 28 * SCALE, 0), new THREE.Vector3(0, 14 * SCALE, 0)], BLUE, 0.96))
  spark.add(line([new THREE.Vector3(0, -14 * SCALE, 0), new THREE.Vector3(0, -28 * SCALE, 0)], ORANGE, 0.96))
  spark.add(line([new THREE.Vector3(-28 * SCALE, 0, 0), new THREE.Vector3(-14 * SCALE, 0, 0)], ORANGE, 0.96))
  spark.add(line([new THREE.Vector3(14 * SCALE, 0, 0), new THREE.Vector3(28 * SCALE, 0, 0)], BLUE, 0.96))
  spark.add(line([new THREE.Vector3(-18 * SCALE, 18 * SCALE, 0), new THREE.Vector3(-9 * SCALE, 9 * SCALE, 0)], MUTED, 0.9))
  spark.add(line([new THREE.Vector3(9 * SCALE, -9 * SCALE, 0), new THREE.Vector3(18 * SCALE, -18 * SCALE, 0)], MUTED, 0.9))
  spark.add(line([
    new THREE.Vector3(0, 12 * SCALE, 0), new THREE.Vector3(4 * SCALE, 4 * SCALE, 0),
    new THREE.Vector3(12 * SCALE, 0, 0), new THREE.Vector3(4 * SCALE, -4 * SCALE, 0),
    new THREE.Vector3(0, -12 * SCALE, 0), new THREE.Vector3(-4 * SCALE, -4 * SCALE, 0),
    new THREE.Vector3(-12 * SCALE, 0, 0), new THREE.Vector3(-4 * SCALE, 4 * SCALE, 0),
    new THREE.Vector3(0, 12 * SCALE, 0),
  ], ORANGE, 0.96))
  const sparkCoreGeometry = new THREE.SphereGeometry(3.5 * SCALE, 16, 16)
  const sparkCoreMaterial = new THREE.MeshBasicMaterial({ color: BLUE, toneMapped: false })
  spark.add(new THREE.Mesh(sparkCoreGeometry, sparkCoreMaterial))
  resources.push(sparkCoreGeometry, sparkCoreMaterial)
  createLetter.add(spark)
  rotors.push({ object: spark, speed: -0.45 })
}

function addHeaderDynamics() {
  if (!backdrop) return
  const header = new THREE.Group()
  const z = -0.57
  const point = (x: number, y: number, depth = z) => new THREE.Vector3(xFromSvg(x), yFromSvg(y), depth)
  const dot = (parent: THREE.Object3D, x: number, y: number, color: string, radius = 0.07, local = false) => {
    const geometry = new THREE.SphereGeometry(radius, 18, 18)
    const material = new THREE.MeshBasicMaterial({ color, toneMapped: false })
    const mesh = new THREE.Mesh(geometry, material)
    mesh.position.copy(local ? new THREE.Vector3(x, y, 0) : point(x, y, z + 0.025))
    parent.add(mesh)
    resources.push(geometry, material)
    return mesh
  }

  const locatePulseGeometry = new THREE.TorusGeometry(8 * SCALE, 0.009, 6, 36)
  const locatePulseMaterial = new THREE.MeshBasicMaterial({ color: ORANGE, transparent: true, opacity: 0.5, toneMapped: false })
  const locatePulse = new THREE.Mesh(locatePulseGeometry, locatePulseMaterial)
  locatePulse.position.copy(point(370, 53, z + 0.025))
  header.add(locatePulse)
  pulses.push({ mesh: locatePulse, material: locatePulseMaterial, phase: 0.2 })
  resources.push(locatePulseGeometry, locatePulseMaterial)

  const observe = new THREE.Group()
  observe.position.copy(point(555, 53, z + 0.02))
  observe.rotation.z = THREE.MathUtils.degToRad(16)
  const miniOrbit = new THREE.EllipseCurve(0, 0, 28 * SCALE, 10 * SCALE, 0, Math.PI * 2)
  const miniOrbitPoints = miniOrbit.getPoints(72).map(value => new THREE.Vector3(value.x, value.y, 0))
  const observeDot = dot(observe, 0, 0, ORANGE, 0.065, true)
  runners.push({ mesh: observeDot, curve: new THREE.CatmullRomCurve3(miniOrbitPoints, true), speed: 0.00009, phase: 0.08 })
  header.add(observe)

  const organize = new THREE.Group()
  organize.position.copy(point(750, 53, z + 0.02))
  const ringCurve = new THREE.EllipseCurve(0, 0, 15 * SCALE, 15 * SCALE, 0, Math.PI * 1.82)
  organize.add(line(ringCurve.getPoints(64).map(value => new THREE.Vector3(value.x, value.y, 0)), INK, 0.95))
  const organizeCore = dot(organize, 0, 0, BLUE, 0.072, true)
  const corePulseGeometry = new THREE.TorusGeometry(7 * SCALE, 0.008, 6, 32)
  const corePulseMaterial = new THREE.MeshBasicMaterial({ color: BLUE, transparent: true, opacity: 0.42, toneMapped: false })
  const corePulse = new THREE.Mesh(corePulseGeometry, corePulseMaterial)
  corePulse.position.copy(organizeCore.position)
  organize.add(corePulse)
  pulses.push({ mesh: corePulse, material: corePulseMaterial, phase: 0.5 })
  resources.push(corePulseGeometry, corePulseMaterial)
  header.add(organize)
  rotors.push({ object: organize, speed: 0.52 })

  const linkStart = point(938, 30, z + 0.02)
  const linkTurn = point(938, 59, z + 0.02)
  const linkControl = point(938, 67, z + 0.02)
  const linkExit = point(946, 67, z + 0.02)
  const linkEnd = point(962, 67, z + 0.02)
  const linkCurve = new THREE.CurvePath<THREE.Vector3>()
  linkCurve.add(new THREE.LineCurve3(linkStart, linkTurn))
  linkCurve.add(new THREE.QuadraticBezierCurve3(linkTurn, linkControl, linkExit))
  linkCurve.add(new THREE.LineCurve3(linkExit, linkEnd))
  for (let dash = 0; dash < 5; dash++) {
    const geometry = new THREE.BufferGeometry()
    const positions = new THREE.BufferAttribute(new Float32Array(6), 3)
    positions.setUsage(THREE.DynamicDrawUsage)
    geometry.setAttribute('position', positions)
    const material = new THREE.LineBasicMaterial({ color: BLUE, transparent: true, opacity: 0.96, toneMapped: false })
    const dashLine = new THREE.Line(geometry, material)
    dashLine.frustumCulled = false
    header.add(dashLine)
    flowDashes.push({ geometry, curve: linkCurve, phase: dash / 5, speed: 0.00028, length: 0.075 })
    resources.push(geometry, material)
  }

  const createRunnerGeometry = new THREE.BufferGeometry()
  createRunnerGeometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array([
    -3 * SCALE, -3 * SCALE, 0,
    4 * SCALE, 0, 0,
    -3 * SCALE, 3 * SCALE, 0,
  ]), 3))
  const createRunnerMaterial = new THREE.MeshBasicMaterial({ color: ORANGE, toneMapped: false })
  const createRunner = new THREE.Mesh(createRunnerGeometry, createRunnerMaterial)
  const exhaustLines: THREE.Line[] = []
  const exhaustSpecs = [
    { y: 2 * SCALE, end: -9 * SCALE, color: BLUE, width: 0.78 },
    { y: 0, end: -12 * SCALE, color: ORANGE, width: 1 },
    { y: -2 * SCALE, end: -9 * SCALE, color: BLUE, width: 0.78 },
  ]
  exhaustSpecs.forEach(spec => {
    const geometry = new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(-5 * SCALE, spec.y, -0.002),
      new THREE.Vector3(spec.end, spec.y, -0.002),
    ])
    const material = new THREE.LineBasicMaterial({ color: spec.color, transparent: true, opacity: 0, toneMapped: false })
    const exhaust = new THREE.Line(geometry, material)
    exhaust.visible = false
    exhaust.scale.y = spec.width
    createRunner.add(exhaust)
    exhaustLines.push(exhaust)
    resources.push(geometry, material)
  })
  createRunner.position.z = z + 0.02
  header.add(createRunner)
  const flightKeyframes = [
    { t: 0, x: 1122, y: 37, rotation: 143.13, scale: 1 },
    { t: .06, x: 1115.42, y: 33.75, rotation: 164.28, scale: 1 },
    { t: .12, x: 1108.11, y: 33.09, rotation: 185.42, scale: 1 },
    { t: .18, x: 1101.06, y: 35.11, rotation: 206.57, scale: 1 },
    { t: .24, x: 1095.21, y: 39.54, rotation: 227.71, scale: 1 },
    { t: .30, x: 1091.35, y: 45.79, rotation: 248.86, scale: 1 },
    { t: .36, x: 1090, y: 53, rotation: 270, scale: 1 },
    { t: .42, x: 1091.35, y: 60.21, rotation: 291.14, scale: 1 },
    { t: .48, x: 1095.21, y: 66.46, rotation: 312.29, scale: 1 },
    { t: .54, x: 1101.06, y: 70.89, rotation: 333.43, scale: 1 },
    { t: .60, x: 1108.11, y: 72.91, rotation: 354.58, scale: 1 },
    { t: .66, x: 1115.42, y: 72.25, rotation: 375.72, scale: 1 },
    { t: .72, x: 1122, y: 69, rotation: 396.87, scale: 1 },
    { t: .76, x: 1122, y: 69, rotation: 396.87, scale: .72 },
    { t: .80, x: 1122, y: 69, rotation: 396.87, scale: 1.08 },
    { t: .8333, x: 1126.3, y: 64.59, rotation: 414.58, scale: 1 },
    { t: .8667, x: 1129.05, y: 59.08, rotation: 432.29, scale: 1 },
    { t: .90, x: 1130, y: 53, rotation: 450, scale: 1 },
    { t: .9333, x: 1129.05, y: 46.92, rotation: 467.71, scale: 1 },
    { t: .9667, x: 1126.3, y: 41.41, rotation: 485.42, scale: 1 },
    { t: 1, x: 1122, y: 37, rotation: 503.13, scale: 1 },
  ].map(keyframe => ({ ...keyframe, x: xFromSvg(keyframe.x), y: yFromSvg(keyframe.y), rotation: THREE.MathUtils.degToRad(keyframe.rotation) }))
  headerFlights.push({ mesh: createRunner, keyframes: flightKeyframes, exhaust: exhaustLines })
  resources.push(createRunnerGeometry, createRunnerMaterial)

  backdrop.add(header)
}

function nightVersionOfSvg(markup: string) {
  const palette: Record<string, string> = {
    '#F2EFE7': '#111722',
    '#171715': '#ECE8DE',
    '#777166': '#A7AFBD',
    '#AFA89B': '#667084',
    '#CEC8BC': '#313B4C',
    '#3155C6': '#6F8DFF',
    '#F05032': '#FF6B49',
  }
  return Object.entries(palette).reduce(
    (result, [day, night]) => result.replace(new RegExp(day, 'gi'), night),
    markup,
  )
}

async function textureFromSvg(markup: string, minimumRasterScale = 2.5) {
  const rasterDocument = new DOMParser().parseFromString(markup, 'image/svg+xml')
  const rasterRoot = rasterDocument.documentElement
  const viewBox = (rasterRoot.getAttribute('viewBox') ?? '0 0 1200 620')
    .trim()
    .split(/[ ,]+/)
    .map(Number)
  const sourceWidth = Number.parseFloat(rasterRoot.getAttribute('width') ?? '') || viewBox[2] || 1200
  const sourceHeight = Number.parseFloat(rasterRoot.getAttribute('height') ?? '') || viewBox[3] || 620
  const rasterScale = Math.min(Math.max(window.devicePixelRatio || 1, minimumRasterScale), 3)
  rasterRoot.setAttribute('width', String(Math.round(sourceWidth * rasterScale)))
  rasterRoot.setAttribute('height', String(Math.round(sourceHeight * rasterScale)))
  const rasterMarkup = new XMLSerializer().serializeToString(rasterRoot)
  const url = URL.createObjectURL(new Blob([rasterMarkup], { type: 'image/svg+xml' }))
  blobUrls.push(url)
  const texture = await new THREE.TextureLoader().loadAsync(url)
  texture.colorSpace = THREE.SRGBColorSpace
  texture.anisotropy = renderer?.capabilities.getMaxAnisotropy() ?? 1
  texture.minFilter = THREE.LinearMipmapLinearFilter
  texture.magFilter = THREE.LinearFilter
  texture.generateMipmaps = true
  resources.push(texture)
  return texture
}

function addNightOverlay(
  geometry: THREE.BufferGeometry,
  texture: THREE.Texture,
  position: THREE.Vector3,
  parent: THREE.Object3D,
  renderOrder: number,
  alphaTest = 0,
) {
  const material = new THREE.MeshBasicMaterial({
    map: texture,
    transparent: true,
    opacity: 0,
    alphaTest,
    depthWrite: false,
    toneMapped: false,
  })
  const plane = new THREE.Mesh(geometry, material)
  plane.position.copy(position)
  plane.renderOrder = renderOrder
  parent.add(plane)
  nightOverlayMaterials.push(material)
  resources.push(material)
  return plane
}

function applyTimeLighting(date: Date) {
  if (!scene || !ambientLight || !keyLight) return
  const hour = date.getHours() + date.getMinutes() / 60 + date.getSeconds() / 3600

  // Local sunrise-to-sunset arc: left light casts right-down shadows in the
  // morning, overhead light casts downward at noon, and vice versa later.
  const sunProgress = THREE.MathUtils.clamp((hour - 6) / 12, 0, 1)
  const sunTravel = THREE.MathUtils.smoothstep(sunProgress, 0, 1)
  const dayLightX = THREE.MathUtils.lerp(-10, 10, sunTravel)
  const dayLightY = 7.5 + Math.sin(sunProgress * Math.PI) * 7

  let nightMix = 0
  if (hour < 5.5 || hour >= 19.5) {
    nightMix = 1
  } else if (hour < 7) {
    nightMix = 1 - THREE.MathUtils.smoothstep(hour, 5.5, 7)
  } else if (hour >= 18) {
    nightMix = THREE.MathUtils.smoothstep(hour, 18, 19.5)
  }
  nightTheme.value = nightMix >= 0.55

  keyLight.position.set(
    THREE.MathUtils.lerp(dayLightX, -4.5, nightMix),
    THREE.MathUtils.lerp(dayLightY, 7.5, nightMix),
    THREE.MathUtils.lerp(14, 11, nightMix),
  )

  // Keep the logo material visually identical throughout the day. Time changes
  // the light direction and the setting, never the hue or brightness of letters.
  keyLight.color.set('#FFFDF8')
  keyLight.intensity = 3.2
  ambientLight.color.set('#FFFDF8')
  ambientLight.groundColor.set('#AAA297')
  ambientLight.intensity = 2.2

  const backgroundColor = new THREE.Color(PAPER).lerp(new THREE.Color('#0D121B'), nightMix)
  ;(scene.background as THREE.Color).copy(backgroundColor)
  nightOverlayMaterials.forEach(material => { material.opacity = nightMix })

  if (backdropShadowMaterial) {
    backdropShadowMaterial.color.copy(new THREE.Color('#403A33')).lerp(new THREE.Color('#101827'), nightMix)
    backdropShadowMaterial.opacity = THREE.MathUtils.lerp(0.14, 0.2, nightMix)
  }
}

async function addSvgSurfaces() {
  if (!artwork || !backdrop) return
  const source = await fetch('/logo.svg').then(response => {
    if (!response.ok) throw new Error('Unable to load logo')
    return response.text()
  })
  const documentSvg = new DOMParser().parseFromString(source, 'image/svg+xml')
  const defs = documentSvg.querySelector('defs')?.outerHTML ?? ''
  const groups = [...documentSvg.querySelectorAll('#loolc-linework > g')]
  const headerSource = documentSvg.querySelector('#header-sequence')
  const headerClone = headerSource?.cloneNode(true) as Element | undefined
  headerClone?.querySelectorAll('.mini-locate-pulse, .mini-observe-dot, .mini-organize-ring, .mini-organize-core, .mini-link-flow, .mini-create-flight, .mini-create-exhaust').forEach(element => element.remove())

  documentSvg.querySelector('#loolc-linework')?.remove()
  documentSvg.querySelector('#header-sequence')?.remove()
  const backgroundSource = new XMLSerializer().serializeToString(documentSvg.documentElement)
  const [backgroundTexture, nightBackgroundTexture] = await Promise.all([
    textureFromSvg(backgroundSource),
    textureFromSvg(nightVersionOfSvg(backgroundSource), 2.25),
  ])
  const backgroundGeometry = new THREE.PlaneGeometry(ART_WIDTH, ART_HEIGHT)
  const backgroundMaterial = new THREE.MeshBasicMaterial({ map: backgroundTexture, toneMapped: false })
  const background = new THREE.Mesh(backgroundGeometry, backgroundMaterial)
  background.position.z = -0.72
  backdrop.add(background)
  const shadowMaterial = new THREE.ShadowMaterial({ color: '#403A33', transparent: true, opacity: 0.14 })
  backdropShadowMaterial = shadowMaterial
  const shadowReceiver = new THREE.Mesh(backgroundGeometry, shadowMaterial)
  shadowReceiver.position.z = -0.69
  shadowReceiver.receiveShadow = true
  shadowReceiver.renderOrder = 2
  backdrop.add(shadowReceiver)
  addNightOverlay(backgroundGeometry, nightBackgroundTexture, new THREE.Vector3(0, 0, -0.715), backdrop, 1)
  resources.push(backgroundGeometry, backgroundMaterial, shadowMaterial)

  if (headerClone) {
    const box = { x: 320, y: 22, w: 840, h: 60 }
    const headerMarkup = `<svg xmlns="http://www.w3.org/2000/svg" width="${box.w}" height="${box.h}" viewBox="${box.x} ${box.y} ${box.w} ${box.h}">${defs}${headerClone.outerHTML}</svg>`
    const [headerTexture, nightHeaderTexture] = await Promise.all([
      textureFromSvg(headerMarkup),
      textureFromSvg(nightVersionOfSvg(headerMarkup), 2.25),
    ])
    const headerGeometry = new THREE.PlaneGeometry(box.w * SCALE, box.h * SCALE)
    const headerMaterial = new THREE.MeshBasicMaterial({ map: headerTexture, transparent: true, alphaTest: 0.02, depthWrite: false, toneMapped: false })
    const headerPlane = new THREE.Mesh(headerGeometry, headerMaterial)
    headerPlane.position.set(xFromSvg(box.x + box.w / 2), yFromSvg(box.y + box.h / 2), -0.6)
    headerPlane.renderOrder = 3
    backdrop.add(headerPlane)
    addNightOverlay(headerGeometry, nightHeaderTexture, new THREE.Vector3(
      headerPlane.position.x,
      headerPlane.position.y,
      -0.598,
    ), backdrop, 4, 0.02)
    resources.push(headerGeometry, headerMaterial)
  }

  await Promise.all(sliceBoxes.map(async (box, index) => {
    const group = groups[index]
    const assembly = letterAssemblies[index]
    if (!group || !assembly) return
    const label = group.querySelector('text')?.cloneNode(true) as Element | undefined
    group.querySelector('text')?.remove()
    if (index === 1) {
      group.querySelectorAll('ellipse, .orbit-dot').forEach(element => element.remove())
    }
    if (index === 2) {
      group.querySelectorAll('.organize-ring, .organize-ring-reverse, .origin').forEach(element => element.remove())
    }
    if (index === 4) {
      group.querySelectorAll('.outline, .create-trace, .create-tip, .create-end-pulse, .create-spark, .create-core').forEach(element => element.remove())
    }
    const markup = `<svg xmlns="http://www.w3.org/2000/svg" width="${box.w}" height="${box.h}" viewBox="${box.x} ${box.y} ${box.w} ${box.h}">${defs}${group.outerHTML}</svg>`
    const labelBox = { x: letterCenters[index]! - 85, y: 432, w: 170, h: 32 }
    const labelMarkup = label
      ? `<svg xmlns="http://www.w3.org/2000/svg" width="${labelBox.w}" height="${labelBox.h}" viewBox="${labelBox.x} ${labelBox.y} ${labelBox.w} ${labelBox.h}">${defs}${label.outerHTML}</svg>`
      : null
    const [texture, labelTexture, nightLabelTexture] = await Promise.all([
      textureFromSvg(markup),
      labelMarkup ? textureFromSvg(labelMarkup) : Promise.resolve(null),
      labelMarkup ? textureFromSvg(nightVersionOfSvg(labelMarkup), 2.25) : Promise.resolve(null),
    ])
    const geometry = new THREE.PlaneGeometry(box.w * SCALE, box.h * SCALE)
    const material = new THREE.MeshBasicMaterial({ map: texture, transparent: true, alphaTest: 0.02, depthWrite: false, toneMapped: false })
    const plane = new THREE.Mesh(geometry, material)
    plane.userData.letterIndex = index
    plane.position.set((box.x + box.w / 2 - letterCenters[index]!) * SCALE, yFromSvg(box.y + box.h / 2), FRONT_Z)
    plane.renderOrder = 4
    assembly.add(plane)
    letterHitTargets.push(plane)
    resources.push(geometry, material)

    if (labelTexture) {
      const labelGeometry = new THREE.PlaneGeometry(labelBox.w * SCALE, labelBox.h * SCALE)
      const labelMaterial = new THREE.MeshBasicMaterial({
        map: labelTexture,
        transparent: true,
        alphaTest: 0.08,
        depthWrite: false,
        toneMapped: false,
      })
      const labelPlane = new THREE.Mesh(labelGeometry, labelMaterial)
      const baseY = yFromSvg(labelBox.y + labelBox.h / 2)
      const baseZ = FRONT_Z + 0.16
      labelPlane.userData.letterIndex = index
      labelPlane.position.set(0, baseY, baseZ)
      labelPlane.renderOrder = 5
      assembly.add(labelPlane)
      letterHitTargets.push(labelPlane)
      floatingLabels.push({ mesh: labelPlane, baseY, baseZ, phase: index * 1.17 })
      resources.push(labelGeometry, labelMaterial)

      if (nightLabelTexture) {
        const nightLabel = addNightOverlay(
          labelGeometry,
          nightLabelTexture,
          new THREE.Vector3(0, baseY, baseZ + 0.003),
          assembly,
          6,
          0.08,
        )
        floatingLabels.push({ mesh: nightLabel, baseY, baseZ: baseZ + 0.003, phase: index * 1.17 })
      }
    }
  }))
}

async function build() {
  if (!canvas.value) return
  try {
    renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true, powerPreference: 'high-performance', alpha: false })
    renderer.setPixelRatio(renderPixelRatio())
    renderer.setSize(innerWidth, innerHeight, false)
    renderer.outputColorSpace = THREE.SRGBColorSpace
    renderer.toneMapping = THREE.ACESFilmicToneMapping
    renderer.toneMappingExposure = 1
    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = THREE.PCFSoftShadowMap

    scene = new THREE.Scene()
    scene.background = new THREE.Color(PAPER)
    camera = new THREE.PerspectiveCamera(32, innerWidth / innerHeight, 0.1, 100)
    camera.position.set(0, 0.2, 30)

    backdrop = new THREE.Group()
    artwork = new THREE.Group()
    scene.add(backdrop, artwork)

    ambientLight = new THREE.HemisphereLight('#FFFDF8', '#AAA297', 2.2)
    keyLight = new THREE.DirectionalLight('#FFF8EB', 3.2)
    keyLight.position.set(-8, 10, 14)
    keyLight.castShadow = true
    keyLight.shadow.mapSize.set(2048, 2048)
    keyLight.shadow.camera.left = -18
    keyLight.shadow.camera.right = 18
    keyLight.shadow.camera.top = 10
    keyLight.shadow.camera.bottom = -10
    keyLight.shadow.bias = -0.0004
    scene.add(ambientLight, keyLight)

    addLetter(makeLShape(), xFromSvg(letterCenters[0]!))
    addLetter(makeOShape(), xFromSvg(letterCenters[1]!))
    addLetter(makeOShape(), xFromSvg(letterCenters[2]!))
    addLetter(makeLShape(), xFromSvg(letterCenters[3]!))
    addLetter(makeCShape(), xFromSvg(letterCenters[4]!))
    await addSvgSurfaces()
    addDynamicLinework()
    addHeaderDynamics()
    applyTimeLighting(new Date())
    lightingMinute = Math.floor(Date.now() / 60_000)

    const resize = () => {
      if (!renderer || !camera || !artwork || !backdrop) return
      const aspect = innerWidth / innerHeight
      camera.aspect = aspect
      camera.updateProjectionMatrix()
      renderer.setPixelRatio(renderPixelRatio())
      renderer.setSize(innerWidth, innerHeight, false)
      fitScale = Math.min(1, aspect / 1.72)
      backdrop.scale.setScalar(fitScale)
      artwork.scale.setScalar(fitScale)
    }
    observer = new ResizeObserver(resize)
    observer.observe(document.documentElement)
    resize()

    let previous = performance.now()
    const stageFrom = new THREE.Vector3()
    const stageTo = new THREE.Vector3()
    const tweenFrom = new THREE.Vector3()
    const tweenTo = new THREE.Vector3()
    const cameraFocus = new THREE.Vector3()
    const getStagePosition = (stage: number, target: THREE.Vector3) => {
      if (stage <= 0) return target.set(0, 0, 0)
      const assembly = letterAssemblies[Math.min(stage - 1, letterAssemblies.length - 1)]!
      return target
        .set(assembly.position.x, 0, Number(assembly.userData.baseZ ?? 0))
        .multiplyScalar(fitScale)
    }
    const getProgressPosition = (progress: number, target: THREE.Vector3) => {
      const fromStage = Math.floor(progress)
      const toStage = Math.min(5, fromStage + 1)
      const stageMix = THREE.MathUtils.smoothstep(progress - fromStage, 0, 1)
      getStagePosition(fromStage, stageFrom)
      getStagePosition(toStage, stageTo)
      return target.lerpVectors(stageFrom, stageTo, stageMix)
    }
    const animate = (now: number) => {
      if (!renderer || !scene || !camera || !artwork || !backdrop) return
      const dt = Math.min(0.04, Math.max(0.001, (now - previous) / 1000))
      previous = now
      const currentLightingMinute = Math.floor(Date.now() / 60_000)
      if (currentLightingMinute !== lightingMinute) {
        lightingMinute = currentLightingMinute
        applyTimeLighting(new Date())
      }
      const activeTween = navigationTween
      let tweenEase: number | null = null
      if (activeTween) {
        const progress = THREE.MathUtils.clamp((now - activeTween.startedAt) / activeTween.duration, 0, 1)
        tweenEase = progress * progress * progress * (progress * (progress * 6 - 15) + 10)
        scrollPosition = THREE.MathUtils.lerp(activeTween.from, activeTween.to, tweenEase)
        if (progress >= 1) navigationTween = null
      } else {
        if (lastWheelAt > 0 && now - lastWheelAt > 220) {
          scrollTarget = Math.round(scrollTarget)
          lastWheelAt = 0
        }
        scrollPosition = damp(scrollPosition, scrollTarget, 4.2, dt)
      }
      artwork.rotation.set(0, 0, 0)
      artwork.position.set(0, 0, 0)
      artwork.scale.setScalar(fitScale)
      backdrop.rotation.set(0, 0, 0)
      backdrop.position.set(0, 0, 0)
      backdrop.scale.setScalar(fitScale)
      letterAssemblies.forEach(assembly => {
        assembly.position.y = 0
        assembly.position.z = Number(assembly.userData.baseZ ?? 0)
        assembly.rotation.set(
          Number(assembly.userData.baseRotationX ?? 0),
          Number(assembly.userData.baseRotationY ?? 0),
          0,
        )
        assembly.scale.setScalar(1)
      })
      floatingLabels.forEach(label => {
        const float = Math.sin(now * 0.00072 + label.phase)
        label.mesh.position.y = label.baseY + float * 0.045
        label.mesh.position.z = label.baseZ + float * 0.025
      })
      artwork.updateMatrixWorld(true)

      if (activeTween && tweenEase !== null) {
        getProgressPosition(activeTween.from, tweenFrom)
        getProgressPosition(activeTween.to, tweenTo)
        cameraFocus.lerpVectors(tweenFrom, tweenTo, tweenEase)
      } else {
        getProgressPosition(scrollPosition, cameraFocus)
      }
      const approach = THREE.MathUtils.smoothstep(THREE.MathUtils.clamp(scrollPosition, 0, 1), 0, 1)
      cameraFocus.y -= approach * 0.34 * fitScale
      const focusDistance = THREE.MathUtils.lerp(30, 14.4 * fitScale, approach)
      camera.position.set(
        cameraFocus.x,
        cameraFocus.y + 0.18 * fitScale,
        cameraFocus.z + focusDistance,
      )
      camera.lookAt(cameraFocus)

      if (locateScan) {
        const cycle = (now * 0.00016) % 1
        locateScan.position.y = yFromSvg(190 + cycle * 158)
        ;(locateScan.material as THREE.MeshBasicMaterial).opacity = Math.sin(cycle * Math.PI) * 0.92
      }
      pulses.forEach(pulse => {
        const cycle = (now * 0.00024 + pulse.phase) % 1
        const scale = 0.62 + cycle * 1.25
        pulse.mesh.scale.setScalar(scale)
        pulse.material.opacity = (1 - cycle) * 0.62
      })
      runners.forEach(runner => {
        const travel = (runner.phase + now * runner.speed) % 1
        const point = runner.curve.getPointAt(travel)
        runner.mesh.position.copy(point)
        if (runner.orient) {
          const tangent = runner.curve.getTangentAt(travel)
          runner.mesh.rotation.z = Math.atan2(tangent.y, tangent.x)
        }
        if (runner.depthCue) {
          const depth = THREE.MathUtils.clamp((point.z / runner.depthCue.range + 1) * 0.5, 0, 1)
          runner.mesh.scale.setScalar(0.76 + depth * 0.34)
          runner.depthCue.material.opacity = 0.54 + depth * 0.46
        }
      })
      flowDashes.forEach(dash => {
        const start = (dash.phase + now * dash.speed) % 1
        const end = Math.min(1, start + dash.length)
        const from = dash.curve.getPointAt(start)
        const to = dash.curve.getPointAt(end)
        const positions = dash.geometry.getAttribute('position') as THREE.BufferAttribute
        positions.setXYZ(0, from.x, from.y, from.z)
        positions.setXYZ(1, to.x, to.y, to.z)
        positions.needsUpdate = true
      })
      headerFlights.forEach(flight => {
        const cycle = (now * 0.0001666667) % 1
        const upperIndex = Math.max(1, flight.keyframes.findIndex(keyframe => keyframe.t >= cycle))
        const from = flight.keyframes[upperIndex - 1]!
        const to = flight.keyframes[Math.min(upperIndex, flight.keyframes.length - 1)]!
        const mix = THREE.MathUtils.clamp((cycle - from.t) / Math.max(0.0001, to.t - from.t), 0, 1)
        flight.mesh.position.x = THREE.MathUtils.lerp(from.x, to.x, mix)
        flight.mesh.position.y = THREE.MathUtils.lerp(from.y, to.y, mix)
        flight.mesh.rotation.z = THREE.MathUtils.lerp(from.rotation, to.rotation, mix)
        flight.mesh.scale.setScalar(THREE.MathUtils.lerp(from.scale, to.scale, mix))
        flight.exhaust.forEach((trail, index) => {
          const isInDashedPhase = cycle > 0.8
          const isOnDashedSide = flight.mesh.position.x > xFromSvg(1122) + 0.002
          const showTrail = isInDashedPhase && isOnDashedSide
          trail.visible = showTrail
          const material = trail.material as THREE.LineBasicMaterial
          const flicker = 0.68 + Math.sin(now * 0.019 + index * 1.8) * 0.2
          material.opacity = showTrail ? flicker : 0
        })
      })
      tracers.forEach(tracer => {
        const head = (tracer.phase + now * tracer.speed) % 1
        const positions = tracer.geometry.getAttribute('position') as THREE.BufferAttribute
        for (let index = 0; index < tracer.count; index++) {
          const amount = index / Math.max(1, tracer.count - 1)
          const travel = Math.max(0, head - tracer.length + tracer.length * amount)
          const point = tracer.curve.getPointAt(travel)
          positions.setXYZ(index, point.x, point.y, point.z)
        }
        positions.needsUpdate = true
      })
      rotors.forEach(rotor => { rotor.object.rotation.z += rotor.speed * dt })
      renderer.render(scene, camera)
      frame = requestAnimationFrame(animate)
    }
    frame = requestAnimationFrame(animate)
    ready.value = true
  } catch (error) {
    console.error(error)
    failed.value = true
    ready.value = true
  }
}

function findLetterAt(clientX: number, clientY: number) {
  if (!camera || !canvas.value) return -1
  const bounds = canvas.value.getBoundingClientRect()
  screenPointer.set(
    ((clientX - bounds.left) / bounds.width) * 2 - 1,
    -((clientY - bounds.top) / bounds.height) * 2 + 1,
  )
  raycaster.setFromCamera(screenPointer, camera)
  const hit = raycaster.intersectObjects(letterHitTargets, false)[0]
  return hit ? Number(hit.object.userData.letterIndex ?? -1) : -1
}

function focusLetter(stage: number, smoothClick = false) {
  const destination = THREE.MathUtils.clamp(Math.round(stage), 0, 5)
  scrollTarget = destination
  lastWheelAt = 0
  const distance = Math.abs(destination - scrollPosition)
  navigationTween = smoothClick && distance > 0.001
    ? {
        from: scrollPosition,
        to: destination,
        startedAt: performance.now(),
        duration: 520 + Math.min(distance, 4) * 140,
      }
    : null
}

function onPointerMove(event: PointerEvent) {
  if (event.pointerType === 'touch') {
    if (event.pointerId !== touchPointerId) return
    const deltaX = event.clientX - touchStartX
    const deltaY = event.clientY - touchStartY
    if (!touchAxis && Math.max(Math.abs(deltaX), Math.abs(deltaY)) >= 7) {
      touchAxis = Math.abs(deltaX) > Math.abs(deltaY) ? 'x' : 'y'
      touchDragging = true
      activeMenuIndex.value = null
    }
    if (!touchAxis) return
    event.preventDefault()
    const travel = touchAxis === 'x' ? deltaX : deltaY
    const travelPerStage = touchAxis === 'x'
      ? Math.max(115, Math.min(170, innerWidth * 0.38))
      : Math.max(130, Math.min(190, innerHeight * 0.22))
    const draggedStage = touchStartScroll - travel / travelPerStage
    scrollTarget = THREE.MathUtils.clamp(draggedStage, 0, 5)
    scrollPosition = scrollTarget
    lastWheelAt = 0
    suppressClickUntil = performance.now() + 450
    return
  }
  ;(event.currentTarget as HTMLCanvasElement).style.cursor = findLetterAt(event.clientX, event.clientY) >= 0 ? 'pointer' : 'default'
}

function onPointerLeave(event: PointerEvent) {
  if (event.pointerType === 'touch') return
  ;(event.currentTarget as HTMLCanvasElement).style.cursor = 'default'
}

function onCanvasClick(event: MouseEvent) {
  if (performance.now() < suppressClickUntil) return
  const letterIndex = findLetterAt(event.clientX, event.clientY)
  if (letterIndex >= 0) {
    activeMenuIndex.value = letterIndex
    focusLetter(letterIndex + 1, true)
  } else {
    activeMenuIndex.value = null
  }
}

function closeLetterMenu() {
  activeMenuIndex.value = null
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') closeLetterMenu()
}

function onPointerDown(event: PointerEvent) {
  if (event.pointerType !== 'touch') return
  navigationTween = null
  touchPointerId = event.pointerId
  touchStartX = event.clientX
  touchStartY = event.clientY
  touchStartScroll = scrollPosition
  touchStartTarget = scrollTarget
  touchStartStage = Math.round(scrollPosition)
  touchAxis = null
  touchDragging = false
  ;(event.currentTarget as HTMLCanvasElement).setPointerCapture(event.pointerId)
}

function onPointerUp(event: PointerEvent) {
  if (event.pointerType !== 'touch' || event.pointerId !== touchPointerId) return
  const deltaX = event.clientX - touchStartX
  const deltaY = event.clientY - touchStartY
  const travel = touchAxis === 'x' ? deltaX : deltaY
  if (touchDragging && Math.abs(travel) >= 42) {
    let destination = Math.round(scrollTarget)
    if (destination === touchStartStage) destination += travel < 0 ? 1 : -1
    focusLetter(destination)
    suppressClickUntil = performance.now() + 450
  } else if (touchDragging) {
    focusLetter(touchStartStage)
    suppressClickUntil = performance.now() + 450
  } else {
    scrollTarget = touchStartTarget
  }
  const target = event.currentTarget as HTMLCanvasElement
  if (target.hasPointerCapture(event.pointerId)) target.releasePointerCapture(event.pointerId)
  touchPointerId = null
  touchAxis = null
  touchDragging = false
}

function onPointerCancel(event: PointerEvent) {
  if (event.pointerId !== touchPointerId) return
  if (touchDragging) focusLetter(Math.round(scrollPosition))
  else scrollTarget = touchStartTarget
  touchPointerId = null
  touchAxis = null
  touchDragging = false
}

function onWheel(event: WheelEvent) {
  navigationTween = null
  activeMenuIndex.value = null
  const pixels = event.deltaMode === WheelEvent.DOM_DELTA_LINE
    ? event.deltaY * 16
    : event.deltaMode === WheelEvent.DOM_DELTA_PAGE
      ? event.deltaY * innerHeight
      : event.deltaY
  const impulse = THREE.MathUtils.clamp(pixels * 0.006, -0.85, 0.85)
  scrollTarget = THREE.MathUtils.clamp(scrollTarget + impulse, 0, 5)
  lastWheelAt = performance.now()
}

onMounted(() => {
  window.addEventListener('keydown', onKeydown)
  build()
})
onBeforeUnmount(() => {
  cancelAnimationFrame(frame)
  observer?.disconnect()
  window.removeEventListener('keydown', onKeydown)
  resources.forEach(resource => resource.dispose())
  blobUrls.forEach(url => URL.revokeObjectURL(url))
  renderer?.dispose()
})
</script>

<template>
  <main class="logo-stage" :class="{ ready, failed, night: nightTheme }">
    <canvas
      ref="canvas"
      aria-label="LOOLC 三维动态标志"
      @click="onCanvasClick"
      @pointerdown="onPointerDown"
      @pointermove="onPointerMove"
      @pointerup="onPointerUp"
      @pointercancel="onPointerCancel"
      @pointerleave="onPointerLeave"
      @wheel.prevent="onWheel"
    />

    <Transition name="letter-menu">
      <nav
        v-if="activeLetterMenu"
        class="letter-submenu"
        :class="`letter-submenu--${activeLetterMenu.accent}`"
        :aria-label="`${activeLetterMenu.title}功能入口`"
      >
        <span class="letter-submenu__accent" aria-hidden="true" />
        <div class="letter-submenu__identity">
          <span>{{ String((activeMenuIndex ?? 0) + 1).padStart(2, '0') }}</span>
          <strong>{{ activeLetterMenu.letter }}</strong>
          <small>{{ activeLetterMenu.title }}</small>
        </div>

        <div class="letter-submenu__entries">
          <a
            v-for="entry in activeLetterMenu.entries"
            :key="entry.href"
            class="letter-submenu__link"
            :href="entry.href"
            :target="entry.external ? '_blank' : undefined"
            :rel="entry.external ? 'noopener noreferrer' : undefined"
          >
            <span>{{ entry.label }}</span>
            <small>{{ entry.external ? '访问' : '进入' }}</small>
            <i aria-hidden="true">↗</i>
          </a>
          <span v-if="activeLetterMenu.entries.length === 0" class="letter-submenu__reserved">
            入口预留
          </span>
        </div>

        <button class="letter-submenu__close" type="button" aria-label="关闭菜单" @click="closeLetterMenu">
          ×
        </button>
      </nav>
    </Transition>

    <section v-if="failed" class="fallback">
      <strong>loolc<span>.com</span></strong>
      <p>三维标志暂时无法载入</p>
    </section>

    <Transition name="fade">
      <div v-if="!ready" class="loader" aria-live="polite">
        <strong>loolc<span>.com</span></strong>
        <i />
      </div>
    </Transition>
  </main>
</template>
