---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python 安全

> 本文件扩展 [common/security.md](../common/security.md) 以包含 Python 特定内容。

## 机密管理

```python
import os
from dotenv import load_dotenv

load_dotenv()

api_key = os.environ["OPENAI_API_KEY"]  # 如果缺少则引发 KeyError
```

## 安全扫描

- 使用 **bandit** 进行静态安全分析：
  ```bash
  bandit -r src/
  ```

## 参考

查看技能：`django-security` 了解 Django 特定的安全指南（如果适用）。
