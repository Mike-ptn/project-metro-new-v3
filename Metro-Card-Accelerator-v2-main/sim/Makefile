## ------------------------------------------------------------------
# Cadence Xcelium (xrun / imc / simvision) Simulation Makefile
#
# Basic run:
#   make all TESTNAME=metro_fallback_miss_test SEED=1
#
# Enable RTL code coverage + SystemVerilog functional coverage:
#   make clean
#   make all COV=ON TESTNAME=metro_metrics_test SEED=1
#
# Run full regression with coverage enabled (regress.cfg controls COV):
#   ./regress.pl
#   make cov_merge
#   make cov_summary
## ------------------------------------------------------------------

TB_NAME   ?= testbench
TESTNAME  ?= metro_fallback_miss_test
VERBOSITY ?= UVM_LOW
RUNARG    ?=
SEED      ?= 1
COV       ?= OFF
UVM_HOME  ?= CDNS-1.2

ifeq ($(strip $(SEED)),random)
  SEED := $(shell date +%s)
endif

# Instrument only the RTL DUT (metro_card_top) for code coverage and collect
# SystemVerilog functional covergroups in the UVM scoreboard via cov.ccf.
ifeq ($(COV),ON)
  COV_BUILD_OPT = -coverage all -covdut metro_card_top -covfile cov.ccf
  COV_RUN_OPT   = -covworkdir log/cov_work -covtest $(TESTNAME)_$(SEED) -covoverwrite
else
  COV_BUILD_OPT =
  COV_RUN_OPT   =
endif

XRUN_COMMON = xrun -64bit -sv -uvm -uvmhome $(UVM_HOME) \
       -timescale 1ns/1ps \
       -access +rwc \
       -f rtl.f \
       -f tb.f \
       -top $(TB_NAME) \
       +define+UVM_OBJECT_MUST_HAVE_CONSTRUCTOR

.PHONY: all build run wave cov_gui cov_merge cov_report cov_summary clean help regress metrics
all: build run

build:
	@mkdir -p log
	$(XRUN_COMMON) $(COV_BUILD_OPT) -elaborate -l xrun_build.log

run:
	@mkdir -p log
	@rm -rf "log/$(TESTNAME)_$(SEED).shm"
	@$(XRUN_COMMON) $(COV_BUILD_OPT) $(COV_RUN_OPT) \
	  -svseed $(SEED) \
	  +UVM_TESTNAME=$(TESTNAME) \
	  +UVM_VERBOSITY=$(VERBOSITY) $(RUNARG) \
	  -input "@database -open waves -into log/$(TESTNAME)_$(SEED).shm -default; probe -create -shm $(TB_NAME) -all -variables -depth all; run; exit" \
	  -l "log/$(TESTNAME)_$(SEED).log"; \
	  status=$$?; \
	  if [ "$(COV)" = "ON" ] && [ -d "log/cov_work/scope/$(TESTNAME)_$(SEED)" ]; then \
	    if command -v imc >/dev/null 2>&1; then \
	      imc -nocopyright -batch -initcmd \
	        "load -run log/cov_work/scope/$(TESTNAME)_$(SEED); report -detail -metrics all -out log/$(TESTNAME)_$(SEED)_coverage.txt; exit" \
	        >/dev/null 2>&1 || \
	        echo "WARNING: Coverage DB saved, but imc report failed; inspect with make cov_gui."; \
	    fi; \
	  fi; \
	  ln -sfn "$(TESTNAME)_$(SEED).log" log/run.log; \
	  exit $$status

wave:
	@shm_dir="log/$(TESTNAME)_$(SEED).shm"; \
	if [ ! -d "$$shm_dir" ]; then \
	  shm_dir=$$(ls -td log/$(TESTNAME)_*.shm 2>/dev/null | head -n 1); \
	fi; \
	test -n "$$shm_dir" && [ -d "$$shm_dir" ] || { \
	  echo "ERROR: No SHM waveform database found for $(TESTNAME) under log/. Run simulation first."; exit 1; }; \
	echo "Opening SimVision with $$shm_dir ..."; \
	simvision "$$shm_dir" -input simvision_dut.tcl &

# Open either an individual testcase coverage run or the merged IP_MERGE database in Cadence IMC GUI.
cov_gui:
ifeq ($(TESTNAME),IP_MERGE)
	@test -d "log/cov_work/scope/IP_MERGE" || { echo "No merged coverage DB: log/cov_work/scope/IP_MERGE. Run make cov_merge first."; exit 1; }
	imc -gui -load log/cov_work/scope/IP_MERGE &
