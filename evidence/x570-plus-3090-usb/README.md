# Evidence: MSI GeForce RTX 3090 Ventus 3X OC 24GB on ASUS TUF GAMING X570-PLUS (WI-FI)

This directory records contributor physical test evidence for an MSI GeForce RTX 3090 Ventus 3X OC 24GB running on an ASUS TUF GAMING X570-PLUS (WI-FI) motherboard under Windows 11 Pro 64-bit with NVIDIA Driver 621.14.

The run demonstrated a complete end-to-end boot chain:
```text
COLD BOOT -> USB EFI -> exact RTX 3090 profile match -> unlock transaction -> PCIe FLR -> firmware cleanup -> direct Windows handoff -> Windows 11 kernel -> NVIDIA driver 621.14 -> GPU operational -> Tensor benchmark uplift -> real NInfer LLM workload improvement
```

Primary tracking issue: [#316](https://github.com/windows-GPU-UNLOCK/Windows-GPU-UNLOCK/issues/316)

---

## Provenance and build scope

- **Build / Source Commit**: `fd8bbea5b1db991805ffcd7ba84e73fa7867bd8a` (Merge PR #139)
- **Profile**: `entryLatchedSs1Full`
- **Boot Method**: Headless USB research route with direct Windows handoff (`bootmgfw.efi`)
- **Gates Evaluated**: 22/22 PASS (watchdog, BDF, BAR0, live PROM hash, cold registers)
- **Single Transaction**: Executed once under guarded conditions, PCIe FLR returned `0`, GSP halted, SEC2 reset released, BME contained.

---

## Machine details

```text
Component             Value
--------------------  -------------------------------------------------------------
GPU Model             MSI GeForce RTX 3090 Ventus 3X OC 24GB
PCI ID                10de:2204
Subsystem ID          1462:3881
Chip ID               0xb72000a1 (GA102 rev A1)
PCI Location          0000:0a:00.0 (BDF 0:10:0.0)
VRAM                  24 GB GDDR6X (Micron)
VBIOS Version         94.02.42.00.2f (factory stock; never cross-flashed)
Motherboard           ASUS TUF GAMING X570-PLUS (WI-FI)
Motherboard BIOS      5021 (AMD AGESA ComboV2PI 1.2.0.B)
Platform              AMD Socket AM4 / X570
CPU Scope             AMD AM4 host CPU; model deliberately excluded from claim scope
Display Connection    DisplayPort directly on RTX 3090; single GPU (no iGPU present)
Operating System      Windows 11 Pro 64-bit (build 26100)
NVIDIA Driver         621.14 (WDDM 3.2, CUDA UMD 13.5)
Fast Startup          Disabled (HiberbootEnabled = 0)
BitLocker             Suspended for controlled experiment; encryption retained
Secure Boot           Disabled in UEFI for USB EFI boot path
```

---

## Recorded hardware runs

From `AUTO000002/REPORT.TXT` and `HLS000002.TXT`:

1. **Pre-flight & Live PROM Matching**:
   - `live PROM hash matched=1` (matches profile header)
   - Cold registers: `SS0=0x70240110`, `SS1=0x0`
2. **Transaction Execution**:
   - Intent selectors written: `SS0target=0x88888888`, `SS1target=0x00000008`
   - PCIe FLR begin: `BME=0 DMA retained, chip=0xb72000a1`
   - PCIe FLR return: `result=0`
   - Post-FLR registers: `SS0=0x88888888`, `SS1=0x00000008`
   - Cleanup: `post-FLR SEC2 reset released, GSP halted, BME=0 pending=0 WPR2=1ffffe00:00000000`
   - Final status: `transaction result=0 containment=0 started=1 quiesced=1 fault=0 retained=0 maps=0 FLR=1`
   - Final exit: `FINAL status=0 stage=transaction attempted=1 driver_release=0 clean=1`
3. **Windows Handoff**:
   - `StartImage(\EFI\Microsoft\Boot\bootmgfw.efi) status=0x0`
   - Windows 11 loaded without power-off or manual intervention.
   - NVIDIA driver 621.14 initialized the GPU without TDR, artifact, or error code 43.

---

## Measurements

### 1. Synthetic PyTorch Tensor Benchmark (GEMM 4096×4096)

Workload: PyTorch 2.11.0+cu128, 4096×4096 GEMM, 3 warmups, 5 measured runs of 50 iterations each. Primary comparison: median TFLOPS.

| Mode | Baseline (Pre-Unlock) | Post-Unlock (Run 1) | Post-Unlock (Run 2) | Delta ($\Delta$) | Uplift Ratio | % Change |
|---|---:|---:|---:|---:|---:|---:|
| **FP16** | 60.38 TFLOPS | **111.74 TFLOPS** | 112.19 TFLOPS | **+51.36 TFLOPS** | **1.85×** | **+85.06%** |
| **BF16** | 60.36 TFLOPS | **110.81 TFLOPS** | 112.19 TFLOPS | **+50.45 TFLOPS** | **1.84×** | **+83.58%** |
| **TF32** | 31.83 TFLOPS | **52.50 TFLOPS** | 52.41 TFLOPS | **+20.67 TFLOPS** | **1.65×** | **+64.94%** |
| **FP32** | 21.74 TFLOPS | **22.35 TFLOPS** | 21.65 TFLOPS | **+0.61 TFLOPS** | **1.03×** | **+2.79%** |

*Interpretation:* The huge FP16/BF16/TF32 uplift (+65% to +85%) accompanied by unchanged FP32 general compute (+2.79% / ~1.00–1.03×) demonstrates that the hardware improvement is specific to Tensor Core execution pipelines and is not an artifact of GPU core clock overclocking.

---

### 2. Real-World LLM Inference Validation (NInfer 3090)

Workload: `Swift-1.5-Qwen3.8-27b.ninfer` (19.03 GB), flags `--spec mtp --draft-tokens 3 --prefill-cublas --prefill-chunk 4096 --fit-context 65536`.

| Request Run | Metric | Baseline (Pre-Unlock) | Post-Unlock | Delta ($\Delta$) | % Change |
|---|---|---:|---:|---:|---:|
| **Run 1: Cold**<br>(Prompt: 66, Out: 100) | Prefill<br>TTFT<br>Decode<br>MTP Acc.<br>Power | 261.4 tok/s<br>261 ms<br>89.3 tok/s<br>60/116 (51.7%)<br>327.4 W | **333.5 tok/s**<br>**215 ms**<br>**92.8 tok/s**<br>**60/116 (51.7%)**<br>**334.2 W** | +72.1 tok/s<br>-46 ms<br>+3.5 tok/s<br>0.0%<br>+6.8 W | **+27.58%**<br>**-17.62%**<br>**+3.92%**<br>Identical<br>In-spec |
| **Run 2: Warm**<br>(Prompt: 71, Out: 150) | Prefill<br>TTFT<br>Decode<br>MTP Acc.<br>Power | 392.5 tok/s<br>203 ms<br>113.1 tok/s<br>102/141 (72.3%)<br>333.6 W | **427.0 tok/s**<br>**200 ms**<br>**115.3 tok/s**<br>**102/141 (72.3%)**<br>**333.8 W** | +34.5 tok/s<br>-3 ms<br>+2.2 tok/s<br>0.0%<br>+0.2 W | **+8.79%**<br>**-1.48%**<br>**+1.95%**<br>Identical<br>In-spec |
| **Run 3: Hot**<br>(Prompt: 81, Out: 200) | Prefill<br>TTFT<br>Decode<br>MTP Acc.<br>Power | 394.5 tok/s<br>254 ms<br>104.2 tok/s<br>131/203 (64.5%)<br>338.7 W | **431.6 tok/s**<br>**238 ms**<br>**105.7 tok/s**<br>**131/203 (64.5%)**<br>**335.9 W** | +37.1 tok/s<br>-16 ms<br>+1.5 tok/s<br>0.0%<br>-2.8 W | **+9.40%**<br>**-6.30%**<br>**+1.44%**<br>Identical<br>In-spec |

*Interpretation:* Prefill GEMMs benefit from the Tensor unlock (+9% to +28% tok/s; lower TTFT). Single-batch autoregressive decode remains constrained by GDDR6X memory bandwidth and speculative verification, as expected by the architecture. MTP draft acceptance rates remained identical, verifying arithmetic correctness.

---

### 3. Negative Control: Strata MoE Cross-Check

Workload: `swift-1.5-iq2_xs` MoE (11,913 experts, prompt 41, gen 200).
- Model load: 52.0s -> 50.0s
- Engine decode: 58.9 -> 59.4 tok/s (+0.8%)
- Client decode: 56.1 -> 56.4 tok/s
- Peak 5-token: 263.3 -> 270.4 tok/s
- Power: 236.6W -> 236.1W

No meaningful decode uplift observed, corroborating that memory-bandwidth/offload-limited workloads do not exhibit artificial speedups.

---

## What is NOT established

1. **Universal Model Support**: This does not establish that all RTX 3090 cards, all MSI Ventus revisions, or all X570 motherboards will succeed.
2. **Permanent Unlock**: The unlock is volatile in UEFI and requires traversal of the boot chain on cold boot.
3. **Gaming Performance**: Gaming frame rates and rasterization performance are not established to increase; Tensor math is the specific beneficiary.
4. **LLM Decode Scaling**: The unlock does not bypass GDDR6X memory bandwidth bottlenecks during batch-1 autoregressive decode.

---

## Excluded items

In accordance with `docs/CONTRIBUTING.en.md`:
- No raw PROM binaries (`PROM.BIN`) or VBIOS ROM dumps are included.
- No firmware binaries or NVIDIA blobs are distributed.
- No BitLocker recovery keys, TPM secrets, or personal paths are stored.
