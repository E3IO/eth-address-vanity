# Ethereum Vanity Address (CUDA)

CUDA 加速的以太坊靓号生成器，支持前缀/后缀匹配、合约/CREATE2/CREATE3 模式，以及“公钥+偏移”云 GPU 搜索模式（公私钥分离）。

## 功能
- 普通地址/合约地址/CREATE2/CREATE3 靓号搜索
- 评分方式：前导零字节、全局零字节、前缀/后缀匹配
- 多 GPU 支持（重复传入 `--device`）
- 公钥 + 偏移搜索：可信环境生成种子并保管私钥，仅把公钥发给云端暴力偏移

## 环境与编译
- 需要 NVIDIA GPU（Compute Capability ≥ 5.2）与 CUDA Toolkit。
- Linux 编译示例（4090 推荐架构 sm_89，可按需调整）：
```bash
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH

nvcc src/main.cu -o eth-vanity-address -std=c++17 -O3 -Xcompiler -pthread -arch=sm_89 --use_fast_math
```

## 运行用法
```bash
./eth-vanity-address [评分/模式] [设备] [前后缀/公钥等参数] [其他选项]
```

### 核心参数
- 设备：`--device <id>`（可重复，多卡）
- 评分方式（互斥）：`--leading-zeros|-lz`、`--zeros|-z`、（前缀/后缀自动启用）
- 前缀/后缀：`--prefix|-p <hex>`，`--suffix|-s <hex>`（40 位地址 hex，建议小写；最多 63 字符）
- 工作规模：`--work-scale|-w <n>`（默认 15，GPU 核心越强可尝试 16/17/18）

### 模式选择
- 普通地址（默认）
- 合约：`--contract|-c`
- CREATE2：`--contract2|-c2`，需 `--bytecode <file>`、`--address <40/42 hex>`
- CREATE3：`--contract3|-c3`，需 `--bytecode <file>`、`--address <origin>`、`--deployer-address <addr>`

### 公钥 + 偏移模式（云 GPU 安全搜索）
- 参数：`--pubkey|-pk <uncompressed hex>`（128/130 字符，可带 0x/04），`--offset-start|-os <uint64>` 起始偏移
- 参数（推荐，256bit）：`--offset-start-hex|-osh 0x...`（64 字节 hex，可带 0x）起始偏移，适合超大区间分片
- 仅支持普通地址模式（mode 0）
- 云端输出 Offset 与地址，不输出私钥；可信端用 `priv = seed + offset (mod n)` 还原

### 示例
- 单卡前缀+后缀：
```bash
./eth-vanity-address --device 0 --prefix abcd --suffix 1234 --work-scale 17
```
- 多卡：
```bash
./eth-vanity-address --device 0 --device 1 --zeros --work-scale 16
```
- CREATE2：
```bash
./eth-vanity-address --device 0 --contract2 --bytecode bytecode.txt \
  --address 0x0000000000000000000000000000000000000000 \
  --prefix dead --work-scale 17
```
- 公钥+偏移（云）：
```bash
./eth-vanity-address --device 0 \
  --pubkey 04<64B_X><64B_Y> \
  --offset-start 0 \
  --prefix cafe --suffix 0bad --work-scale 17
```

## 安全提示
- 私钥生成使用系统 CSPRNG（Linux `/dev/urandom`，Windows BCrypt），含曲线阶拒绝采样；务必在可信环境运行。
- 公钥+偏移模式：种子/私钥仅在可信端；云端只接触公钥与偏移，不输出私钥。
- 关闭或谨慎处理日志，避免泄露前缀/后缀目标或偏移。
- 云部署建议使用专属裸金属或 TEE，避免快照/内存被窃取；如需持久化结果请加密。
- 启动日志会打印前/后缀长度（host/device），便于确认匹配条件是否正确传入 GPU。

## 已知建议
- work-scale 过大可能增显存与单 kernel 时长，可根据 GPU 调整。
- 建议显式设置 `-arch` 与 `--use_fast_math`（视需求而定）。
