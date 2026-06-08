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

