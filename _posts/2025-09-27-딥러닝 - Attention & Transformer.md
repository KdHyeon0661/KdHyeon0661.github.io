---
layout: post
title: 딥러닝 - Attention & Transformer
date: 2025-09-27 21:25:23 +0900
category: 딥러닝
---
# 1.10 Attention & Transformer 핵심  
**쿼리/키/값 · 멀티헤드 · 포지셔널 인코딩 · 마스킹(causal/패딩) · 디코딩 전략(그리디/빔/샘플링)**

## A. Scaled Dot-Product Attention

### A-1. 핵심 직관
- 입력 시퀀스의 각 토큰은 **내가 지금 누구를 봐야 하는가?**(Query) / **나를 얼마나 참고할 것인가?**(Key) / **참고할 때 어떤 정보를 줄 것인가?**(Value)로 분해되어 상호작용합니다.
- 한 쿼리 벡터 $$\mathbf{q}\in\mathbb{R}^{d_k}$$ 와 모든 키 $$\mathbf{K}\in\mathbb{R}^{T\times d_k}$$ 의 유사도(점곱)를 소프트맥스로 정규화하여 가중합으로 값을 모읍니다.

### A-2. 수식
쿼리/키/값: $$\mathbf{Q}=\mathbf{X}\mathbf{W}^Q,\quad \mathbf{K}=\mathbf{X}\mathbf{W}^K,\quad \mathbf{V}=\mathbf{X}\mathbf{W}^V$$  
스코어/어텐션/출력:
$$
\mathbf{S}=\frac{\mathbf{Q}\mathbf{K}^\top}{\sqrt{d_k}},\quad
\mathbf{A}=\mathrm{softmax}(\mathbf{S} + \mathbf{M}),\quad
\mathrm{Attn}(\mathbf{Q},\mathbf{K},\mathbf{V})=\mathbf{A}\mathbf{V}.
$$
- $$\mathbf{M}$$: **마스크(additive)**. 금지 위치는 $$-\infty$$(또는 아주 작은 수)로 채워 softmax에서 0 확률을 강제.

### A-3. 왜 스케일링 $$\sqrt{d_k}$$?
- $$\langle \mathbf{q},\mathbf{k}\rangle$$ 의 분산을 고정시켜 **큰 차원에서 softmax가 포화**하지 않도록 예방 → 안정적 그라디언트.

---

## B. Multi-Head Attention (MHA)

### B-1. 동시 다각도 뷰
- 하나의 큰 공간에서 한 번 계산하는 대신, $$h$$개의 헤드로 **부분 공간**을 나누어 병렬로 어텐션:
$$
\mathrm{MHA}(\mathbf{X})=\mathrm{Concat}(\mathrm{head}_1,\dots,\mathrm{head}_h)\mathbf{W}^O,
$$
$$
\mathrm{head}_i=\mathrm{Attn}(\mathbf{X}\mathbf{W}_i^Q,\ \mathbf{X}\mathbf{W}_i^K,\ \mathbf{X}\mathbf{W}_i^V).
$$
- 보통 $$d_{\text{model}} = h\cdot d_k = h\cdot d_v$$.

### B-2. 복잡도
- 시간/메모리 복잡도는 **$$\mathcal{O}(T^2 d)$$**(셀프-어텐션). 긴 시퀀스에서 비용 급증 → FlashAttention/저랭크/스파스/선형 어텐션 등 실전 최적화가 등장.

---

## C. 포지셔널 인코딩(Positional Encoding)

어텐션은 순서를 모릅니다. → **위치 정보**를 주입해야 합니다.

### C-1. 사인/코사인(고정형; Transformer 원 논문)
$$
\mathrm{PE}_{(pos,2i)}=\sin\!\Big(\frac{pos}{10000^{2i/d_{\text{model}}}}\Big),\quad
\mathrm{PE}_{(pos,2i+1)}=\cos\!\Big(\frac{pos}{10000^{2i/d_{\text{model}}}}\Big).
$$
- **장점**: 외삽(긴 길이) 가능, 파라미터 없음.

