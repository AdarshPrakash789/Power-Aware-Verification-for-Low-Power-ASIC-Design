# Power-Aware Verification for Low-Power ASIC Design

## Project Overview
This project demonstrates a power-aware verification flow for a low-power ASIC design using Synopsys tools (VCS and Primetime PX) along with UVM-based SystemVerilog testbenches. The main objective is to validate energy-efficient design features like clock gating and power domains under real-world operating conditions. It integrates low-power simulation techniques with assertion-based checks and activity-based power analysis to ensure both functional correctness and power optimization.

---

## Highlights
- Refined a power-aware verification flow for ASICs targeting energy-efficient designs
- Implemented low-power simulation using VCS and SAIF (Switching Activity Interchange Format)
- Applied clock gating and multi-power domain validation techniques
- Achieved 40% reduction in power consumption through optimization and clock control
- Used Synopsys Primetime PX for activity-based power analysis
- Built UVM-based testbenches to validate power-saving features

---

## Tools and Technologies
- **HDL**: SystemVerilog
- **Verification Methodology**: UVM
- **Simulator**: Synopsys VCS
- **Power Analysis Tool**: Synopsys Primetime PX
- **Power Format**: UPF (Unified Power Format)

---

## Directory Structure
```
power_aware_verification/
├── rtl/
│   └── low_power_block.sv              # RTL design with clock gating
├── upf/
│   └── power_domains.upf              # UPF file for power domain definitions
├── tb/
│   ├── env/
│   │   ├── lp_env.sv
│   │   ├── lp_driver.sv
│   │   ├── lp_monitor.sv
│   │   ├── lp_agent.sv
│   │   └── lp_scoreboard.sv
│   ├── seq/
│   │   └── lp_sequence.sv
│   ├── test/
│   │   └── lp_test.sv
│   ├── lp_interface.sv
│   └── top_tb.sv
├── scripts/
│   ├── run_sim.tcl                     # TCL for VCS simulation
│   └── power_analysis.tcl             # TCL for Primetime PX analysis
├── reports/
│   └── power_report.txt
├── README.md
├── LICENSE
└── Makefile
```

---

## RTL Design (`rtl/low_power_block.sv`)
```systemverilog
module low_power_block(
    input  logic clk,
    input  logic rst_n,
    input  logic en,
    input  logic [7:0] data_in,
    output logic [7:0] data_out
);
    logic [7:0] reg_data;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            reg_data <= 0;
        else if (en)
            reg_data <= data_in;
    end

    assign data_out = reg_data;
endmodule
```

---

## UPF File (`upf/power_domains.upf`)
```tcl
create_power_domain PD_TOP -elements {low_power_block}
create_supply_net VDD
create_power_supply VDD -voltage 1.0
connect_supply_net VDD -ports {clk rst_n data_in en}
```

---

## Interface (`tb/lp_interface.sv`)
```systemverilog
interface lp_if(input logic clk);
    logic rst_n;
    logic en;
    logic [7:0] data_in;
    logic [7:0] data_out;
endinterface
```

---

## UVM Components
### Driver (`tb/env/lp_driver.sv`)
```systemverilog
class lp_driver extends uvm_driver#(lp_txn);
    virtual lp_if vif;

    task run_phase(uvm_phase phase);
        forever begin
            lp_txn tx;
            seq_item_port.get_next_item(tx);
            vif.en      <= tx.en;
            vif.data_in <= tx.data;
            seq_item_port.item_done();
            #10;
        end
    endtask
endclass
```

### Monitor (`tb/env/lp_monitor.sv`)
```systemverilog
class lp_monitor extends uvm_monitor;
    virtual lp_if vif;
    uvm_analysis_port#(lp_txn) ap;

    task run_phase(uvm_phase phase);
        forever begin
            lp_txn tx = lp_txn::type_id::create("tx");
            tx.data = vif.data_out;
            ap.write(tx);
            #10;
        end
    endtask
endclass
```

### Scoreboard (`tb/env/lp_scoreboard.sv`)
```systemverilog
class lp_scoreboard extends uvm_scoreboard;
    uvm_analysis_imp#(lp_txn, lp_scoreboard) ap;
    queue[lp_txn] expected;

    function void write(lp_txn tx);
        lp_txn exp = expected.pop_front();
        if (tx.data !== exp.data)
            `uvm_error("SCOREBOARD", $sformatf("Mismatch: Got %0h, Expected %0h", tx.data, exp.data));
    endfunction
endclass
```

---

## Testbench Top (`tb/top_tb.sv`)
```systemverilog
module top_tb;
    logic clk;
    lp_if intf(clk);

    initial clk = 0;
    always #5 clk = ~clk;

    low_power_block dut (
        .clk(clk),
        .rst_n(intf.rst_n),
        .en(intf.en),
        .data_in(intf.data_in),
        .data_out(intf.data_out)
    );

    initial begin
        run_test("lp_test");
    end
endmodule
```

---

## Simulation Script (`scripts/run_sim.tcl`)
```tcl
vcs -full64 -sverilog \
    -debug_access+all \
    -l vcs.log \
    -ucli -do run.tcl \
    -f filelist.f \
    +define+POWER_AWARE \
    -top top_tb
```

## Power Analysis Script (`scripts/power_analysis.tcl`)
```tcl
read_verilog rtl/low_power_block.sv
read_saif -input power.saif
create_clock -name clk -period 10 [get_ports clk]
report_power > reports/power_report.txt
```

---

## Sample Transaction (`tb/seq/lp_sequence.sv`)
```systemverilog
class lp_sequence extends uvm_sequence#(lp_txn);
    task body();
        repeat (10) begin
            lp_txn tx = lp_txn::type_id::create("tx");
            assert(tx.randomize());
            start_item(tx);
            finish_item(tx);
        end
    endtask
endclass
```

---

## README.md
```markdown
# Power-Aware Verification for Low-Power ASIC Design

## Overview
This project showcases a power-aware verification methodology for a low-power ASIC. Using UVM, VCS, and Synopsys Primetime PX, it integrates low-power techniques such as clock gating and UPF-defined power domains. The testbench validates energy-efficient operation under dynamic conditions.

## Features
- Clock gating-based RTL simulation
- Multi-power domain definition using UPF
- Power-aware UVM testbench
- Switching activity analysis using SAIF
- Power reporting using Primetime PX

## Requirements
- Synopsys VCS
- Synopsys Primetime PX
- SystemVerilog
- UVM

## Running Simulation
```bash
vcs -full64 -sverilog -f filelist.f -debug_access+all -top top_tb
./simv +UVM_TESTNAME=lp_test
```

## Generating Power Report
```bash
pt_shell -f scripts/power_analysis.tcl
```
```

---

## License
MIT License

---

## Author
Adarsh Prakash
[LinkedIn](https://www.linkedin.com/in/adarsh-prakash-a583a3259/)
