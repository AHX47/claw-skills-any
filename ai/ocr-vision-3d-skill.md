# OCR, Vision & 3D Skills

## OCR — Text Extraction from Images
```python
import pytesseract
from PIL import Image, ImageFilter, ImageEnhance
import cv2, numpy as np

def preprocess_for_ocr(img_path: str) -> np.ndarray:
    img  = cv2.imread(img_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    # Denoise
    gray = cv2.fastNlMeansDenoising(gray, h=10)
    # Adaptive threshold (better than global for uneven lighting)
    binary = cv2.adaptiveThreshold(gray, 255,
                                    cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
                                    cv2.THRESH_BINARY, 11, 2)
    # Deskew
    coords = np.column_stack(np.where(binary > 0))
    angle  = cv2.minAreaRect(coords)[-1]
    if angle < -45: angle += 90
    (h, w) = binary.shape
    M = cv2.getRotationMatrix2D((w//2, h//2), angle, 1.0)
    return cv2.warpAffine(binary, M, (w, h), flags=cv2.INTER_CUBIC,
                           borderMode=cv2.BORDER_REPLICATE)

def extract_text(img_path: str, lang="ara+eng") -> str:
    processed = preprocess_for_ocr(img_path)
    config    = "--psm 3 --oem 3"  # auto page seg + LSTM
    return pytesseract.image_to_string(processed, lang=lang, config=config)

def extract_structured(img_path: str) -> list[dict]:
    """Extract text with bounding boxes and confidence."""
    img  = Image.open(img_path)
    data = pytesseract.image_to_data(img, output_type=pytesseract.Output.DICT)
    results = []
    for i, word in enumerate(data["text"]):
        if word.strip() and int(data["conf"][i]) > 30:
            results.append({
                "text": word,
                "confidence": data["conf"][i],
                "x": data["left"][i], "y": data["top"][i],
                "w": data["width"][i], "h": data["height"][i],
            })
    return results
```

## Vision with Claude API
```python
import anthropic, base64
from pathlib import Path

client = anthropic.Anthropic()

def analyze_image(img_path: str, question: str) -> str:
    img_data = base64.standard_b64encode(Path(img_path).read_bytes()).decode()
    ext      = Path(img_path).suffix.lstrip(".").lower()
    media    = {"jpg":"image/jpeg","jpeg":"image/jpeg","png":"image/png","webp":"image/webp"}.get(ext,"image/jpeg")

    resp = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role":"user","content":[
            {"type":"image","source":{"type":"base64","media_type":media,"data":img_data}},
            {"type":"text","text":question},
        ]}])
    return resp.content[0].text

def extract_table_from_image(img_path: str) -> list[dict]:
    """Extract structured table data from an image of a table."""
    result = analyze_image(img_path,
        "Extract the table from this image. Return ONLY a JSON array of objects. "
        "Each row becomes an object with column headers as keys. No markdown.")
    import json, re
    text = re.sub(r"```json|```","", result).strip()
    return json.loads(text)

def ocr_with_vision(img_path: str, lang_hint="Arabic") -> str:
    return analyze_image(img_path,
        f"Extract ALL text from this image exactly as written. "
        f"The document is in {lang_hint}. Preserve formatting with newlines. "
        f"Output ONLY the text, nothing else.")
```

## 3D Frontend Design (Three.js)
```javascript
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { GLTFLoader }    from "three/addons/loaders/GLTFLoader.js";

// Scene setup
const scene    = new THREE.Scene();
scene.background = new THREE.Color(0x0a0e1a);
const camera   = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.5, 5);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

// Controls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; controls.dampingFactor = 0.05;

// Lighting
scene.add(new THREE.AmbientLight(0x404040, 0.6));
const light = new THREE.DirectionalLight(0x00d4ff, 1.2);
light.position.set(5,10,5); light.castShadow = true;
scene.add(light);

// Glowing sphere
const geo    = new THREE.SphereGeometry(1, 64, 64);
const mat    = new THREE.MeshStandardMaterial({
    color: 0x00d4ff, emissive: 0x003344,
    metalness: 0.8, roughness: 0.2,
});
const sphere = new THREE.Mesh(geo, mat);
sphere.castShadow = true;
scene.add(sphere);

// Particle system
const N = 5000;
const pos = new Float32Array(N * 3);
for (let i = 0; i < N*3; i++) pos[i] = (Math.random()-0.5)*50;
const particles = new THREE.Points(
    new THREE.BufferGeometry().setAttribute("position", new THREE.BufferAttribute(pos, 3)),
    new THREE.PointsMaterial({ color: 0x00d4ff, size: 0.05, transparent: true, opacity: 0.6 })
);
scene.add(particles);

// Load GLB model
const loader = new GLTFLoader();
loader.load("./model.glb", gltf => {
    gltf.scene.traverse(child => {
        if (child.isMesh) { child.castShadow=true; child.receiveShadow=true; }
    });
    scene.add(gltf.scene);
});

// Animate
function animate() {
    requestAnimationFrame(animate);
    sphere.rotation.y += 0.005;
    particles.rotation.y += 0.0003;
    controls.update();
    renderer.render(scene, camera);
}
animate();

// Resize
window.addEventListener("resize", () => {
    camera.aspect = window.innerWidth/window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
```

## 3D CSS Effects
```css
/* 3D card flip */
.card-3d { perspective: 1000px; width: 300px; height: 200px; }
.card-3d-inner {
    position: relative; width: 100%; height: 100%;
    transform-style: preserve-3d;
    transition: transform 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}
.card-3d:hover .card-3d-inner { transform: rotateY(180deg); }
.card-3d-front, .card-3d-back {
    position: absolute; inset: 0; backface-visibility: hidden;
    border-radius: 16px; display: flex; align-items: center; justify-content: center;
}
.card-3d-back { transform: rotateY(180deg); background: linear-gradient(135deg, #00d4ff, #0088ff); }

/* Tilt on hover (JavaScript + CSS) */
.tilt { transform-style: preserve-3d; transition: transform 0.1s; }
```

## Document Layout Analysis (LayoutLMv3)
```python
from transformers import LayoutLMv3Processor, LayoutLMv3ForTokenClassification
from PIL import Image
import torch

processor = LayoutLMv3Processor.from_pretrained("microsoft/layoutlmv3-base")
model     = LayoutLMv3ForTokenClassification.from_pretrained("microsoft/layoutlmv3-base")

def analyze_document(img_path: str) -> dict:
    image  = Image.open(img_path).convert("RGB")
    # OCR first to get words + boxes
    import pytesseract
    ocr    = pytesseract.image_to_data(image, output_type=pytesseract.Output.DICT)
    words  = [w for w in ocr["text"] if w.strip()]
    boxes  = [[ocr["left"][i], ocr["top"][i],
               ocr["left"][i]+ocr["width"][i],
               ocr["top"][i]+ocr["height"][i]]
              for i,w in enumerate(ocr["text"]) if w.strip()]

    encoding = processor(image, words, boxes=boxes, return_tensors="pt",
                          truncation=True, padding=True)
    with torch.no_grad():
        outputs  = model(**encoding)
    predictions = outputs.logits.argmax(-1).squeeze().tolist()
    return {"words": words, "labels": predictions}
```