### C-2. 학습형 포지션 임베딩(learned)
- 위치마다 임베딩 벡터를 **학습**. 구현 단순, 길이 고정(max_len 안)에서 강력.

### C-3. RoPE(회전 포지셔널 인코딩; Rotary)
- $$\mathbf{q},\mathbf{k}$$ 의 2차원 쌍을 **각도 회전**으로 위치를 주입 → **상대적 거리**에 민감, 길이 스케일링/외삽에 유리. (간단 구현 포함)

---

## D. 마스킹(Masking)

### D-1. 패딩 마스크(Padding mask)
- 배치 내 문장 길이가 다르면 **PAD 토큰**을 채웁니다. PAD 위치는 **어텐션 확률 0**이어야 합니다.
- 보통 shape: `(B, 1, 1, S)`(브로드캐스트) 또는 `(B, 1, S, S)`.

### D-2. 카우절 마스크(Causal mask)
- **미래 토큰 금지**(오토리그레시브 LM; GPT류):
$$
\mathbf{M}_{\text{causal}}[i,j]=
\begin{cases}
0 & j\le i\\
-\infty & j> i
\end{cases}
$$

### D-3. 결합
- **self-attn**: `mask = causal_mask OR padding_mask(키 쪽)`  
- **cross-attn(디코더→인코더)**: 디코더 쿼리는 과거만 보되, **인코더 key/value에서 PAD만** 막기.

---

## E. 파이토치로 MHA를 “처음부터” 구현

아래 구현은 **Pre-LN** Transformer 블록에 바로 쓸 수 있게 작성합니다.  
(실전에서는 PyTorch `nn.MultiheadAttention`/FlashAttention 사용 권장)

```python
import torch, torch.nn as nn, torch.nn.functional as F

def make_pad_mask(seq_lens, max_len=None):
    """
    seq_lens: (B,) 실제 길이(패딩 제외)
    return: (B, 1, 1, L) 형태의 boolean mask (True==mask)
    """
    B = seq_lens.size(0)
    L = int(seq_lens.max().item()) if max_len is None else max_len
    ar = torch.arange(L, device=seq_lens.device).unsqueeze(0).expand(B, L)  # (B,L)
    mask = ar >= seq_lens.unsqueeze(1)  # PAD 위치 True
    return mask.unsqueeze(1).unsqueeze(1)  # (B,1,1,L)

def make_causal_mask(L, device=None):
    """
    return: (1,1,L,L) upper-tri True mask (j>i)
    """
    m = torch.triu(torch.ones(L, L, device=device, dtype=torch.bool), diagonal=1)
    return m.unsqueeze(0).unsqueeze(0)  # (1,1,L,L)

def combine_masks(*masks):
    """
    additive mask를 만들 수 있도록 boolean mask들을 OR로 결합
    return: bool mask or None
    """
    masks = [m for m in masks if m is not None]
    if not masks: return None
    out = masks[0]
    for m in masks[1:]:
        out = out | m
    return out

def apply_attention_mask(scores, mask):
    """
    scores: (B, h, T_q, T_k), mask: broadcastable bool (True ==> mask)
    """
    if mask is None: return scores
    # float16/bfloat16 환경 안전한 -inf
    minus_inf = torch.finfo(scores.dtype).min
    return scores.masked_fill(mask, minus_inf)

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.0, bias=True):
        super().__init__()
        assert d_model % num_heads == 0
        self.d_model = d_model
        self.h = num_heads
        self.d_head = d_model // num_heads
        self.wq = nn.Linear(d_model, d_model, bias=bias)
        self.wk = nn.Linear(d_model, d_model, bias=bias)
        self.wv = nn.Linear(d_model, d_model, bias=bias)
        self.wo = nn.Linear(d_model, d_model, bias=bias)
        self.attn_drop = nn.Dropout(dropout)
        self.proj_drop = nn.Dropout(dropout)

    def forward(self, q, k, v, mask=None):
        """
        q,k,v: (B, T, d_model)
        mask: broadcastable bool, True==mask (ex: (B,1,1,T_k) or (1,1,T_q,T_k))
        """
        B, Tq, _ = q.shape
        Tk = k.shape[1]
        # [B,T, d_model] -> [B,h,T,d_head]
        Q = self.wq(q).view(B, Tq, self.h, self.d_head).transpose(1, 2)
        K = self.wk(k).view(B, Tk, self.h, self.d_head).transpose(1, 2)
        V = self.wv(v).view(B, Tk, self.h, self.d_head).transpose(1, 2)
        # scaled dot-product
        scores = (Q @ K.transpose(-2, -1)) / (self.d_head ** 0.5)  # (B,h,Tq,Tk)
        scores = apply_attention_mask(scores, mask)
        A = F.softmax(scores, dim=-1)
        A = self.attn_drop(A)
        out = A @ V  # (B,h,Tq,d_head)
        out = out.transpose(1, 2).contiguous().view(B, Tq, self.d_model)
        out = self.proj_drop(self.wo(out))
        return out, A  # A는 분석/시각화용
```

