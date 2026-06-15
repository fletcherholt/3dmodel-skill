# Real-time 3D on the web (Three.js)

Reusable, generic patterns. Import Three.js + addons via an importmap from a CDN (jsDelivr/unpkg). No build step needed.

```html
<script type="importmap">
{ "imports": {
  "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
  "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
}}
</script>
```

## Scene skeleton with realistic defaults

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';

const renderer = new THREE.WebGLRenderer({ antialias:true, alpha:true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

const scene = new THREE.Scene();
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture; // reflections

const camera = new THREE.PerspectiveCamera(32, innerWidth/innerHeight, 0.1, 80);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; controls.dampingFactor = 0.08;
controls.minDistance = 4; controls.maxDistance = 30;

const key = new THREE.DirectionalLight(0xfff1dd, 2.1); key.position.set(4,7,7); scene.add(key);
key.castShadow = true; key.shadow.mapSize.set(2048,2048);
key.shadow.radius = 11; key.shadow.blurSamples = 24; key.shadow.normalBias = 0.02;
Object.assign(key.shadow.camera, {near:1, far:46, left:-9, right:9, top:6, bottom:-6});
scene.add(new THREE.DirectionalLight(0xffffff, 0.7).translateX(-6));
scene.add(new THREE.AmbientLight(0xffffff, 0.4));

// soft contact shadow that grounds the object
const ground = new THREE.Mesh(new THREE.PlaneGeometry(50,28),
  new THREE.ShadowMaterial({ opacity:0.16, transparent:true }));
ground.rotation.x = -Math.PI/2; ground.position.y = -1.05; ground.receiveShadow = true; scene.add(ground);

addEventListener('resize', () => { camera.aspect = innerWidth/innerHeight;
  camera.updateProjectionMatrix(); renderer.setSize(innerWidth, innerHeight); });
```

Set `castShadow`/`receiveShadow` on meshes: `obj.traverse(o => { if (o.isMesh){ o.castShadow = true; o.receiveShadow = true; } });` (turn `castShadow` off on additive/glow planes).

## Spherical camera fly (arcs, never through the object)

```js
const SPH = new THREE.Spherical(), tmp = new THREE.Vector3();
let from=null, to=null, t=1;
function sph(camPos, tgt){ return { s:new THREE.Spherical().setFromVector3(camPos.clone().sub(tgt)), tgt:tgt.clone() }; }
function flyTo(camVec, tgtVec){
  from = sph(camera.position, controls.target);
  to = sph(camVec, tgtVec);
  let d = to.s.theta - from.s.theta;            // shortest way round
  while (d >  Math.PI) d -= 2*Math.PI;
  while (d < -Math.PI) d += 2*Math.PI;
  to.s.theta = from.s.theta + d; t = 0;
}
const ease = x => x<0.5 ? 2*x*x : 1 - Math.pow(-2*x+2,2)/2;
// in the loop:
function updateFly(dt){
  controls.enabled = t >= 1;                      // don't fight the fly
  if (t < 1){ t = Math.min(1, t + dt*1.25); const k = ease(t);
    SPH.radius = THREE.MathUtils.lerp(from.s.radius, to.s.radius, k);
    SPH.phi    = THREE.MathUtils.lerp(from.s.phi,    to.s.phi,    k);
    SPH.theta  = THREE.MathUtils.lerp(from.s.theta,  to.s.theta,  k);
    const tg = from.tgt.clone().lerp(to.tgt, k);
    camera.position.setFromSpherical(SPH).add(tg); controls.target.copy(tg);
  }
}
```

## X-ray reveal (shell fades, wireframe stays, internals show)

Keep shells fully OPAQUE when not revealing (clean look, no sort glitches). Only switch to transparent + `depthWrite=false` during the fade, and fully hide when invisible. Add an `EdgesGeometry` outline that fades in.

```js
const edges = new THREE.LineSegments(new THREE.EdgesGeometry(shell.geometry, 40),
  new THREE.LineBasicMaterial({ color:0x4a4a50, transparent:true, opacity:0 }));
// per frame, g = 0..1 reveal amount:
const solid = g < 0.002;
for (const m of [shell.material, glass.material]) { m.transparent = !solid; m.depthWrite = solid; }
shell.material.opacity = solid ? 1 : 1 - g;
shell.visible = g < 0.985;
edges.material.opacity = 0.55 * g;
```
Light internals that face into a cavity with their own light from that side, or they read as a dark void.

## Callouts: fixed lanes + leader lines (stable, no overlap)

```js
// HTML: one absolutely-positioned .label div per part + a full-screen <svg id="wires">
const NS = 'http://www.w3.org/2000/svg', svg = document.getElementById('wires');
const mkLine = (c,w) => { const l=document.createElementNS(NS,'line');
  l.setAttribute('stroke',c); l.setAttribute('stroke-width',w); l.setAttribute('stroke-linecap','round'); svg.appendChild(l); return l; };
// PARTS[k] = { obj, el, lane:'L'|'R', row:0..2, lead, dot }
function layoutCallouts(W, H){
  const laneX = { L:30, R:W-242 }, rowY = [H*0.27, H*0.44, H*0.61];
  for (const k in PARTS){ const p = PARTS[k];
    p.obj.getWorldPosition(tmp); tmp.project(camera);
    const x = (tmp.x*0.5+0.5)*W, y = (-tmp.y*0.5+0.5)*H, front = tmp.z < 1;
    const lx = laneX[p.lane], ly = rowY[p.row];
    p.el.style.left = lx+'px'; p.el.style.top = ly+'px';
    p.el.style.opacity = front ? 1 : 0;
    const ex = p.lane==='L' ? lx+212 : lx, ey = ly+20;          // leader from label edge to part
    p.lead.setAttribute('x1',ex); p.lead.setAttribute('y1',ey);
    p.lead.setAttribute('x2',x);  p.lead.setAttribute('y2',y);
    p.dot.setAttribute('cx',x);   p.dot.setAttribute('cy',y);
    p._x=x; p._y=y; p._front=front;
  }
}
```

## Flow lines (what connects to what, colour-coded)

```js
// FLOWS = [{ a:'partKey', b:'partKey', c:'#d98a2b', t:'5V' }, ...]
for (const f of FLOWS){ const A=PARTS[f.a], B=PARTS[f.b], ok = A._front && B._front;
  f.line.setAttribute('opacity', ok?0.9:0);
  f.line.setAttribute('x1',A._x); f.line.setAttribute('y1',A._y);
  f.line.setAttribute('x2',B._x); f.line.setAttribute('y2',B._y);
  f.lbl.setAttribute('opacity', ok?1:0);
  f.lbl.setAttribute('x',(A._x+B._x)/2); f.lbl.setAttribute('y',(A._y+B._y)/2-5);
}
```
Add a small legend mapping colours to categories (power / data / signal). Give SVG `<text>` a background-coloured stroke with `paint-order:stroke` so it stays legible over busy geometry.

Keep the two line types visually distinct or they blur together: draw **leader lines** thin, faint and dashed (they are just label-to-part pointers), and draw **flow lines** bold and solid with a dark casing line underneath for contrast against busy/coloured geometry, plus **animated marching dashes** (decrement `stroke-dashoffset` each frame) so they read as flow and show direction. Different categories get different hues; leaders stay neutral.

## Hide the overlay while moving (the anti-glitch rule)

```js
const lastCam = new THREE.Vector3(), lastTgt = new THREE.Vector3(); let lastMoveT = 0;
// per frame (now = ms):
const moved = camera.position.distanceToSquared(lastCam) > 1e-5
           || controls.target.distanceToSquared(lastTgt) > 1e-5;
if (moved || flying) lastMoveT = now;
const still = (now - lastMoveT) > 200;
overlay.style.opacity = (revealed && still) ? 1 : 0;   // CSS transition does the fade
// ... update line/label positions every frame anyway so they are correct when shown ...
lastCam.copy(camera.position); lastTgt.copy(controls.target);
```

## Click vs drag

```js
let dx, dy, dt;
dom.addEventListener('pointerdown', e => { dx=e.clientX; dy=e.clientY; dt=performance.now(); });
dom.addEventListener('pointerup', e => {
  if (Math.hypot(e.clientX-dx, e.clientY-dy) > 6 || performance.now()-dt > 320) return; // a drag
  raycaster.setFromCamera({ x:(e.clientX/innerWidth)*2-1, y:-(e.clientY/innerHeight)*2+1 }, camera);
  if (raycaster.intersectObject(model, true).length) toggleReveal();
});
```

## Frame-swap displays (LED matrices, screens)

glTF cannot animate textures. Either generate N emissive textures and toggle which mesh/plane is visible on a JS timer, or build a real LED matrix as an `InstancedMesh` of dots plus an additive-blended glow `InstancedMesh`, and set per-instance colour/scale to draw glyphs.

## Keynote-grade pipeline: cinematic stage + post-processing

```js
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { GTAOPass } from 'three/addons/postprocessing/GTAOPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
import { SMAAPass } from 'three/addons/postprocessing/SMAAPass.js';

// studio stage: gradient backdrop, glossy floor, softbox lights
const bg = document.createElement('canvas'); bg.width=16; bg.height=512;
{ const g=bg.getContext('2d'); const grd=g.createLinearGradient(0,0,0,512);
  grd.addColorStop(0,'#181b21'); grd.addColorStop(0.5,'#0e1014'); grd.addColorStop(1,'#08090b');
  g.fillStyle=grd; g.fillRect(0,0,16,512); }
const bt=new THREE.CanvasTexture(bg); bt.colorSpace=THREE.SRGBColorSpace; scene.background=bt;

const floor = new THREE.Mesh(new THREE.PlaneGeometry(120,80),
  new THREE.MeshStandardMaterial({ color:0x0a0b0d, roughness:0.38, metalness:0.0 }));
floor.rotation.x = -Math.PI/2; floor.position.y = -1.06; floor.receiveShadow = true; scene.add(floor);

RectAreaLightUniformsLib.init();
const s1 = new THREE.RectAreaLight(0xffffff, 3.4, 11, 6); s1.position.set(-5,6,8); s1.lookAt(0,0,0); scene.add(s1);
const s2 = new THREE.RectAreaLight(0xbcd2ff, 2.2, 8, 5); s2.position.set(7,3,4); s2.lookAt(0,0,0); scene.add(s2);

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
const gtao = new GTAOPass(scene, camera, innerWidth, innerHeight);
gtao.updateGtaoMaterial({ radius:0.5, distanceExponent:1.0, thickness:1.0, scale:1.0, samples:16 });
composer.addPass(gtao);
composer.addPass(new UnrealBloomPass(new THREE.Vector2(innerWidth, innerHeight), 0.42, 0.5, 0.82));
composer.addPass(new OutputPass());
composer.addPass(new SMAAPass(innerWidth*renderer.getPixelRatio(), innerHeight*renderer.getPixelRatio()));
// render loop: composer.render();  resize: composer.setSize(innerWidth, innerHeight);
```

Idle auto-rotate that pauses on user input (track input time, not camera movement, or it fights itself):
```js
let lastInput = -9999; const bump = () => lastInput = performance.now();
for (const ev of ['pointerdown','pointermove','wheel']) dom.addEventListener(ev, bump, {passive:true});
controls.autoRotateSpeed = 0.5;
// per frame: controls.autoRotate = resting && (performance.now() - lastInput > 2600);
```
Add a CSS radial-gradient vignette div over the canvas for cinematic framing, and fade the UI chrome up ~1s after load.

## Verifying with a headless browser

Serve the folder (`python3 -m http.server`), drive it with Playwright: navigate, wait, screenshot, exercise each state (closed, mid-transition, open, dragging) and Read each screenshot. Most 3D bugs only show in transitions.
