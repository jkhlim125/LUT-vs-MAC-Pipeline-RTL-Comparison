# LUT vs MAC Inference Pipeline — RTL + Cycle-Accurate Latency

Two hardware inference pipelines in Verilog — **LUT-based** (lookup-table) and **MAC-based**
(multiply-accumulate) — processing the same input under identical control, so the only variable is
the datapath. Latency is measured from real signal events, not assumed pipeline depth.

## Result

**LUT = 4 cycles, MAC = 5 cycles — the same +1 across all 18 runs** (`results/comparison_results.csv`,
every row `4,5`). The extra cycle is traceable to exactly one added register stage in the MAC
scoring datapath (`rtl/class_scoring_neuron_mac.v`). Small, but cleanly controlled and verified.

![Event-based latency: latency_lut=4, latency_mac=5](results/latency_comparsion.png)

The LUT output asserts `out_valid` one cycle before the MAC output; `latency_lut` settles to 4 and
`latency_mac` to 5.

## How latency is measured

Event-based, one transaction per run to guarantee correct pairing:

```
start = rising edge of in_valid
end   = out_valid_lut  → latency_lut
        out_valid_mac  → latency_mac
latency = end_cycle − start_cycle          # logged as run_id, latency_lut, latency_mac
```

This measures the real signal timing rather than trusting the designed depth — the point of the project.

## Datapath

```
Input → Register → Feature extraction → Scoring → Aggregation → Decision → Output
                                          └─ MAC path adds one registered stage here (+1 cycle)
```

![Pipeline overview](results/pipeline_overview.png)

| LUT path | MAC path |
|---|---|
| ![LUT path](results/lut_path.png) | ![MAC path](results/mac_path.png) |

## Modules

```
rtl/
  inference_top.v / inference_top_mac.v   top level, instantiates both pipelines
  lut_feature_layer.v / _neuron.v         LUT feature extraction
  class_scoring_layer.v / _neuron.v       LUT scoring
  class_scoring_layer_mac.v / _neuron_mac.v   MAC scoring (the extra register stage)
  class_aggregator.v · decision_logic.v
tb/
  inference_top_tb.v                      drives one transaction per run, logs latency
results/
  comparison_results.csv                  18 runs, all 4 vs 5
  *.png                                   waveforms
```

## Run

```bash
iverilog -o simv tb/inference_top_tb.v rtl/*.v
vvp simv
gtkwave wave.vcd      # optional, to inspect signals
```

## Takeaways

- Pipeline depth determines latency — and it must be measured from real events, not assumed.
- Valid-signal alignment is critical in multi-stage pipelines.
- A one-register structural change produces a deterministic, measurable +1 cycle.