---

## F. 포지셔널 인코딩 구현(사인/코사인, RoPE)

### F-1. 사인/코사인(고정형)
```python
class SinusoidalPE(nn.Module):
    def __init__(self, d_model, max_len=10000):
        super().__init__()
        self.d_model = d_model
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)
        div = torch.exp(torch.arange(0, d_model, 2).float() * (-torch.log(torch.tensor(10000.0))/d_model))
        pe[:, 0::2] = torch.sin(position * div)
        pe[:, 1::2] = torch.cos(position * div)
        self.register_buffer('pe', pe)  # (max_len, d_model)

    def forward(self, x, start_pos=0):
        """
        x: (B,T,d_model)
        """
        T = x.size(1)
        return x + self.pe[start_pos:start_pos+T]
```

### F-2. RoPE(간이 구현; Q/K에 회전만 부여)
```python
def rope_rotate(x, base=10000):
    """
    x: (..., d_model) where d_model is even
    returns rotated x with pairwise rotation
    """
    d = x.size(-1)
    half = d // 2
    x1, x2 = x[..., :half], x[..., half:]
    # make angles for each pair index
    freq = torch.arange(half, device=x.device, dtype=x.dtype)
    inv = 1.0 / (base ** (2*freq/ d))
    # broadcast over sequence positions using implicit pos index via outer-product trick
    # require explicit pos indices:
    T = x.size(-2)
    pos = torch.arange(T, device=x.device, dtype=x.dtype).unsqueeze(-1)  # (T,1)
    angles = pos * inv  # (T, half)
    sin = torch.sin(angles)[None, None, ...]  # broadcast for (B,h or ?) match later if needed
    cos = torch.cos(angles)[None, None, ...]
    # here assume x shape (B, T, d) -> align (B, T, half)
    # rotate (x1, x2): (x1*cos - x2*sin, x1*sin + x2*cos)
    x1r = x1*cos.squeeze(0).squeeze(0) - x2*sin.squeeze(0).squeeze(0)
    x2r = x1*sin.squeeze(0).squeeze(0) + x2*cos.squeeze(0).squeeze(0)
    return torch.cat([x1r, x2r], dim=-1)

# 실전에서는 헤드별/차원별로 RoPE를 적용하고, 캐싱 시 각 step에 맞춰 회전 각을 업데이트합니다.
```
> 주: 위 RoPE 코드는 **개념 시연용**입니다(실전에서는 헤드 차원 포함형 및 캐시형 구현을 권장).

---

## G. Transformer 블록(Pre-LN) 구현

### G-1. Feed-Forward (FFN)
$$
\mathrm{FFN}(\mathbf{x}) = \mathbf{W}_2\,\phi(\mathbf{W}_1\mathbf{x}+\mathbf{b}_1)+\mathbf{b}_2,
$$
보통 $$d_{\text{ff}} \approx 4\cdot d_{\text{model}}$$, 활성은 **GELU** 선호.

```python
class FeedForward(nn.Module):
    def __init__(self, d_model, d_ff, p_drop=0.0):
        super().__init__()
        self.fc1 = nn.Linear(d_model, d_ff)
        self.fc2 = nn.Linear(d_ff, d_model)
        self.act = nn.GELU()
        self.drop = nn.Dropout(p_drop)

    def forward(self, x):
        return self.drop(self.fc2(self.act(self.fc1(x))))
```

