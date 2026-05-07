---
title: "A Backend Engineer's Path to Learning Computer Vision - From Image Recognition Basics to Vision APIs"
date: 2026-03-10
categories: [CS Fundamentals]
tags: [computer-vision, vision-api, study-log]
mermaid: true
---

## Why Study CV

While pondering what AI could add to a half-finished climbing app I once started, the idea of "automatically recognizing hold colors from photos" came up. But I'm a backend engineer, not an ML engineer. Can't I just call the Vision API? Sure, but without understanding the basics, I wouldn't know why things fail or how to write proper prompts.

I have no intention of learning CV "at an expert level." I've organized **the minimum fundamentals needed to use Vision APIs effectively**. Following this roadmap, it should take about 16 hours.

---

## Learning Roadmap

```mermaid
flowchart TD
    A["1. 이미지가 뭔가?<br>(픽셀, 채널, 텐서)"] --> B["2. 전통적 이미지 처리<br>(필터, 엣지 검출, 색상 공간)"]
    B --> C["3. CNN 기초<br>(합성곱, 풀링, 특징 추출)"]
    C --> D["4. 이미지 분류 vs 객체 탐지<br>(Classification vs Detection)"]
    D --> E["5. 전이 학습<br>(Pre-trained 모델 활용)"]
    E --> F["6. Vision API 활용<br>(Claude Vision, GPT-4V)"]
    F --> G["7. 한계 이해 + 커스텀 모델 판단"]
```

Rather than going deep into everything, the approach was to **understand "why this is necessary" at each stage and move on to the next**.

---

## Step 1: The Essence of Images -- Pixels and Tensors

To a computer, an image is a **numerical array (tensor)**.

```text
흑백 이미지 (3x3):     컬러 이미지 (3x3):
┌─────────────┐        ┌─────────────────────┐
│ 0   128  255 │        │ R채널 │ G채널 │ B채널 │
│ 64  192   32 │        │ (3x3) │ (3x3) │ (3x3) │
│ 128  96  160 │        └─────────────────────┘
└─────────────┘         → shape: (3, 3, 3)
→ shape: (3, 3)
```

- Each pixel is an integer between 0-255
- A color image has 3 RGB channels --> a 3D tensor of (height, width, 3)
- **A 4000x3000 photo = 36 million numbers** --> feeding this directly to an LLM causes costs to explode

Key insight: **Resizing images is fundamental to cost optimization.** Sending originals as-is to Vision APIs leads to unnecessarily high token counts.

---

## Step 2: Traditional Image Processing -- Color Spaces

To recognize climbing hold colors, you need to understand **color spaces**.

```mermaid
flowchart LR
    A["RGB<br>(Red, Green, Blue)"] --> B["인간 직관과 다름<br>'빨간색'을 RGB로 정의하기 어려움"]
    C["HSV<br>(Hue, Saturation, Value)"] --> D["색상 필터링에 적합<br>'빨간색 = H값 0~10'으로 정의 가능"]
```

| Color Space | Strengths | Climbing Application |
|------------|-----------|---------------------|
| **RGB** | Standard, default for input/output | Displaying on screen |
| **HSV** | Easy to filter by hue (H) | Classifying hold colors |
| **LAB** | Robust against lighting changes | In dimly lit climbing gyms |

```python
import cv2
import numpy as np

# 이미지 읽기 + HSV 변환
img = cv2.imread('climbing_wall.jpg')
hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)

# 빨간색 홀드 필터링 (HSV 범위)
lower_red = np.array([0, 100, 100])
upper_red = np.array([10, 255, 255])
mask = cv2.inRange(hsv, lower_red, upper_red)

# 빨간색 영역의 픽셀 수
red_pixels = cv2.countNonZero(mask)
```

Knowing just this much lets you debug "why the Vision API got the color wrong." You can understand that in dim lighting, the V (brightness) in HSV drops, making color identification difficult.

---

## Step 3: CNN Basics -- Why Neural Networks Excel at Image Recognition

```mermaid
flowchart LR
    A["입력 이미지<br>(224x224x3)"] --> B["Conv Layer 1<br>엣지, 색상 감지"]
    B --> C["Conv Layer 2<br>패턴, 텍스처 감지"]
    C --> D["Conv Layer 3<br>부분 형태 감지"]
    D --> E["Fully Connected<br>최종 분류"]
    E --> F["출력: '빨간 홀드'"]
```

The key concepts of CNN (Convolutional Neural Network):

- **Convolution**: Sliding small filters across the image to extract features
- **Hierarchical feature learning**: Low-level (edges) --> mid-level (patterns) --> high-level (objects)
- **Pooling**: Reducing spatial dimensions for computational efficiency

**Why do you need to know this?** Vision APIs use this architecture internally. To understand "why it confuses similar colors" or "why it misses small holds," you need a sense of how CNNs work.

---

## Step 4: Image Classification vs Object Detection

There are two possible approaches for analyzing climbing wall photos:

