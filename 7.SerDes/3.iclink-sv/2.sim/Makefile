# SPDX-FileCopyrightText: 2026 Chili.CHIPS
#
# SPDX-License-Identifier: BSD-3-Clause
#
# Unit tests of the IC Link RTL, Verilator; the BFM's own tests (Icarus too) via ../../1.sim-model.
#   make            all unit tests
#   make sim_8b10b  the reference 8b/10b (simulation only) against LiteX's vectors
#                   (vectors/*.hex, committed)
#   make sim_prbs   PRBS7/15/31 against LiteX's vectors, and pause
#   make sim_phy_oneway  the SerDes PHY on the SerDes BFM, TX looped to RX
#                        (WAVES=1: also writes phy_oneway.<options>.fst, for WaveCrux)
#   make sim_phy_twoway  two PHYs on two SerDes BFMs, serial lines crossed
#                        (WAVES=1: also writes phy_twoway.<options>.fst)
#   make sim_phy_rates   both PHY tests at 1250, 2500 and 5000 Mb/s
#   make sim_serwb_tx  SerWB packetizer against LiteICLink's Packetizer (vectors/serwb_*)
#   make sim_serwb_rx  SerWB depacketizer against LiteICLink's CRCDepacketizer (vectors/serwb_rx_*)
#   make sim_serwb_pump  FIFO pump, two sides on the digital bus (SerDes out of the loop), flow control
#   make sim_serwb_w4  the three SerWB tests with W = 4 words a clock (5 Gb/s), through gearboxes
#                      (SERWB_W=4 does the same for any one of them)
#   make sim_top    the board top: self-test, SerDes link and FIFO pump, clean and with errors
#   make sim_link_ppm  two sides PPM apart (default 200): clock correction keeps every word;
#                      without its bursts (NO_CLKCOR) the same test must fail
#   make sim_modes, sim_icarus  the SerDes BFM's own tests, in ../../1.sim-model (its Makefile)
#   make vectors    regenerate vectors/*.hex from LiteX (needs LiteX in python3)
#   make clean
# Settings (make ... NAME=VALUE):
#   CDR_MODEL=NONE|FAST|ACCURATE  the SerDes BFM mode, PHY, top and link_ppm tests (default NONE)
#   SEED=N        the simulator seed, PHY and top tests: it sets the BFM's random RX word
#                 phase. None: Verilator's fixed default seed; 0: a new seed every run
#   RANDOM_PHASE=0  the RX word phase fixed at 0, PHY, top and link_ppm tests (default: random)
#   LINE_RATE=1250|2500|5000  Mb/s, PHY tests (default 2500)
#   WAVES=1       also write an FST waveform, PHY tests, named by the options (WAVE_TAG)
#   SERWB_W=2|4   SerWB words a clock, SerWB tests (default 2)
#   SIM=verilator|janushdl  the simulator (default verilator). janushdl: JanusHDL (jhdl 0.17
#                 or later), which compiles at each run; WAVES=1 is its --fst; no SEED;
#                 sim_modes and sim_icarus stay on Verilator and Icarus

# a simulator that fails in a pipe (| tee) fails the target
SHELL       := /bin/bash
.SHELLFLAGS := -o pipefail -ec

SIM  ?= verilator
JHDL ?= jhdl
ifeq ($(filter verilator janushdl,$(SIM)),)
  $(error SIM must be verilator or janushdl (got "$(SIM)"))
endif
VLT := verilator --binary --assert --timing --timescale 1ns/1ps -Wall -Wno-DECLFILENAME -Wno-UNUSEDSIGNAL

# $(call sim,TOP,OBJ_DIR,OPTIONS,SOURCES,RUN_ARGS,LOG): build TOP, run it into LOG.
# JanusHDL: one jhdl run; of OPTIONS it takes -G (as -g) and +define+, no .vlt
ifeq ($(SIM),verilator)
define sim
$(VLT) --top-module $1 -Mdir $2 $3 $4
./$2/V$1 $5 | tee $6
endef
else
define sim
$(JHDL) --top $1 $(patsubst -G%,-g%,$(filter -G% +define+%,$3)) $(filter-out %.vlt,$4) $5 | tee $6
endef
endif

HW  := ../1.hw
BFM := ../../1.sim-model/ccfpga_serdes_pkg.sv ../../1.sim-model/ccfpga_serdes.BFM.sv
CDR_MODEL    ?=
RANDOM_PHASE ?=
BFM_DEF := $(if $(CDR_MODEL),+define+CDR_MODEL=$(CDR_MODEL)) \
           $(if $(RANDOM_PHASE),+define+BHV_RX_WORD_PHASE_RANDOM=$(RANDOM_PHASE))