### G-2. 디코더 전용 블록(GPT 스타일; 마스킹된 self-attn)
```python
class DecoderBlock(nn.Module):
    def __init__(self, d_model=512, nhead=8, d_ff=2048, attn_drop=0.0, resid_drop=0.0):
        super().__init__()
        self.ln1 = nn.LayerNorm(d_model)
        self.mha = MultiHeadAttention(d_model, nhead, dropout=attn_drop)
        self.ln2 = nn.LayerNorm(d_model)
        self.ff = FeedForward(d_model, d_ff, p_drop=resid_drop)
        self.resid_drop = nn.Dropout(resid_drop)

    def forward(self, x, attn_mask=None):
        # x: (B,T,d)
        h = self.ln1(x)
        h, _ = self.mha(h, h, h, mask=attn_mask)
        x = x + self.resid_drop(h)
        h = self.ln2(x)
        h = self.ff(h)
        x = x + self.resid_drop(h)
        return x
```

### G-3. 전체 디코더(언어모델 헤드 포함)
```python
class GPTMini(nn.Module):
    def __init__(self, vocab_size, d_model=512, nlayer=6, nhead=8, d_ff=2048, max_len=2048, p_drop=0.1, learned_pos=False):
        super().__init__()
        self.tok_emb = nn.Embedding(vocab_size, d_model)
        self.pos_emb = (nn.Embedding(max_len, d_model) if learned_pos else SinusoidalPE(d_model, max_len))
        self.blocks = nn.ModuleList([DecoderBlock(d_model, nhead, d_ff, attn_drop=p_drop, resid_drop=p_drop) for _ in range(nlayer)])
        self.ln_f = nn.LayerNorm(d_model)
        self.head = nn.Linear(d_model, vocab_size, bias=False)
        self.max_len = max_len
        self.learned_pos = learned_pos

    def forward(self, idx, attn_mask=None):
        """
        idx: (B,T) int64
        attn_mask: (1,1,T,T) or (B,1,T,T) causal/pad 결합 마스크 (True==mask)
        """
        B, T = idx.shape
        x = self.tok_emb(idx)  # (B,T,d)
        if self.learned_pos:
            pos = torch.arange(T, device=idx.device)
            x = x + self.pos_emb(pos)[None, :, :]
        else:
            x = self.pos_emb(x)  # sinusoidal: x에 더해줌

        for blk in self.blocks:
            x = blk(x, attn_mask=attn_mask)
        x = self.ln_f(x)
        logits = self.head(x)   # (B,T,V)
        return logits
```

### G-4. 마스크 만들기(패딩+카우절)
```python
def build_decoder_mask(input_ids, pad_id=None):
    """
    input_ids: (B,T)
    pad_id: 패딩 토큰 id (없으면 None)
    return: (B or 1, 1, T, T) bool mask
    """
    B, T = input_ids.shape
    causal = make_causal_mask(T, device=input_ids.device)  # (1,1,T,T)
    pad_mask = None
    if pad_id is not None:
        # 키 쪽 패딩을 가린다: (B,1,1,T)
        key_pad = (input_ids == pad_id).unsqueeze(1).unsqueeze(1)
        # causal과 브로드캐스트 가능하게 결합
        pad_mask = key_pad.expand(B, 1, T, T)
    return combine_masks(causal, pad_mask)
```

---

## H. 엔코더/디코더 개요(번역 등)

- **Encoder**: 패딩 마스크만 사용, 자기 자신을 전부 볼 수 있음.  
- **Decoder**: **마스킹된 self-attn**(미래 금지) + **cross-attn**(인코더 출력과 상호작용) + 패딩 마스크(인코더쪽 PAD 금지).

> 본 장에서는 GPT류(디코더 전용)에 초점을 맞추지만, cross-attn은 **`Q=디코더, K/V=인코더 출력`** 으로 바로 연결됩니다.

---

## I. 학습: 토이 언어모델(문자 단위)

