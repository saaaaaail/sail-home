---
title: 好听的音乐
date: 2020-04-19 14:35:09
type: "music"
---

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/aplayer@1.10/dist/APlayer.min.css">
<script src="https://cdn.jsdelivr.net/npm/aplayer@1.10/dist/APlayer.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/meting@1.2/dist/Meting.min.js"></script>


{% meting "386703174" "netease" "playlist" "theme:#555" "mutex:true" "listmaxheight:340px" "preload:auto" %}


# hexo添加hexo-tag-aplayer插件page

## 安装
```
npm install --save hexo-tag-aplayer
```

## 创建新页面

```
hexo new page music
```

## meting支持(aplayer3.0+)
### 配置
在hexo博客根目录配置文件中添加配置

```
aplayer:
  meting: true
```

### 参数

|  选项   | 默认值  | 描述 |
| :-: |  :-:  | :-: |
| id  | 必须值 | 歌曲 id / 播放列表 id / 相册 id / 搜索关键字 |
| server  | 必须值 | 音乐平台: netease, tencent, kugou, xiami, baidu |
| type  | 必须值 | song, playlist, album, search, artist |
| fixed  | false | 开启固定模式 |
| mini | false | 开启迷你模式 |
| loop  | all | 列表循环模式：all, one,none |
| volume  | 0.7 | 播放器音量 |
| lrctype  | 0 | 歌词格式类型 |
| listfolded  | false | 指定音乐播放列表是否折叠 |
| storagename  | metingjs | LocalStorage 中存储播放器设定的键名 |
| autoplay  | true | 自动播放，移动端浏览器暂时不支持此功能 |
| mutex  | true | 该选项开启时，如果同页面有其他 aplayer 播放，该播放器会暂停 |
| listmaxheight  | 340px | 播放列表的最大长度 |
| preload  | auto | 音乐文件预载入模式，可选项： none, metadata, auto |
| theme  | #ad7a86 | 播放器风格色彩设置 |

### 用法

```
{% meting "627070825" "netease" "playlist" "theme:#555" "mutex:true" "listmaxheight:340px" "preload:auto" %}

```

### 编辑音乐页面

```
---
title: 好听的音乐
date: 2020-04-19 14:35:09
type: "music"
---

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/aplayer@1.10/dist/APlayer.min.css">
<script src="https://cdn.jsdelivr.net/npm/aplayer@1.10/dist/APlayer.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/meting@1.2/dist/Meting.min.js"></script>


{% meting "627070825" "netease" "playlist" "theme:#555" "mutex:true" "listmaxheight:340px" "preload:auto" %}

```