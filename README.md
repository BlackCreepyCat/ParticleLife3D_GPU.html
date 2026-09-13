# Particle Life

Two self-contained implementations of particle life: a few colored particles, an asymmetric
attraction matrix, and no rule beyond that. Chains, cells, membranes and predator swarms are
not programmed anywhere. They come out of the matrix.

<img width="2276" height="1075" alt="image" src="https://github.com/user-attachments/assets/80767993-6f0d-411a-86ef-ebc1cb5978dc" />


Both builds are single HTML files. No dependency, no build step, no server. Open the file in a
browser and it runs.

| | `particle-life.html` | `particle-life-gpu.html` |
|---|---|---|
| Engine | JavaScript, canvas 2D | WebGPU compute shaders |
| Typical population | 2 500 at 60 fps | 60 000 to 500 000 at 60 fps |
| Glow | blur on a quarter-res buffer | HDR bloom, `rgba16float` target |
| Requirements | any modern browser | Chrome/Edge 113+, Safari 18+, Firefox with WebGPU |

Setups are interchangeable between the two: the JSON schema is identical.

## The model

Each particle belongs to one of up to ten colors. For every pair closer than `rMax`, the force
along the separation is

```
r = distance / rMax

r < beta        f = r / beta - 1                                  hard repulsion, color-blind
beta <= r < 1   f = A[i][j] * (1 - |2r - 1 - beta| / (1 - beta))   triangular ramp from the matrix
r >= 1          f = 0
```

`A[i][j]` is the coefficient of the interaction matrix, in `[-1, 1]`. It is **asymmetric**:
`A[i][j]` and `A[j][i]` are independent, so red can chase green while green flees red. That
asymmetry is what makes the system interesting; a symmetric matrix gives you a conventional
molecular fluid.

Velocity is damped with a half-life rather than a per-step coefficient, so the friction stays
the same when you change the time step:

```
v ← v · 2^(-dt / halfLife) + f · rMax · forceFactor · dt
p ← p + v · dt
```

Space is a torus by default. The neighborhood is found through a uniform grid whose cell size
equals `rMax`, rebuilt every frame by counting sort, so cost is linear in population and
quadratic in local density.

## Controls

| | |
|---|---|
| Left click | attract |
| Right click | repel |
| Wheel | zoom on the cursor |
| Middle click, or Shift + drag | pan |
| Double click, or `0` | reframe |
| `Space` | pause |
| `R` | respawn |
| `M` | new random matrix |
| `H` | hide the panel |

Drag vertically inside a matrix cell to change its coefficient, double-click to zero it. A
`.json` setup file can be dropped straight onto the view.

## Tuning

**Range is the cost knob, not population.** The grid holds `count / cells` particles per cell,
and each particle examines nine cells. Doubling the range roughly quadruples the work, while
doubling the population only doubles it. The GPU build shows the current density at the bottom
of the panel: keep it under about 60 and it stays fast.

Useful starting points:

- **Cells and membranes** — six colors, symmetric matrix, range around 0.05, damping 0.04.
- **Chains and worms** — the `Chains` preset, strong self-attraction with repulsion from
  everything else, low force.
- **Chase** — the `Chase` preset, cyclic matrix, this is where asymmetry shows best.
- **Crystals** — high hard core (0.5), low force, high damping.

If the simulation explodes when you raise the force, raise `Substeps` in the GPU build rather
than lowering the time step: the physics is integrated several times per displayed frame.

## Setup files

```json
{
  "format": "particle-life",
  "version": 1,
  "saved": "2026-09-13T12:00:00.000Z",
  "engine": "webgpu",
  "params": { "types": 4, "count": 60000, "rMax": 0.014, "beta": 0.3, "wrap": true },
  "matrix": [[1, -0.35, 0.2, 0], [0.4, 1, -0.6, 0.1], [0, 0.3, 1, -0.5], [-0.2, 0, 0.45, 1]]
}
```

The matrix is stored as rows so it stays readable in a text editor. Loading is deliberately
tolerant: unknown keys are ignored, missing keys keep their current value, non-finite numbers
are discarded, coefficients are clamped to `[-1, 1]`, and every setting is pushed back through
its slider so the UI bounds act as guard rails. A `count` of 800 000 loaded on a card that
cannot hold it is silently reduced to what the hardware allows.

Setups can also travel as a URL: `Copy link` puts the whole state in the fragment.

## How the GPU build works

Five compute passes per frame, chained in a single command encoder, with nothing read back to
the CPU:

1. **clear** — zero the per-cell counters and the scatter cursors.
2. **count** — one thread per particle, `atomicAdd` on its cell.
3. **scan** — exclusive prefix sum over the cells, single workgroup of 256 threads, each
   handling a contiguous chunk followed by a Hillis-Steele scan over the chunk sums.
4. **scatter** — one thread per particle, `atomicAdd` on the cursor gives the destination slot.
   Positions and colors are written in cell order, which makes the neighbor reads contiguous.
5. **forces** — the 3×3 neighborhood loop, then integration. The pass reads the sorted buffer
   and writes the unsorted one, so there is no read-write hazard and no barrier is needed
   beyond the implicit one between dispatches.

Rendering never touches the CPU either: instanced quads whose vertex shader reads the position
buffer directly, into an `rgba16float` target, then a bloom chain (4× downsample, separable
blur, additive composite with an exposure curve).

Both engines were cross-checked against each other: starting from an identical state and matrix,
positions agree to 3 × 10⁻⁸ world units after a step, which is float32 rounding noise.

## Limitations

- The PNG export of the GPU build goes through `toBlob` on a WebGPU canvas. It works on Chrome
  but can return a black image depending on the driver.
- Minimum zoom is 1×: the camera cannot pull back beyond the world.
- The canvas 2D build caps out around 4 000 particles at 60 fps. That is the point of the other one.

## License

MIT.