아주 작은 예로 **숫자/기호** 토큰 사전을 만들고 **다음 토큰 예측**을 수행합니다.

```python
import torch, torch.nn as nn, torch.nn.functional as F
torch.manual_seed(0)

# 1) Toy vocab & 데이터
chars = list("0123456789+-= ")
stoi = {ch:i for i,ch in enumerate(chars)}
itos = {i:ch for ch,i in stoi.items()}
V = len(chars); pad_id = stoi[" "]

def encode(s): return torch.tensor([stoi[c] for c in s], dtype=torch.long)
def decode(ids): return "".join([itos[int(i)] for i in ids])

# 랜덤 산식 같은 문자열 만들기
def make_batch(B=64, T=32):
    X = torch.full((B,T), pad_id, dtype=torch.long)
    for b in range(B):
        s = ""
        while len(s) < T:
            a, b2 = torch.randint(0,10,(1,)).item(), torch.randint(0,10,(1,)).item()
            s += f"{a}+{b2}="
            s += str((a+b2)%10)  # 단순화
            s += " "
        s = s[:T]
        X[b] = encode(s)
    return X

# 2) 모델
model = GPTMini(vocab_size=V, d_model=192, nlayer=4, nhead=6, d_ff=768, max_len=128, p_drop=0.1, learned_pos=True)
opt = torch.optim.AdamW(model.parameters(), lr=3e-4, weight_decay=0.05)

# 3) 학습 루프 (LM: shift target)
for step in range(200):
    xb = make_batch(B=64, T=64)
    # LM target: 다음 토큰
    x = xb[:, :-1]; y = xb[:, 1:]
    mask = build_decoder_mask(x, pad_id=pad_id)  # (B,1,T-1,T-1)
    logits = model(x, attn_mask=mask)
    loss = F.cross_entropy(logits.reshape(-1, V), y.reshape(-1), ignore_index=pad_id, label_smoothing=0.05)
    opt.zero_grad(set_to_none=True); loss.backward()
    torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
    opt.step()
    if (step+1) % 50 == 0:
        print(f"step {step+1} | loss {loss.item():.3f}")
```

---

## J. 디코딩 전략(생성 Inference)

생성 모델에서 **다음 토큰을 어떻게 고를 것인가**가 품질/다양성에 큰 영향을 줍니다.

### J-1. 그리디(Argmax)
- 매 스텝 $$\arg\max p(\cdot|{\small \text{prefix}})$$ 선택. **결정적** / 반복·루프 위험.

```python
@torch.no_grad()
def greedy_decode(model, prompt_ids, max_new_tokens=50, pad_id=None):
    x = prompt_ids.clone()
    for _ in range(max_new_tokens):
        T = x.size(1)
        mask = build_decoder_mask(x, pad_id=pad_id)
        logits = model(x, attn_mask=mask)[:, -1, :]  # 마지막 토큰 분포
        next_id = logits.argmax(dim=-1, keepdim=True)
        x = torch.cat([x, next_id], dim=1)
    return x
```

### J-2. 빔 서치(Beam Search)
- 상위 $$B$$개의 **부분 시퀀스**를 유지하며 확장 → **우도 최대** 근사.
- **길이 편향**을 막기 위해 **length penalty** $$\frac{\log P}{(5+|y|)^\alpha/(5+1)^\alpha}$$ 등을 사용.

```python
@torch.no_grad()
def beam_search(model, prompt_ids, beam_size=4, max_new_tokens=50, pad_id=None, alpha=0.7):
    B = prompt_ids.size(0)
    beams = [(prompt_ids[b:b+1], 0.0) for b in range(B)]  # (tokens, logprob)
    for _ in range(max_new_tokens):
        candidates = []
        for tokens, logp in beams:
            mask = build_decoder_mask(tokens, pad_id=pad_id)
            logits = model(tokens, attn_mask=mask)[:, -1, :]
            log_probs = F.log_softmax(logits, dim=-1)  # (1,V)
            topk = torch.topk(log_probs, k=beam_size, dim=-1)
            for k in range(beam_size):
                nid = topk.indices[0, k].view(1,1)
                nlp = topk.values[0, k].item()
                cand = (torch.cat([tokens, nid], dim=1), logp + nlp)
                candidates.append(cand)
        # 길이 패널티 적용하여 상위 beam_size 선택
        scored = []
        for tok, lp in candidates:
            L = tok.size(1)
            lp_ = lp / (((5+L)**alpha)/((5+1)**alpha))
            scored.append((tok, lp_, lp))  # (tokens, score, raw_lp)
        scored.sort(key=lambda x: x[1], reverse=True)
        beams = [(t, raw) for (t, _, raw) in scored[:beam_size]]
    # 최종 최고점
    beams.sort(key=lambda x: x[1], reverse=True)
    return beams[0][0]
```

