# 阅读器右侧翻章竖条（方案记录，未实施）

状态：已实施并在小米 13 实测通过（v5，2026-11-14）。目标样本：Android v2.1.11 自测分支。

## 1. 现状（已查证）

- 阅读器页面：路由 `/comic/detail/read`，构建产物 `assets/public/static/js/7522.e48de9ca.chunk.js`（压缩）。原始源码在同类名 `.map` 里（`pages/Comic/Read.tsx`、`components/Comic/ReadNav.tsx`），`.map` 不打进 APK，且内容是官方原始版本，只作阅读参考。
- 翻章唯一入口在 `ReadNav`：底部固定栏 `fixed left-0 bottom-0`，第一行是「上一章箭头 / 页码 / 进度条 / 总页数 / 下一章箭头」。两个箭头是 `KeyboardDoubleArrowLeft/RightIcon`，无额外 padding。
- 该底栏默认随 UI 收起（`closeNav`）；阅读区大 `div` 的 `onClick` 只做 `setCloseNav(!closeNav)`。所以现状翻章 = 先点屏幕唤出底栏，再点屏幕最底边的小箭头；命中区小，且贴着系统手势条。
- 翻页路径：纵向为 window 滚动（Virtuoso），横向为 Swiper；音量键仅翻页。没有任何翻章手势区。
- 章节可用性由 `changeCurrentPage.{prev,next}` 控制；为 false 时箭头只是 `opacity-0`，元素仍可点（`handlePageChange` 找不到目标就静默返回）。

## 2. 方案（A）

在 `ReadNav` 内新增一个右侧竖向触控条，点击直接复用现有 `handlePageChange("next")`（内部已完成：关闭 express、按 `detailList.series` 找下一章、`replace` 跳转、写阅读记录）。

**唤起逻辑：与底栏共用 `closeNav`。** 底栏收起时竖条不渲染；点屏幕唤出底栏时一起出现。不做「滚到章末再滑一下」的 overscroll 手势。

理由：

1. 零新增 state、零新增手势监听，只有一个渲染条件；
2. 不破坏「点屏幕切换 UI」的既有约定，不会出现「想唤出底栏却整章跳走」；
3. 阅读时画面上不出现常驻覆盖层；
4. overscroll 方案要分别处理纵向 window 滚动与横向 Swiper，还要和系统下拉/侧滑返回冲突，改动面大。

## 3. 实现落点

压缩产物中 `ReadNav` 组件的解构映射（`const z=e=>{const{...}=e`）：

| 压缩变量 | 含义 |
|---|---|
| `a` | `isLoading` |
| `o` | `changeCurrentPage` |
| `f` | `handlePageChange` |
| `y` | `closeNav` |
| `p` | `dialogOpen` |
| `S` | `showSnackbar` |
| `I.A` | `KeyboardDoubleArrowLeftIcon` |
| `N.A` | `KeyboardDoubleArrowRightIcon` |
| `D` | `jsx` 运行时 |

插入位置：`ReadNav` 返回的 Fragment 子节点数组里、`p.folder && ...` 之前。唯一锚点：

```
),p.folder&&(0,D.jsx)(R.A,{folderList:H.folder_list
```

在 `p.folder&&` 前插入（`D.jsx` 写法，行内 style 避免依赖预编译 Tailwind）：

```js
,!y&&o.next&&!a&&(0,D.jsx)("div",{
  onClick:()=>f("next"),
  className:"fixed right-0 z-20",
  style:{top:"16%",bottom:"18%",width:48,display:"flex",alignItems:"center",
         justifyContent:"center",background:"rgba(0,0,0,0.18)",
         borderTopLeftRadius:24,borderBottomLeftRadius:24},
  children:(0,D.jsx)(N.A,{sx:{fontSize:32,color:"#ffffff",opacity:.45}})
})
```

要点：

- `chunk id -> 文件名` 映射在 `main.9a45a3ff.js` 内为 `7522:"e48de9ca"`，该串是文件名不是内容校验，**保持文件名不变**即可。
- 项目 CSS 为预编译 Tailwind，新 class 不会生效；位置/尺寸用行内 `style`（`fixed`/`right-0`/`z-20`/`opacity-0`/`pointer-events-none` 等已存在于 `main.fe5d614d.css`）。
- 高度区间取 `top:16%`、`bottom:18%`（2026-11-14 实测调长；初版 30%/32% 太短）：避开顶部栏（`fixed top-0`）与底栏（两行约 150px），同时避开底部系统手势区。
- `z-20` 与底栏同级，低于章节弹层（z-50）、评论弹层（z-30）、顶栏（z-40），弹层打开时自然被遮住。
- 竖条是覆盖层，从竖条起手的横滑不会传给 Swiper；浏览器会在位移后吞掉 click，所以滑到一半松手不会误跳章。竖条宽度限制在 48px 中段以控制影响。
- 可选（同一处 1 行）：把无下一章时的底栏箭头从「透明但仍可点」改成真禁用，`opacity-0` -> `opacity-0 pointer-events-none`。

## 4. 无下一章时的行为

**竖条不出现，也不给「已到最后一章」提示。**

- `changeCurrentPage.next === false` 有三种来源：`detailList.series` 为空、当前 `readId` 不在 series 中、确实是最后一章（`sort === series.length`）。按「无下一章就提示最后一章」写会误报前两种。要做提示必须再加严格判定，为一个提示新增分支，不划算。
- 「到头」提示项目已有：`Read.tsx` 的 `readTrack` 弹窗条件为 `!changeCurrentPage.next && scrollProgress === images.length && !localStorage.dontShowExpiry`，即最后一章最后一页自动弹追踪/完结提示，再提示重复。
- 与底栏一致：无下一章时底栏下一章箭头本就 `opacity-0`（视觉消失），竖条同样消失，用户心智统一。

若后续确实需要反馈，最小做法是复用现有 i18n key `novel.is_last_chapter`（tw `已經是最後一篇！` / cn `已经是最后一篇！`）弹 snackbar，但文案是「篇」不适合漫画，且仍需先加严格判定。

## 5. 明确不做

- 章末 overscroll / 再滑一次翻章的手势拦截。
- 常驻隐形右侧热区（会与「点屏幕唤出底栏」冲突，误触代价是整章跳转）。
- 新增 i18n key、新增设置开关、新增依赖。
- 竖条分上下两段（上=上一章、下=下一章）：上一章误触代价更高，仍保留在底栏。
- 进度条 `max = images.length + 3` 把 3 个广告位计入（总页数偏大 3）属于既有问题，不在本次范围。

## 6. 验证清单

模拟器（Android 16 / arm64）：

1. 纵向滚动模式、横向 Swiper 模式下竖条位置正确、可点。
2. 第一章：竖条正常（`prev` 无关）；最后一章：竖条不出现。
3. 加载中（`isLoading`）竖条不渲染。
4. 点屏幕唤出/收起底栏时，竖条同步出现/消失。
5. 章节弹层、评论弹层打开时竖条被遮挡且不可点。
6. 从竖条起手横滑不误跳章。
7. 跳章后系统返回与左上角返回仍直接回详情页（沿用已有 `replace` 行为）。
8. 跳章后阅读记录写入 `localStorage.read` 正常。

## 7. 打包

沿用现有流程：`zip` 写入（`resources.arsc` 与图片资源 STORED）-> `zipalign` 4 字节 -> `apksigner` V1/V2/V3（`signing/jmcomic-noads.keystore`），APK 命名 `JMComic3v{version}-{YYYYMMDD}v{x}.apk`。