```mermaid
flowchart TD
    A["클라이밍 벽 사진"] --> B{"어떤 태스크?"}
    B --> C["이미지 분류<br>(Classification)"]
    B --> D["객체 탐지<br>(Object Detection)"]
    
    C --> E["이 사진에 빨간 홀드가 있는가?<br>→ Yes/No"]
    D --> F["빨간 홀드가 어디에 몇 개 있는가?<br>→ 위치 + 개수"]
```

| Task | Output | Climbing Application |
|------|--------|---------------------|
| **Classification** | "This photo is a red route" | Simple, high accuracy, identifies single route |
| **Detection** | "8 red holds, positions (x,y)" | Complex, identifies multiple routes simultaneously |
| **Segmentation** | "These pixels are red holds" | Most accurate, most complex |

For an MVP, **classification level** is sufficient. All you need is to determine "what color routes are in this photo?" Vision APIs can handle this much.

---

## Step 5: Transfer Learning -- When a Custom Model Is Needed

When the limitations of Vision APIs become clear, a custom model is necessary. Instead of **training from scratch, you use transfer learning**.

```mermaid
flowchart TD
    A["ImageNet으로 학습된<br>ResNet/EfficientNet"] --> B["마지막 분류 레이어만<br>교체"]
    B --> C["클라이밍 홀드 데이터로<br>fine-tuning"]
    C --> D["클라이밍 특화 모델 완성"]
```

```python
import torch
from torchvision import models

# Pre-trained ResNet 로드
model = models.resnet50(pretrained=True)

# 마지막 레이어만 교체 (1000개 클래스 → 8개 홀드 색상)
model.fc = torch.nn.Linear(model.fc.in_features, 8)

# 기존 레이어는 동결, 마지막만 학습
for param in model.parameters():
    param.requires_grad = False
model.fc.requires_grad_(True)
```

With transfer learning, **you can build a usable model with just a few hundred images**. The efficiency is incomparable to training from scratch, which requires tens of thousands.

### Data Acquisition Strategy

Building a custom model requires data:

| Strategy | Data Volume | Feasibility |
|----------|-----------|-------------|
| Self-photograph + label | Hundreds possible | Labor-intensive |
| User correction data from the app | Requires the app to exist | **Most ideal** |
| Crowdsourcing | Requires a community | Long-term |

**The app's "suggestion + user correction" flow naturally generates training data.** When the Vision API suggests "3 red" and the user corrects it to "2 red, 1 orange," that correction becomes labeling data.

---

## Step 6: Vision API Usage -- Practical Choices

With the above fundamentals understood, you can now judge what Vision APIs are good and bad at.

```mermaid
flowchart TD
    A["Vision API가 잘하는 것"]
    A --> B["일반적인 객체 인식"]
    A --> C["텍스트 읽기 (OCR)"]
    A --> D["장면 설명"]
    
    E["Vision API가 못하는 것"]
    E --> F["도메인 특화 분류<br>(홀드 vs 볼트 구분)"]
    E --> G["미세한 색상 차이<br>(주황 vs 빨강)"]
    E --> H["조명이 나쁜 환경"]
```

**Knowing the basics changes your prompts:**

```text
❌ (기초 없이): "이 사진에서 색상을 분석해줘"

✅ (기초 있으면): "이 볼더링 벽 사진에서 홀드를 분석해줘.
   주의: 홀드와 볼트(회색 육각 나사)를 구분할 것.
   조명이 어두우므로 채도가 낮은 색상도 고려할 것.
   테이프 마킹은 홀드 색상이 아닌 루트 구분용임."
```

When you know how CNNs recognize colors, you can specify in the prompt that "dim lighting reduces saturation."

---

## Study Plan Summary

Here's the organized study sequence and resources:

| Order | Topic | Resource | Estimated Time |
|-------|-------|----------|---------------|
| 1 | Image basics | OpenCV official tutorial (Python) | 3 hours |
| 2 | Color spaces | OpenCV HSV filtering hands-on | 2 hours |
| 3 | CNN concepts | 3Blue1Brown "Neural Networks" series | 2 hours |
| 4 | CNN hands-on | PyTorch official tutorial (image classification) | 4 hours |
| 5 | Transfer learning | PyTorch Transfer Learning tutorial | 3 hours |
| 6 | Vision API | Claude/GPT-4V docs + experiments | 2 hours |
| **Total** | | | **~16 hours** |

**16 hours should be enough to reach "a level where you can properly use Vision APIs."** The goal isn't to become a CV expert, but to reach a level where you can make informed decisions as an API user.

---

## What I Realized While Organizing This Roadmap

### Without the Basics, You Won't Be Able to Use APIs Properly
Vision APIs are black boxes. But if you roughly understand what's happening inside, you can diagnose failures and compensate with better prompts. An attitude of "just call the API" will plateau at 80% accuracy.

### ML Fundamentals Are Worth the Investment for Backend Engineers
You don't need to become an ML expert. But knowing what a CNN is and what transfer learning does enables you to judge "does this problem need a custom model, or is an API enough?" Without this judgment ability, you'd have to ask an ML engineer every time.

### A Practical Goal Determines Learning Efficiency
"Let's study CV" is open-ended. But "I want to recognize climbing hold colors with a Vision API" makes it clear what to study. Goalless learning is a waste of time; goal-oriented learning can open doors in 16 hours.
