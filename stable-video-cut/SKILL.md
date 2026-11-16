# stable-video-cut

从一段(或多段)实拍素材中自动找出"画面稳定"的片段,剪出来,再按稳定度排序拼成一条带转场的成片。全程只用 ffmpeg + Python(numpy),无需任何剪辑软件。

## 何时使用

- "找稳定的画面剪一下 / 画面晃动的太多,挑稳的"
- "把这些素材合成一个视频,拼接处加转场"
- 手机/手持拍摄的工厂、产品展示、案例视频,需要挑出能直接用的段落

## 环境依赖

- ffmpeg / ffprobe:先 `which ffmpeg ffprobe`(或 `where ffmpeg`)确认已安装。Windows 可用 winget(`winget install Gyan.FFmpeg`)或从 gyan.dev 下载;Mac 用 `brew install ffmpeg`。下文命令一律用 `ffmpeg` / `ffprobe` 可执行文件名,不再写死绝对路径;若不在 PATH,先导出 `FF=/path/to/ffmpeg`、`FP=/path/to/ffprobe` 再用 `"$FF"` 等替代。
- Python 3 + numpy(`python --version` / `python -c "import numpy"` 确认;缺则 `pip install numpy`)。
- 涉及封面标题时另需 Pillow(`pip install pillow`)。

## 流程

### 1. 探测素材

对每个视频跑 ffprobe,拿到 分辨率 / 帧率 / 时长:

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate,duration \
  -show_entries format=duration -of default=noprint_wrappers=1 <file>
```

记录各片规格——不同片拼接前必须统一(步骤 4)。

### 2. 稳定性分析 + 剪片段

把下面的脚本存到素材目录(如 `_stability_cut.py`)再运行;它完成:帧差分析 → 自适应阈值找稳定段 → 剪成 mp4(带原音)→ 输出 `稳定剪辑/分析报告.json`。

关键参数都在文件开头,可按需调:

```python
import subprocess, numpy as np, os, json, shutil
SRC = r"E:\你的素材文件夹"          # 改成实际路径
OUT = os.path.join(SRC, "稳定剪辑")
FF  = shutil.which("ffmpeg")        # 不在 PATH 时改成实际路径
FPS = 5          # 采样帧率:分析用,越大越精细越慢
W   = 160        # 分析灰度图宽度
MIN_SEC = 4.0    # 最短保留片段秒数
TOP_N = 4        # 每个原片最多保留几段
THR_PCT = 45     # 自适应阈值 = 本片帧差分布的 P45(越小说出的段越稳但越少)

