# gem5 + NVmain
---

## q1 - build-up
```bash
cd ~/gem5
./build/X86/gem5.opt configs/example/se.py \
  -c tests/test-progs/hello/bin/x86/linux/hello \
  --cpu-type=TimingSimpleCPU --caches --l2cache \
  --l1i_size=32kB --l1d_size=32kB --l2_size=128kB \
  --mem-type=NVMainMemory \
  --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**result**
Hello world!

## q2 - enable L3 cache
```bash
./build/X86/gem5.opt configs/example/se.py \
  -c tests/test-progs/hello/bin/x86/linux/hello \
  --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache \
  --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB \
  --l3_assoc=4 \
  --mem-type=NVMainMemory \
  --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**result**
l3 cache exits

## q3 - 2-way and full-way in l3 cache (quicksort benchmark)
```bash
# 2-way
./build/X86/gem5.opt --outdir=./m5out_2way configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --l3_assoc=2 --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

# full-way
./build/X86/gem5.opt --outdir=./m5out_fullway configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --l3_assoc=16384 --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**result**
| index | 2-way | Fully-associative |
|-------|-------|------------------|
| L3 miss rate | 0.4334 | 0.4706 |

**conclusion**
理論上full-way 的miss rate 應該要小於2-way，因為沒有conflict miss
但是在benchmark 是quicksort 的情況下，
gem5 的fully-associative = 把整個 1MB L3 變成 單一 set（16384-way），全部由 LRU 管理。
quicksort 的存取是跳躍式、非線性的，working set 很大，LRU 在 16384-way 的巨大 set 裡反而容易把「等一下還會用到的 block」提早淘汰，製造更多 miss。 

簡單來説LRU + quicksort 的跳躍存取在超大set 下失效，所以實驗結果與理論相反。

## q4 - LRU v.s. LFU replacement policy (quicksort benchmark)
```python
# LRU（baseline）
replacement_policy = Param.BaseReplacementPolicy(LRURP(), "Replacement policy")
整行改成:
replacement_policy = LRURP()

執行:
./build/X86/gem5.opt --outdir=./m5out_lru configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --l3_assoc=16384 --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

# LFU (frequency-based)
replacement_policy = LRURP() 改成 replacement_policy = LFURP()

執行:
./build/X86/gem5.opt configs/example/se.py -c ./quicksort \
  --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache \
  --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB \
  --l3_assoc=16384 \
  --mem-type=NVMainMemory \
  --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**result**
| index | LRU (Baseline) | LFU (Frequency-based) |
|-------|---------------|----------------------|
| sim_seconds | 0.533567 | 0.534684 |
| L3 miss rate | 0.4706 | 0.5500 |

**conclusion**
LFU 的miss rate 較高，因為quicksort benchmark 不適合frequency-based policy
quicksort 透過Pivot 分割陣列，在遞迴過程中(temporal locality)，會重複存取，但是遞迴結束後這些資料就不會被存取了。
LRU (Least Recently used) 只看最近有沒有用到，當移動到下一個子陣列時，原本的陣列最近沒有被存取，會被LRU 踢走，釋出空間給新的資料，因此miss rate 比較低。

## q5 - write back & write through (multiply benchmark)
```python 
# write-back
replacement_policy = LFURP() 改成 replacement_policy = LRURP()

執行:
./build/X86/gem5.opt --outdir=./m5out_writeback configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --l3_assoc=4 --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

抓數據:
grep "sim_seconds\|num_writes::total" ~/gem5/m5out_writeback/stats.txt

# write-through
caches.py: write_buffers = 0

./build/X86/gem5.opt --outdir=./m5out_writeback configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --l3_assoc=4 --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

抓數據:
grep "sim_seconds\|num_writes::total" ~/gem5/m5out_writethrough/stats.txt
                                        
```

**result**
| index | Write-back | Write-through |
|-------|-----------|--------------|
| sim_seconds | 1.962470 | 18446744（timeout） |
| PCM writes | 691 | 0（deadlock） |

**conclusion**       
write-through 對 PCM 來說是災難性的，所以 write-back 是正確選擇。
write-through 因 write_buffers=0 造成所有寫入請求必須Pass through 快取，直接到達主記憶體，PCM 本身具有極高的Write Latency, 在multiply benchmark 會產生海量的同步，會在瞬間導致 gem5 內部的快取控制器、匯流排與記憶體控制器嚴重阻塞，引發Dirty Eviction 無法排出，
因此產生Deadlock。

因此在真實的硬體設計中，如果要採用write-through policy, cache 必須搭配很大的write buffer.
這個數據也證明了在現代的Non-Volatile Memory Architecture, 會優先採用write-back.

## bonus - Design last level cache policy to reduce the energy consumption of pcm_based main memory (Baseline: LRU)

**新增檔案**
src/mem/cache/replacement_policies/wa_lru_rp.hh
src/mem/cache/replacement_policies/wa_lru_rp.cc

**修改檔案**
src/mem/cache/replacement_policies/ReplacementPolicies.py  ← 註冊 WALRURP
src/mem/cache/replacement_policies/SConscript              ← 加入編譯
                                                     
**編譯指令**
```bash
cd ~/gem5
scons build/X86/gem5.opt -j12
```

**切換 policy 指令：**
```bash
# Baseline（LRU）
sed -i 's/WALRURP()/LRURP()/' configs/common/Caches.py

# Proposed（WALRU）
sed -i 's/LRURP()/WALRURP()/' configs/common/Caches.py
```bash
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --l3_assoc=4 --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**result**
| 指標 | LRU (Baseline) | WALRU (Proposed) | 改善幅度 |
|------|---------------|-----------------|---------|
| PCM writes | 91,095 | **89,582** | **↓ 1.66%** |
| PCM reads | 110,086 | 112,651 | +2.33% |
| sim_seconds | 0.5346 | 0.5315 | ↓ 0.58% |

**conclusion**
WALRU 透過優先淘汰 clean block，減少 PCM 寫入次數 1.66%，有效降低 PCM 能耗。
