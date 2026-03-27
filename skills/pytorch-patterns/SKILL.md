---
name: pytorch-patterns
description: PyTorch 深度学习模式和最佳实践，用于构建健壮、高效且可复现的训练流水线、模型架构和数据加载。
origin: ECC
---

# PyTorch 开发模式

构建健壮、高效且可复现深度学习应用的惯用 PyTorch 模式和最佳实践。

## 何时启用

- 编写新的 PyTorch 模型或训练脚本
- 审查深度学习代码
- 调试训练循环或数据流水线
- 优化 GPU 内存使用或训练速度
- 设置可复现实验

## 核心原则

### 1. 设备无关代码

始终编写无需硬编码设备即可在 CPU 和 GPU 上运行的代码。

```python
# 好：设备无关
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = MyModel().to(device)
data = data.to(device)

# 坏：硬编码设备
model = MyModel().cuda()  # 无 GPU 时崩溃
data = data.cuda()
```

### 2. 可复现性优先

设置所有随机种子以获得可复现结果。

```python
# 好：完整的可复现性设置
def set_seed(seed: int = 42) -> None:
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    np.random.seed(seed)
    random.seed(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

# 坏：无种子控制
model = MyModel()  # 每次运行权重不同
```

### 3. 显式形状管理

始终记录和验证张量形状。

```python
# 好：带形状注释的前向传播
def forward(self, x: torch.Tensor) -> torch.Tensor:
    # x: (batch_size, channels, height, width)
    x = self.conv1(x)    # -> (batch_size, 32, H, W)
    x = self.pool(x)     # -> (batch_size, 32, H//2, W//2)
    x = x.view(x.size(0), -1)  # -> (batch_size, 32*H//2*W//2)
    return self.fc(x)    # -> (batch_size, num_classes)

# 坏：无形状跟踪
def forward(self, x):
    x = self.conv1(x)
    x = self.pool(x)
    x = x.view(x.size(0), -1)  # 这是什么尺寸？
    return self.fc(x)           # 这能工作吗？
```

## 模型架构模式

### 干净的 nn.Module 结构

```python
# 好：组织良好的模块
class ImageClassifier(nn.Module):
    def __init__(self, num_classes: int, dropout: float = 0.5) -> None:
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
        )
        self.classifier = nn.Sequential(
            nn.Dropout(dropout),
            nn.Linear(64 * 16 * 16, num_classes),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.features(x)
        x = x.view(x.size(0), -1)
        return self.classifier(x)

# 坏：所有东西都在 forward 中
class ImageClassifier(nn.Module):
    def __init__(self):
        super().__init__()

    def forward(self, x):
        x = F.conv2d(x, weight=self.make_weight())  # 每次调用都创建权重！
        return x
```

### 正确的权重初始化

```python
# 好：显式初始化
def _init_weights(self, module: nn.Module) -> None:
    if isinstance(module, nn.Linear):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
        if module.bias is not None:
            nn.init.zeros_(module.bias)
    elif isinstance(module, nn.Conv2d):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
    elif isinstance(module, nn.BatchNorm2d):
        nn.init.ones_(module.weight)
        nn.init.zeros_(module.bias)

model = MyModel()
model.apply(model._init_weights)
```

## 训练循环模式

### 标准训练循环

```python
# 好：遵循最佳实践的完整训练循环
def train_one_epoch(
    model: nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    criterion: nn.Module,
    device: torch.device,
    scaler: torch.amp.GradScaler | None = None,
) -> float:
    model.train()  # 始终设置训练模式
    total_loss = 0.0

    for batch_idx, (data, target) in enumerate(dataloader):
        data, target = data.to(device), target.to(device)

        optimizer.zero_grad(set_to_none=True)  # 比 zero_grad() 更高效

        # 混合精度训练
        with torch.amp.autocast("cuda", enabled=scaler is not None):
            output = model(data)
            loss = criterion(output, target)

        if scaler is not None:
            scaler.scale(loss).backward()
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            scaler.step(optimizer)
            scaler.update()
        else:
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()

        total_loss += loss.item()

    return total_loss / len(dataloader)
```

### 验证循环

```python
# 好：正确的评估
@torch.no_grad()  # 比 torch.no_grad() 块包裹更高效
def evaluate(
    model: nn.Module,
    dataloader: DataLoader,
    criterion: nn.Module,
    device: torch.device,
) -> tuple[float, float]:
    model.eval()  # 始终设置评估模式 — 禁用 dropout，使用运行中的 BN 统计
    total_loss = 0.0
    correct = 0
    total = 0

    for data, target in dataloader:
        data, target = data.to(device), target.to(device)
        output = model(data)
        total_loss += criterion(output, target).item()
        correct += (output.argmax(1) == target).sum().item()
        total += target.size(0)

    return total_loss / len(dataloader), correct / total
```

## 数据流水线模式

### 自定义 Dataset

```python
# 好：带有类型提示的干净 Dataset
class ImageDataset(Dataset):
    def __init__(
        self,
        image_dir: str,
        labels: dict[str, int],
        transform: transforms.Compose | None = None,
    ) -> None:
        self.image_paths = list(Path(image_dir).glob("*.jpg"))
        self.labels = labels
        self.transform = transform

    def __len__(self) -> int:
        return len(self.image_paths)

    def __getitem__(self, idx: int) -> tuple[torch.Tensor, int]:
        img = Image.open(self.image_paths[idx]).convert("RGB")
        label = self.labels[self.image_paths[idx].stem]

        if self.transform:
            img = self.transform(img)

        return img, label
```