else
	@cov_dir="log/cov_work/scope/$(TESTNAME)_$(SEED)"; \
	if [ ! -d "$$cov_dir" ]; then \
	  cov_dir=$$(ls -td log/cov_work/scope/$(TESTNAME)_* 2>/dev/null | head -n 1); \
	fi; \
	test -n "$$cov_dir" && [ -d "$$cov_dir" ] || { \
	  echo "No coverage DB found for $(TESTNAME). Run with COV=ON first."; exit 1; }; \
	imc -gui -load "$$cov_dir" &
endif

cov_report:
	@cov_dir="log/cov_work/scope/$(TESTNAME)_$(SEED)"; \
	if [ ! -d "$$cov_dir" ]; then \
	  cov_dir=$$(ls -td log/cov_work/scope/$(TESTNAME)_* 2>/dev/null | head -n 1); \
	fi; \
	test -n "$$cov_dir" && [ -d "$$cov_dir" ] || { \
	  echo "No coverage DB found for $(TESTNAME). Run with COV=ON first."; exit 1; }; \
	imc -nocopyright -batch -initcmd \
	  "load -run $$cov_dir; report -detail -metrics all -out log/$(TESTNAME)_$(SEED)_coverage.txt; exit"
	@echo "Detailed coverage report: log/$(TESTNAME)_$(SEED)_coverage.txt"

# Merge all per-test coverage databases into log/cov_work/scope/IP_MERGE, excluding any prior merge.
cov_merge:
	@set -e; \
	  mkdir -p log; \
	  runs=$$(find log/cov_work/scope -mindepth 1 -maxdepth 1 -type d ! -name 'IP_MERGE*' -print 2>/dev/null | sort); \
	  test -n "$$runs" || { echo "No testcase coverage databases under log/cov_work/scope/. Run tests with COV=ON first."; exit 1; }; \
	  rm -rf log/cov_work/scope/IP_MERGE log/IP_MERGE_coverage_summary.txt log/IP_MERGE_coverage_details.txt; \
	  imc -nocopyright -batch -initcmd \
	    "merge $$runs -out log/cov_work/scope/IP_MERGE -metrics all -initial_model union_all -overwrite; \
	     load -run log/cov_work/scope/IP_MERGE; \
	     report -summary -metrics all -out log/IP_MERGE_coverage_summary.txt; \
	     report -detail -metrics all -out log/IP_MERGE_coverage_details.txt; \
	     exit"; \
	  echo "Merged Coverage DB : log/cov_work/scope/IP_MERGE"; \
	  echo "Summary            : log/IP_MERGE_coverage_summary.txt"; \
	  echo "Details            : log/IP_MERGE_coverage_details.txt"

# Reprint summary from the already-merged coverage database (all merged testcases).
cov_summary:
	@test -d "log/cov_work/scope/IP_MERGE" || { echo "No merged coverage DB. Run make cov_merge first."; exit 1; }
	@set -e; \
	  imc -nocopyright -batch -initcmd \
	    "load -run log/cov_work/scope/IP_MERGE; \
	     report -summary -metrics all -out log/IP_MERGE_coverage_summary.txt; \
	     exit"; \
	  cat log/IP_MERGE_coverage_summary.txt; \
	  echo "Summary saved to log/IP_MERGE_coverage_summary.txt"

clean:
	rm -rf xcelium.d INCA_libs cov_work .simvision work log *.ini *.log *.key *.history *.trace *.wlf *.ucdb *.shm transcript vsim.dbg tmp_report

help:
	@echo "make build                        : compile & elaborate RTL + UVM testbench with xrun"
	@echo "make run                          : run current TESTNAME with xrun"
	@echo "make all                          : build and run"
	@echo "make all COV=ON TESTNAME=...      : collect RTL code + functional coverage"
	@echo "make wave TESTNAME=...            : open SHM waveform in Cadence SimVision"
	@echo "make cov_report TESTNAME=...      : generate text coverage report via Cadence IMC"
	@echo "make cov_gui TESTNAME=...         : open testcase coverage in Cadence IMC GUI"
	@echo "make cov_merge                    : merge testcase coverage DBs into IP_MERGE"
	@echo "make cov_gui TESTNAME=IP_MERGE    : open merged coverage in Cadence IMC GUI"
	@echo "make cov_summary                  : print summary for all merged tests"
	@echo "make regress                      : run all 9 testcases via ./regress.pl"
	@echo "make metrics                      : compute AI/classification metrics from CSV"
	@echo "make clean                        : remove generated simulation data"

regress:
	./regress.pl

metrics:
	python3 ../metrics/ai_metrics.py log/metro_metrics_results.csv