### J-3. 샘플링(Temperature · Top-k · Nucleus Top-p)
- **Temperature** $$\tau$$: $$\mathrm{softmax}(z/\tau)$$, $$\tau>1$$ → 평탄화(**창의**↑), $$\tau<1$$ → 날카로움(**정확**↑)
- **Top-k**: 상위 k 토큰만 남기고 재정규화.  
- **Top-p(Nucleus)**: 누적 확률이 p가 될 때까지 토큰 집합 선택 → 적응적 k.

```python
def top_k_top_p_filtering(logits, top_k=0, top_p=1.0, filter_value=-float('inf')):
    """
    logits: (B,V)
    """
    B, V = logits.shape
    if top_k > 0:
        kth = torch.topk(logits, top_k, dim=-1).values[:, -1].unsqueeze(-1)  # (B,1)
        mask = logits < kth
        logits = logits.masked_fill(mask, filter_value)
    if top_p < 1.0:
        sorted_logits, sorted_idx = torch.sort(logits, descending=True, dim=-1)
        cum_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)
        mask = cum_probs > top_p
        # 맨 첫 토큰은 항상 허용
        mask[:, 0] = False
        # 원래 인덱스로 되돌려 mask
        scatter_mask = torch.zeros_like(logits, dtype=torch.bool).scatter_(1, sorted_idx, mask)
        logits = logits.masked_fill(scatter_mask, filter_value)
    return logits

@torch.no_grad()
def sampling_decode(model, prompt_ids, max_new_tokens=50, pad_id=None, temperature=1.0, top_k=0, top_p=1.0):
    x = prompt_ids.clone()
    for _ in range(max_new_tokens):
        mask = build_decoder_mask(x, pad_id=pad_id)
        logits = model(x, attn_mask=mask)[:, -1, :]
        if temperature != 1.0:
            logits = logits / temperature
        logits = top_k_top_p_filtering(logits, top_k=top_k, top_p=top_p)
        probs = F.softmax(logits, dim=-1)
        next_id = torch.multinomial(probs, num_samples=1)
        x = torch.cat([x, next_id], dim=1)
    return x
```

### J-4. 반복 방지(간단 버전)
- **repetition penalty**(반복된 토큰의 로짓을 나눔/곱함), **no-repeat-ngram**(최근 n-그램 재출현 금지) 등.

```python
def apply_repetition_penalty(logits, generated, penalty=1.2):
    """
    logits: (B,V), generated: (B,T) 지금까지 생성한 토큰
    """
    for b in range(logits.size(0)):
        uniq = torch.unique(generated[b])
        logits[b, uniq] /= penalty
    return logits
```

---

## K. 캐싱(Incremental Decoding)

디코딩 시 매 스텝 **다시 T×T**를 계산하면 비효율.  
→ 과거의 $$\mathbf{K},\mathbf{V}$$ 를 **캐시**해 새 쿼리 부분만 계산.

아이디어:
1) 첫 스텝: 전체 K,V 계산 후 저장  
2) 다음 스텝: 새 토큰의 q/k/v만 계산, K/V를 **concat**하여 길이 1씩 확장  
3) 스코어는 **최근 q** vs **누적 K** 로만 계산

> 본 장의 수기 MHA에 캐시를 넣으려면 `forward(q,k,v, cache_kv=None)` 시그니처로 확장하고, 내부에서 `K = torch.cat([cache.K, K_new], dim=2)` 처리를 합니다(메모리 관리 필수).