### 高效的 DataLoader 配置

```python
# 好：优化的 DataLoader
dataloader = DataLoader(
    dataset,
    batch_size=32,
    shuffle=True,            # 训练时打乱
    num_workers=4,           # 并行数据加载
    pin_memory=True,         # 更快的 CPU->GPU 传输
    persistent_workers=True, # 在 epoch 之间保持 workers 存活
    drop_last=True,          # 为 BatchNorm 提供一致的批次大小
)

# 坏：慢的默认值
dataloader = DataLoader(dataset, batch_size=32)  # num_workers=0, 无 pin_memory
```

### 用于变长数据的自定义 Collate

```python
# 好：在 collate_fn 中填充序列
def collate_fn(batch: list[tuple[torch.Tensor, int]]) -> tuple[torch.Tensor, torch.Tensor]:
    sequences, labels = zip(*batch)
    # 填充到批次中的最大长度
    padded = nn.utils.rnn.pad_sequence(sequences, batch_first=True, padding_value=0)
    return padded, torch.tensor(labels)

dataloader = DataLoader(dataset, batch_size=32, collate_fn=collate_fn)
```

## 检查点模式

### 保存和加载检查点

```python
# 好：包含所有训练状态的完整检查点
def save_checkpoint(
    model: nn.Module,
    optimizer: torch.optim.Optimizer,
    epoch: int,
    loss: float,
    path: str,
) -> None:
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "loss": loss,
    }, path)

def load_checkpoint(
    path: str,
    model: nn.Module,
    optimizer: torch.optim.Optimizer | None = None,
) -> dict:
    checkpoint = torch.load(path, map_location="cpu", weights_only=True)
    model.load_state_dict(checkpoint["model_state_dict"])
    if optimizer:
        optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
    return checkpoint

# 坏：只保存模型权重（无法恢复训练）
torch.save(model.state_dict(), "model.pt")
```

## 性能优化

### 混合精度训练

```python
# 好：使用 GradScaler 的 AMP
scaler = torch.amp.GradScaler("cuda")
for data, target in dataloader:
    with torch.amp.autocast("cuda"):
        output = model(data)
        loss = criterion(output, target)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
    optimizer.zero_grad(set_to_none=True)
```

### 大模型的梯度检查点

```python
# 好：用计算换内存
from torch.utils.checkpoint import checkpoint

class LargeModel(nn.Module):
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # 在反向传播时重新计算激活以节省内存
        x = checkpoint(self.block1, x, use_reentrant=False)
        x = checkpoint(self.block2, x, use_reentrant=False)
        return self.head(x)
```

### torch.compile 加速

```python
# 好：编译模型以加快执行速度 (PyTorch 2.0+)
model = MyModel().to(device)
model = torch.compile(model, mode="reduce-overhead")

# 模式: "default" (安全), "reduce-overhead" (更快), "max-autotune" (最快)
```

## 快速参考：PyTorch 惯用语

| 惯用语 | 描述 |
|--------|------|
| `model.train()` / `model.eval()` | 训练/评估前始终设置模式 |
| `torch.no_grad()` | 禁用推理时的梯度 |
| `optimizer.zero_grad(set_to_none=True)` | 更高效的梯度清除 |
| `.to(device)` | 设备无关的张量/模型放置 |
| `torch.amp.autocast` | 混合精度，2x 加速 |
| `pin_memory=True` | 更快的 CPU→GPU 数据传输 |
| `torch.compile` | JIT 编译加速 (2.0+) |
| `weights_only=True` | 安全的模型加载 |
| `torch.manual_seed` | 可复现实验 |
| `gradient_checkpointing` | 用计算换内存 |

## 要避免的反模式

```python
# 坏：验证时忘记 model.eval()
model.train()
with torch.no_grad():
    output = model(val_data)  # Dropout 仍然激活！BatchNorm 使用批次统计！

# 好：始终设置评估模式
model.eval()
with torch.no_grad():
    output = model(val_data)

# 坏：破坏 autograd 的原地操作
x = F.relu(x, inplace=True)  # 可能破坏梯度计算
x += residual                  # 原地加法破坏 autograd 图

# 好：非原地操作
x = F.relu(x)
x = x + residual

# 坏：在训练循环中重复将数据移到 GPU
for data, target in dataloader:
    model = model.cuda()  # 每次迭代都移动模型！

# 好：在循环前移动一次模型
model = model.to(device)
for data, target in dataloader:
    data, target = data.to(device), target.to(device)

# 坏：在 backward 前使用 .item()
loss = criterion(output, target).item()  # 从图分离！
loss.backward()  # 错误：无法通过 .item() 反向传播

# 好：仅用于日志记录时调用 .item()
loss = criterion(output, target)
loss.backward()
print(f"Loss: {loss.item():.4f}")  # backward 后的 .item() 没问题

# 坏：不正确地使用 torch.save
torch.save(model, "model.pt")  # 保存整个模型（脆弱，不可移植）

# 好：保存 state_dict
torch.save(model.state_dict(), "model.pt")
```

__记住__：PyTorch 代码应该是设备无关、可复现且内存敏感的。如有疑问，使用 `torch.profiler` 进行性能分析，使用 `torch.cuda.memory_summary()` 检查 GPU 内存。
