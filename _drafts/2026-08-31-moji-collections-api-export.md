---
title: MOJi 收藏接口拉取与导出
date: 2026.08.31
categories: [Othernotes]
tags: [moji, api, network, csv]
---

通过 MOJi 网页版的收藏接口，可以读取收藏夹及其全部收藏，不需要从安卓抓包或解密 HTTPS，也不需要使用网站自带导出。下面记录 2026 年 8 月 31 日实际验证的接口约定；后续复现时，以浏览器新抓到的请求为准。

## 获取登录请求

登录 [MOJi 网页收藏页](https://www.mojidict.com/collection)，打开浏览器开发者工具的 Network，刷新页面或进入一个收藏夹，筛选 `folder-fetchContentWithRelatives`，复制该请求及其 Payload。鉴权使用请求体里的 `_SessionToken`，同时保留 `_InstallationId`；二者都从当前登录请求取得，不写入笔记或公开仓库。登录失效时重新登录并抓取，不复用旧凭据。

## 接口与字段

请求方法是 `POST`，地址为 [folder-fetchContentWithRelatives](https://api.mojidict.com/parse/functions/folder-fetchContentWithRelatives)。请求体是 JSON 文本，但实抓请求的 `Content-Type` 为 `text/plain`；`Origin` 为 `https://www.mojidict.com`，`Referer` 为 `https://www.mojidict.com/`。复现时优先保留浏览器复制出的完整请求头，包括 `User-Agent`，确认成功后再精简。本次验证中，过度精简请求头会得到 HTML 403，不能仅凭这个状态认定登录失效；不要通过关闭 TLS 校验解决请求问题。

决定读取范围的是 `fid`：根目录使用 `ROOT#com.mojitec.mojidict#zh-CN_ja`，具体收藏夹使用根目录记录中的 `targetId`。分页参数使用 `count=30`、`sortType=4`，`pageIndex` 从 `1` 开始。其余请求体字段沿用当前浏览器请求，本次验证值为 `_ClientVersion="js4.3.1"`、`_ApplicationId="E62VyFVLMiW7kvbtVq3p"`、`g_os="PCWeb"`、`g_ver="4.17.5"`，并附带当前 `_SessionToken` 和 `_InstallationId`。根目录 ID 与语言环境有关，更换语言设置后也应重新确认。

响应 JSON 的 `result` 是包含分页信息的对象，真正的记录数组位于 `result.result`。成功响应的 `result.code` 为 `200`，同时核对返回的 `fid` 和 `pageIndex`；HTTP 200 本身不代表业务成功。记录的 `targetType=1000` 表示收藏夹，`102` 表示单词，`103` 表示例句。读取子收藏夹时使用 `targetId`，不要误用当前收藏记录自身的 `objectId`；后者标识的是收藏关系。具体内容位于记录的 `target` 对象，常用字段包括 `spell`、`pron`、`accent`、`excerpt`，例句常用 `title` 和 `trans`。

## 完整拉取与保存

从根目录开始逐页读取，直到 `result.result` 实际为空；收集其中所有收藏夹的 `targetId`，对每个收藏夹重复同样的分页过程，发现子收藏夹则继续递归，并用已访问的收藏夹 ID 集合避免重复遍历。`size`、`totalPage` 和收藏夹的计数字段可能与实际返回不符，不能作为唯一终止条件。每夹核对收藏记录 ID 是否重复、`parentFolderId` 是否正确；遇到重复分页或异常响应应停止核查，避免把不完整数据标为成功。对计数异常的目录重新拉取并比较记录，抓取期间尽量不要修改收藏。

先保存所有分页响应的完整 JSON 结构及请求参数，再把同一收藏夹的所有分页合并成一份 CSV，一条收藏对应一行。根目录直接收藏的条目另存一份，不漏掉例句，也不按单词 ID 跨收藏夹去重。CSV 可提取标题、读音、音调、摘要或译文、收藏时间、收藏记录 ID、目标 ID 和收藏夹信息，使用 UTF-8 BOM，并正确处理字段里的逗号、引号和换行。原始 JSON 保留其他字段，不额外请求单词详情；缺失字段留空，不自行补写。

保存或分享前，将 `_SessionToken`、`_InstallationId`、`authData` 以及其他认证信息脱敏。分页 JSON 用于保留原始返回，CSV 用于按收藏夹阅读；脱敏后的 JSON 不再是逐字节原始报文。私人的收藏内容仍需妥善保存，实际可导出的范围以当前账号及接口可访问的数据为限。
