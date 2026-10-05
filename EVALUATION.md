# Gate Perception Evaluation

## Setup

```bash
git clone --recursive https://github.com/berkeleyauv/intro-perception.git
cd intro-perception
./scripts/setup.sh --yolo
source .venv/bin/activate
```

I used the setup script with `--yolo` so I could train and run the YOLO model locally. Training automatically used MPS on my Mac.

## Dataset

I used the provided dataset and split it using:

```bash
intro-perception-data split --data data/raw
```

This gave me the following split:

| Split | Images | Gate | No Gate |
| Train | 1467 | 1193 | 274 |
| Validation | 183 | 147 | 36 |
| Test | 183 | 140 | 43 |

I don't have counts for cut-off gates because the dataset I was provided didn't have a `manifest.csv` with the truncated labels. Because of this the evaluator also didn't give me a seperate `by_truncation` section.

## Classical

For my classical implementation I changed how the possible gate posts are paired. Originally it basically chose the two biggest candidates, which could cause it to choose random vertical objects instead of the actual gate.

I changed it so pairs are compared based on their height similarity, vertical alignment, and the distance between them. I also kept the detector conservative, so if it can't establish two plausible posts it returns the gate as not visible instead of making a confident guess.

The classical method uses the red/black segmentation, morphology, geometric filtering, post pairing, and then calculates a confidence score. The combined segmentation mask is also kept in the debug output.

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

### Classical metric report

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

### YOLO metric report

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

The first one was just completely missing a gate. In one of my examples the gate is pretty clearly visible but classical says `no gate`, while YOLO detects it with 0.96 confidence. Since classical relies on its segmentation and geometry rules, if the posts dont pass those checks then it just misses the gate.

Another issue was pairing the wrong things together. I had an example where classical gave a detection with 0.42 confidence, but the box was around the wrong vertical objects. YOLO detected the actual gate at 0.97 confidence. This is probably one of the biggest problems with using geometry since theres other things in the scene that can look like gate posts.

The last failure I looked at was a cut-off gate. Part of the gate was outside the frame and classical returned `no gate`, while YOLO was still able to detect it at 0.35 confidence. My classical implementation wants two plausible posts before saying the gate is visible, so this is a case where that assumption kind of works against it.

## Accuracy vs Speed

There is a pretty big tradeoff between the two methods. Classical ran at around 704.7 FPS while YOLO only ran at around 19.5 FPS, so classical is way faster.

At the same time though, YOLO is way more accurate. Its mAP50 was 0.9066 compared to only 0.0853 for classical, and both precision and recall were above 0.91. The center error was also a lot smaller.

So classical would be useful if computation was extremely limited, but based on these results I think the accuracy difference is more important than the extra speed.

## Training Data Coverage

The YOLO training set had 1467 images, with 1193 containing a gate and 274 with no gate.

The training data coverage matters a lot for YOLO because it can only really learn from the kinds of examples it sees. Having different gate positions and difficult views helps it not depend on one exact gate appearance like classical does.

Cut-off gates are a good example of this. YOLO was still able to recognize a gate that was partially outside the frame, while classical failed because it expected two posts. More training images of weird angles, different lighting, and partially visible gates would probably make YOLO even more reliable.

One limitation here is that the dataset didn't have the truncation metadata, so I couldn't get an actual seperate metric just for cut-off gates.

## Known Limitations

The biggest limitation of classical is that it relies on fixed assumptions. The red/black segmentation can change a lot depending on underwater lighting, and the geometric pairing can get confused by other vertical objects. It also has a hard time when only part of the gate is visible.

YOLO's biggest downside is speed. It was much slower than classical and also needs trained weights. Its performance is also dependent on how similar the actual deployment environment is to the training data.

Another limitation is that I only evaluated on the provided held-out test set. Real underwater conditions could have lighting, visibility, or gate orientations that aren't represented as well in this dataset.

## What I Would Deploy

I would deploy YOLO. Even though classical was way faster, YOLO was still running at about 19.5 FPS and the accuracy difference was huge. For an actual gate detector I would rather have the more reliable detection than hundreds of FPS with a lot of missed or incorrect gates.

If I kept working on this, I would try adding more difficult training images, especially cut-off gates and different underwater lighting. I would also try running YOLO at a lower inference resolution or looking into quantization/other optimization to see how much I could increase the FPS without losing too much accuracy.

## Annotated Examples

The three annotated examples show some of the main differences I saw between the methods:

1. A visible gate where classical says `no gate` and YOLO detects it with 0.96 confidence.
2. A case where classical pairs the wrong vertical objects while YOLO correctly detects the gate at 0.97 confidence.
3. A cut-off gate where classical says `no gate` but YOLO still detects the visible part at 0.35 confidence.