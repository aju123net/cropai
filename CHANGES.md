# KrishiSaarthi AI — All Code Changes Log

> This file tracks every code change made to the project, with before/after
> comparisons and reasons. Keep it updated for future reference.

---

## Table of Contents

1. [Image-First Simplified Advisory Flow](#1-image-first-simplified-advisory-flow)
2. [Fix: RAG Retrieval Prioritizes Same-Crop Guides](#2-fix-rag-retrieval-prioritizes-same-crop-guides)
3. [Fix: Detected Crop Shown in Image Observations](#3-fix-detected-crop-shown-in-image-observations)
4. [Fix: Deprecated Vision Model Replaced](#4-fix-deprecated-vision-model-replaced)
5. [Fix: Smart Crop Fallback from Symptoms Text](#5-fix-smart-crop-fallback-from-symptoms-text)
6. [Fix: Input Guardrail Skipped When Image Uploaded](#6-fix-input-guardrail-skipped-when-image-uploaded)

---

## 1. Image-First Simplified Advisory Flow

**Goal:** Farmers don't know technical terms. Reduce 10+ input fields to just
"upload a photo and get advice."

### 1a. `be-MVP5/app/schemas.py` — Make crop & symptoms optional

**Why:** Farmer should only need to upload a photo. Crop is auto-detected from
the image, symptoms are optional.

```python
# ─── BEFORE ───────────────────────────────────────────────
class AdvisoryCreate(BaseModel):
    farmer_name: str = "Farmer"
    crop: str                          # REQUIRED
    variety: str = ""
    growth_stage: str = "Vegetative"
    location: str = "Kolkata, West Bengal"
    region: str = "West Bengal"
    language: str = "English"
    soil_type: str = ""
    previous_fertilizer: str = ""
    previous_pesticide: str = ""
    symptoms: str = Field(min_length=5, max_length=4000)  # MUST be 5+ chars

# ─── AFTER ────────────────────────────────────────────────
class AdvisoryCreate(BaseModel):
    farmer_name: str = "Farmer"
    crop: str = ""                     # OPTIONAL — auto-detected from image
    variety: str = ""
    growth_stage: str = ""             # Empty = auto-set from catalog
    location: str = "Kolkata, West Bengal"
    region: str = "West Bengal"
    language: str = "English"
    soil_type: str = ""
    previous_fertilizer: str = ""
    previous_pesticide: str = ""
    symptoms: str = Field(default="", max_length=4000)  # Optional
```

### 1b. `be-MVP5/app/models.py` — Database columns nullable

**Why:** DB schema must match API. If crop is optional in API, DB must allow NULL.

```python
# ─── BEFORE ───────────────────────────────────────────────
class AdvisoryCase(Base):
    crop = Column(String(80), index=True, nullable=False)       # NOT NULL
    symptoms = Column(Text, nullable=False)                      # NOT NULL

# ─── AFTER ────────────────────────────────────────────────
class AdvisoryCase(Base):
    crop = Column(String(80), index=True, nullable=True, default="")
    symptoms = Column(Text, nullable=True, default="")
```

### 1c. `fe-MVP5/src/pages/NewAdvisory.jsx` — Complete rewrite

**Why:** Old form had 10+ fields (crop, variety, growth stage, location, region,
language, soil type, fertilizer, pesticide, symptoms). New design: upload photo →
optional description → get advice.

**BEFORE (key parts):**
```jsx
const initial = {
  farmer_name: 'Ramesh Farmer',
  crop: 'Tomato',              // Must know crop name
  variety: '',                  // What variety?
  growth_stage: 'Fruiting',    // Technical term
  soil_type: 'Alluvial loam',  // Technical term
  previous_fertilizer: '',     // Farmer may not know
  previous_pesticide: '',      // Farmer may not know
  symptoms: '',                // Required 10+ chars
};
// Multi-step form with sections 1, 2, 3
```

**AFTER (complete rewrite):**
```jsx
export default function NewAdvisory() {
  const [image, setImage] = useState(null);
  const [imagePreview, setImagePreview] = useState(null);
  const [symptoms, setSymptoms] = useState('');       // Optional
  const [crop, setCrop] = useState('');                // Optional dropdown
  const [location, setLocation] = useState('Kolkata'); // Optional
  const [language, setLanguage] = useState('English');  // Optional

  const submit = async (event) => {
    // 1. Create advisory with minimal data
    const { data: created } = await api.post('/advisories', {
      crop: crop || '',        // Empty → auto-detected from image
      symptoms: symptoms || '',// Empty → image carries info
      has_image: true,         // Tells backend: relax guardrail
    });
    // 2. Upload image
    await api.post(`/advisories/${created.id}/image`, imageData);
    // 3. Run analysis (crop auto-detected at this point)
    await api.post(`/advisories/${created.id}/analyze`);
  };

  return (
    // BIG photo upload area (primary action)
    // Optional crop dropdown
    // Optional symptoms textarea with voice input
    // Optional location + language
    // "Get Advice" button
  );
}
```

### 1d. `fe-MVP5/src/pages/FarmerDashboard.jsx` — Updated hero text

**Why:** Align landing page message with simpler flow.

```python
# ─── BEFORE ───────────────────────────────────────────────
<h1>Protect your crop with faster, explainable AI guidance.</h1>
<p>Upload a crop image, describe the symptoms, and receive a grounded advisory
   using vision analysis, disease rules, agricultural knowledge, weather,
   and soil context.</p>
<Link to="/new-advisory">Start new advisory</Link>

# ─── AFTER ────────────────────────────────────────────────
<h1>Protect your crop with a single photo.</h1>
<p>Snap or upload a photo of your crop. Our AI detects the crop type,
   identifies possible diseases, and gives you grounded, actionable guidance
   — no technical knowledge needed.</p>
<Link to="/new-advisory">Get crop advice</Link>
```

### 1e. `fe-MVP5/src/styles.css` — New CSS for upload dropzone

**Why:** Style the new image-first upload area.

```css
/* ─── NEW ────────────────────────────────────────────────── */
.simple-upload-hero{display:flex;flex-direction:column;align-items:center;gap:14px;margin-bottom:20px}
.upload-dropzone{width:100%;max-width:420px;min-height:220px;border:2.5px dashed var(--border);border-radius:var(--radius);display:flex;flex-direction:column;align-items:center;justify-content:center;gap:10px;cursor:pointer;transition:.2s;background:var(--sunken);color:var(--muted);padding:24px;text-align:center}
.upload-dropzone:hover{border-color:var(--primary);background:var(--primary-light);color:var(--primary)}
.upload-dropzone.has-image{border-style:solid;border-color:var(--primary);padding:0;overflow:hidden;background:#fff}
.upload-preview{width:100%;max-height:280px;object-fit:cover;display:block}
.simple-fields{display:flex;flex-direction:column;gap:14px;margin-bottom:18px}
.form-grid-2{display:grid;grid-template-columns:1fr 1fr;gap:14px}
.analysis-status{display:flex;align-items:center;gap:8px;padding:12px 14px;background:#eff6ff;border:1px solid #bfdbfe;border-radius:10px;color:var(--info);font-weight:600;font-size:13px;margin-bottom:14px}
```

---

## 2. Fix: RAG Retrieval Prioritizes Same-Crop Guides

**Problem:** Potato Late Blight query returned "Tomato Late Blight Guide"
because TF-IDF matched the disease name without penalizing cross-crop results.

**Why:** The disease "Late Blight" exists for both Potato and Tomato. Without
crop-aware scoring, the retrieval picks whichever document has higher text
similarity, which happened to be the Tomato guide.

### `be-MVP5/app/services.py` — `_retrieve_with_tfidf()`

```python
# ─── BEFORE ───────────────────────────────────────────────
ranked = sorted(
    zip(chunks, scores),
    key=lambda item: item[1],
    reverse=True,
)[:top_k]

# ─── AFTER ────────────────────────────────────────────────
crop_lower = (crop or "").strip().lower()
boosted = []
for idx, (chunk, score) in enumerate(zip(chunks, scores)):
    doc_crop = (chunk.document.crop or "").strip().lower()
    boosted_score = float(score)
    if crop_lower and doc_crop == crop_lower:
        boosted_score += 0.5          # Same crop gets big boost
    elif doc_crop == "all":
        boosted_score += 0.1          # "All crops" gets small boost
    if disease and doc_disease == disease.lower():
        boosted_score += 0.2          # Exact disease match bump
    boosted.append((chunk, boosted_score))

ranked = sorted(boosted, key=lambda item: item[1], reverse=True)[:top_k]
```

### `be-MVP5/app/services.py` — `_retrieve_with_embeddings()`

Same crop-priority re-ranking added after Chroma retrieval:

```python
# ─── NEW (after Chroma retrieval) ─────────────────────────
crop_lower = (crop or "").strip().lower()
def _crop_priority(r: dict) -> int:
    doc_crop = (r.get("crop") or "").strip().lower()
    if doc_crop == crop_lower: return 0
    if doc_crop == "all": return 1
    return 2
results.sort(key=_crop_priority)
return results[:top_k]
```

**Result:** Potato Late Blight now returns:
```
✅ Potato Blight Guide (crop=Potato)
   General Irrigation Safety (crop=All)
   General Pesticide Safety (crop=All)
```

---

## 3. Fix: Detected Crop Shown in Image Observations

**Problem:** Farmer couldn't see what crop the AI detected from the image.

### `be-MVP5/app/services.py` — `analyze_case()`

```python
# ─── BEFORE ───────────────────────────────────────────────
case.image_observations = json.dumps(
    {**observations, "file_metadata": image_meta}, ensure_ascii=False,
)

# ─── AFTER ────────────────────────────────────────────────
case.image_observations = json.dumps(
    {**observations, "detected_crop": case.crop or "Unknown", "file_metadata": image_meta},
    ensure_ascii=False,
)
```

### `fe-MVP5/src/pages/AdvisoryResult.jsx`

```jsx
// ─── NEW (in Context section) ─────────────────────────────
{data.image_observations?.detected_crop && (
  <div className="detected-crop-badge">
    <b>Detected crop:</b> {data.image_observations.detected_crop}
  </div>
)}
```

### `fe-MVP5/src/styles.css`

```css
/* ─── NEW ────────────────────────────────────────────────── */
.detected-crop-badge{display:inline-flex;align-items:center;gap:6px;padding:6px 12px;margin-bottom:10px;background:var(--primary-light);border:1px solid #86efac;border-radius:8px;color:var(--primary-dark);font-size:13px;font-weight:600}
.source-mismatch{color:var(--warning);font-weight:600}
```

---

## 4. Fix: Deprecated Vision Model Replaced

**Problem:** The old vision model `Llama-3.2-90B-Vision-Instruct` was
deprecated (410 error). All image analysis failed silently.

### `be-MVP5/.env`

```
# ─── BEFORE ───────────────────────────────────────────────
VISION_MODEL=azure_ai/genailab-maas-Llama-3.2-90B-Vision-Instruct

# ─── AFTER ────────────────────────────────────────────────
VISION_MODEL=genailab-maas-gpt-4o
```

---

## 5. Fix: Smart Crop Fallback from Symptoms Text

**Problem:** When LLM vision fails, crop defaulted to "Tomato" regardless of
what the farmer actually grows.

**Why:** Need to try detecting crop from the farmer's text description before
falling back to a hardcoded default.

### `be-MVP5/app/services.py` — `analyze_case()`

```python
# ─── BEFORE ───────────────────────────────────────────────
# If crop still empty after auto-detect, use a safe default
if not case.crop:
    case.crop = "Tomato"
    if not case.growth_stage:
        case.growth_stage = "Vegetative"
    db.commit()

# ─── AFTER ────────────────────────────────────────────────
if not case.crop:
    from .catalog import CROP_CATALOG
    symptoms_lower = (case.symptoms or "").lower()
    inferred_crop = ""
    # Check if crop name appears in symptoms
    for crop_name in CROP_CATALOG:
        if crop_name.lower() in symptoms_lower:
            inferred_crop = crop_name
            break
    # Also check Hindi/Bengali aliases
    if not inferred_crop:
        aliases = {"paddy": "Rice", "aloo": "Potato", "tamatar": "Tomato",
                   "gobhi": "Wheat", "sarson": "Mustard", "lanka": "Chilli"}
        for alias, crop in aliases.items():
            if alias in symptoms_lower:
                inferred_crop = crop
                break
    case.crop = inferred_crop or "Tomato"  # Only defaults if nothing matches
    if not case.growth_stage:
        stages = CROP_CATALOG.get(case.crop, [])
        case.growth_stage = stages[1] if len(stages) > 1 else (stages[0] if stages else "Vegetative")
    db.commit()
```

**Result:**
- "Potato leaves have spots" → detects **Potato**
- "Rice paddy yellow leaves" → detects **Rice**
- "leaves are brown" → defaults to **Tomato** (no crop name found)

---

## 6. Fix: Input Guardrail Skipped When Image Uploaded

**Problem:** When farmer uploads image and clicks "Get Advice" with empty
symptoms, the guardrail rejected it:
```
"Symptom field must describe a visible crop/plant problem, not a request to analyze an image."
```

**Why:** The guardrail validates symptom text even when an image is present.
The image carries diagnostic info, so symptoms should be fully optional.

### 6a. `be-MVP5/app/schemas.py` — Added `has_image` flag

```python
# ─── NEW ──────────────────────────────────────────────────
class AdvisoryCreate(BaseModel):
    ...
    symptoms: str = Field(default="", max_length=4000)
    has_image: bool = False  # Frontend tells backend: image will follow
```

### 6b. `be-MVP5/app/services.py` — Skip guardrail when image present

```python
# ─── NEW in _hard_input_guardrail_errors() ────────────────
def _hard_input_guardrail_errors(values, has_image=False):
    errors = {}
    symptoms = values.get("symptoms", "")
    if not has_image and len(symptoms) < 10:  # Only enforce when NO image
        errors["symptoms"] = "..."

# ─── NEW in _deterministic_input_guardrail() ──────────────
def _deterministic_input_guardrail(values, crop, has_image=False):
    errors = _hard_input_guardrail_errors(values, has_image)
    if has_image:
        errors.pop("symptoms", None)  # Skip all symptom checks
        return errors

# ─── NEW in validate_advisory_input() ─────────────────────
if has_image:
    mode = "image-upload-simplified"
    model_used = "none"
    # Skip entire LLM guardrail
```

### 6c. `be-MVP5/app/main.py` — Filter has_image before DB insert

```python
# ─── BEFORE ───────────────────────────────────────────────
case = AdvisoryCase(owner_username=context["username"], **validation["cleaned"])

# ─── AFTER ────────────────────────────────────────────────
cleaned = {k: v for k, v in validation["cleaned"].items() if k != "has_image"}
case = AdvisoryCase(owner_username=context["username"], **cleaned)
```

### 6d. `fe-MVP5/src/pages/NewAdvisory.jsx` — Send has_image flag

```jsx
// ─── BEFORE ───────────────────────────────────────────────
const { data: created } = await api.post('/advisories', {
  symptoms: symptoms || 'Please analyze the crop in the uploaded image.',
});

// ─── AFTER ────────────────────────────────────────────────
const { data: created } = await api.post('/advisories', {
  crop: crop || '',
  symptoms: symptoms || '',
  has_image: true,  // Tells backend: relax guardrail
});
```

### 6e. `be-MVP5/app/services.py` — RAG retrieval improved

```python
# ─── BEFORE ───────────────────────────────────────────────
query = (
    f"{case.crop} {match['disease']} {case.growth_stage} "
    f"{case.symptoms} {json.dumps(observations, ensure_ascii=False)}"
)

# ─── AFTER ────────────────────────────────────────────────
observation_summary = observations.get("observations_summary", "")
query = (
    f"{case.crop} {match['disease']} {case.growth_stage} "
    f"{case.symptoms} {observation_summary}"  # Clean text, not JSON
)
```

### 6f. `be-MVP5/app/services.py` — Advisory prompt strengthened

```python
# ─── BEFORE ───────────────────────────────────────────────
"Return Markdown with possible condition, confidence, immediate actions, "
"irrigation, prevention, and safety note."

# ─── AFTER ────────────────────────────────────────────────
"Using ONLY the retrieved agricultural context above, generate a grounded "
"farmer-friendly advisory. Do not invent facts, dosages, or treatments "
"not present in the retrieved context. Return Markdown with possible condition, "
"confidence, immediate actions, irrigation, prevention, and safety note."
```

---

## Summary: Before vs After

| Aspect | Before | After |
|--------|--------|-------|
| **Fields farmer fills** | 10+ (crop, variety, stage, location, region, language, soil, fertilizer, pesticide, symptoms) | 0 required (photo + optional crop/description) |
| **Crop identification** | Farmer must select from dropdown | Auto-detected from image by vision model |
| **Symptoms** | Required, minimum 10 characters | Optional — image carries information |
| **Input guardrail** | Always validates symptoms | Skipped when image is uploaded |
| **RAG retrieval** | Returns wrong crop's guide for same disease name | Prioritizes same-crop guides |
| **Vision model** | Llama-3.2-90B (deprecated) | GPT-4o (working) |
| **Crop fallback** | Always defaults to "Tomato" | Infers from symptoms text first |
| **Image observations** | No crop info shown | Shows "Detected crop: Potato" |
| **Grounding** | Generic LLM prompt | Explicit "use ONLY retrieved context" |
