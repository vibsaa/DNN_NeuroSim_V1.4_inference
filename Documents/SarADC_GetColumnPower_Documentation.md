# SarADC::GetColumnPower Function Documentation

## Overview
The `GetColumnPower` function in the SarADC (Successive Approximation Register Analog-to-Digital Converter) class calculates the energy consumption for reading a single column in a neuromorphic computing array.

## Location
- **File**: `Inference_pytorch/NeuroSIM/SarADC.cpp`
- **Lines**: 157-244
- **Header Declaration**: `Inference_pytorch/NeuroSIM/SarADC.h`, Line 66

## Function Signature
```cpp
double SarADC::GetColumnPower(double columnRes)
```

## Parameters
- `columnRes` (double): The resistance of the column in ohms (Ω)

## Return Value
- Returns `Column_Energy` (double): The energy consumed for reading one column in Joules (J)

## Function Workflow

### 1. Initialization
```cpp
double Column_Power = 0;
double Column_Energy = 0;
```
Two variables are initialized:
- `Column_Power`: Instantaneous power consumption (Watts)
- `Column_Energy`: Total energy consumed (Joules)

### 2. Voltage Adjustment
```cpp
columnRes *= 0.5/param->readVoltage;
```
The column resistance is adjusted to account for the difference between:
- **Fixed simulation voltage**: 0.5V (used in Cadence simulations)
- **User-defined read voltage**: `param->readVoltage`

This scaling ensures the power model remains accurate across different read voltages.

### 3. Edge Case Handling

#### Case A: Infinite Resistance (1/columnRes == 0)
```cpp
if ((double) 1/columnRes == 0) { 
    Column_Power = 1e-6;  // 1 µW
}
```
When resistance is extremely high (approaching infinity), a minimal power of 1 µW is assigned.

#### Case B: Zero Resistance
```cpp
else if (columnRes == 0) {
    Column_Power = 0;
}
```
Zero resistance means no power consumption.

### 4. Normal Case: Power Calculation Based on Technology Node

The power calculation uses empirically-derived formulas that vary based on:
- **Device Roadmap**: High Performance (HP) or Low Power (LP)
- **Technology Node**: Ranging from 130nm down to 1nm

#### Formula Structure
For each technology node, the power is calculated as:
```
Column_Power = (A*log2(levelOutput) + B) * 1e-6 + C*exp(-D*log10(columnRes))
```

Where:
- **A, B**: Constants related to the ADC bit resolution
- **C, D**: Constants related to the column resistance
- **levelOutput**: Number of ADC output levels (2^bits)
- **1e-6**: Converts to microwatts (µW)

#### High Performance (HP) Roadmap
Technology nodes: 130nm, 90nm, 65nm, 45nm, 32nm, 22nm, 14nm, 10nm, 7nm

Example for 32nm HP:
```cpp
Column_Power = (1.0157*log2(levelOutput)+7.6286)*1e-6;
Column_Power += 0.083709*exp(-2.313*log10(columnRes));
```

#### Low Power (LP) Roadmap
Technology nodes: 130nm, 90nm, 65nm, 45nm, 32nm, 22nm, 14nm, 10nm, 7nm, 5nm, 3nm, 2nm, 1nm

Example for 7nm LP:
```cpp
Column_Power = (0.1061*log2(levelOutput)+0.8847)*1e-6;
Column_Power += 0.043555*exp(-2.303*log10(columnRes));
```

**Note**: The 1.4 update extended SAR ADC power projections down to 1nm node for LP technology.

### 5. Temperature Compensation
```cpp
Column_Power *= (1+1.3e-3*(param->temp-300));
```
The power is adjusted for temperature effects:
- **Base temperature**: 300K (≈27°C)
- **Temperature coefficient**: 1.3×10⁻³ per Kelvin
- Power increases linearly with temperature

### 6. Energy Calculation
```cpp
Column_Energy = Column_Power * (log2(levelOutput)+1)*1e-9;
```
Energy is calculated by multiplying power by the conversion time:
- **Conversion time**: `(log2(levelOutput)+1) * 1e-9` seconds (nanoseconds)
- This represents the time for SAR ADC successive approximation cycles
- Number of cycles = log2(levels) + 1 (extra cycle for setup)

## Usage Context

The function is called from `SarADC::CalculatePower`:
```cpp
void SarADC::CalculatePower(const vector<double> &columnResistance, double numRead) {
    readDynamicEnergy = 0;
    for (double i=0; i<columnResistance.size(); i++) {
        double E_Col = 0;
        E_Col = GetColumnPower(columnResistance[i]);
        readDynamicEnergy += E_Col;
    }
    readDynamicEnergy *= numRead;
}
```

## Key Insights

### 1. Technology Scaling
- As technology nodes shrink (130nm → 1nm), power consumption decreases
- LP roadmap generally consumes less power than HP roadmap
- Modern nodes (7nm, 5nm, 3nm, 2nm, 1nm) show significant power improvements

### 2. Resolution Impact
- Power increases logarithmically with ADC resolution
- Higher bit-width ADCs (larger `levelOutput`) consume more power
- The `log2(levelOutput)` term captures this relationship

### 3. Column Resistance Effect
- Power has an exponential relationship with column resistance
- Lower resistance (higher current) → higher power consumption
- The `exp(-2.3*log10(columnRes))` term models this behavior

### 4. Energy-Delay Product
- Energy is proportional to conversion time
- More ADC bits → longer conversion time → higher energy
- Time complexity: O(log2(resolution))

## Related Functions
- `SarADC::CalculatePower()`: Parent function that iterates over all columns
- `SarADC::CalculateLatency()`: Calculates timing for SAR ADC operations
- `SarADC::Initialize()`: Sets up ADC parameters including `levelOutput`

## Dependencies
- **Global Param Object**: `extern Param *param`
  - `param->readVoltage`: Read voltage setting
  - `param->temp`: Operating temperature
  - `param->technode`: Technology node (nm)
  - `param->deviceroadmap`: 1 for HP, other for LP

## Empirical Model Basis
The power formulas are derived from:
- Cadence SPICE circuit simulations
- Fixed 0.5V read voltage in simulations
- Real transistor models for various technology nodes
- Characterization across different column resistance values

## Important Notes
1. This is an **empirical model** based on circuit simulations
2. Power values are in **Watts**, energy in **Joules**
3. The model accounts for technology scaling, resolution, and load
4. Temperature compensation uses a linear approximation
5. Edge cases (zero and infinite resistance) are handled gracefully
