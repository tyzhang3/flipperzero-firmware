# Wiegand 34 位门禁降级风险审计清单（结合代码实现）

> 目的：用于在**合法授权**范围内，验证门禁系统是否将高安全凭证降级为“Wiegand34 号码鉴权”。
>
> 范围：本清单基于仓库中 LF RFID/HID/Wiegand 相关实现整理，适用于现场安全审计与整改验收。

---

## 1. 背景与代码证据

### 1.1 LF HID 复制能力来自“编码/解码 + 可写标签”链路

仓库中的 LF 协议实现对 HID 相关格式进行了：
- 前导码与曼彻斯特编码检查；
- 位流解码与重新编码；
- 写入 T5577 数据块。

关键实现：
- `lib/lfrfid/protocols/protocol_hid_generic.c`
  - `protocol_hid_generic_can_be_decoded()`
  - `protocol_hid_generic_decode()`
  - `protocol_hid_generic_write_data()`
- `lib/lfrfid/protocols/protocol_hid_ex_generic.c`
  - `protocol_hid_ex_generic_can_be_decoded()`
  - `protocol_hid_ex_generic_decode()`
  - `protocol_hid_ex_generic_write_data()`
- `lib/lfrfid/tools/t5577.c`
  - `t5577_write()`
  - `t5577_write_block_pass()`

### 1.2 Wiegand 34 在代码中是“位流解析/转换”，不是加密认证

关键实现：
- `applications/debug/accessor/helpers/wiegand.cpp`
  - `WIEGAND::GetCardId(..., bitlength)` 对 34 位路径做数据拼装
  - `WIEGAND::DoWiegandConversion()` 处理 24/26/32/34/37/40 位帧

结论：Wiegand34 本身作为输出格式不提供加密认证能力；如果控制器仅按 W34 编号放行，会形成“同号替代”风险。

---

## 2. 风险判定模型（给审计报告可直接引用）

**判定语句（建议原文）**：

> 本次风险不等同于“高安全卡核心算法被完整破解”，而是门禁链路存在以 Wiegand34 编号作为最终放行依据的降级路径，导致同号凭证替代风险。

**风险级别建议**：
- 若存在“同号不同介质均可放行”：`高`
- 若仅在个别通道/时段触发降级：`中`
- 若全链路强制高安全认证、W34 仅日志用途：`低`

---

## 3. 现场排查清单（Checklist）

> 使用方式：逐项打勾；证据需包含截图、配置导出、日志片段或抓包摘要。

### A. 读卡器侧

- [ ] 确认读卡器当前输出协议：Wiegand34 / OSDP / 其他
- [ ] 确认是否启用 legacy 兼容输出（仅卡号模式）
- [ ] 确认是否存在“多技术并行”且低安全格式优先
- [ ] 导出读卡器配置并留档（版本号、策略模板）

**证据要求**：
- 配置页面截图（隐藏敏感密钥）
- 设备型号、固件版本、配置导出文件 hash

### B. 控制器侧

- [ ] 控制器是否将 `Facility Code + Card Number` 作为唯一放行条件
- [ ] 是否校验读卡器上传的安全认证状态字段
- [ ] 是否存在“认证失败回退卡号放行”逻辑
- [ ] 是否按门/时段应用了不同鉴权策略

**证据要求**：
- 控制器规则截图/导出
- 规则变更日志（最近 90 天）

### C. 平台与日志侧

- [ ] 日志是否记录“仅卡号事件”与“安全认证事件”两类
- [ ] 是否可区分同号不同介质/不同读头来源
- [ ] 是否有 anti-passback、时空冲突、频率异常告警
- [ ] 告警是否真正联动阻断（而非仅通知）

**证据要求**：
- 审计日志样本（脱敏）
- 告警策略与处置闭环记录

### D. 对抗验证（授权测试）

- [ ] 样本 A（原卡）在标准流程可通过
- [ ] 样本 B（同号测试凭证）在所有门点都被拒绝
- [ ] 断网/离线模式下仍不出现号码直通
- [ ] 夜间/跨时段策略与白天一致

**判定**：
- 任一门点出现“同号测试凭证可放行” => 降级风险成立

---

## 4. 树莓派辅助验证（合规版）

> 仅用于记录读卡器输出行为与日志一致性，不用于未授权复制或绕过。

### 建议硬件
- 树莓派 4/5
- 合法授权 USB 读卡器（PC/SC）
- 串口/网络日志采集通道（只读）

### 建议采集字段
- 时间戳、门点、读头 ID
- 输出位长（如 34）、编号字段
- 控制器判定结果（pass/deny）
- 后台事件类型（卡号事件/认证事件）

### 关键核验
- 同一人员在不同介质下是否被识别为同一“号码源”
- 失败事件是否被错误降级为“卡号放行”

---

## 5. 整改与验收清单

### P0（立即）
- [ ] 禁止“仅卡号放行”规则
- [ ] 关闭 legacy/兼容降级模式
- [ ] 对高风险门点临时启用双因子（卡 + PIN/生物）

### P1（1~2 周）
- [ ] 统一读头与控制器鉴权策略
- [ ] 打通“认证状态”字段到平台判定链路
- [ ] 上线同号多介质冲突告警与自动阻断

### P2（持续）
- [ ] 发卡与挂失流程纳入密钥生命周期治理
- [ ] 季度渗透测试与配置漂移审计
- [ ] 形成安全基线模板并固化到交付规范

### 验收标准（建议）
- [ ] 任意门点对“同号测试凭证”均拒绝
- [ ] 所有放行日志均可关联到有效认证状态
- [ ] 随机抽检 10 个门点策略一致

---

## 6. 附录：关联代码路径

- `documentation/file_formats/LfRfidFileFormat.md`
- `lib/lfrfid/protocols/lfrfid_protocols.c`
- `lib/lfrfid/protocols/protocol_hid_generic.c`
- `lib/lfrfid/protocols/protocol_hid_ex_generic.c`
- `lib/lfrfid/tools/t5577.h`
- `lib/lfrfid/tools/t5577.c`
- `applications/debug/accessor/helpers/wiegand.cpp`
- `applications/debug/accessor/helpers/wiegand.h`

---

## 7. 法律与合规声明

本清单用于授权安全评估与防御整改。任何未授权复制、绕过或非法使用凭证的行为均可能违反法律法规与组织制度。
