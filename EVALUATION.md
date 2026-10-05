# Gate Perception Evaluation

## Setup

```bash
git clone --recursive https://github.com/berkeleyauv/intro-perception.git
cd intro-perception
./scripts/setup.sh --yolo
source .venv/bin/activate
```

I used the setup script with `--yolo` so I could train and run YOLO locally. Training automatically used MPS on my Mac.

## Dataset

I used the provided labeled dataset and split it using:

```bash
intro-perception-data split --data data/raw
```

This gave me the following split:

| Split | Images | Gate | No Gate | Cut-off Gates |
|---|---:|---:|---:|---:|
| Train | 1467 | 1193 | 274 | 619 |
| Validation | 183 | 147 | 36 | 79 |
| Test | 183 | 140 | 43 | 71 |

The provided manifest includes truncation metadata, so I also counted the cut-off gates in each of my existing splits. There were 619 cut-off gates in train, 79 in validation, and 71 in test.

## Classical

For my classical implementation I changed how the possible gate posts are paired. Originally it basically chose the two biggest candidates, which could cause it to choose random vertical objects instead of the actual gate.

I changed it so pairs are compared based on their height similarity, vertical alignment, and the distance between them. I also kept the detector conservative, so if it can't establish two plausible posts it returns the gate as not visible instead of making a confident guess.

The classical method uses red/black segmentation, morphology, geometric filtering, post pairing, and then calculates a confidence score. The combined segmentation mask is also kept in the debug output.

## YOLO Training

For YOLO I trained the default pretrained YOLO26 nano model for 30 epochs with an image size of 640.

The full training command was:

```bash
intro-perception-train \
  --data data/generated/dataset.yaml \
  --model yolo26n.pt \
  --epochs 30 \
  --imgsz 640 \
  --output artifacts/yolo/best.pt
```

The training automatically selected Apple MPS. The best weights were copied to `artifacts/yolo/best.pt`. I did not commit the weights.

## Evaluation

I ran both methods on the same 183 images in the held-out test set.

```bash
intro-perception-run \
  --data data/generated/test/images \
  --output outputs/final_comparison \
  --method both \
  --model artifacts/yolo/best.pt
```

Then I evaluated both prediction files:

```bash
python -m intro_perception.evaluate \
  --test-dir data/generated/test \
  --predictions outputs/final_comparison/predictions_classical.jsonl

python -m intro_perception.evaluate \
  --test-dir data/generated/test \
  --predictions outputs/final_comparison/predictions_yolo.jsonl
```

## Results

| Metric | Classical | YOLO |
|---|---:|---:|
| Precision @ IoU 0.50 | 0.2472 | 0.9161 |
| Recall @ IoU 0.50 | 0.1571 | 0.9357 |
| mAP50 | 0.0853 | 0.9066 |
| Normalized Center Error | 0.0445 | 0.0059 |
| FPS | 704.7 | 19.5 |

### Classical Metric Report

```json
{
  "fps": 704.6963106532863,
  "frames": 183,
  "map50": 0.08529866291465,
  "mean_center_error": 0.044548740397698405,
  "precision_iou50": 0.24719101123595505,
  "recall_iou50": 0.15714285714285714
}
```

### YOLO Metric Report

```json
{
  "fps": 19.519015273405696,
  "frames": 183,
  "map50": 0.9066018824425671,
  "mean_center_error": 0.005896849039145762,
  "precision_iou50": 0.916083916083916,
  "recall_iou50": 0.9357142857142857
}
```

## Failure Modes

There were a few pretty clear failure modes when comparing classical and YOLO.

### 1. Missing a Gate

In one example the gate is pretty clearly visible but classical says `no gate`, while YOLO detects it with 0.96 confidence. Since classical relies on its segmentation and geometry rules, if the posts dont pass those checks then it can completely miss the gate.

### 2. Pairing the Wrong Posts

Another issue was pairing the wrong things together. I had an example where classical gave a detection with 0.42 confidence, but the box was around the wrong vertical objects. YOLO detected the actual gate at 0.97 confidence.

This is probably one of the biggest problems with the classical approach since theres other vertical objects in the scene that can look like gate posts.

### 3. Cut-Off Gates

The last failure I looked at was a cut-off gate. Part of the gate was outside the frame and classical returned `no gate`, while YOLO was still able to detect it at 0.35 confidence.

My classical implementation wants two plausible posts before saying the gate is visible, so this is a case where that assumption kind of works against it. This is also important for this dataset since the manifest shows that a pretty large amount of the gate examples are cut-off.

## Accuracy vs Speed

There is a pretty big tradeoff between the two methods. Classical ran at around 704.7 FPS while YOLO ran at around 19.5 FPS, so classical is way faster.

At the same time though, YOLO is way more accurate. Its mAP50 was 0.9066 compared to only 0.0853 for classical. YOLO also had much higher precision and recall, and its normalized center error was a lot smaller.

So classical would be useful if computation was extremely limited, but based on these results I think the accuracy difference is more important than the extra speed.

## Training Data Coverage

The YOLO training set had 1467 images, with 1193 gate images and 274 no-gate images. Of the training images, 619 were marked as cut-off in the manifest.

The training data coverage matters a lot for YOLO because it learns the gate appearance from the examples its given. Having different gate positions and difficult views helps it not depend on one exact gate appearance like classical does.

The amount of cut-off examples is especially useful here. A gate isn't always going to be completely inside the camera frame, and YOLO was able to detect the partially visible gate in my example even when classical failed. More examples with different underwater lighting, distances, angles, and partial gates would probably make it even more reliable.

At the same time, the model is still dependent on the provided training data. If the real deployment environment looks very different from this dataset, the accuracy could drop.

## Classical Assumptions

The classical detector makes a few assumptions about what a gate should look like. It depends on red/black segmentation to find possible parts of the gate and then looks for vertical candidates that can be paired together.

This works really fast, but underwater lighting can change how the colors look. There can also be other vertical objects that pass the geometry checks and get paired incorrectly.

The two-post assumption is especially limiting for cut-off gates. If one post is outside the frame, classical may not have two plausible candidates anymore and returns `no gate`. YOLO doesn't explicitly need to find and pair two posts, which is probably part of why it handled my cut-off example better.

## Known Limitations

The biggest limitation of classical is that it relies on fixed color and geometry assumptions. Lighting changes can mess with the segmentation, unrelated vertical objects can be mistaken for posts, and partial gates can fail the two-post requirement.

YOLO's biggest downside is speed. It was much slower than classical and it also needs trained weights. Its performance depends on how similar the actual deployment environment is to the training data too.

Another limitation is that I only evaluated these methods on the provided held-out test set. Real underwater conditions could have lighting, visibility, backgrounds, or gate orientations that aren't represented as well in this dataset.

## What I Would Deploy

I would deploy YOLO. Even though classical was way faster, YOLO was still running at about 19.5 FPS and the accuracy difference was huge. For an actual gate detector I would rather have the more reliable detection than hundreds of FPS with a lot of missed or incorrectly localized gates.

If I kept working on this, I would add more difficult training images, especially different underwater lighting and partial gates. I would also try a lower inference resolution, quantization, or another optimized deployment format to see how much I could increase YOLO's FPS without losing too much accuracy.

## Annotated Examples

The three annotated examples I selected show the main differences I saw between the methods:

1. A visible gate where classical says `no gate` and YOLO detects it with 0.96 confidence.
2. A case where classical pairs the wrong vertical objects while YOLO correctly detects the gate at 0.97 confidence.
3. A cut-off gate where classical says `no gate` but YOLO still detects the visible part at 0.35 confidence.