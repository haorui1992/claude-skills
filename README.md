# claude-skills

通用的 Claude Code skills 合集,方便团队共享和复用。

## 安装

```bash
# 克隆本仓库
git clone <repo-url>

# 把需要的 skill 文件夹拷到你的 Claude Code skills 目录
# Windows: C:\Users\<你的用户名>\.claude\skills\
# macOS/Linux: ~/.claude/skills/
```

拷完后重启 Claude Code 会话,新 skill 就会出现在可用列表中。

## 包含的 skill

### stable-video-cut

从实拍素材中自动找出"画面稳定"的片段,剪出来,按稳定度排序拼成带转场的成片。
全程只用 ffmpeg + Python(numpy),无需剪辑软件。适合手机/手持拍摄的工厂、产品展示、案例视频。

- 位置:`stable-video-cut/SKILL.md`
- 依赖:ffmpeg/ffprobe + Python 3 + numpy(封面标题环节另需 Pillow)
- 详见该文件夹内 `SKILL.md` 的完整流程说明
