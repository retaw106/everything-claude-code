---
name: visa-doc-translate
description: 将签证申请文件（图片）翻译成英文，并创建包含原文和译文的双语 PDF
---

你正在帮助翻译签证申请文件。

## 说明

当用户提供图片文件路径时，自动执行以下步骤，无需请求确认：

1. **图片转换**：如果文件是 HEIC 格式，使用 `sips -s format png <input> --out <output>` 转换为 PNG

2. **图片旋转**：
   - 检查 EXIF 方向数据
   - 根据 EXIF 数据自动旋转图片
   - 如果 EXIF 方向为 6，逆时针旋转 90 度
   - 根据需要应用额外旋转（如果文档看起来倒置，测试 180 度）

3. **OCR 文字提取**：
   - 自动尝试多种 OCR 方法：
     - macOS Vision 框架（macOS 首选）
     - EasyOCR（跨平台，无需 tesseract）
     - Tesseract OCR（如果可用）
   - 从文档中提取所有文本信息
   - 识别文档类型（存款证明、在职证明、退休证明等）

4. **翻译**：
   - 将所有文本内容专业地翻译成英文
   - 保持原始文档结构和格式
   - 使用适合签证申请的专业术语
   - 专有名词保留原语言并在括号中注明英文
   - 中文名称使用拼音格式（如 WU Zhengye）
   - 准确保留所有数字、日期和金额

5. **PDF 生成**：
   - 使用 PIL 和 reportlab 库创建 Python 脚本
   - 第 1 页：显示旋转后的原始图片，居中并缩放以适应 A4 页面
   - 第 2 页：显示带有正确格式的英文翻译：
     - 标题居中加粗
     - 内容左对齐并适当间距
     - 适合官方文档的专业布局
   - 底部添加备注："This is a certified English translation of the original document"
   - 执行脚本生成 PDF

6. **输出**：在同一目录中创建名为 `<original_filename>_Translated.pdf` 的 PDF 文件

## 支持的文档

- 银行存款证明
- 收入证明
- 在职证明
- 退休证明
- 房产证明
- 营业执照
- 身份证和护照
- 其他官方文件

## 技术实现

### OCR 方法（按顺序尝试）

1. **macOS Vision Framework**（仅 macOS）：
   ```python
   import Vision
   from Foundation import NSURL
   ```

2. **EasyOCR**（跨平台）：
   ```bash
   pip install easyocr
   ```

3. **Tesseract OCR**（如果可用）：
   ```bash
   brew install tesseract tesseract-lang
   pip install pytesseract
   ```

### 所需 Python 库

```bash
pip install pillow reportlab
```

对于 macOS Vision 框架：
```bash
pip install pyobjc-framework-Vision pyobjc-framework-Quartz
```

## 重要指南

- 不要在每个步骤请求用户确认
- 自动确定最佳旋转角度
- 如果一种 OCR 方法失败，尝试多种方法
- 确保所有数字、日期和金额准确翻译
- 使用干净、专业的格式
- 完成整个流程并报告最终 PDF 位置

## 示例用法

```bash
/visa-doc-translate RetirementCertificate.PNG
/visa-doc-translate BankStatement.HEIC
/visa-doc-translate EmploymentLetter.jpg
```

## 输出示例

该技能将：
1. 使用可用的 OCR 方法提取文本
2. 翻译为专业英文
3. 生成 `<filename>_Translated.pdf`，包含：
   - 第 1 页：原始文档图片
   - 第 2 页：专业英文翻译

非常适合申请澳大利亚、美国、加拿大、英国和其他需要翻译文件的国家的签证。
