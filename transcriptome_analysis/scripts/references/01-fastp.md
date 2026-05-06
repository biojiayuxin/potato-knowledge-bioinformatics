# fastp 质控参数参考

## 功能
双端测序数据质量控制：去除接头、低质量序列、N 过高序列

## 命令模板

```bash
fastp -i {INPUT_R1} -o {OUTPUT_R1} \
      -I {INPUT_R2} -O {OUTPUT_R2} \
      -j {OUTPUT_JSON} \
      -h {OUTPUT_HTML} \
      -w {THREADS}
```

## 参数说明

| 参数 | 说明 | 必需 |
|------|------|------|
| `-i` | 输入 R1 文件 | ✅ |
| `-o` | 输出 R1 文件 | ✅ |
| `-I` | 输入 R2 文件 | ✅ |
| `-O` | 输出 R2 文件 | ✅ |
| `-j` | JSON 报告输出路径 | ✅ |
| `-h` | HTML 报告输出路径 | ✅ |
| `-w` | 线程数 | 推荐 |
| `-q` | 低质量碱基阈值（默认 15） | 可选 |
| `-l` | 最小序列长度（默认 30） | 可选 |
| `--detect_adapter_for_pe` | 自动检测双端接头 | 可选 |
| `--cut_tail` | 切除尾部低质量 | 可选 |

## 输出文件

- `{sample}_R1.fq.gz` - 质控后 R1
- `{sample}_R2.fq.gz` - 质控后 R2
- `{sample}.json` - JSON 格式报告
- `{sample}.html` - HTML 格式报告

## 批量处理示例

```bash
INPUT_DIR="<原始数据目录>"
OUTPUT_DIR="./00-fastp"
THREADS=8

mkdir -p "$OUTPUT_DIR"

for R1 in "$INPUT_DIR"/*_R1.fq.gz; do
    sample=$(basename "$R1" | sed 's/_R1\.fq\.gz//')
    R2="$INPUT_DIR/${sample}_R2.fq.gz"
    
    fastp -i "$R1" -o "$OUTPUT_DIR/${sample}_R1.fq.gz" \
          -I "$R2" -O "$OUTPUT_DIR/${sample}_R2.fq.gz" \
          -j "$OUTPUT_DIR/${sample}.json" \
          -h "$OUTPUT_DIR/${sample}.html" \
          -w "$THREADS"
done
```

## 注意事项

- 默认启用接头检测和去除
- 默认启用质量修剪
- 输出文件自动 gzip 压缩