---

## L. 수치 안정/성능 팁

1) **마스크 dtype**: 반드시 **bool**로 만들고, scores에 `masked_fill(True, min)` 적용(amp 안전).  
2) **LayerNorm 위치**: **Pre-LN**가 안정(ln→subblock→residual).  
3) **Dropout**: attn probs / out proj / FFN 사이에 적절히(0.0~0.3).  
4) **Weight Decay**: AdamW + no_decay(=bias, norm) 그룹 필수.  
5) **AMP + Grad Clip**: 대형 모델 수렴/안정에 중요.  
6) **시퀀스 길이**: 메모리 $$\propto T^2$$; 필요시 **FlashAttention**(커스텀 커널)·**압축** 기법 고려.

---

## M. 체크리스트(실전)

- [ ] **로짓에 softmax 이중 적용 금지**(CE는 로짓 입력).  
- [ ] **마스크 합성**: causal ∨ padding.  
- [ ] **포지션**: Sinusoidal(간편/외삽) vs Learned(간단/고정 길이) vs RoPE(상대성).  
- [ ] **디코딩**: Greedy(결정) / Beam(정확) / Sampling(다양) — **temperature/top-k/p** 조절.  
- [ ] **캐싱**: KV 캐시로 O(T^2)→O(T) per step.  
- [ ] **Pre-LN + AdamW + Cosine/Warmup + Clip** 기본 레시피.

---

## N. 수식 요약(한 페이지)

- **Scaled Dot-Product**
$$
\mathbf{A}=\mathrm{softmax}\Big(\frac{\mathbf{Q}\mathbf{K}^\top}{\sqrt{d_k}}+\mathbf{M}\Big),\quad
\mathbf{O}=\mathbf{A}\mathbf{V}.
$$

- **Multi-Head**
$$
\mathrm{head}_i=\mathrm{Attn}(\mathbf{X}\mathbf{W}_i^Q,\mathbf{X}\mathbf{W}_i^K,\mathbf{X}\mathbf{W}_i^V),\quad
\mathrm{MHA}=\mathrm{Concat}(\mathrm{head}_1,\dots,\mathrm{head}_h)\mathbf{W}^O.
$$

- **Sinusoidal PE**
$$
\mathrm{PE}_{(pos,2i)}=\sin\!\Big(\frac{pos}{10000^{2i/d}}\Big),\quad
\mathrm{PE}_{(pos,2i+1)}=\cos\!\Big(\frac{pos}{10000^{2i/d}}\Big).
$$

- **Causal Mask**
$$
M_{ij}=\begin{cases}0 & j\le i\\ -\infty & j>i\end{cases}
$$

- **Beam Length Penalty(ex.)**
$$
\mathrm{score}(y)=\frac{\log P(y)}{((5+|y|)^\alpha/(5+1)^\alpha)}.
$$

---

## O. 연습 과제

1) 위 `GPTMini`로 **N-그램 반복 금지**를 추가해 문장 반복성을 줄여보세요.  
2) RoPE를 **헤드별로** 정확히 적용하는 버전을 작성하고, 길이 외삽 성능을 비교하세요.  
3) Cross-Attn을 추가해 **Encoder-Decoder** 번역 미니 모델을 완성하세요(패딩 마스크·카우절/크로스 마스크 결합).  
4) **Temperature/Top-k/Top-p**를 조합해 생성 스타일 변화를 관찰하세요(샘플 텍스트 로그).  
5) KV 캐시를 구현하고, 같은 프롬프트에서 디코딩 속도 비교(토큰/초).

---

### 마무리
**어텐션**은 “**가중 합으로 문맥을 모으는 연산**”, **Transformer**는 이를 **정규화/FFN/잔차**와 결합한 **범용 시퀀스 블록**입니다.  
포지션·마스크·디코딩·캐시까지 이해하면 **사전학습 LM/RAG/번역/비전 트랜스포머** 등 대부분의 현대 딥러닝 모델을 **읽고, 수정하고, 직접 구현**할 수 있습니다.