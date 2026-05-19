# MITRE ATT&CK for ICS — Technique Mapping

## T0836 — Modify Parameter
**Tactic**: Impair Process Control
**Indicator**: Large deviation in rolling mean of control sensors
**Subsystems**: P1 (Boiler), P2 (Turbine), C (Control Valves)
**Detection**: rolling_mean deviation > 3σ from baseline

## T0838 — Modify Alarm Settings
**Tactic**: Inhibit Response Function
**Indicator**: Sensor reads constant value — rolling std ≈ 0 for >30s
**Subsystems**: FI (Flow), LI (Level), any process sensor
**Detection**: rolling_std < 0.001 for 30+ consecutive seconds

## T0855 — Unauthorized Command Message
**Tactic**: Impair Process Control
**Indicator**: Cross-sensor correlation breaks — sensors that
normally move together suddenly decouple
**Subsystems**: Any correlated sensor pair
**Detection**: correlation_delta heatmap shows anomaly
