# Termini fork of crates.io `rig-core` 0.42.0

只记跟官方 **0.42.0** 源码的差异。基底是 crates.io `.crate`，不是 git 快照。

GitHub：<https://github.com/chengpengvb/rig-core>  
`main` 是这份 0.42；`v0.40.0` 是上一份 0.40 存根。

Termini 挂载：

```toml
[patch.crates-io]
rig-core = { path = "../../archive/rig" }
```

`rig` 门面只做 re-export，不 patch。`rig-agent` 现走 crates.io。诊断 tool call 那一刀在 **rig-agent** 的 `src/agent/run/streamed.rs`，本仓库没有。

---

## 1. `stringified_json::serialize`：`Null` → `"{}"`

文件：`src/json_utils.rs`

百炼要求 `function.arguments` 必须是 JSON 对象。官方把 `Null` 序列化成 `"null"`，历史里已存的脏数据回放会 400。直连百炼：`"null"` → 400；`""` / `"{}"` → 200。

```rust
let s = match value {
    serde_json::Value::Null => "{}".to_string(),
    other => other.to_string(),
};
serializer.serialize_str(&s)
```

测例：`test_stringified_json_serialize_null_as_empty_object`。

anthropic 一线不用改，官方已有 `src/providers/anthropic/completion.rs` 的 `coerce_tool_input`。这层曾在一次升版本时丢过，升级时确认还在。

---

## 2. 流式 `delta.images` → `Unknown`，不进 assistant 历史

OpenRouter 风格生成图放在 delta 同级 `images`。官方流式不认这个字段，serde 丢掉。本补丁收进来，交给 Termini 画图，不写进对话历史。

| 文件 | 改什么 |
| --- | --- |
| `src/providers/openai/completion/streaming.rs` | `StreamingDelta.images`，`CompatibleChoiceData` 带上 |
| `src/providers/internal/openai_chat_completions_compatible.rs` | `CompatibleChoice` / `CompatibleChoiceData` 加字段，`From` 透传，`emit_chunk` 之后 `Unknown` |
| `src/test_utils/internal_streaming_profiles.rs` | 字面量补 `images: Vec::new()` |

0.42 没有 `null_or_vec`，用旁边字段同一套 `null_or_default`。`Unknown` 要 `.into()` 成 `UnknownPayload`。0.42 copilot 不再拼 `CompatibleChoiceData`，不用改 copilot。

```rust
#[serde(default, deserialize_with = "json_utils::null_or_default")]
images: Vec<serde_json::Value>,
```

```rust
for image in choice.images {
    out.push(Ok(RawStreamingChoice::Unknown(image.into())));
}
```

`Unknown` 不折进累积的 assistant 消息。OpenRouter 也不接受把生成图当 assistant 内容回传。

Termini 消费：`StreamedAssistantContent::Unknown`，用 `.value()` 读，`handle_image.rs` 解析 `image_url.url`。没有 `image_url` 的 Unknown 忽略。

---

## 3. `rustls` feature：`reqwest/rustls-no-provider`

文件：`Cargo.toml`（cargo 用这个，不是 `Cargo.toml.orig`）

官方：

```toml
rustls = [
    "reqwest/rustls",
    "tokio-tungstenite?/rustls-tls-webpki-roots",
]
```

本地：

```toml
rustls = [
    "reqwest/rustls-no-provider",
    "tokio-tungstenite?/rustls-tls-webpki-roots",
]
```

Termini 的 `ProvideFactory` 自建 rustls + ring，再 `tls_backend_preconfigured` 塞进客户端。官方 `reqwest/rustls` 自带 crypto provider，接不上。

---

## 不在本仓库

诊断 tool call 空参数 `Null` → `{}`：`name_delta_diagnostic_tool_call`，0.42 在 **rig-agent** `src/agent/run/streamed.rs`。Skip / Retry 回放进历史。配套测例也在 rig-agent：`streamed_invalid_name_delta_without_args_records_object_arguments`、`streamed_invalid_name_delta_unparseable_args_normalize_to_object`。

`.cargo_vcs_info.json`、`Cargo.toml.orig` 是 crates.io 包装文件，不是补丁。`Cargo.lock` 不是补丁。
