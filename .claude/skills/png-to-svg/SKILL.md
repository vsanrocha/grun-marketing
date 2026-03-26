---
name: "PNG to SVG Vectorizer"
description: "Vectorize a PNG image into a clean SVG using imagemagick and potrace. Handles transparent backgrounds correctly by extracting the alpha channel, and solid-color backgrounds via luminance threshold. Use this skill whenever the user asks to convert, vectorize, trace, or export any raster image (PNG, JPG, logo) to SVG — even if they say 'I need a vector version', 'make it scalable', 'convert to vector', or 'I need this for print/Figma/Illustrator'. Always use this skill for any image-to-vector conversion task."
---

# PNG to SVG Vectorizer

Converts a raster PNG into a clean, scalable SVG using `imagemagick` and `potrace`. Uses only imagemagick — no Python/PIL required.

## Requirements

Verify tools are installed before proceeding:

```bash
which potrace convert identify || echo "MISSING TOOLS"
# Install if missing:
sudo apt-get install -y potrace imagemagick
```

## Core Workflow

### Step 1 — Detect background type

Use `identify` to check for alpha channel (no PIL needed):

```bash
identify -format "%[channels]" input.png
# Contains "alpha" → transparent background
# Does not contain "alpha" → solid background
```

Or check Type field:

```bash
identify -format "%[type]" input.png
# "TrueColorAlpha" or "GrayscaleAlpha" → has transparency
# "TrueColor", "Grayscale" → solid background
```

### Step 2 — Create bitmap mask

**Transparent background (most logos — Photoroom, cutout images):**

Extract the alpha channel and negate so the logo becomes black (potrace traces dark areas):

```bash
convert input.png -alpha extract -negate mask.pbm
```

Why this works:
- Alpha=255 (opaque logo pixel) → white after extract → black after negate → potrace traces it ✓
- Alpha=0 (transparent background) → black after extract → white after negate → ignored ✓
- Without this, transparent pixels become black and potrace traces the background instead of the logo

**Solid white/colored background:**

Flatten onto white, then threshold. Green at ~44% luminance becomes black:

```bash
convert input.png -background white -flatten -threshold 50% mask.pbm
```

### Step 3 — Vectorize

```bash
potrace mask.pbm -s -o output.svg
```

Useful flags:
- `-O 0.2` — lower tolerance = more accurate curves (default is 1.0)
- `-s` — SVG output

### Step 4 — Sample fill color (imagemagick only, no PIL)

Potrace outputs black paths. Sample the original logo color by finding the center of the non-transparent bounding box:

```bash
# Get bounding box of non-transparent content
TRIM=$(convert input.png -trim -format "%wx%h+%x+%y" info:)
W=$(echo $TRIM | grep -oP '^\d+')
H=$(echo $TRIM | grep -oP 'x\K\d+')
OX=$(echo $TRIM | grep -oP '\+\K\d+' | head -1)
OY=$(echo $TRIM | grep -oP '\+\K\d+' | tail -1)
CX=$((OX + W/2))
CY=$((OY + H/2))

# Sample color at center of logo
COLOR=$(convert input.png -format "%[hex:u.p{${CX},${CY}}]" info: | head -c6)
echo "Detected color: #$COLOR"

# Apply to SVG
sed -i "s/fill=\"#000000\"/fill=\"#${COLOR}\"/g" output.svg
```

> **Why bounding box center?** Sampling at fixed coordinates (e.g., 100,100) frequently hits transparent pixels and returns `00000000`. Using `-trim` to find actual content bounds guarantees we sample inside the logo.

### Step 5 — Verify and cleanup

```bash
# Verify SVG has paths
grep -c "<path" output.svg
echo "Paths found: $(grep -c '<path' output.svg)"

# Cleanup temp file
rm mask.pbm
```

## Full Example (auto-detects background)

```bash
#!/bin/bash
INPUT="$1"
OUTPUT="${INPUT%.png}.svg"
MASK="/tmp/mask_$$.pbm"

# Detect background type
TYPE=$(identify -format "%[type]" "$INPUT")

if [[ "$TYPE" == *"Alpha"* ]]; then
  echo "Transparent background — using alpha channel"
  convert "$INPUT" -alpha extract -negate "$MASK"
else
  echo "Solid background — using luminance threshold"
  convert "$INPUT" -background white -flatten -threshold 50% "$MASK"
fi

# Vectorize
potrace "$MASK" -s -O 0.2 -o "$OUTPUT"

# Sample color from logo center
TRIM=$(convert "$INPUT" -trim -format "%wx%h+%x+%y" info:)
W=$(echo $TRIM | grep -oP '^\d+')
H=$(echo $TRIM | grep -oP 'x\K\d+')
OX=$(echo $TRIM | grep -oP '\+\K\d+' | head -1)
OY=$(echo $TRIM | grep -oP '\+\K\d+' | tail -1)
CX=$((OX + W/2))
CY=$((OY + H/2))
COLOR=$(convert "$INPUT" -format "%[hex:u.p{${CX},${CY}}]" info: | head -c6)

sed -i "s/fill=\"#000000\"/fill=\"#${COLOR}\"/g" "$OUTPUT"

# Verify
PATHS=$(grep -c "<path" "$OUTPUT" 2>/dev/null || echo 0)
echo "Done: $OUTPUT ($PATHS paths, fill #$COLOR)"

rm -f "$MASK"
```

Usage: `bash vectorize.sh logo.png`

## Multi-color logos

Potrace is single-color. For logos with multiple colors:
1. Isolate each color layer by hue range: `convert input.png -fuzz 20% -fill black -opaque "#COLOR" mask.pbm`
2. Run potrace on each layer with its corresponding fill color
3. Combine all `<path>` elements into one SVG with different fills
