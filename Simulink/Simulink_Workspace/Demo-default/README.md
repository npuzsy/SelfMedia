# Demo_Default_Microgrid

This folder contains a generated Simulink model for a simplified DC microgrid.

Files:

- `Demo_Default_Microgrid.slx`: generated model.
- `build_Demo_Default_Microgrid.m`: rebuilds the model and runs a 10 s verification simulation.
- `init_Demo_Default_Microgrid.m`: parameter values used by the model and build script.
- `verification_summary.txt`: generated after the build script runs.

Model contents:

- Two lithium battery subsystems with averaged bidirectional DC/DC converters.
- Line-aware droop control targeting `Pbat1:Pbat2 ~= 2:1`.
- Different battery-to-bus line resistances: `Rline_bat1 = 0.03 ohm`, `Rline_bat2 = 0.05 ohm`.
- A supercapacitor modeled as the DC bus capacitor state.
- A variable resistor load that steps from `40 ohm` to `20 ohm` at `5 s`.
- Scope legends are driven by named signal lines such as `Vbus_V`, `Iload_A`, `Pbat1_kW`, `Pbat2_kW`, and `P_ratio_Bat1_over_Bat2`.

To rebuild:

```matlab
run('build_Demo_Default_Microgrid.m')
```

