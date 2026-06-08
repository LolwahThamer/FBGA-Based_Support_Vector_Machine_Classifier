# 🧠 FPGA-Based Support Vector Machine (SVM) Classifier

**Hardware implementation of a binary classifier on FPGA**  
Classifies 2D points (x, y) using three modes: **Vertical**, **Horizontal**, and **Inclined** thresholding.  
Built with Verilog, simulated in ModelSim, and synthesized for the **DE2-115** board.

![Verilog](https://img.shields.io/badge/Verilog-1995-blueviolet)
![FPGA](https://img.shields.io/badge/FPGA-DE2--115-orange)
![Simulation](https://img.shields.io/badge/Simulation-ModelSim-green)
![License](https://img.shields.io/badge/License-MIT-yellow)

---

## 📑 Table of Contents
- [Project Overview](#project-overview)
- [System Architecture & Flowchart](#system-architecture--flowchart)
- [Modules Description and Verilog Code](#modules-description-and-verilog-code)
  - [1. Input Reader](#1-input-reader)
  - [2. Mode Selector](#2-mode-selector)
  - [3. SVM Core](#3-svm-core)
  - [4. Output Writer](#4-output-writer)
  - [5. Output Interface](#5-output-interface)
  - [6. System Tick](#6-system-tick)
  - [7. Counter](#7-counter)
  - [8. SEG7_LUT](#8-seg7_lut)
- [Top-Level Integration](#top-level-integration)
  - [svm_system](#svm_system)
  - [Project_ASIC (FPGA Top Wrapper)](#project_asic-fpga-top-wrapper)
- [Simulation & Testing](#simulation--testing)
  - [Testbench](#testbench)
  - [Simulation Waveforms](#simulation-waveforms)
- [FPGA Implementation](#fpga-implementation)
- [Results and Analysis](#results-and-analysis)
- [Engineering Constraints](#engineering-constraints)
- [Future Work](#future-work)
- [References](#references)

---

## Project Overview

This project implements a **binary classifier** on an FPGA that categorizes data points represented by (x, y) coordinates. The classification is performed using one of three modes:

- **Vertical Mode** – compares `x` against a fixed threshold (`x_threshold = 128`)
- **Horizontal Mode** – compares `y` against a fixed threshold (`y_threshold = 100`)
- **Inclined Mode** – computes `y_calc = m*x + c` (with `m = 1`, `c = 20`) and compares `y` against `y_calc`

The system reads x and y values from text files (`x_value.txt`, `y_value.txt`), processes them sequentially, and writes the classification results (0 or 1) to `result.txt`. Results are also displayed on LEDs and 7‑segment displays.

**Key objectives:**
- Modular Verilog design (input reader, mode selector, SVM core, output writer, display interface)
- Accurate classification for all three modes
- Simulation and hardware verification on DE2‑115 FPGA
- Real‑time visual feedback via LEDs and HEX displays

---

## System Architecture & Flowchart

The system follows a pipelined, modular architecture. The high‑level workflow is shown below:

```mermaid
flowchart TD
    A[Start] --> B[Read x, y from memory using index]
    B --> C{Mode selector}
    C -- Mode 00 --> D[Vertical: compare x >= x_th]
    C -- Mode 01 --> E[Horizontal: compare y >= y_th]
    C -- Mode 10 --> F[Inclined: compute y_calc = m*x + c, compare y >= y_calc]
    D --> G[Store result in memory]
    E --> G
    F --> G
    G --> H{All 256 points processed?}
    H -- No --> B
    H -- Yes --> I[Write results to result.txt and display on LEDs]
    I --> J[Stop]
Block diagram of the complete system:












Modules Description and Verilog Code
All modules are written in Verilog and tested individually before integration.

1. Input Reader
Reads x and y values from memory arrays initialized from x_value.txt and y_value.txt using $readmemh. Outputs data sequentially based on index idx.

verilog
module input_reader (
    input wire clk, rst, start,
    output reg [7:0] x_data, y_data,
    output reg done,
    output reg LED,
    input [7:0] idx
);
    reg [7:0] x_mem [0:255];
    reg [7:0] y_mem [0:255];
    reg [7:0] Q;

    initial begin
        $readmemh("x_value.txt", x_mem);
        $readmemh("y_value.txt", y_mem);
        $display("Debug: Memory Initialization");
        $display("x_mem[0]: %h, y_mem[0]: %h", x_mem[0], y_mem[0]);
        $display("x_mem[255]: %h, y_mem[255]: %h", x_mem[255], y_mem[255]);
    end

    always @(posedge clk, negedge rst) begin
        if (!rst) begin
            x_data <= 8'b0;
            y_data <= 8'b0;
            done <= 1'b0;
            LED <= 1;
        end else if (start && !done) begin
            x_data <= x_mem[idx];
            y_data <= y_mem[idx];
            $display("Index: %d, X: %h, Y: %h", idx, x_data, y_data);
            if (idx == 8'd255) begin
                done <= 1'b1;
                $display("Done signal asserted at index %d", idx);
            end else begin
                LED <= 0;
            end
        end
    end
endmodule
2. Mode Selector
Selects thresholds or calculates y_calc based on mode.

verilog
module mode_selector (
    input wire [1:0] mode,
    input wire [7:0] x,
    output reg [7:0] x_threshold,
    output reg [7:0] y_threshold,
    output reg [15:0] y_calc
);
    localparam [7:0] x_fixed_threshold = 8'd128;
    localparam [7:0] y_fixed_threshold = 8'd100;
    localparam [7:0] m = 8'd1;
    localparam [7:0] c = 8'd20;

    always @(*) begin
        case (mode)
            2'b00: begin // Vertical mode
                x_threshold = x_fixed_threshold;
                y_threshold = 8'b0;
                y_calc = 16'b0;
            end
            2'b01: begin // Horizontal mode
                x_threshold = 8'b0;
                y_threshold = y_fixed_threshold;
                y_calc = 16'b0;
            end
            2'b10: begin // Inclined mode
                x_threshold = 8'b0;
                y_threshold = 8'b0;
                y_calc = (m * x) + c;
                $display("Debug: y_calc = %h for x = %h, m = %h, c = %h", y_calc, x, m, c);
            end
            default: begin
                x_threshold = 8'b0;
                y_threshold = 8'b0;
                y_calc = 16'b0;
            end
        endcase
    end
endmodule
3. SVM Core
Compares input values against thresholds or y_calc to produce classification result.

verilog
module svm_core (
    input wire clk, rst, start,
    input wire [7:0] x_data, y_data, threshold_x, threshold_y,
    input wire [15:0] y_calc,
    input wire [1:0] mode,
    output reg result
);
    always @(posedge clk or negedge rst) begin
        if (!rst) begin
            result <= 1'b0;
        end else if (start) begin
            case (mode)
                2'b00: result <= (x_data >= threshold_x) ? 1'b1 : 1'b0; // Vertical mode
                2'b01: result <= (y_data >= threshold_y) ? 1'b1 : 1'b0; // Horizontal mode
                2'b10: result <= (y_data >= y_calc) ? 1'b1 : 1'b0;      // Inclined mode
                default: result <= 1'b0;
            endcase
        end
    end
endmodule
4. Output Writer
Stores results in memory and writes to result.txt after completion.

verilog
module output_writer (
    input wire clk, rst, start,
    input wire result,
    input wire [7:0] idx
);
    reg [7:0] result_mem [0:255];
    integer i;

    initial begin
        for (i = 0; i < 256; i = i + 1) begin
            result_mem[i] = 8'b0;
        end
    end

    always @(posedge clk or negedge rst) begin
        if (!rst) begin
            for (i = 0; i < 256; i = i + 1) begin
                result_mem[i] <= 8'b0;
            end
        end else if (start) begin
            result_mem[idx] <= {7'b0, result};
        end
    end

    always @(posedge clk) begin
        if (start) begin
            $writememh("result.txt", result_mem);
        end
    end
endmodule
5. Output Interface
Drives LEDs based on mode and result.

verilog
module output_interface (
    input wire result,
    input wire [1:0] mode,
    output reg [2:0] leds
);
    always @(*) begin
        leds = 3'b000;
        case (mode)
            2'b00: leds[0] = result; // LED 0 for vertical mode
            2'b01: leds[1] = result; // LED 1 for horizontal mode
            2'b10: leds[2] = result; // LED 2 for inclined mode
        endcase
    end
endmodule
6. System Tick
Generates a periodic tick (1 second for a 25 MHz clock; adjustable via parameters).

verilog
module System_Tick (
    input Clock, Reset_n,
    output reg tick
);
    parameter n = 25;
    parameter [n-1:0] k = 25000000;
    reg [n-1:0] Q;

    always @(posedge Clock or negedge Reset_n) begin
        if (!Reset_n) begin
            Q <= 1'd0;
            tick <= 1'd0;
        end else if (Q == k-1) begin
            Q <= 1'd0;
            tick <= 1'd1;
        end else begin
            Q <= Q + 1'b1;
            tick <= 1'd0;
        end
    end
endmodule
7. Counter
8‑bit up counter with enable.

verilog
module Counter (
    input clk, rst, En,
    output reg [7:0] Q
);
    always @(posedge clk, negedge rst) begin
        if (!rst)
            Q <= 0;
        else if (En)
            Q <= Q + 1'b1;
    end
endmodule
8. SEG7_LUT
Lookup table for 7‑segment display (hexadecimal to segment pattern).

verilog
module SEG7_LUT (
    input [3:0] iDIG,
    output reg [6:0] oSEG
);
    always @(iDIG) begin
        case (iDIG)
            4'h0: oSEG = 7'b1000000;
            4'h1: oSEG = 7'b1111001;
            4'h2: oSEG = 7'b0100100;
            4'h3: oSEG = 7'b0110000;
            4'h4: oSEG = 7'b0011001;
            4'h5: oSEG = 7'b0010010;
            4'h6: oSEG = 7'b0000010;
            4'h7: oSEG = 7'b1111000;
            4'h8: oSEG = 7'b0000000;
            4'h9: oSEG = 7'b0011000;
            4'ha: oSEG = 7'b0001000;
            4'hb: oSEG = 7'b0000011;
            4'hc: oSEG = 7'b1000110;
            4'hd: oSEG = 7'b0100001;
            4'he: oSEG = 7'b0000110;
            4'hf: oSEG = 7'b0001110;
        endcase
    end
endmodule
Top-Level Integration
svm_system
The main system module instantiates all submodules and connects them.

verilog
module svm_system (
    input wire clk, rst, start, En,
    input wire [1:0] mode,
    output wire done,
    output wire [2:0] leds,
    output wire result, LED,
    output [6:0] X0, X1, Y0, Y1,
    output [6:0] X_th0, X_th1, Y_th0, Y_th1,
    output [8:0] y_calc0
);
    wire [7:0] x_data, y_data, x_threshold, y_threshold;
    wire [15:0] y_calc;
    wire [7:0] idx;
    wire tick;

    Counter u6 (.clk(tick), .rst(rst), .En(En), .Q(idx));
    input_reader u1 (.clk(tick), .rst(rst), .start(start), .x_data(x_data), .y_data(y_data), .done(done), .LED(LED), .idx(idx));
    mode_selector u2 (.mode(mode), .x(x_data), .x_threshold(x_threshold), .y_threshold(y_threshold), .y_calc(y_calc));
    svm_core u3 (.clk(clk), .rst(rst), .start(start), .x_data(x_data), .y_data(y_data), .threshold_x(x_threshold), .threshold_y(y_threshold), .y_calc(y_calc), .mode(mode), .result(result));
    output_writer u4 (.clk(clk), .rst(rst), .start(start), .result(result), .idx(x_data));
    output_interface u5 (.result(result), .mode(mode), .leds(leds));

    // 7‑segment displays
    SEG7_LUT HEX0_Display (.oSEG(X0), .iDIG(x_data[3:0]));
    SEG7_LUT HEX1_Display (.oSEG(X1), .iDIG(x_data[7:4]));
    SEG7_LUT HEX2_Display (.oSEG(Y0), .iDIG(y_data[3:0]));
    SEG7_LUT HEX3_Display (.oSEG(Y1), .iDIG(y_data[7:4]));
    SEG7_LUT HEX4_Display (.oSEG(X_th0), .iDIG(x_threshold[3:0]));
    SEG7_LUT HEX5_Display (.oSEG(X_th1), .iDIG(x_threshold[7:4]));
    SEG7_LUT HEX6_Display (.oSEG(Y_th0), .iDIG(y_threshold[3:0]));
    SEG7_LUT HEX7_Display (.oSEG(Y_th1), .iDIG(y_threshold[7:4]));

    System_Tick tick_gen (.Clock(clk), .Reset_n(rst), .tick(tick));
    assign y_calc0 = y_calc[8:0];
endmodule
Project_ASIC (FPGA Top Wrapper)
Top module connecting the system to the DE2‑115 board peripherals.

verilog
module Project_ASIC (
    input CLOCK_50,
    input CLOCK2_50,
    input CLOCK3_50,
    output [8:0] LEDG,
    output [17:0] LEDR,
    input [3:0] KEY,
    input [17:0] SW,
    output [6:0] HEX0,
    output [6:0] HEX1,
    output [6:0] HEX2,
    output [6:0] HEX3,
    output [6:0] HEX4,
    output [6:0] HEX5,
    output [6:0] HEX6,
    output [6:0] HEX7,
    inout [35:0] GPIO
);
    wire done;
    svm_system DUT (
        .clk(CLOCK_50),
        .rst(KEY[0]),
        .En(SW[2]),
        .start(SW[3]),
        .mode(SW[1:0]),
        .done(LEDG[0]),
        .leds(LEDR[2:0]),
        .X0(HEX0), .X1(HEX1), .Y0(HEX2), .Y1(HEX3),
        .X_th0(HEX4), .X_th1(HEX5), .Y_th0(HEX6), .Y_th1(HEX7),
        .LED(LEDR[8]),
        .y_calc0(LEDR[17:9])
    );
endmodule
Simulation & Testing
Testbench
The testbench instantiates the svm_system, applies reset, asserts start, and waits for done. It can be configured for any mode.

verilog
`timescale 1ns / 1ns
module tb_svm_system;
    reg clk, rst, start;
    wire done;

    // Select mode by uncommenting the desired line
    // svm_system DUT (.clk(clk), .rst(rst), .start(start), .mode(2'b00), .done(done)); // Vertical
    // svm_system DUT (.clk(clk), .rst(rst), .start(start), .mode(2'b01), .done(done)); // Horizontal
    svm_system DUT (.clk(clk), .rst(rst), .start(start), .mode(2'b10), .done(done));   // Inclined

    always #5 clk = ~clk;

    initial begin
        clk = 0;
        rst = 1;
        start = 0;
        #10 rst = 0;
        #10 start = 1;
        wait(done);
        $display("Simulation completed. Results written to result.txt.");
        $stop;
    end
endmodule
Simulation Waveforms
Simulation waveforms (from ModelSim) confirm correct operation:

Inclined mode – shows y_calc computation and comparison

Vertical mode – shows x_data compared with threshold

Horizontal mode – shows y_data compared with threshold

All waveforms match expected results from pre‑computed Excel tables.

Debugging
Fixed file path issues by using absolute paths in $readmemh

Corrected mode selection in testbench (explicitly set mode bits)

Ensured signal matching between output_interface and LED transitions

FPGA Implementation
The project was synthesized for the DE2‑115 board using Quartus Prime.

Clock: 50 MHz

Inputs: Switches (mode, start, enable), push button (reset)

Outputs:

LEDG[0] : done

LEDR[2:0] : classification per mode

HEX0–HEX7 : display x, y, thresholds

The design meets timing constraints and consumes minimal logic resources.

Results and Analysis
Output Example (Inclined Mode)
X (dec)	Y (dec)	y_calc	Output (Z)
48	114	68	1
80	179	100	1
120	122	140	0
...	...	...	...
A chart comparing input points and y_calc values confirms correct classification.

Accuracy: 100% for all modes when compared with expected results from Excel.

Engineering Constraints
Modular design – each module simulated independently before integration.

Dynamic mode selection – user‑configurable via switches.

Resource awareness – optimized for low power on DE2‑115.

Portability issues – absolute file paths for input data remain a limitation (needs updating when moving between systems).

Future Work
LCD integration – replace 7‑segment with a character LCD for better user feedback.

Real‑time input – read from sensors or UART instead of pre‑loaded files.

More classification modes – add non‑linear kernels (polynomial, RBF).

Larger datasets – support 1024+ points with external memory.

Power optimization – clock gating for unused modules.

References
Hosseini, H. G. (2015). Hardware Implementations of SVM on FPGA: A State-of-the-Art Review.

Afifi, S. et al. (2020). FPGA Implementations of SVM Classifiers: A Review. SN Computer Science.

Chu, P. P. (2008). FPGA Prototyping by VHDL Examples.

Harris, S. & Harris, D. (2015). Digital Design and Computer Architecture.

Talib, M. A. et al. (2020). A systematic literature review on hardware implementation of AI algorithms.