OUT := obj_8b10b

WAVES ?= 0
SERWB_W ?= 2
TRACE := $(if $(filter 1,$(WAVES)),--trace-fst --trace-structs waves.vlt,)
SEED  ?=
# the FST name carries the run options: phy_oneway.<CDR_MODEL>.<LINE_RATE>.s<SEED>[.rp<RANDOM_PHASE>].fst
# (s-: no SEED, Verilator's fixed default; s0: a new seed each run)
WAVE_TAG  = $(or $(CDR_MODEL),NONE).$(LINE_RATE).s$(or $(SEED),-)$(if $(RANDOM_PHASE),.rp$(RANDOM_PHASE))
ifeq ($(SIM),verilator)
PLUS   = $(if $(filter 1,$(WAVES)),+waves=$(WAVE_TAG),) $(if $(SEED),+verilator+seed+$(SEED))
else
# $dumpfile would write a VCD: JanusHDL's --fst records every signal
PLUS   = $(if $(filter 1,$(WAVES)),$(if $(filter sim_phy_%,$@),--fst $(@:sim_%=%).$(WAVE_TAG).fst))
endif

.PHONY: all sim_8b10b sim_prbs sim_phy_oneway sim_phy_twoway sim_phy_rates sim_serwb_tx sim_serwb_rx sim_serwb_pump sim_serwb_w4 sim_top sim_link_ppm sim_modes sim_icarus vectors clean

all: sim_icarus sim_8b10b sim_prbs sim_phy_oneway sim_phy_twoway sim_serwb_tx sim_serwb_rx sim_serwb_pump sim_serwb_w4 sim_top sim_link_ppm sim_modes

sim_8b10b:
	$(call sim,tb_8b10b,$(OUT),, \
	  $(HW)/iclink_pkg.sv iclink_8b10b_pkg.sv iclink_8b10b_enc.sv iclink_8b10b_dec.sv tb_8b10b.sv,,sim_8b10b.log)
	@grep -q "TB_8B10B: PASS" sim_8b10b.log

sim_prbs:
	$(call sim,tb_prbs,obj_prbs,, \
	  $(HW)/iclink_pkg.sv $(HW)/iclink_prbs_gen.sv $(HW)/iclink_prbs_chk.sv $(HW)/iclink_prbs_tx.sv $(HW)/iclink_prbs_rx.sv tb_prbs.sv,,sim_prbs.log)
	@grep -q "TB_PRBS: PASS" sim_prbs.log

# line rate in Mb/s (1250, 2500, 5000)
LINE_RATE ?= 2500
RATE_G    := -GLINE_RATE=$(LINE_RATE)

# both PHY testbenches at every rate
sim_phy_rates:
	@for rate in 1250 2500 5000; do \
	  $(MAKE) --no-print-directory sim_phy_oneway LINE_RATE=$$rate > /dev/null; \
	  $(MAKE) --no-print-directory sim_phy_twoway LINE_RATE=$$rate > /dev/null; \
	  echo "$$rate Mb/s: $$(grep -h PASS sim_phy_oneway.log sim_phy_twoway.log | tr '\n' ' ')"; \
	done | tee sim_phy_rates.log

sim_phy_oneway:
	$(call sim,tb_phy_oneway,obj_phy,$(TRACE) $(RATE_G) $(BFM_DEF),sim-model.vlt rtl.vlt \
	  $(HW)/iclink_pkg.sv $(HW)/iclink_phy_pkg.sv \
	  $(HW)/iclink_prbs_gen.sv $(HW)/iclink_prbs_chk.sv $(HW)/iclink_prbs_tx.sv $(HW)/iclink_prbs_rx.sv \
	  $(HW)/iclink_serdes_gm.sv $(BFM) \
	  tb_phy_oneway.sv,$(PLUS),sim_phy_oneway.log)
	@grep -q "TB_PHY_ONEWAY: PASS" sim_phy_oneway.log

sim_phy_twoway:
	$(call sim,tb_phy_twoway,obj_phy2,$(TRACE) $(RATE_G) $(BFM_DEF),sim-model.vlt rtl.vlt \
	  $(HW)/iclink_pkg.sv $(HW)/iclink_phy_pkg.sv \
	  $(HW)/iclink_prbs_gen.sv $(HW)/iclink_prbs_chk.sv $(HW)/iclink_prbs_tx.sv $(HW)/iclink_prbs_rx.sv \
	  $(HW)/iclink_serdes_gm.sv $(BFM) \
	  tb_phy_twoway.sv,$(PLUS),sim_phy_twoway.log)
	@grep -q "TB_PHY_TWOWAY: PASS" sim_phy_twoway.log

