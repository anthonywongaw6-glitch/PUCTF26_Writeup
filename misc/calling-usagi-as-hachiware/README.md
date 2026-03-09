# Challenge Info

| Field | Value |
| --- | --- |
| **Challenge Name** | Calling Usagi as Hachiware |
| **Author** | Paco |
| **Category** | AI / Misc |
| **Description** | ula ula ula ulaaa<br>Translation: model_a is correct image classifier for usagi and hachiware<br>PS: Usagi description: https://chiikawa.fandom.com/wiki/Usagi<br>Hachiware description: https://chiikawa.fandom.com/wiki/Hachiware |

---

# TL;DR

The challenge uses two image classification models:

- `model_a.pt`
- `model_b.pt`

To obtain the flag, **both models must classify the uploaded image as `usagi` with confidence above a hidden threshold**.

Testing `model_b.pt` locally shows that it **incorrectly classifies Usagi images as `hachiware` with confidence 1.0**. This means a normal Usagi image will fail the server check.

To solve this, we generate a **targeted adversarial example** that forces `model_b` to output `usagi` while still looking like a normal Usagi image so `model_a` also predicts `usagi`.

Submitting the adversarial image reveals the flag.

---

# Final Flag

```text
PUCTF26{you_N0W_Can_id3nTiFy_USa9i_anD_hacHIWAR3e_kCQCUA71lecXz6YX76rvlg2e0foUdEb9}
```

---

# Understanding the Server Logic

The provided `test.py` file contains the core validation logic.

Models are loaded with Ultralytics YOLO:

```python
_models = {}
THRESH = "0.XX"

def load_models():
        from ultralytics import YOLO
        if _models:
                return _models
        a_path = os.path.join(os.getcwd(), "models", "model_a.pt")
        b_path = os.path.join(os.getcwd(), "models", "model_b.pt")
        _models['a'] = YOLO(a_path)
        _models['b'] = YOLO(b_path)
        return _models
```

Each uploaded image is evaluated by both models:

```python
results_a = models['a'](path)
results_b = models['b'](path)
```

The flag condition is:

```python
if class_name_a == class_name_b:
        if class_name_a == "usagi":
                if top1_conf_a > THRESH and top1_conf_b > THRESH:
                        ok = True
```

So the required condition is:

```
model_a(image) = usagi
model_b(image) = usagi
confidence > threshold
```

---

# Inspecting `model_b`

We only received `model_b.pt`, so we tested it locally.

```python
from ultralytics import YOLO
m = YOLO("model_b.pt")
print(m.names)
```

Output:

```
{0: 'hachiware', 1: 'usagi'}
```

Testing a normal Usagi image:

```python
from ultralytics import YOLO

m = YOLO("model_b.pt")
r = m("usagi.jpg")[0]

print("top1 =", r.names[r.probs.top1])
print("conf =", float(r.probs.top1conf))
```

Result:

```
top1 = hachiware
conf = 1.0
```

This shows that **model_b consistently misclassifies Usagi images as Hachiware**.

---

# Attempting Simple Image Transformations

We first tried basic modifications:

- horizontal flip  
- brightness adjustment  
- contrast adjustment  
- sharpening  
- center crop  

However, all variations still produced:

```
hachiware 1.0
```

So the model is extremely confident in the incorrect class.

---

# Crafting an Adversarial Example

Instead of manually modifying the image, we used a **targeted adversarial attack**.

Goal:

```
maximize P(model_b = usagi)
while keeping the image visually similar
```

We implemented a **PGD-style gradient attack**.

```python
from ultralytics import YOLO
from PIL import Image
import torch
import torch.nn.functional as F
import numpy as np

TARGET = 1
EPS = 24/255
ALPHA = 1/255
STEPS = 200

m = YOLO("model_b.pt")
net = m.model.eval()

img = Image.open("usagi.jpg").convert("RGB").resize((224, 224))
x0 = torch.tensor(np.array(img), dtype=torch.float32).permute(2,0,1).unsqueeze(0)/255.0
x = x0.clone().detach()

for i in range(STEPS):
    x.requires_grad_(True)

    logits = net(x)
    if isinstance(logits,(list,tuple)):
        logits = logits[0]

    loss = F.cross_entropy(logits, torch.tensor([TARGET]))

    net.zero_grad()
    if x.grad is not None:
        x.grad.zero_()
    loss.backward()

    with torch.no_grad():
        x = x - ALPHA * x.grad.sign()
        x = torch.max(torch.min(x, x0 + EPS), x0 - EPS)
        x = x.clamp(0,1).detach()

adv = (x.squeeze(0).permute(1,2,0).cpu().numpy()*255).clip(0,255).astype(np.uint8)
Image.fromarray(adv).save("adv_usagi.png")
```

---

# Verifying the Result

Testing the generated image:

```python
from ultralytics import YOLO

m = YOLO("model_b.pt")
r = m("adv_usagi.png")[0]

print(r.names[r.probs.top1], float(r.probs.top1conf))
```

Output:

```
usagi 1.0
```

The adversarial example successfully fools `model_b`.

---

# Getting the Flag

We uploaded `adv_usagi.png` to the challenge website.

Now both models agree:

```
usagi
```

The server returns the flag.

---

# Key Takeaways

This challenge demonstrates **adversarial examples in machine learning models**.

Neural networks can be highly sensitive to small perturbations, allowing attackers to manipulate predictions with minimal visual changes.

The optimization goal was:

```
argmax_x P_model(target_class | x)
subject to ||x − original|| < ε
```

This allowed us to flip the prediction from **hachiware → usagi** while keeping the image visually unchanged.

---

# Final Flag

```
PUCTF26{you_N0W_Can_id3nTiFy_USa9i_anD_hacHIWAR3e_kCQCUA71lecXz6YX76rvlg2e0foUdEb9}
