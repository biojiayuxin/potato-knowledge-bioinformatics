---
name: extract-cds-pep-from-gff3
description: 从马铃薯基因组 FASTA 和 GFF3 注释中提取 CDS 与蛋白 FASTA，特别适用于 DMv8.2 这类存在全量转录本注释和代表转录本注释两套文件的数据集。
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [potato, genomics, gff3, fasta, cds, protein, translation, dm8.2]
    related_skills: [potato-domain-router]
---

# 从 GFF3 提取 CDS 和蛋白序列

## 适用场景

当用户提供：
- 基因组序列 FASTA
- 基因注释 GFF3

并要求生成：
- CDS fasta
- 蛋白 fasta

尤其适用于马铃薯 DMv8.2 数据：
- `DM8.2_genome_HiFi_ONT.fasta`
- `DM8.2_genome_HiFi_ONT_gene.gff3`
- `DMv8.2.repre.gff3`

## 先做的数据判定

先检查用户到底要：

1. **gene-level 一基因一条序列**
   - 优先寻找代表转录本 GFF3
   - 对 DMv8.2，优先用 `DMv8.2.repre.gff3`

2. **transcript-level 保留全部 isoform**
   - 用全量注释 GFF3
   - 对 DMv8.2，用 `DM8.2_genome_HiFi_ONT_gene.gff3`

### DMv8.2 的经验结论

- `DM8.2_genome_HiFi_ONT_gene.gff3` 是全量注释，含 gene / mRNA / exon / CDS
- `DMv8.2.repre.gff3` 是代表转录本注释，只有 mRNA + CDS
- DMv8.2 中代表转录本数约等于有 CDS 的 gene 数，因此如果用户说“基因的 CDS/蛋白”，通常应该优先推荐 `DMv8.2.repre.gff3`

## 优先方案：gffread

若环境有 `gffread`，优先使用。

### 当前服务器可复用的安装方式（经验）

若系统未安装 `gffread`，可优先检查公用 `micromamba`。在当前环境中可用路径为：

```bash
/opt/micromamba/bin/micromamba
```

推荐显式设置 root prefix，避免把环境装到不可控位置：

```bash
export MAMBA_ROOT_PREFIX=/mnt/data/potato_agent/.micromamba
/opt/micromamba/bin/micromamba create -y -n dmv8-gffread -c conda-forge -c bioconda gffread
/opt/micromamba/bin/micromamba run -n dmv8-gffread gffread --version
```

之后直接用 `micromamba run -n dmv8-gffread` 执行，不必依赖 shell activate。

### 代表转录本（推荐）
```bash
gffread DMv8.2.repre.gff3 \
  -g DM8.2_genome_HiFi_ONT.fasta \
  -x DMv8.2.cds.fa \
  -y DMv8.2.pep.fa
```

### 全部转录本
```bash
gffread DM8.2_genome_HiFi_ONT_gene.gff3 \
  -g DM8.2_genome_HiFi_ONT.fasta \
  -x DM8.2.all_transcripts.cds.fa \
  -y DM8.2.all_transcripts.pep.fa
```

## 环境缺少常用工具时的回退方案

如果 `gffread / bedtools / seqkit / samtools / Biopython` 都没有，但有 `python3`，直接写纯 Python 脚本：

1. 读入 genome FASTA
2. 解析 GFF3 的 `mRNA` 与 `CDS`
3. 按 `Parent` 聚合 CDS 到 transcript
4. 按坐标排序并拼接 CDS
5. 负链做 reverse complement
6. 用标准密码子表翻译蛋白
7. 输出 `.cds.fa` 和 `.pep.fa`

## 纯 Python 验证要点

生成结果后至少检查：

- CDS 长度是否都能被 3 整除
- 蛋白是否存在内部终止密码子
- 是否具有合理终止位点

### DMv8.2 代表转录本的已验证特征

对 `DMv8.2.repre.gff3`：
- CDS 长度均可被 3 整除
- 无内部 stop codon
- 翻译结果均有终止位点

因此该文件适合直接作为 gene-level CDS/蛋白提取输入。

## 实操检查命令

### 检查环境工具
```bash
for cmd in gffread agat_sp_extract_sequences.pl bedtools seqkit samtools python3 perl; do
  printf '%s\t%s\n' "$cmd" "$(command -v "$cmd" 2>/dev/null || echo NA)"
done
```

### 统计 GFF3 特征数量
```bash
python3 - <<'PY'
from collections import Counter
for fn in ['DM8.2_genome_HiFi_ONT_gene.gff3','DMv8.2.repre.gff3']:
    c=Counter()
    with open(fn) as f:
        for line in f:
            if line.startswith('#') or not line.strip():
                continue
            parts=line.rstrip('\n').split('\t')
            if len(parts) >= 3:
                c[parts[2]] += 1
    print(fn, dict(c))
PY
```

## 常见坑

- 用户口头上可能会说 `DMv8.2.fa` / `DMv8.2.gff3`，但实际目录中的文件名常常不是这两个名字；对当前 DMv8.2 数据，实际常见文件是：
  - `DM8.2_genome_HiFi_ONT.fasta`
  - `DM8.2_genome_HiFi_ONT_gene.gff3`
  - `DMv8.2.repre.gff3`
  开始前一定先列目录确认真实文件名，不要按口头文件名直接执行。
- 用户说“基因序列”，但全量 GFF3 实际输出是 transcript-level，不是 gene-level
- `repre.gff3` 可能没有 `gene` feature，只有 `mRNA` + `CDS`，这是正常的
- FASTA header 应与 GFF3 第 1 列染色体名完全一致
- 输出 header 默认常是 transcript ID；若用户想改成 gene ID，需要额外改名
- `gffread` 首次运行时可能自动创建 genome `.fai` 索引文件，这是正常现象

## 结果核对

运行完至少检查：

```bash
python3 - <<'PY'
from pathlib import Path
for fn in ['DMv8.2.cds.fa','DMv8.2.pep.fa']:
    p=Path(fn)
    print(fn, 'exists=', p.exists(), 'size=', p.stat().st_size if p.exists() else None)
    n=0
    heads=[]
    with p.open() as f:
        for line in f:
            if line.startswith('>'):
                n += 1
                if len(heads) < 3:
                    heads.append(line.strip())
    print('seq_count=', n)
    print('\n'.join(heads))
PY
```

对当前 DMv8.2 代表转录本文件，预期两份 FASTA 都约为 **37658 条序列**。

## 决策规则

- 用户没特别说明时：默认推荐代表转录本版本
- 用户明确要所有 isoform 时：使用全量 GFF3
- 环境无现成工具时：直接走纯 Python 回退，不必卡在安装软件
