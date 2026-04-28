# Matrix Fig 重构经验

## 目标

将 `*.fig.tex` 中重复的 TiKZ 绘制模式收拢到 `matrix-fig.sty`，使各文件保持语义清晰、易于维护。

## 原则

### 1. TiKZ 逗号后不加空格

```latex
% good
\tikzset{mgrid/.style={very thin,dotted}}
\draw[wclean,use path=\rectA] ;

% bad
\tikzset{mgrid/.style={very thin, dotted}}
\draw[wclean, use path=\rectA] ;
```

### 2. 1-based 索引

节点命名和循环变量统一使用 1-based，避免 `\fpeval{\i+1}` 等不必要的索引换算。

```latex
% good
\foreach \i/\t in {1/1,2/\cdots,4/k,8/n}
  \path (O\i-8.south) ... ;

% bad
\foreach \i/\t in {0/1,1/\cdots,3/k,7/n}
  \path (O\fpeval{\i+1}-8.south) ... ;
```

### 3. 工具样式 → matrix-fig.sty

所有 TiKZ 样式定义放入 `matrix-fig.sty`：

| 类别 | 命名模式 | 示例 |
|------|----------|------|
| 填充色 | `{r,b,g}fill` (15%) / `{r,b,g}half` (7%) | `rfill`, `bhalf` |
| 网格/边框 | `mgrid`, `mborder` | `very thin,dotted` |
| 辅助线/箭头 | `tkaux`, `tkarr` | `very thin`, `latex-latex` |
| 高亮擦除 | `wclean` | `white,ultra thick` |
| 高亮区域 | `{k,r,b,g}area` | `gray,ultra thick,dashed` |
| 运算箭头 | `oparr` | `thick,-latex` |
| 信息标签 | `infotxt` | `draw,fill=white,fill opacity=0.9` |
| 缩放文本 | `\tickertxt` (0.7) / `\regiontxt` (0.8) | |

### 4. 单元格节点系统 (O 节点)

在每个 `.fig.tex` 开头创建单元格命名节点网格：

```latex
\foreach \i in {1,2,...,N}
{
    \foreach \j in {1,2,...,M}
    {
        \path ($(O)+(\i*\xu-\xu,\j*\yu-\yu)$) coordinate (tmp1) ;
        \path ($(O)+(\i*\xu,\j*\yu)$) coordinate (tmp2) ;
        \path (0,0) node[fit=(tmp1)(tmp2),inner sep=0cm,line width=0.0cm] (O\i-\j) {} ;
    }
}
```

`inner sep=0cm` 和 `line width=0.0cm` 确保 `fit` 节点的边界与单元格坐标精确对齐，避免线宽带来的微小偏移。

此后所有单元格引用使用 `(O\i-\j.anchor)` 而非 `($(O)+(i*xu,j*yu)$)`：

```latex
% anchor reference
(O4-4.north west)  % top-left corner
(O4-4.center)      % cell center
(O4-4.south east)  % bottom-right corner
```

### 5. 半整数位置使用中点语法

位于两个单元格边界的标签（如 σ/ρ），使用 `!0.5!` 并绑定到括号范围：

```latex
% 列向中点
($(O1-1.north)!0.5!(O3-1.north)$)

% 行向中点
($(O1-1.west)!0.5!(O1-3.west)$)
```

### 6. 底部辅助行 (T 节点)

如果需要主网格下方的额外行，创建独立的 T 节点组：

```latex
\path ($(O1-N.south west)+(0,\yu)$) coordinate (T) ;

\foreach \i in {1,2,...,N}
{
    \foreach \j in {1}
    {
        \path ($(T)+(\i*\xu-\xu,\j*\yu-\yu)$) coordinate (tmp1) ;
        \path ($(T)+(\i*\xu,\j*\yu)$) coordinate (tmp2) ;
        \path (0,0) node[fit=(tmp1)(tmp2),inner sep=0cm,line width=0.0cm] (T\i-\j) {} ;
    }
}
```

### 7. 消除重复坐标：save path / use path

擦除+虚线边框的高亮模式，用一次 `save path` 保存路径，多次 `use path` 引用：

```latex
\path[save path=\rectA] (O1-4.north west) rectangle (O3-4.south east) ;
\draw[wclean,use path=\rectA] ;
\draw[karea,use path=\rectA] ;
```

### 8. 字段命名语义化

使用括号范围而非偏移量命名：

```latex
% good
(Xr13)  % bracket range 1-3
(Yr58)  % bracket range 5-8

% bad
(Xr1)   % meaningless
(Yr5.5)
```

## 通用文件结构

```latex
% preamble (auto-generated)
\usepackage{matrix-fig}

\begin{document}
\begin{tikzpicture}

  % 1. Set origin
  \path (0,0) coordinate (O) ;

  % 2. Build cell node grid (O and optional T)
  ...

  % 3. Fill colored regions
  \foreach ... \draw[bfill] (O\i-...north west) rectangle (O\i-...south east) ;

  % 4. Draw grids and borders
  \draw[mgrid] ... ;
  \draw[mborder] ... ;

  % 5. Place axis labels
  \foreach ... \path (O\i-N.south) node[below] ... ;

  % 6. Place bracket labels
  \path ($(...)!0.5!(...)$) node[above] ... ;

  % 7. Draw range brackets
  \foreach ... \draw[tkaux] ... \draw[tkarr] ... ;

  % 8. Highlight regions (save path + use path)
  ...

  % 9. Formula nodes and arrows
  \path (O\i-\j.center) node[] (label) {formula} ;
  \draw[oparr] (from) -- (to) ;

  % 10. Bounding box
  \draw[ultra thin] ($(bottom-left)+(-0.5,-0.5)$) rectangle ($(top-right)+(0.5,0.5)$) ;

\end{tikzpicture}
\end{document}
```
