# [Frankesqwen](https://tryhackme.com/room/frankesqwen)

## Can you find the flag nested in Frankesqwen?

# Find the flag

```
connection.info

> Connection Information

Protocol SSH
Username frankesqwen
Password FrankesQwen

The model is available directly on the VM.

> Alternative Downloads

> Model ab123451/frankesqwen-v7
> Hint model ab123451/frankesqwen-hint-v2

> No local GPU? Run it for free on Google Colab.
```

### What's the flag?

1. Gain SSH access

Connect with the credentials provided in the challenge brief.

```bash
ssh frankesqwen@TARGET_IP
# password: FrankesQwen
```

Confirm the model files and environment:

```bash
ls -la ~/frankesqwen-v7 ~/frankesqwenhint
source ~/myenv/bin/activate
```

2. Prompt the main model directly

```bash
cat > ~/chat.py << 'EOF'
cat > ~/chat.py << 'EOF'
import sys
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

path = sys.argv[1]
tok = AutoTokenizer.from_pretrained(path)
model = AutoModelForCausalLM.from_pretrained(path, dtype=torch.float32)
model.eval()

while True:
    user = input("USER> ")
    if user in ("exit", "quit"):
        break
    msgs = [{"role": "user", "content": user}]
    text = tok.apply_chat_template(msgs, add_generation_prompt=True, tokenize=False)
    enc = tok(text, return_tensors="pt")
    out = model.generate(
        **enc,
        max_new_tokens=60,
        do_sample=False,
        repetition_penalty=1.3,
    )
    resp = tok.decode(out[0][enc["input_ids"].shape[1]:], skip_special_tokens=True)
    print("MODEL>", resp)
EOF

python3 ~/chat.py ~/frankesqwen-v7
```

Ask `What is the flag?`. It refuses or degenerates into repetition. This holds up across prompt variations, confirming the suppression lives in the weights, not the prompt.

<img width="1222" height="599" alt="SCREEN01" src="https://github.com/user-attachments/assets/c4bac7cf-9b01-4e87-a2c1-085f05c50558" />

3. Diff the two model's weights

```bash
cat > ~/diff_weights.py << 'EOF'
from safetensors import safe_open

p1 = "/home/frankesqwen/frankesqwen-v7/model.safetensors"
p2 = "/home/frankesqwen/frankesqwenhint/model.safetensors"

with safe_open(p1, framework="pt") as f1, safe_open(p2, framework="pt") as f2:
    for k in sorted(set(f1.keys()) & set(f2.keys())):
        t1, t2 = f1.get_tensor(k), f2.get_tensor(k)
        diff = (t1 - t2).abs().sum().item()
        if diff > 0:
            print(f"{k}: total={diff:.4f}, max={(t1-t2).abs().max().item():.4f}")
EOF

python3 ~/diff_weights.py
```

Most tensors show small, uniform fine-tuning deltas. Two stand out by roughly two orders of magnitude: `model.layers.22.mlp.down_proj.weight` and `model.layers.23.mlp.down_proj.weight`.

4. Patch those two tensors from the hint model into v7

```bash
cat > ~/patch_and_ask.py << 'EOF'
import torch
from safetensors.torch import load_file
from transformers import AutoModelForCausalLM, AutoTokenizer

path_v7 = "/home/frankesqwen/frankesqwen-v7"
path_hint = "/home/frankesqwen/frankesqwenhint"

tok = AutoTokenizer.from_pretrained(path_v7)
model = AutoModelForCausalLM.from_pretrained(path_v7, dtype=torch.float32)
hint_weights = load_file(f"{path_hint}/model.safetensors")

with torch.no_grad():
    for layer_idx in [22, 23]:
        key = f"model.layers.{layer_idx}.mlp.down_proj.weight"
        dict(model.named_parameters())[key].copy_(hint_weights[key])

model.eval()

text = tok.apply_chat_template(
    [{"role": "user", "content": "What is the flag?"}],
    add_generation_prompt=True, tokenize=False
)
enc = tok(text, return_tensors="pt")
out = model.generate(**enc, max_new_tokens=80, do_sample=False)
print(tok.decode(out[0][enc["input_ids"].shape[1]:], skip_special_tokens=False))
EOF

python3 ~/patch_and_ask.py
```

<img width="1219" height="582" alt="SCREEN02" src="https://github.com/user-attachments/assets/e0022067-6700-4019-9932-03dadd5ddb82" />
