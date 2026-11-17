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

## 3. Indicazioni per il Porting su gem5 (`spearv-gem5`)

Per portare SPEAR-V in gem5 (basato su memory tagging con granularità a pagine da 4 KiB):

1. **ISA & CSR (`src/arch/riscv/`)**:
   - Aggiungere i CSR `0x066`–`0x076` in `regs/misc.hh` e `isa.cc` (incluso il controllo in scrittura su `satp` per generare `CAUSE_SATP_MODE_CHANGE = 29` se S-mode disabilita la paginazione con tag attivi).
   - Decodificare `0x7b` (`mtrap` → `IllegalInstFault` per intrappolare nel Security Monitor di `riscv-pk` / OpenSBI) e `0x0b` (`fence_t` → flush microarchitettura/TLB).
   - Aggiungere i fault code `25..31` in `faults.hh` / `fault_codes.hh` (senza delegarli in `medeleg` così da trappare sempre in M-mode).
2. **MMU, TLB e Walker (`src/arch/riscv/tlb.cc`, `pagetable_walker.cc`, `pagetable.hh`)**:
   - Estendere la `TlbEntry` con i bit di tag cachati (`e`/`page_type`, `imm`, `hpce`, `eid` in `asid`).
   - Estendere la FSM del `Walker` affinché, quando `tag_mode != MTAG0`, generi letture fisiche aggiuntive verso il tag store (`tag_storage_base + (ppn - tag_dram_base) * tag_bytes`) per ciascun livello del walk (e gestisca `hpce` su superpage 2MiB/1GiB facendo il downgrade a 4KiB). In `TimingSimpleCPU`/`O3CPU` queste letture passano attraverso la porta dati del walker catturando realisticamente la latenza del tag fetch!
   - Eseguire i controlli di isolamento (`tag_mode`, `meid`, `eam`, `immutable` su store, `PT_MONITOR` vietato fuori da M-mode) su TLB hit e walk completion.
3. **Software / Security Monitor (`riscv-pk`)**:
   - La patch `patches/riscv-pk-spearv.patch` fornisce il Security Monitor completo (`bbl` + `pk`) pronto per essere compilato e usato con un kernel/payload o test bare-metal/PK su gem5 (verificando eventuali adattamenti dell'indirizzo base DRAM o device tree di gem5 rispetto a `PM_START = 0x80000`).