def gray(path):
    p = subprocess.run([FF,"-hide_banner","-loglevel","error","-i",path,
        "-vf",f"scale={W}:-2,fps={FPS}","-f","rawvideo","-pix_fmt","gray","-"],
        stdout=subprocess.PIPE)
    fp = FF.replace("ffmpeg","ffprobe")
    if shutil.which("ffprobe"):
        fp = shutil.which("ffprobe")
    pr = subprocess.run([fp,"-v","error",
        "-select_streams","v:0","-show_entries","stream=width,height","-of","json",path],
        capture_output=True,text=True)
    w,h = json.loads(pr.stdout)["streams"][0]["width"], json.loads(pr.stdout)["streams"][0]["height"]
    h = (h*W//w)//2*2
    return np.frombuffer(p.stdout,np.uint8).reshape(-1,h,W).astype(np.float32)

def smooth(x,w=5):
    k=np.ones(w)/w; return np.convolve(x,k,mode="same")

report={}
for name in sorted(f for f in os.listdir(SRC) if f.lower().endswith((".mp4",".mov",".m4v"))):
    f = gray(os.path.join(SRC,name)); n=f.shape[0]
    D = np.mean(np.abs(f[1:]-f[:-1]),axis=(1,2))   # 相邻帧平均差 = 波动
    S = smooth(D)
    thr = float(np.percentile(D,THR_PCT))
    good = S<thr
    runs,start=[],None
    for i in range(len(good)):
        if good[i] and start is None: start=i
        if not good[i] and start is not None: runs.append((start,i)); start=None
    if start is not None: runs.append((start,len(good)))
    runs=[(a,b) for a,b in runs if (b-a)>=FPS*MIN_SEC]
    runs.sort(key=lambda ab: float(S[ab[0]:ab[1]].mean()))   # 按最稳排序
    top=runs[:TOP_N]; top.sort(key=lambda ab:ab[0])
    os.makedirs(OUT,exist_ok=True)
    base=os.path.splitext(name)[0]
    for i,(a,b) in enumerate(top,1):
        outp=os.path.join(OUT,f"{base}_稳定{i}_{a/FPS:05.1f}s-{b/FPS:05.1f}s.mp4")
        subprocess.run([FF,"-hide_banner","-loglevel","error","-y","-ss",f"{a/FPS:.2f}",
            "-i",os.path.join(SRC,name),"-t",f"{(b-a)/FPS:.2f}",
            "-c:v","libx264","-preset","veryfast","-crf","20",
            "-c:a","aac","-b:a","128k","-movflags","+faststart",outp],check=True)
    report[name]=[{"片段":i,"起":a/FPS,"止":b/FPS,"时长s":(b-a)/FPS,
                   "波动":round(float(S[a:b].mean()),2)} for i,(a,b) in enumerate(top,1)]
    print(name,"->",len(top),"段", [(f"{a/FPS:.1f}-{b/FPS:.1f}s",round(float(S[a:b].mean()),1)) for a,b in top])
json.dump(report, open(os.path.join(OUT,"分析报告.json"),"w",encoding="utf-8"), ensure_ascii=False, indent=2)
```

### 3. 排"稳定度总表"

把所有片子的段按 `波动` 升序汇总成一张表(排名、原片、时间区间、时长、推荐用途),写成 `稳定剪辑/稳定度排名总表.md`。

经验:
- 波动值越小越稳,但它是**相对本片**的量纲,跨片只能排序参考,不能当绝对值。
- "重点用哪支"一般就是波动最低的那支;最长且稳的段适合当主画面。
- 只有稳定段偏短(<4s 居多)时,把 `THR_PCT` 调大到 55-60 重跑。

### 4. 统一规格 + 拼接转场

**先把全部片段归一化**(分辨率、帧率、音频参数必须完全一致,xfade 才不会报错):

```bash
for f in 片段*.mp4; do
  ffmpeg -y -i "$f" -vf "scale=720:1280:force_original_aspect_ratio=decrease,pad=720:1280:(ow-iw)/2:(oh-ih)/2,setsar=1" \
    -r 30 -c:v libx264 -crf 20 -pix_fmt yuv420p -c:a aac -b:a 128k -ar 44100 -ac 2 "_norm/$(printf '%02d' $i).mp4"
done
```

- 竖屏素材用 `720:1280`;横屏用 `1280:720`;方屏 `1080:1080`。
- 源片比目标更清晰时,`force_original_aspect_ratio=decrease` 会缩到刚好放进目标画框,`pad` 居中补黑边,不变形。
- 归一化文件一律用 **ASCII 文件名**(01.mp4…)放 `_norm/` 子目录——避免下面 concat 的中文路径坑。

**xfade 拼接**:12 段、每段 0.5s 转场时,filter 这样构造:

```
[0:v][1:v]xfade=transition=<T1>:duration=0.5:offset=<o1>[v1];
[v1][2:v]xfade=transition=<T2>:duration=0.5:offset=<o2>[v2]; ... 最后 -> [vout]
[0:a][1:a]acrossfade=d=0.5[a1]; [a1][2:a]acrossfade=d=0.5[a2]; ... -> [aout]
```

offset 递推公式(转场时长 `d`,各片时长 `D1..Dn`):

```
o1 = D1 - d
o2 = o1 + D2 - d
o3 = o2 + D3 - d   # 即前一段输出时长 - d,逐段累加
```

D 用 ffprobe 精确取(注意有些是 16.400023 这种,别手算 16.4)。总时长 = ΣDi - (n-1)·d。

音频用 `acrossfade` 同步淡化,避免拼接处"咔嚓"声。一次 filter_complex 全部搞定,不要逐对 concat demuxer(那样加不了转场)。

常用转场(每次挑不同的,避免重复感):
`fade`(叠化,最安全)、`fadeblack`(换分组/收尾)、`slideleft`/`slideright`/`smoothleft`、`circleopen`、`radial`、`hlslice`、`zoomin`。11 段以上建议把 fadeblack 用在第 4-5 处和最后,做分组。

```bash
ffmpeg -y -i _norm/01.mp4 ... -i _norm/12.mp4 -filter_complex "$FILTER" \
  -map "[vout]" -map "[aout]" -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p \
  -c:a aac -b:a 128k -movflags +faststart 成片名.mp4
```

### 5. 封面标题 + 背景音乐(成片包装)

**核心技巧:标题文字用 PIL 生成透明 PNG,不要用 drawtext 画中文**——drawtext 在 Windows 下中文路径转义极难调试(冒号要 `\:`、`textfile` 路径不能带空格/中文、`filter_complex_script` 在 ffmpeg 9 里已不存在),而 PIL 一次成功:

```python
from PIL import Image, ImageDraw, ImageFont
F = "C:/Windows/Fonts/msyhbd.ttc"        # 微软雅黑粗体, 支持中文; Linux/macOS 换用思源黑体等
img = Image.new("RGBA", (720, 1280), (0,0,0,0))
d = ImageDraw.Draw(img)
fnt = ImageFont.truetype(F, 62)
# 先画阴影再画正文, 保证在亮/暗背景上都清晰
d.text((x+3, y+3), "宠物洗护流水线", font=fnt, fill=(0,0,0,160))
d.text((x, y), "宠物洗护流水线", font=fnt, fill=(255,255,255,255))
img.save("title.png")
```

视频里叠加(前 4.5s 淡入淡出, 透明 PNG 用 loop 拉成 4.5s):

```
[1:v]loop=loop=449:size=1:start=0,fps=30,format=rgba,
      fade=t=in:st=0:d=0.4:alpha=1,fade=t=out:st=3.8:d=0.7:alpha=1,
      trim=duration=4.5,setpts=PTS-STARTPTS[ov];
[0:v][ov]overlay=0:0:eof_action=pass,format=yuv420p[vout]
```

BGM 混音(原声压低 0.45, BGM 0.9, 首尾 1-2s 淡入淡出):

```
[0:a]volume=0.45[a1];
[1:a]atrim=0:<总时长>,afade=t=in:d=1,afade=t=out:st=<总时长-2>:d=2,volume=0.9[b1];
[a1][b1]amix=inputs=2:duration=first:normalize=0[aout]
```

- BGM 自备:先用 ffprobe 看时长,不够长用 `-stream_loop -1` 循环。注意使用免版权音乐(如 lofi/CC0 音源)。
- **封面图**:取最稳段中间一帧,同样的 PIL 标题叠上去,720×1280,存 PNG/JPG。

### 6. 输出约定

- 素材目录/`稳定剪辑/` ← 各段稳定片段 + `分析报告.json` + `稳定度排名总表.md` + `成片_标题+BGM.mp4` + `封面_720x1280.png`
- 归一化中间文件 `_norm/`、临时 PNG 用完删掉

## 常见坑

1. **concat demuxer + 中文/空格路径**:ffmpeg 报 `Illegal byte sequence`。解法就是上面说的 ASCII 中间文件 + 相对路径 concat.txt。
2. **xfade 要求两输入完全同规格**(分辨率/帧率/像素格式/采样率/声道数不一致就报错或结果错乱)——第 4 步的归一化就是干这个的。
3. **Bash on Windows 里中文输出成乱码**(`΢`、`????`)不影响实际剪出的文件(文件名是对的),只是 console 显示问题;别被吓到,用 `ls` 验证即可。
4. 分析耗时随视频时长线性,几支 1-4 分钟的手持视频,整个流程(分析+剪辑+拼接)约 3-5 分钟。
5. **中文标题别用 drawtext**(见第 5 步),PIL 生成透明 PNG 再 overlay 才稳。
