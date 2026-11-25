# Analisi dell'implementazione di SPEAR-V e Baseline Upstream

Questo documento riassume l'analisi della repository SPEAR-V (commit originale `2d292c4a3d6f9b5f03276d7bf52fc62a514c5ed2` su `master`), identifica le versioni upstream originali dei software modificati e cataloga tutti i file modificati o aggiunti per implementare **SPEAR-V** (TEE RISC-V basato su page-granular memory tagging, ASIA CCS '23), in vista del porting su **gem5** nel branch `spearv-gem5`.

---

## 1. Versioni Upstream Originali Recuperate

La repository contiene tre alberi software con le modifiche SPEAR-V già integrate direttamente nel commit iniziale (`riscv-isa-sim`, `riscv-opcodes`, `riscv-pk`) più il submodule hardware `cva6` (`davidschrammel/cva6.git`, branch `spearv`).

Confrontando gli hash SHA-1 dei blob Git rispetto ai repository upstream ufficiali, sono stati individuati gli esatti commit di partenza:

| Componente | Repository Upstream | Commit Base Upstream | Data Commit | File Modificati / Aggiunti da SPEAR-V |
| :--- | :--- | :--- | :--- | :--- |
| **`riscv-isa-sim` (Spike)** | `riscv-software-src/riscv-isa-sim` | `06f630cb30c60c32452058710e98c7b05106b7f5` (*Merge PR #649 from ben-marshall/scalar-crypto-fix*) | 2021-03-08 | **23 file** (15 modificati, 8 aggiunti) su 1.226 (1.218 baseline) |
| **`riscv-opcodes`** | `riscv/riscv-opcodes` | `ad36fb9f80a84154b6028592142988f0344054cc` (*Merge PR #63 from ben-marshall/scalar-crypto*) | 2021-03-08 | **4 file** (4 modificati) su 38 |
| **`riscv-pk` (BBL / SM / PK)** | `riscv-software-src/riscv-pk` | `108c414a7730e9666a787d1bc55229d5cb07f7ce` (*pk: remove linear VA mapping constraint*) con `b7f64fe5584094a6fe65d94d5e2382ff9289cbda` (*Add statx syscall #234*) escluso/revertito | 2021-02-10 / 2021-03-25 | **26 file** (17 modificati, 9 aggiunti) su 444 (435 baseline) |
| **`cva6` (Submodule RTL)** | `pulp-platform/cva6` (fork `davidschrammel/cva6` branch `spearv`) | `6265f1d7151ac2ae75441b943af602178a7cf054` (*vcs: Compatibility improvements #626*, 2021-03-05) → commit SPEAR-V `19eaf312051db59453a0b30df04504cfab820ba3` | 2023-09-04 (SPEAR-V) | **25 file** (25 modificati, +1449 / -101 righe) |

### Patch generate in `patches/`

- `patches/spearv-full.patch` — Patch completa (53 file) applicabile dalla root del repo sopra il commit di baseline (`b291c08`).
- `patches/riscv-isa-sim-spearv.patch` — Patch standalone per Spike (`riscv-isa-sim` @ `06f630cb30c6`).
- `patches/riscv-opcodes-spearv.patch` — Patch standalone per `riscv-opcodes` @ `ad36fb9f80a8`.
- `patches/riscv-pk-spearv.patch` — Patch standalone per BBL/Security Monitor/PK (`riscv-pk` @ `108c414a7730` senza `b7f64fe`).
- `patches/cva6-spearv.patch` — Diff hardware RTL di CVA6 estratto dal submodule (`6265f1d..19eaf31`) come riferimento architetturale per il porting hardware su gem5.

---

## 2. Analisi delle Modifiche per Componente

### 2.1 `riscv-opcodes` (4 file modificati)

Definisce le istruzioni custom, i CSR e i codici di eccezione di SPEAR-V usati per generare `encoding.h` in Spike e `riscv-pk`.

| File | Stato | Modifiche Principali |
| :--- | :--- | :--- |
| `opcodes-custom` | MOD | Aggiunte le istruzioni custom: `mtrap` (`14..12=0, 6..2=0x1E, 1..0=3` -> `0x7b`, usato come porta per le SM API call), `rdtlb` (`0x307b`), `ritlb` (`0x407b`) su `custom-3`, `fence_t` (`6..2=0x02, 1..0=3` -> `0xb`, opcode `custom-0` per il flush microarchitetturale). Presenti commentate `stag` e `ppage`. |
| `parse_opcodes` | MOD | Aggiunti **7 cause code custom** (`0x19`–`0x1f`): `mpkey mismatch fault` (`0x19`), `enclave access fault` (`0x1a`), `meid mismatch fault` (`0x1b`), `enc integrity fault` (`0x1c`), `satp mode change` (`0x1d`), `st imm fault` (`0x1e`), `hpce fault` (`0x1f`). Aggiunti **11 CSR custom** (`0x066`–`0x076`): `u_tag_size`, `u_tag_mode`, `u_tag_storage_base`, `u_mpkey_config`, `u_eam`, `u_meid`, `u_tlb_size`, `u_emp`, `u_tcs`, `u_tag_control`, `u_tag_dram_base`. |
| `encoding.h` | MOD | Rigenerato con match/mask, `CSR_U_*` e `CAUSE_*` SPEAR-V; ripristinate costanti di piattaforma (`DEFAULT_RSTVEC`, `CLINT_BASE`, `DRAM_BASE`, ecc.). |
| `Makefile` | MOD | Disabilitata generazione `OPENOCD_H` (`OPENOCD_H :=`) e aggiunto target `clean`. |

---

### 2.2 `riscv-isa-sim` / Spike (23 file: 15 modificati, 8 aggiunti)

Spike emula i controlli hardware di SPEAR-V su traduzione MMU, tag store in RAM, CSR e fault.

#### Moduli e header di tagging (aggiunti)
- `riscv/tag.h`, `riscv/tag.cc`:
  - Classe `tagging`: legge i tag dal tag store in RAM (`get_tag`, `get_tag_from_ppn`), calcolando `tag_addr = tag_storage_base + (ppn - tag_dram_base) * tag_size_bytes` per `MTAG32` (4B), `MTAG64` (8B) e `MTAG128` (16B).
  - Metodo `tagcheck(...)`:
    - Bypassa i check in `PRV_M`.
    - Ignora indirizzi fisici fuori dall'intervallo taggabile `[tag_dram_base, (tag_storage_base/4096) - 1]`.
    - Impedisce `STORE` su pagine con `tag.bit.immutable == 1` (lancia `trap_enclave_access`, a meno che `TAG_CTRL_DISABLE_IMM_WRITE_CHECK` sia impostato).
    - Blocca qualsiasi accesso U/S a pagine `PT_MONITOR` (`trap_enclave_access`).
    - Se `meid != 0` e la pagina è `PT_NORMAL`: consente l'accesso solo se `eam == EAM_NORMAL`, altrimenti `trap_enclave_access` (protezione della memoria host da enclave).
    - Se la pagina è `PT_ENCLAVE`: verifica `meid != 0` e `meid == tag.bit.id` (`trap_enclave_access`), e in `MTAG64`/`MTAG128` verifica che `tag.bit.vpn` e `tag.bit.pte_perms` corrispondano alla traduzione corrente (`trap_enc_integrity_fault`).
    - In `MTAG128`: verifica le chiavi MPK tramite `mpk_match()` contro i 4 slot di `mpk_config` (`trap_mpkey_mismatch`).

#### Istruzioni custom (aggiunte)
- `riscv/insns/mtrap.h`, `riscv/insns/stag.h`, `riscv/insns/ppage.h`: lanciano `trap_illegal_instruction(0)` in modo che l'istruzione `.word 0x7b` (`mtrap`) intrappoli in M-mode dove il Security Monitor (`tag_trap`) gestisce la SM API call.
- `riscv/insns/fence_t.h`: verifica privilegio S/M (o `MSTATUS_TVM`) e chiama `MMU.flush_tlb()`.
- `riscv/insns/rdtlb.h`, `riscv/insns/ritlb.h`: stub di debug che restituiscono `0xDEAD`.

#### File modificati in Spike
- `riscv/mmu.h`, `riscv/mmu.cc`: `mmu_t` istanzia `tagging_`; `walk()` restituisce anche `leaf_pte` e `pte_level`; `translate()` invoca `tag_check()` quando `proc->state.tag_storage_base != 0` prima di `pmp_ok()`.
- `riscv/processor.h`, `riscv/processor.cc`:
  - Aggiunti a `state_t`: `tag_addr`, `tag_storage_base`, `tag_dram_base`, `tag_mode`, `mpk_config`, `meid`, `tcs`, `eam`, `emp`, `tag_control`.
  - Aggiunto `archen_t` a `insn_desc_t` per il filtraggio RV32/RV64.
  - Gestione lettura/scrittura in `set_csr()` / `get_csr()` per i CSR `CSR_U_*` (`0x067`–`0x076`, con `CSR_U_TAG_SIZE` in sola lettura in `get_csr()`, mentre `CSR_U_TLB_SIZE` è definito in `encoding.h` ma non gestito in `processor.cc`; nota anche che `CSR_U_TAG_DRAM_BASE` è referenziato in `processor.cc` ma mancava ancora dalla rigenerazione di `riscv-isa-sim/riscv/encoding.h` pur essendo presente in `riscv-opcodes/parse_opcodes` e `riscv-pk/machine/encoding.h`).
  - Su scrittura di `CSR_SATP`: se `state.tag_storage_base != 0` e si tenta di spegnere la memoria virtuale (`SATP64_MODE` da != `OFF` a `SATP_MODE_OFF`), lancia `trap_satp_mode_change()`.
- `riscv/trap.h`: dichiarate le macro `DECLARE_MEM_TRAP` / `DECLARE_TRAP` per `mpkey_mismatch`, `enclave_access`, `meid_mismatch`, `hpce`, `satp_mode_change`, `enc_integrity_fault`.
- `riscv/encoding.h`: aggiornato con istruzioni, CSR e cause SPEAR-V.
- `riscv/decode.h`: aggiunta macro `REQUIRE_TAG` e piccole modifiche alle macro `dirty_*_state`/`require_*`.
- `riscv/common.h`, `riscv/interactive.cc`, `riscv/sim.h`, `riscv/sim.cc`, `spike_main/spike.cc`, `spike_main/spike-log-parser.cc`: parametri CLI `--tag-width`, `--tag-granularity`, `--log-level`, comando interattivo `debugnoise`/`dn`, e propagazione di `tag_width`/`tag_granularity` a `sim_t`/`processor_t`/`mmu_t`.
- `riscv/riscv.mk.in`, `fesvr/fesvr.mk.in`: aggiunti `tag.h`, `tag.cc`, `mtrap`, `fence_t` alla build (commentate alcune estensioni non usate come V/Zfh/B/Q/K per velocizzare la compilazione); abilitato `fesvr_install_shared_lib = yes`.

---

### 2.3 `riscv-pk` / Security Monitor & Proxy Kernel (26 file: 17 modificati, 9 aggiunti)

Contiene il **Security Monitor (SM)** in M-mode (sotto `machine/`, compilato sia per `bbl` che per `pk`) e adattamenti al Proxy Kernel (`pk/`).

#### Security Monitor (`machine/`)
- `machine/tag_api.h` (aggiunto):
  - Definizione di `union memory_tag` (128 bit: `id` [16b], `page_type` [3b], `hpce` [1b], `page_level` [2b], `validated` [1b], `immutable` [1b], `vpn` [36b], `pte_perms` [4b], `mpk` [10b], `unused` [54b]).
  - Enum `TagMode` (`MTAG0..MTAG128`), `EnclaveAccessMode`, `PageType` (`PT_NORMAL=0`, `PT_ENCLAVE=1`, `PT_MONITOR=2`, `PT_SHARED=3`, `PT_PAGETABLE=4`), `PageLevel`.
  - Bit di controllo `TAG_CTRL_*` per `CSR_U_TAG_CONTROL`.
  - Numeri di API SM (`LTAG=0`, `STAG=1`, `PPAGE=2`, `ECREATE=3`, `EENTER=4`, `EEXIT=5`, `EDESTROY=6`, `LTAG2=7`, `EADD=8`, `EREMOVE=9`, `TAGRESET=10`, `API_DEBUG=11`, `API_SET_FENCE=12`, `SHM_CREATE=13`, `SHM_ACK=14`, `SHM_DESTROY=15`) e macro inline `ourthing_1..7` che emettono `.word 0x7b` (`mtrap`).
- `machine/tag.h`, `machine/tag.c` (aggiunti):
  - Strutture metadati enclave: `e_metapage` (ECS: stato, `id`, `parent_id`, `entry`, `codesize`, `satp`, `tcs`, `sharing_array[64]`, puntatori per nesting `current_child_status_page`, `root_parent_status_page`, `parent_status_page`, `child_ctr`) e `e_tcs_t` (stack, salvataggio registri host/enclave e CSR).
  - Allocatore EID a bitmap (`_allocate_eid`, `_deallocate_eid`) su 16 bit.
  - Dispatcher `trap_m_call`: gestisce `ECREATE`, `EENTER`, `EEXIT`, `EDESTROY`, `EADD`, `EREMOVE`, `PPAGE`, `LTAG`, `LTAG2`, `STAG`, `TAGRESET`, `API_DEBUG`, `API_SET_FENCE`.
  - Protezione delle page table (`ppage`) e controllo anti-aliasing ricorsivo (`ppn_is_mapped`, con ottimizzazione tramite bit `validated` sulle PT che contengono solo pagine enclave).
  - Gestione fault su pagine immutabili (`enclave_access_trap`, `emulate_sd_to_immutable_page` per emulare `sd`/`c.sd`/`amoswap.d` su PTE non-enclave in PT immutabili) e huge page (`hpce_trap`, `_tag_megapage`, `_clear_megapage_tag_unlocked`).
  - Context switch sicuro e supporto **enclave nesting** + **interrupt handling** (`_switch_context_host_to_enclave`, `_switch_context_enclave_to_host`, `trap_vector_enclave_c`, salvataggio/ripristino `mideleg`/`medeleg`/`mtvec` e forwarding di `MTIP`/`MSIP`/`MEIP` verso `STIP`/`SSIP`/`SEIP`).
- `machine/tag_tagstore.c` (aggiunto):
  - `get_max_tagstorage_size()`, `tag_init()` (azzera il tag store alla fine della DRAM fisica, imposta `CSR_U_TAG_STORAGE_BASE`, `CSR_U_TAG_MODE`, `CSR_U_TAG_CONTROL` e **auto-protegge** le pagine del tag store impostando `.id = EID_INVALID`, `.immutable = 1`), `read_tag()`, `write_tag()`.
- `machine/tag_virtual_memory.c` (aggiunto):
  - Page table walk in M-mode (`walk_next`, `walk`, `getPPN`, `getPPN4K`, `walk_ppns`) per risolvere VA → PPN/permessi e gestire demand paging tramite `access_range_va`.
- `machine/flush.h` (aggiunto):
  - Supporto all'istruzione `fence.t` (`0xB` con mask `FLUSH_IF`, `FLUSH_ID`, `FLUSH_EX`, `FLUSH_DCACHE`, `FLUSH_ICACHE`, `FLUSH_TLB`, `FLUSH_BP`, ecc.) per svuotare i buffer microarchitetturali all'ingresso/uscita dall'enclave, più `sfence_vma` su aggiornamento tag.
- `machine/tag_debug.h`, `machine/tag_debug.c` (vuoto), `machine/flush_icache.h` (aggiunti).
- File modificati in `machine/` e `bbl/`:
  - `machine/emulation.c`: sostituisce l'entry `custom-3` (`0x7b`) in `illegal_insn_trap_table` con `tag_trap` → `trap_m_call()`.
  - `machine/mentry.S`: aggiunge i vettori `25..31` in `trap_table` (`mpkey_mismatch_trap`, `enclave_access_trap`, `enc_integrity_trap`, `satp_mode_change_trap`, `hpce_trap`) e il nuovo entry point `trap_vector_enclave` per gli interrupt durante l'esecuzione dell'enclave.
  - `machine/minit.c`: esporta `mem_size_final` e `mem_base`, chiama `tag_init(mem_base, mem_size, MTAG0, 0)` in `memory_init()`.
  - `machine/fdt.c`, `machine/fdt.h`, `bbl/bbl.c`: aggiunge `filter_tagged_memory()` per ridurre la dimensione della RAM visibile nel Device Tree (FDT) sottraendo la memoria riservata al tag store.
  - `machine/mtrap.c`, `machine/mtrap.h`, `machine/encoding.h`, `machine/machine.mk.in`: handler dei trap (`mpkey_mismatch_trap`, `enc_integrity_trap`, `satp_mode_change_trap`), dump dei registri in `bad_trap`, dichiarazione sorgenti tag.

#### Proxy Kernel (`pk/`)
- `pk/mmap.c`, `pk/mmap.h`: usa `mem_size_final` invece di `mem_size` per l'allocatore di pagine e kernel mapping, evitando di allocare le pagine fisiche riservate al tag store; aggiunge `debug_pte()`.
- `pk/handlers.c`, `pk/syscall.c`, `pk/syscall.h`: aggiunge `_sys_set_timer` (`SYS_set_timer`), `_sys_debug_pte` (`SYS_debug_pte`), `_sys_debug` (`SYS_debug`) e gestione `SIP_STIP` in `handle_interrupt`.
- `pk/pk.c`, `.gitignore`: usa `__riscv_flush_icache()`.

---

### 2.4 `cva6` Hardware RTL (`patches/cva6-spearv.patch`, 25 file)

Il riferimento hardware CVA6 implementa in RTL i meccanismi che andranno replicati nel modello di **gem5**:

- `include/riscv_pkg.sv`, `include/ariane_pkg.sv`: definizioni `mtag_t`, `tlb_update_tagbits_t`, `CSR_U_*`, eccezioni `25..31`, `mtag_control_t`, `mpkey_config_t`, `eam_t`, `tagmode_t`. (Nota: in `cva6` `PT_PAGETABLE = 3` anziché `4`, e `CSR_U_TAG_MODE` = `0x067`).
- `src/csr_regfile.sv`: registri CSR di memory tagging (`tag_mode_q`, `tag_storage_base_q`, `tag_dram_base_q`, `eam_q`, `meid_q`, `tcs_q`, `tag_control_q`, `mpkey_config_q`) e controllo `satp_mode_exception` quando S-mode tenta di disattivare la traduzione (`ModeOff`).
- `src/ptw.sv`: estensione della FSM del Page Table Walker con gli stati `TAG_LOOKUP`, `TAG_FINAL`, `PROPAGATE_ENC_ERROR`:
  - Calcola `ppn_to_tag_paddr()` ad ogni livello di paging (partendo da `satp_ppn_i` fino al leaf).
  - Propaga l'immutabilità delle page table (`pt_tag_wd_q & tag.immutable`).
  - Verifica che per pagine `PT_ENCLAVE` / `PT_MONITOR` le PT siano immutabili (`!pt_tag_wd_q`), che `tag.page_level` corrisponda al livello del PTW e (con `INCLUDE_FUTURE`) che `tag.vpn` e `tag.pte_perms` combacino.
  - Gestisce `tag.hpce` sulle megapage (`LVL2`) forzando un ulteriore fetch del tag 4KiB e il downgrade della entry TLB a 4KiB (`LVL3`).
  - Marca le richieste PTW come non-cacheable (`req2_port_o.nc = tag_control_i.nc_ptw_in_e`) dentro l'enclave (propagato a `wt_dcache_ctrl.sv` / `wt_cache_subsystem.sv`).
- `src/tlb.sv`: memorizza nei tag TLB `e` (enclave), `imm` (immutable), `hpce`, e usa `tags_q[i].asid` per memorizzare `EID` quando `tags_q[i].e == 1`, confrontandolo con `meid_i` durante il lookup.
- `src/mmu.sv`: esegue i controlli su ITLB/DTLB hit (`itlb_enclave_acc_err`, `dtlb_enclave_acc_err`, `dtlb_imm_st_err`, `dtlb_hpce_err`, `mpkey_mismatch_err`) in parallelo ai check PMP/PTE standard.
- `src/decoder.sv`, `src/commit_stage.sv`, `src/controller.sv`, `src/cache_subsystem/*`: decodifica ed esecuzione di `fence.t` (`CUSTOM_FENCE`) e istruzioni debug `rdtlb`/`ritlb` (`MTAGDEBUG`).

---

## 3. Piano di Porting su gem5

### 3.1 Scelta dell'Implementazione di Riferimento: RTL (`cva6`) vs Spike (`riscv-isa-sim`)

Per il porting di SPEAR-V su **gem5**, la scelta architetturale ottimale è **basarsi primariamente sull'implementazione hardware RTL di `cva6`**, integrandola puntualmente con le definizioni ABI e i corner case funzionali di **Spike (`riscv-isa-sim`) / `riscv-pk`** laddove il codice RTL presenta semplificazioni da prototipo FPGA o disallineamenti rispetto al Security Monitor.

#### Motivazioni della scelta

1. **Isomorfismo strutturale tra CVA6 e il sottosistema MMU di gem5**:
   - **Spike** è un simulatore puramente funzionale (ISA-level): in `riscv-isa-sim/riscv/mmu.cc` e `riscv/tag.cc`, `mmu_t::translate()` invoca `tagging_->tagcheck()` **ad ogni singolo accesso in memoria**, leggendo il tag direttamente dall'array RAM dell'host (`sim->addr_to_mem(tag_addr)`) in tempo zero e senza memorizzare alcun metadato di tagging nelle entry della TLB.
   - **gem5** (`src/arch/riscv/tlb.hh/.cc`, `pagetable.hh`, `pagetable_walker.hh/.cc`), esattamente come **CVA6** (`src/tlb.sv`, `src/ptw.sv`, `src/mmu.sv`), separa nettamente:
     - il **Page Table Walker (`RiscvISA::Walker::WalkerState`)**, una macchina a stati finiti che emette pacchetti di lettura `PacketPtr` (`MemCmd::ReadReq`) sulla `WalkerPort` verso la gerarchia di cache/memoria;
     - la **TLB (`RiscvISA::TLB` e `RiscvISA::TlbEntry`)**, che memorizza le traduzioni completate e applica i controlli di protezione ad ogni TLB hit.
2. **Fedeltà del modello prestazionale e del meccanismo chiave di SPEAR-V**:
   - Il contributo architetturale centrale di SPEAR-V (ASIA CCS '23) consiste nell'eseguire il fetch dei memory tag dal Tag Store in DRAM **esclusivamente durante il Page Table Walk (su TLB miss)** e nel cachare i bit di sicurezza (`e`, `imm`, `hpce`, `eid`, `mpk`, `pte_perms`) dentro la entry TLB, rendendo i controlli su TLB hit a costo zero.
   - Replicare l'approccio di Spike in gem5 (leggendo il tag ad ogni `TLB::translate`) snaturerebbe il design hardware: se implementato tramite pacchetti di memoria, introdurrebbe un accesso extra in RAM per ogni load/store; se implementato tramite letture funzionali *backdoor*, falserebbe completamente le statistiche temporali (`TimingSimpleCPU`, `MinorCPU`, `O3CPU`) azzerando il costo reale dei fetch dei tag durante il PTW (1 lettura da 8B per `MTAG32`/`MTAG64`, 2 letture da 8B per `MTAG128` ad ogni livello di paginazione) e l'effetto del flag non-cacheable `TAG_CTRL_NC_PTW_IN_E`.
3. **Meccanismi di sicurezza presenti solo nell'RTL di CVA6 (assenti o commentati in Spike)**:
   - **Immutabilità delle Page Table durante il walk (`pt_tag_wd_q`)**: CVA6 (`src/ptw.sv`) legge il tag di ogni livello dell'albero di paginazione (a partire dalla root PT puntata da `satp.ppn`), accumulando `pt_tag_wd_q &= tag.immutable` e sollevando `ENC_INTEGRITY_FAULT` (`28`) se una foglia `PT_ENCLAVE` o `PT_MONITOR` viene raggiunta attraversando page table non immutabili. Spike (`mmu.cc` / `tag.cc`) controlla unicamente il tag della pagina fisica finale, ignorando i tag delle page table intermedie.
   - **Gestione delle Huge Page (`HPCE` — Huge Page Containing Enclave)**: in Spike il controllo `hpce` in `tag.cc` (righe 155–162) è commentato perché `get_tag()` ricalcola sempre il tag 4KiB ad ogni accesso. In gem5 (dove `TlbEntryTrie` effettua il match sull'intera superpagina tramite `logBytes`), una megapage da 2MiB (`logBytes = 21`) inserita in TLB produrrebbe hit su tutte le 512 sottopagine da 4KiB senza più invocare il PTW. È quindi indispensabile il meccanismo RTL di `cva6/src/ptw.sv`: quando un walk al livello 2MiB (`LVL2`) rileva `tag.hpce == 1`, il PTW deve eseguire un ulteriore fetch del tag della sottopagina 4KiB (`pte.ppn | vaddr[20:12]`) e inserire nella TLB una entry degradata a 4KiB (`LVL3`, `logBytes = 12`), oltre a sollevare `HPCE_FAULT` (`31`) su eventuale hit TLB 2MiB con `hpce == 1`.
   - **Isolamento microarchitetturale (`fence.t` e `TAG_CTRL_NC_PTW_IN_E`)**: CVA6 implementa in `src/ptw.sv` e `src/cache_subsystem/wt_dcache_ctrl.sv` il bypass della cache per le richieste del PTW quando `nc_ptw_in_e` è attivo e in `src/controller.sv` il flush microarchitetturale selettivo su `fence.t`, mentre Spike riduce `fence.t` a un semplice `MMU.flush_tlb()`.

---

### 3.2 Disallineamenti Critici tra CVA6 RTL, Spike e `riscv-pk` da Risolvere nel Porting

L'analisi incrociata delle patch ha messo in luce **5 discrepanze** tra il codice SystemVerilog di `cva6` e il codice C del Security Monitor (`riscv-pk`) / `Spike`. Nel porting su gem5, la struttura microarchitetturale seguirà `cva6`, ma queste specifiche costanti e condizioni dovranno essere allineate a `riscv-pk`:

| # | Aspetto | Implementazione `cva6` (RTL) | Implementazione `riscv-pk` / `Spike` | Scelta per gem5 |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Valori enum `PageType`** | `PT_NORMAL=0`, `PT_ENCLAVE=1`, `PT_MONITOR=2`, `PT_PAGETABLE=3` (`cva6/include/riscv_pkg.sv`, manca `PT_SHARED`). | `PT_NORMAL=0`, `PT_ENCLAVE=1`, `PT_MONITOR=2`, `PT_SHARED=3`, `PT_PAGETABLE=4` (`riscv-pk/machine/tag_api.h`). | **Allineare a `riscv-pk` (`PT_SHARED=3`, `PT_PAGETABLE=4`)**, poiché il Security Monitor scrive `4` nei tag delle page table in `ppage()`. |
| **2** | **Valori e semantica di `CSR_U_EAM`** | `EAM_NORMAL=0`, `EAM_ENCLAVE=1`, `EAM_SHARED=2` (`riscv_pkg.sv`). In `mmu.sv`, se `eam == EAM_NORMAL` dentro un'enclave, gli accessi DTLB a `PT_ENCLAVE` sono bloccati. | `EAM_NORMAL=0`, `EAM_SHM=1`, `EAM_ENCLAVE=2` (`tag_api.h`). `tag.c` scrive `EAM_ENCLAVE=2`. In `tag.cc`, `EAM_NORMAL` consente all'enclave di accedere sia a `PT_NORMAL` sia alle proprie pagine `PT_ENCLAVE`. | **Adottare i valori di `tag_api.h` (`EAM_ENCLAVE=2`)**; consentire DTLB hit su `PT_ENCLAVE` (con `meid == eid`) sia con `EAM_ENCLAVE` sia con `EAM_NORMAL`, e su `PT_NORMAL` dentro enclave solo con `EAM_NORMAL` (solo per DTLB, mai per ITLB). |
| **3** | **Protezione di `PT_MONITOR` in TLB** | `ptw.sv` comprime `PT_ENCLAVE` e `PT_MONITOR` in un unico bit `e = 1` con `eid = tag.id`. Poiché `trap_ecreate` in `tag.c` assegna alla `statuspage` (`PT_MONITOR`) lo stesso `id = e_id` dell'enclave, l'enclave in U-mode potrebbe leggerla. | `tag.cc` blocca incondizionatamente ogni accesso non-M-mode a `PT_MONITOR` (`trap_enclave_access`), indipendentemente da `meid`. | **Salvare l'intero `page_type` in `TlbEntry` di gem5**, bloccando su TLB hit qualsiasi accesso U/S a `PT_MONITOR` con `ENCLAVE_ACCESS_FAULT` (`26`). |
| **4** | **Range PPN taggati e Self-Protection del Tag Store** | `ppn_has_tag()` in `ptw.sv` è cablato a `1'b1` (range check commentato). | `tag_tagstore.c` riserva il Tag Store in cima alla DRAM e lo **auto-protegge** scrivendo nei tag corrispondenti `.id = EID_INVALID, .immutable = 1`. In Spike (`tag.cc`), tuttavia, `max_ppn = (tag_storage_base/4096) - 1` escludeva il Tag Store stesso dai check. | **Verificare `ppn >= tag_dram_base && ppn < tag_storage_end_ppn`** (coprendo fino alla fine della DRAM / fine del Tag Store), così che la *self-protection* del Tag Store configurata da `tag_init()` sia realmente applicata dall'hardware. |
| **5** | **Codifica `page_level` sulle `PT_PAGETABLE`** | `ptw.sv` confronta `LVL1->PL_GIGA(2)`, `LVL2->PL_MEGA(1)`, `LVL3->PL_KILO(0)` sia per `PT_ENCLAVE` sia per `PT_PAGETABLE`. | In `tag.c` (`ppage`), `depth` vale `0` per la foglia 4KiB, ma `1, 2, 3` per i 3 livelli di PT (scrivendo `page_level = 3` sulla root PT). Perciò `tag_init()` imposta di default `TAG_CTRL_DISABLE_IMM_PT_CHECKS`. | In gem5 (`level` `2=1G`, `1=2M`, `0=4K` coincide numericamente con `PL_GIGA=2, PL_MEGA=1, PL_KILO=0`), verificare `tag.page_level == level` sui PTE foglia `PT_ENCLAVE` e condizionare i check sulle PT intermedie a `!(tag_control & TAG_CTRL_DISABLE_IMM_PT_CHECKS)`. |

---

### 3.3 Mappatura File-per-File verso gem5 (`src/arch/riscv/`)

La tabella seguente mappa ogni file modificato nei componenti SPEAR-V (`riscv-opcodes`, `cva6`, `riscv-isa-sim`) sul corrispondente file del modello RISC-V di **gem5**:

| File Sorgente SPEAR-V (`cva6` / `Spike` / `opcodes`) | Ruolo in SPEAR-V | File Target in gem5 (`src/arch/riscv/`) | Modifiche da Apportare in gem5 |
| :--- | :--- | :--- | :--- |
| `riscv-pk/machine/tag_api.h`, `cva6/include/riscv_pkg.sv`, `cva6/include/ariane_pkg.sv` | Strutture `memory_tag` / `mtag_t`, enum `TagMode`, `PageType`, `PageLevel`, `EnclaveAccessMode`, `mpkey_config_t`, `TAG_CTRL_*`. | **`spearv.hh`** *(nuovo)* | Definizione di `BitUnion` / struct per `MemoryTag` (128 bit), enum `TagMode`, `PageType`, `PageLevel`, `EnclaveAccessMode`, flag `TagControl` e helper di calcolo indirizzo tag. |
| `riscv-opcodes/parse_opcodes`, `cva6/include/riscv_pkg.sv`, `riscv-isa-sim/riscv/trap.h` | Cause code di eccezione custom `25..31` (`0x19`–`0x1f`). | **`fault_codes.hh`**, **`faults.hh`**, **`faults.cc`** | Aggiunta di `MPKEY_MISMATCH_FAULT = 25`, `ENCLAVE_ACCESS_FAULT = 26`, `MEID_MISMATCH_FAULT = 27`, `ENC_INTEGRITY_FAULT = 28`, `SATP_MODE_CHANGE = 29`, `ST_IMM_FAULT = 30`, `HPCE_FAULT = 31` in `ExceptionCode` (non delegabili in `DELEGABLE_EXCPS`). |
| `riscv-opcodes/parse_opcodes`, `cva6/src/csr_regfile.sv`, `riscv-isa-sim/riscv/processor.cc` | 11 CSR custom (`0x066`–`0x076`) e trap su disattivazione `SATP` in S-mode. | **`regs/misc.hh`**, **`isa.hh`**, **`isa.cc`** | Registrazione `MISCREG_U_TAG_SIZE`..`MISCREG_U_TAG_DRAM_BASE` in `MiscRegIndex` e `CSRData`; logica read/write in `ISA::readMiscReg()` / `ISA::setMiscReg()`; controllo cambio `SATP.mode -> BARE` in `PRV_S` quando il tagging è attivo. |
| `cva6/include/ariane_pkg.sv` (`tlb_update_tagbits_t`), `cva6/src/tlb.sv` | Estensione entry TLB con metadati del tag (`e`, `imm`, `hpce`, `pt`, `eid`, `mpk`, `pte_perms`). | **`pagetable.hh`**, **`pagetable.cc`** | Estensione di `struct TlbEntry` con i campi di sicurezza SPEAR-V e relativo supporto in `reset()`, `serialize()` e `unserialize()`. |
| `cva6/src/tlb.sv`, `cva6/src/mmu.sv`, `riscv-isa-sim/riscv/tag.cc` | Controllo tag su ITLB/DTLB hit (isolamento enclave `meid`/`eid`, `eam`, store su `imm`, `PT_MONITOR`, `mpk`, `hpce`). | **`tlb.hh`**, **`tlb.cc`** | Implementazione di `TLB::checkSpearVPermissions()` chiamata in `TLB::doTranslate()` (e `Walker::startWalkWrapper()`) su TLB hit; gestione del matching `eid` vs `asid`. |
| `cva6/src/ptw.sv`, `cva6/src/cache_subsystem/wt_dcache_ctrl.sv` | FSM del Page Table Walker con stati `TAG_LOOKUP` / `TAG_FINAL`, verifica immutabilità PT, VPN/permessi PTE, downgrade 2MiB→4KiB su `hpce` e flag non-cacheable `nc_ptw_in_e`. | **`pagetable_walker.hh`**, **`pagetable_walker.cc`** | Estensione di `Walker::WalkerState` per emettere letture `PacketPtr` verso il Tag Store ad ogni step del walk (sia in `Atomic`/`Functional` sia in `Timing`), validare i tag intermedi/foglia, gestire `hpce` e impostare `Request::UNCACHEABLE` se `TAG_CTRL_NC_PTW_IN_E`. |
| `riscv-opcodes/opcodes-custom`, `cva6/src/decoder.sv`, `cva6/src/commit_stage.sv`, `cva6/src/controller.sv`, `riscv-isa-sim/riscv/insns/{mtrap,fence_t,rdtlb,ritlb}.h` | Istruzioni custom `mtrap` (`0x7b`), `fence.t` (`0x0b`), `rdtlb` (`0x307b`), `ritlb` (`0x407b`). | **`isa/decoder.isa`** | Decodifica di `fence_t` su `custom-0` (flush TLB + serializzazione pipeline), `rdtlb`/`ritlb` su `custom-3` (`funct3 = 3, 4`) e generazione `IllegalInstFault` per `mtrap` (`0x7b`, `funct3 = 0`) per intrappolare nel Security Monitor. |
| `riscv/riscv.mk.in` | Build system. | **`SConscript`** | Registrazione di eventuali nuovi file sorgente (es. `spearv.cc` se separato) e flag di debug (`PageTableWalker`, `TLB`). |

---

### 3.4 Piano Operativo di Implementazione Passo-Passo

#### Fase 1: Definizione Tipi Tag, CSR Custom e Codici di Eccezione
1. **Creazione di `src/arch/riscv/spearv.hh`**:
   - Definire la struttura `MemoryTag` a 128 bit (`uint64_t word[2]`):
     - **Parola 0 (bit 0..63)**: `id` (`[15:0]`), `page_type` (`[18:16]`), `hpce` (`[19]`), `page_level` (`[21:20]`), `validated` (`[22]`), `immutable` (`[23]`), `vpn` (`[59:24]`, 36 bit), `pte_perms` (`[63:60]`, 4 bit `u, x, w, r` corrispondenti a `(pte >> 1) & 0xf`).
     - **Parola 1 (bit 64..127)**: `mpk` (`[73:64]`, 10 bit), `unused` (`[127:74]`, 54 bit).
   - Definire `MpKeyConfig` (4 slot da 11 bit in `uint64_t`: `slot0_mpk [9:0]`, `slot0_wd [10]`, ..., `slot3_mpk [41:32]`, `slot3_wd [42]`).
   - Definire le costanti `TAG_CTRL_DISABLE_TAG_COMPARE_CHECKS (1 << 0)`, `TAG_CTRL_DISABLE_IMM_PT_CHECKS (1 << 1)`, `TAG_CTRL_NC_PTW_IN_E (1 << 2)`, `TAG_CTRL_DISABLE_IMM_WRITE_CHECK (1 << 3)`.
2. **Aggiunta dei CSR in `src/arch/riscv/regs/misc.hh` e `src/arch/riscv/isa.cc`**:
   - Aggiungere in `MiscRegIndex` e nella mappa `CSRData` gli 11 CSR (`0x066`–`0x076`):
     - `CSR_U_TAG_SIZE` (`0x066`): sola lettura, restituisce `0`, `32`, `64` o `128` in base al valore corrente di `MISCREG_U_TAG_MODE`.
     - `CSR_U_TAG_MODE` (`0x067`): 2 bit (`MTAG0=0`, `MTAG32=1`, `MTAG64=2`, `MTAG128=3`), inizializzato a `MTAG0` al reset.
     - `CSR_U_TAG_STORAGE_BASE` (`0x068`), `CSR_U_MPKEY_CONFIG` (`0x069`), `CSR_U_EAM` (`0x070`), `CSR_U_MEID` (`0x071`), `CSR_U_EMP` (`0x073`), `CSR_U_TCS` (`0x074`), `CSR_U_TAG_CONTROL` (`0x075`), `CSR_U_TAG_DRAM_BASE` (`0x076`).
     - `CSR_U_TLB_SIZE` (`0x072`): sola lettura, restituisce la dimensione configurata della TLB (es. `16` o `size`).
   - Nota sui privilegi CSR: benché gli indirizzi `0x066`–`0x076` abbiano i bit `[9:8] == 2'b00` (spazio U-mode nello standard RISC-V, scelti così dagli autori in `cva6` per prototipazione rapida), nel caso si voglia restringerne l'accesso o mantenerlo fedele a `cva6`/`Spike`, va verificato che le letture/scritture eseguite da `riscv-pk` in M-mode non vengano bloccate dal check sui permessi CSR di gem5 (in particolare per `CSR_U_TAG_SIZE` e `CSR_U_TLB_SIZE`).
   - In `ISA::setMiscReg()`, intercettare la scrittura a `MISCREG_SATP`: se `prv == PRV_S`, `readMiscRegNoEffect(MISCREG_U_TAG_MODE) != MTAG0`, `old_satp.mode != AddrXlateMode::BARE` e `new_satp.mode == AddrXlateMode::BARE`, non aggiornare `SATP` e sollevare il fault `SATP_MODE_CHANGE` (`29`) (o segnalarlo all'istruzione `csrw satp`).
3. **Aggiunta delle eccezioni custom in `src/arch/riscv/fault_codes.hh` e `src/arch/riscv/faults.hh/.cc`**:
   - Aggiungere i codici `25..31` in `enum ExceptionCode`.
   - Non inserirli in `DELEGABLE_EXCPS`: qualsiasi eccezione SPEAR-V (`25..31`) deve sempre intrappolare in M-mode (`trap_table` in `riscv-pk/machine/mentry.S`) salvando l'indirizzo virtuale faultante in `mtval` (`mbadaddr`).

#### Fase 2: Estensione della TLB e Controlli su TLB Hit (`pagetable.hh`, `tlb.hh`, `tlb.cc`)
1. **Estensione di `TlbEntry` (`src/arch/riscv/pagetable.hh`)**:
   - Aggiungere i campi:
     - `bool tagValid = false;`
     - `bool e = false;` (`page_type == PT_ENCLAVE`)
     - `bool imm = false;` (`tag.immutable`)
     - `bool hpce = false;` (`tag.hpce`)
     - `uint8_t pageType = 0;` (`PT_NORMAL`, `PT_ENCLAVE`, `PT_MONITOR`, `PT_SHARED`, `PT_PAGETABLE`)
     - `uint16_t eid = 0;` (`tag.id`)
     - `uint16_t mpk = 0;` (`tag.mpk`, 10 bit)
     - `uint8_t tagPtePerms = 0;` (`tag.pte_perms`, 4 bit)
2. **Gestione ASID / EID in `TLB::lookup()` e `TLB::insert()` (`src/arch/riscv/tlb.cc`)**:
   - In `cva6/src/tlb.sv`, una entry con `e == 1` sostituisce `asid` con `eid` e fa hit quando `tags_q[i].e && tags_q[i].asid == meid_i`.
   - In gem5, poiché `riscv-pk` impiega lo stesso spazio di indirizzamento del processo host (`assert(statuspage_pa->satp == CSRR(CSR_SATP))` in `trap_eenter`), `satp.asid` rimane invariato tra host ed enclave. Pertanto, mantenere `entry.asid = satp.asid` nella chiave `buildKey(vpn, asid)` del `TlbEntryTrie` e verificare separatamente `entry.eid == meid` in `checkSpearVPermissions()` garantisce sia l'assenza di collisioni nel trie sia il corretto isolamento tra enclave e host.
3. **Implementazione di `TLB::checkSpearVPermissions()` (`src/arch/riscv/tlb.cc`)**:
   - Invocare il controllo in `TLB::doTranslate()` (subito dopo `checkPermissions()` e prima dell'accesso fisico) e in `Walker::startWalkWrapper()` (prima di fare squash di un walk in coda):
     1. Se `memaccess.priv == PRV_M` o `u_tag_mode == MTAG0` o `!entry->tagValid`, ritorna `NoFault`.
     2. **Store su pagina immutabile (`dtlb_imm_st_err`)**: se `!(tag_control & TAG_CTRL_DISABLE_IMM_WRITE_CHECK) && mode == BaseMMU::Write && entry->imm`, ritorna `std::make_shared<AddressFault>(vaddr, ExceptionCode::ENCLAVE_ACCESS_FAULT)` (necessario affinché `enclave_access_trap` in `riscv-pk/machine/tag.c` sul vettore `26` emuli le scritture `sd`/`c.sd`/`amoswap.d` sulle page table immutabili).
     3. **Protezione `PT_MONITOR`**: se `entry->pageType == PT_MONITOR`, ritorna `AddressFault(vaddr, ExceptionCode::ENCLAVE_ACCESS_FAULT)`.
     4. **Controlli MPK (`MTAG128`)**: se `u_tag_mode == MTAG128 && !(tag_control & TAG_CTRL_DISABLE_TAG_COMPARE_CHECKS)`, verifica che `entry->mpk` corrisponda ad almeno uno dei 4 slot `slot_i_mpk` in `u_mpkey_config` con `(mode == BaseMMU::Write ? !slot_i_wd : true)`; in caso contrario ritorna `AddressFault(vaddr, ExceptionCode::MPKEY_MISMATCH_FAULT)` (`25`).
     5. **Isolamento Enclave (`dtlb_enclave_acc_err` / `itlb_enclave_acc_err`)**: se `!(tag_control & TAG_CTRL_DISABLE_TAG_COMPARE_CHECKS)`:
        - Sia `bool inside_enclave = (meid != 0)`.
        - Se `mode == BaseMMU::Execute` (ITLB): l'accesso è consentito solo se `(!inside_enclave && !entry->e) || (inside_enclave && entry->e && meid == entry->eid)`; altrimenti ritorna `AddressFault(vaddr, ExceptionCode::ENCLAVE_ACCESS_FAULT)` (`26`).
        - Se `mode != BaseMMU::Execute` (DTLB): l'accesso è consentito solo se:
          - `!inside_enclave && !entry->e` (host accede a pagina non-enclave), oppure
          - `inside_enclave && entry->e && meid == entry->eid && (eam == EAM_ENCLAVE || eam == EAM_NORMAL)` (enclave accede a propria pagina enclave), oppure
          - `inside_enclave && !entry->e && eam == EAM_NORMAL` (enclave accede a memoria host con `EAM_NORMAL` attivo);
          altrimenti ritorna `AddressFault(vaddr, ExceptionCode::ENCLAVE_ACCESS_FAULT)` (`26`).
     6. **Controllo Huge Page (`HPCE`)**: se `entry->hpce && entry->logBytes > PageShift`, ritorna `AddressFault(vaddr, ExceptionCode::HPCE_FAULT)` (`31`).

#### Fase 3: Estensione del Page Table Walker (`pagetable_walker.hh`, `pagetable_walker.cc`)
1. **Stati e registri interni di `Walker::WalkerState`**:
   - Aggiungere in `WalkerState` un enum per la sotto-fase di lettura tag (es. `enum TagWalkPhase { ReadRootPtTagLow, ReadRootPtTagHigh, ReadPte, ReadTargetTagLow, ReadTargetTagHigh, ReadHpce4kTagLow, ReadHpce4kTagHigh }`):
     - All'avvio di `walkOneStage(vaddr)`: se `u_tag_mode != MTAG0` e `ppnHasTag(satp.ppn)`, il primo pacchetto `read` generato non legge subito il PTE di livello 2, ma legge i primi 8 byte del tag della root page table (`ppnToTagPaddr(satp.ppn, false)`), impostando `ptImmAccum = true` e `tagPhase = ReadRootPtTagLow` (esattamente come `tag_first_n = 1'b1; read_tag_n = 1'b1;` in `cva6/src/ptw.sv`, cfr. `patches/cva6-spearv.patch` righe 2426–2436).
2. **Transizioni in `WalkerState::stepWalk()` (supporto `Atomic`, `Functional` e `Timing`)**:
   - **`ReadRootPtTagLow` (e `ReadRootPtTagHigh` se `MTAG128`)**:
     - Legge il tag della root PT (`satp.ppn`), aggiorna `ptImmAccum &= rootTag.immutable`.
     - Se `!(tag_control & TAG_CTRL_DISABLE_TAG_COMPARE_CHECKS) && rootTag.page_type == PT_PAGETABLE && rootTag.page_level != 2` (con la cautela sulla codifica `depth` di `ppage`), solleva `ENC_INTEGRITY_FAULT` (`28`).
     - Prepara quindi il pacchetto `read` per il PTE di livello `level = 2` (`setupWalk(vaddr)`) e passa a `ReadPte`.
   - **`ReadPte`**:
     - Esegue i consueti controlli PMA/PMP e `checkPTEPermissions(pte, stepWalkFlags, level)`.
     - Se il tagging è disabilitato (`u_tag_mode == MTAG0`) o `!ppnHasTag(pte.ppn)`, procede come nel PTW standard di gem5.
     - Se il tagging è attivo: salva `savedPte = pte`, calcola l'indirizzo del tag del PPN puntato dal PTE (`ppnToTagPaddr(pte.ppn, false)`) e passa a `ReadTargetTagLow` (senza concludere ancora il walk anche se `pteIsLeaf == true`).
   - **`ReadTargetTagLow` (e `ReadTargetTagHigh` se `MTAG128`)**:
     - Assembla `currentTag` (64 bit in `MTAG32`/`MTAG64`, 128 bit in `MTAG128` dopo il secondo pacchetto con offset `+8`).
     - **Caso PTE non-foglia (`!savedPte.r && !savedPte.x`)**:
       - Aggiorna `ptImmAccum &= currentTag.immutable`.
       - Verifica eventuale consistenza `PT_PAGETABLE`, decrementa `level--`, calcola l'indirizzo del PTE del livello successivo `(savedPte.ppn << PageShift) + (idx * sizeof(PTESv39))` e torna a `ReadPte`.
     - **Caso PTE foglia (`savedPte.r || savedPte.x`)**:
       - **Controllo 1 (Immutabilità Page Table)**: se `!(tag_control & TAG_CTRL_DISABLE_IMM_PT_CHECKS) && (currentTag.page_type == PT_ENCLAVE || currentTag.page_type == PT_MONITOR) && !ptImmAccum`, termina il walk e ritorna `std::make_shared<AddressFault>(entry.vaddr, ExceptionCode::ENC_INTEGRITY_FAULT)` (`28`).
       - **Controllo 2 (VPN e Permessi PTE in `MTAG64`/`MTAG128`)**: se `!(tag_control & TAG_CTRL_DISABLE_TAG_COMPARE_CHECKS) && currentTag.page_type == PT_ENCLAVE && (u_tag_mode == MTAG64 || u_tag_mode == MTAG128)`:
         - Verifica `currentTag.vpn == getVPNFromVAddr(entry.vaddr, satp.mode)` e `currentTag.pte_perms == ((savedPte >> 1) & 0xf)`. In caso di mismatch, termina il walk e ritorna `AddressFault(entry.vaddr, ExceptionCode::ENC_INTEGRITY_FAULT)` (`28`).
       - **Controllo 3 (Page Level Confusion)**: se `!(tag_control & TAG_CTRL_DISABLE_TAG_COMPARE_CHECKS) && currentTag.page_type == PT_ENCLAVE && currentTag.page_level != level`, termina il walk e ritorna `AddressFault(entry.vaddr, ExceptionCode::ENC_INTEGRITY_FAULT)` (`28`).
       - **Controllo 4 (Downgrade Huge Page `HPCE`)**: se `currentTag.hpce && level == 1` (megapage 2MiB contenente pagine enclave/protette):
         - Come in `cva6/src/ptw.sv` (cfr. `patches/cva6-spearv.patch` righe 2516–2523 e 2543–2549), degrada la traduzione a pagina 4KiB: imposta `level = 0`, aggiorna `savedPte.ppn = savedPte.ppn | bits(entry.vaddr, 20, 12)` e avvia la lettura del tag 4KiB specifico (`ReadHpce4kTagLow` / `ReadHpce4kTagHigh`), applicando poi i controlli 1–3 al tag 4KiB.
       - **Completamento (`TAG_FINAL`)**:
         - Popola `entry.pte = savedPte`, `entry.paddr = savedPte.ppn`, `entry.logBytes = PageShift + (level * SV39_LEVEL_BITS)` (che sarà `12` se degradato da `hpce`), `entry.tagValid = true`, `entry.e = (currentTag.page_type == PT_ENCLAVE)`, `entry.imm = currentTag.immutable`, `entry.hpce = currentTag.hpce`, `entry.pageType = currentTag.page_type`, `entry.eid = currentTag.id`, `entry.mpk = currentTag.mpk`, `entry.tagPtePerms = currentTag.pte_perms`.
         - Esegue `TLB::checkSpearVPermissions()` prima dell'inserimento/ritorno e inserisce `entry` nella TLB (`walker->tlb->insert(vpn, entry)`).
3. **Gestione `TAG_CTRL_NC_PTW_IN_E`**:
   - In `WalkerState::createReqPacket(Addr paddr, MemCmd cmd, size_t bytes)`, leggere `CSR_U_TAG_CONTROL`: se il bit `TAG_CTRL_NC_PTW_IN_E` è impostato, aggiungere `Request::UNCACHEABLE` ai `Request::Flags` del pacchetto (evitando che le letture di PTE e tag durante l'esecuzione dell'enclave lascino tracce nella D-Cache).

#### Fase 4: Decodifica delle Istruzioni Custom (`src/arch/riscv/isa/decoder.isa`)
1. **`mtrap` (`.word 0x7b`, `custom-3`, `funct3 = 0`)**:
   - Nel quadrante `0x3` (`opcode[1:0] == 2'b11`), `0x1e` (`opcode[6:2] == 5'b11110`, cioè `custom-3`), `funct3 == 0x0`: restituire `IllegalInstFault("mtrap SM call", machInst)` in modo che il trap handler M-mode in `riscv-pk/machine/emulation.c` (`illegal_insn_trap` → `tag_trap` → `trap_m_call`) riceva l'eccezione `CAUSE_ILLEGAL_INSTRUCTION` (`2`) con `mtval` valorizzato e dispacci la chiamata SM API (`ECREATE`, `EENTER`, `EEXIT`, ecc.).
2. **`fence.t` (`.word 0x...0b`, `custom-0` = `0x0b`)**:
   - Nel quadrante `0x3`, `0x02` (`opcode[6:2] == 5'b00010`, cioè `custom-0`):
     - Verificare che `xc->readMiscReg(MISCREG_PRV) != PRV_U` e che non sia attivo `STATUS.tvm` in `PRV_S` (altrimenti `IllegalInstFault`).
     - Estrarre `uint32_t imm20 = bits(machInst, 31, 12)`.
     - Se `imm20 & (1 << 6)` (`FLUSH_TLB`) o `imm20 != 0`, invocare `xc->tcBase()->getMMUPtr()->flushAll()`.
     - Contrassegnare l'istruzione con i flag `[IsSerializing, IsSerializeAfter, IsNonSpeculative]` per garantire il flush della pipeline in `MinorCPU` / `O3CPU`.
3. **`rdtlb` (`0x307b`, `funct3 = 3`) e `ritlb` (`0x407b`, `funct3 = 4`) su `custom-3`**:
   - Decodificare in `0x1e` (`custom-3`) per `funct3 == 0x3` e `funct3 == 0x4` restituendo in `Rd` i bit del tag TLB all'indice `Rs1` (o `0xDEAD` come in Spike).

#### Fase 5: Piano di Verifica e Validazione
1. **Boot e Inizializzazione Tag Store (`MTAG0` → `MTAG32`/`MTAG64`/`MTAG128`)**:
   - Avviare gem5 in Full System con il `bbl` compilato da `riscv-pk` (che invoca `tag_init(mem_base, mem_size, MTAG0, 0)` in `memory_init()` e filtra il Device Tree tramite `filter_tagged_memory()`).
   - Eseguire un test user-space su `pk` che invoca `api_tagreset(MTAG64)` e verificare tramite trace `--debug-flags=TLB,PageTableWalker` che:
     - `CSR_U_TAG_STORAGE_BASE`, `CSR_U_TAG_MODE` e `CSR_U_TAG_CONTROL` vengano configurati;
     - le pagine del Tag Store vengano auto-protette (`id = EID_INVALID, immutable = 1`);
     - i successivi Page Table Walk in S/U-mode emettano le letture verso il Tag Store e popolino le `TlbEntry`.
2. **Ciclo di Vita Enclave e Nesting**:
   - Testare la sequenza completa `api_ecreate` → `api_eadd` → `api_ppage` → `api_eenter` → `api_eexit` → `api_eremove` → `api_edestroy` (incluso il nesting padre-figlio e la gestione degli interrupt timer durante l'esecuzione dell'enclave tramite `trap_vector_enclave`), sia con `AtomicSimpleCPU` sia con `TimingSimpleCPU` / `O3CPU`.
3. **Suite di Test di Sicurezza (Verifica Eccezioni `25..31`)**:
   - **Fault `26` (`ENCLAVE_ACCESS_FAULT`) — Emulazione PT immutabile**: proteggere una page table con `api_ppage(va, 1)` e verificare che una scrittura del kernel (`sd`/`c.sd`/`amoswap.d`) su una PTE non-enclave nella stessa page table sollevi il fault `26`, venga emulata da `enclave_access_trap` in `riscv-pk` e riprenda l'esecuzione.
   - **Fault `26` (`ENCLAVE_ACCESS_FAULT`) — Isolamento**: verificare che un accesso dell'host a `PT_ENCLAVE`, un accesso U/S a `PT_MONITOR` (inclusa la `statuspage` dell'enclave stessa) o un accesso dell'enclave a `PT_NORMAL` con `EAM_ENCLAVE` vengano bloccati.
   - **Fault `28` (`ENC_INTEGRITY_FAULT`)**: verificare che il remap malevolo di una pagina fisica `PT_ENCLAVE` a un VPN differente o con permessi PTE alterati venga intercettato dal PTW durante `ReadTargetTagLow`.
   - **Fault `29` (`SATP_MODE_CHANGE`)**: verificare che `csrw satp, zero` in S-mode con tagging attivo sollevi l'eccezione `29`.
   - **Fault `25` (`MPKEY_MISMATCH_FAULT`)**: in modalità `MTAG128`, verificare il blocco di lettura/scrittura al cambio di `CSR_U_MPKEY_CONFIG`.
   - **Downgrade Huge Page (`HPCE`)**: con `SM_TAG_MEGAPAGES = 1`, mappare una pagina enclave dentro una megapage da 2MiB e verificare che il PTW degradi la entry TLB a 4KiB (`logBytes = 12`).