sim_serwb_tx:
	$(call sim,tb_serwb_tx,obj_serwb_tx_w$(SERWB_W),-GW=$(SERWB_W), \
	  rtl.vlt $(HW)/iclink_serwb_pkg.sv $(HW)/iclink_serwb_tx.sv tb_serwb_tx.sv,,sim_serwb_tx.log)
	@grep -q "TB_SERWB_TX: PASS" sim_serwb_tx.log

sim_serwb_rx:
	$(call sim,tb_serwb_rx,obj_serwb_rx_w$(SERWB_W),-GW=$(SERWB_W), \
	  rtl.vlt $(HW)/iclink_serwb_pkg.sv $(HW)/iclink_serwb_rx.sv tb_serwb_rx.sv,,sim_serwb_rx.log)
	@grep -q "TB_SERWB_RX: PASS" sim_serwb_rx.log

PUMP_SRC := rtl.vlt $(HW)/iclink_pkg.sv $(HW)/iclink_phy_pkg.sv $(HW)/iclink_serwb_pkg.sv \
  $(HW)/iclink_serwb_fifo.sv $(HW)/iclink_serwb_tx.sv \
  $(HW)/iclink_serwb_rx.sv $(HW)/iclink_serwb_line.sv $(HW)/iclink_serwb_pump.sv

sim_serwb_pump:
	$(call sim,tb_serwb_pump,obj_serwb_pump_w$(SERWB_W),-Wno-UNUSEDPARAM -GW=$(SERWB_W), \
	  $(PUMP_SRC) tb_serwb_pump.sv,,sim_serwb_pump.log)
	@grep -q "TB_SERWB_PUMP: PASS" sim_serwb_pump.log

sim_serwb_w4:
	$(MAKE) SERWB_W=4 sim_serwb_tx sim_serwb_rx sim_serwb_pump

# two boards on two oscillators, PPM apart: the PHYs, the pumps and the
# CC_SERDES clock correction must keep every word. Then the same without
# the clock-correction bursts (NO_CLKCOR), which must fail
PPM ?= 200
LINK_SRC := sim-model.vlt rtl.vlt $(HW)/iclink_pkg.sv $(HW)/iclink_phy_pkg.sv $(HW)/iclink_serwb_pkg.sv \
  $(HW)/iclink_prbs_gen.sv $(HW)/iclink_prbs_chk.sv $(HW)/iclink_prbs_tx.sv $(HW)/iclink_prbs_rx.sv \
  $(HW)/iclink_serdes_gm.sv $(HW)/iclink_serwb_fifo.sv $(HW)/iclink_serwb_tx.sv $(HW)/iclink_serwb_rx.sv \
  $(HW)/iclink_serwb_line.sv $(HW)/iclink_serwb_pump.sv $(BFM) tb_link_ppm.sv
LINK_OPT := -Wno-UNUSEDPARAM -GPPM=$(PPM) $(RATE_G) $(BFM_DEF)

sim_link_ppm:
	$(call sim,tb_link_ppm,obj_link,$(LINK_OPT),$(LINK_SRC),,sim_link_ppm.log)
	@grep -q "TB_LINK_PPM: PASS" sim_link_ppm.log
	$(call sim,tb_link_ppm,obj_link_neg,$(LINK_OPT) +define+NO_CLKCOR=1,$(LINK_SRC),,sim_link_ppm_neg.log)
	@grep -q "TB_LINK_PPM: FAIL" sim_link_ppm_neg.log && echo "NO_CLKCOR: FAIL, as it must"

sim_top:
	$(call sim,tb_top,obj_top,$(RATE_G) $(BFM_DEF),sim-model.vlt rtl.vlt \
	  $$(sed -e "s|\$${HW_SRC}|$(HW)|" -e "/^#/d" -e "/^$$/d" $(HW)/top.filelist) \
	  $(BFM) tb_top.sv,$(PLUS),sim_top.log)
	@grep -q "TB_TOP: PASS" sim_top.log

# The SerDes BFM's own tests live with it, in ../../1.sim-model; its logs
# stay there
sim_modes sim_icarus:
	$(MAKE) -C ../../1.sim-model $@

vectors:
	mkdir -p vectors && python3 gen_8b10b_vectors.py vectors && python3 gen_prbs_vectors.py vectors && python3 gen_serwb_vectors.py vectors && python3 gen_serwb_rx_vectors.py vectors

# build directories (obj_*, recursively) and outputs
# the waves phy_*.fst stay: they are committed, for WaveCrux
clean:
	find . -maxdepth 1 -name 'obj_*' -exec find {} -delete \;
	rm -f sim_*.log pump_levels_w*.csv modes_*.txt
