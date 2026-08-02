# FEA Process Documentation, 5-Jaw Radial Gripper

**Software:** Ansys Mechanical 2026 R1 (Workbench)
**Model:** 5-jaw radial gripper with central pushrod actuation
**Analysis of record:** Static Structural, two-step loading

---

## 1. Gripper description

The assembly is a 5-jaw radial gripper built around a central pushrod (slider-crank
actuation):

- **Central pushrod (slider):** moves axially to actuate all 5 jaws simultaneously.
  75 mm long, stroke from 11.69 mm (fully open) to 45.64 mm (fully closed) — about
  33.95 mm of linear travel.
- **5 jaw assemblies:** at 72° intervals. Each jaw pivots on a fixed pin in the bracket
  and is driven by a crank link connected to the pushrod.
- **5 crank links:** 20 mm pin-to-pin, converting the pushrod's linear motion into jaw
  rotation.
- **Central bracket:** houses the fixed jaw pivots and the pushrod guide. Ø90 mm base.
- **Mounting flange:** disc with 5× Ø12 mm bolt holes for the robot arm.

The mechanism is a slider-crank: pushrod down closes all five jaws inward, pushrod up
opens them. At the fully open position the crank link is perpendicular (90°) to the
pushrod axis — the lowest mechanical-advantage point in the stroke. Overall size is
roughly 150 mm from mounting flange to jaw tips.

---

## 2. Load case development

No load spec existed, so one was derived from first principles.

**Assumed payload:** 5 kg.

**Required grip force per jaw:**

```
F_grip = (m · g · SF) / (μ · n)
       = (5 · 9.81 · 3.0) / (0.30 · 5)
       ≈ 98 N per jaw
```

where m = 5 kg, g = 9.81 m/s², SF = 3.0, μ = 0.30, n = 5 jaws.

For the FEA, **50 N per jaw** was used as an initial test value, applied as outward
radial forces on the inside face of each jaw tip — the reaction from a gripped object
pushing back against the closed jaws.

**Mechanical advantage.** Because the gripper is a slider-crank, the pushrod force
needed to hold a given grip force varies across the stroke. It is worst at the fully
open position, where the crank link is perpendicular to the pushrod axis, so that is
the position an actuator-sizing calculation should target. An interactive linkage
force calculator was built from the linkage geometry to estimate mechanical advantage
across the stroke and locate this worst case.

Linkage geometry used (estimates from a reference model — to be refined with exact CAD):

- Pivot offset (centerline to fixed jaw pivot): ~18 mm
- Crank link pin-to-pin: 20 mm (measured)
- Jaw pivot to link-pin: 14 mm
- Jaw pivot to tip: 38 mm
- Link-pin angle offset: ~30–40°

---

## 3. FEA model setup

### 3.1 Two analyses: transient (initial), static (adopted)

The model was first built as a **Transient Structural** analysis (standard gravity,
fixed support on the flange, 50 N radial force per jaw tip, a translational joint
driving the pushrod, revolute joints at the pins, large deflection on, weak springs
on, time integration on).

It was then rebuilt as **Static Structural**, and static is the analysis of record.
Both systems still exist in the project, but the static one is the trustworthy result.

**Why static:** a grip check holds the pushrod in a fixed closed position with the
object pushing on the jaws — there is no meaningful acceleration or time-dependent
behavior. Transient solves F = ma at every step, adding inertia effects, longer solves,
and convergence difficulty without adding useful information. (In the transient system,
time-integration effects were in fact turned off, making it a quasi-static solve in a
transient wrapper — another reason to prefer the cleaner static setup. The transient
system also applies closing and grip load in a single load step, which is the setup
that produced the misleading early deflection described in §5.)

### 3.2 Static Structural configuration

| Setting | Value | Reason |
|---------|-------|--------|
| Analysis type | Static Structural | No inertia/time effects needed for a static grip check |
| Large deflection | On | Mechanism undergoes large rotation/translation while closing |
| Weak springs | Off | Weak springs can mask an under-constrained model and inject artificial forces |
| Number of steps | 2 | Separate mechanism closing (step 1) from grip loading (step 2) |
| Gravity | Standard earth gravity | Included |

### 3.3 Two-step loading strategy

**Step 1 — mechanism closing (time 0 → 1):** pushrod joint displacement ramps through
the full stroke; jaw forces held at 0 N. Closes the gripper with no external load.
Substeps: initial 50, min 20, max 200.

**Step 2 — grip load (time 1 → 2):** pushrod displacement holds at the closed position;
jaw forces ramp 0 → 50 N each. Applies the grip reaction with the mechanism locked
closed. Substeps: initial 20, min 10, max 100.

**Why two steps:** the original single-step approach ramped displacement and force
together, so the jaw forces were trying to push the jaws open before the pushrod had
fully closed and the linkage could resist. Separating closing from loading removes that
non-physical transient.

### 3.4 Connections

Current state of the static model:

- **Pushrod:** 1 translational joint (drives the closing displacement).
- **Pins:** 10 revolute joints at the articulating pivots.
- **Non-moving interfaces:** bonded contacts.
- All joints use MPC (multi-point constraint) formulation — the default for Ansys
  revolute joints.

The articulating pivots are still revolute joints (they can rotate), so the mechanism
is free to close; the bonded contacts sit on interfaces that are not meant to move.
The planned upgrade (§4.2) is to replace the MPC pin joints with frictional cylindrical
contacts to remove the residual over-constraint warnings and enable real pin-bore
bearing results.

### 3.5 Material

Structural steel assigned to all bodies: **E = 200 GPa, ν = 0.3, ρ = 7850 kg/m³**.
With a steel yield around 250 MPa, the part is far from stress-limited at this load (see
§5).

---

## 4. Convergence

### 4.1 Current status — clean

The static, two-step model **converges cleanly**. Both load steps solve, each substep
reaching equilibrium in about 3–4 iterations, with displacement residuals several orders
of magnitude below criterion, and the run completes in ~3.5 minutes. This is the behavior
of record.

### 4.2 Earlier oscillation and its cause (resolved)

An earlier configuration showed the force residual oscillating between roughly 0.3 N and
20,000+ N over ~90 cumulative iterations, never settling below criterion — the signature
of contact chatter (contact regions opening and closing each iteration and flipping the
force state). This was tied to the MPC over-constraint condition below. Moving to the
static two-step configuration removed the oscillation.

### 4.3 Residual MPC over-constraint warnings (non-fatal)

The solver still reports MPC/Lagrange over-constraint warnings, because the revolute
joints and several bonded contacts are MPC-based and overlap on shared nodes. In the
current setup these warnings **no longer prevent convergence** — the solve completes
through them — but they are the reason results at or near the pin bores should not yet be
treated as final.

**Root cause:** with 5 jaws at 72°, each pivot pin points in a different direction, so no
single global axis defines the free rotation DOF for all joints, and MPC creates rigid
node couplings that conflict where they overlap.

**Recommended permanent fix (not yet implemented):** replace the revolute pin joints
with **frictional cylindrical contacts using Augmented Lagrange** (pin outer surface as
contact, hole inner surface as target; μ ≈ 0.15 steel-on-steel; symmetric behavior).
Augmented Lagrange uses penalty-based stiffness instead of rigid constraints, and the
cylindrical geometry naturally allows rotation about the pin axis without needing a
defined axis or local coordinate systems — eliminating the MPC conflict and giving real
bearing pressures at each pin-bore interface.

### 4.4 Reference convergence value

An earlier "reference convergence value may be less than the threshold" warning occurred
because the solver's reference force dropped near zero when directional loads were very
small, making tiny residuals look large. It was addressed through the nonlinear force
convergence controls so the reference could not collapse to near-zero.

---

## 5. Results (static, two-step)

Evaluated at time = 2 s (mechanism closed, full 50 N/jaw grip load applied).

| Quantity | Value | Interpretation |
|----------|-------|----------------|
| Peak von Mises stress | **14.12 MPa** | Very low for steel (yield ~250 MPa → ~18× margin even taken at face value). If the peak lies on a sharp claw edge it is a mesh singularity and would need a fillet + mesh-convergence study, but it does not change the conclusion. |
| Total deformation | **37.12 mm** | **Dominated by rigid-body closing motion, not elastic flex** — see below. |
| Deflection under grip load | small (step 1 → step 2 change) | The figure that actually matters for grip accuracy; see below. |

### The deformation reading — the key point

For a mechanism, "total deformation" conflates two very different things: the jaws
**travelling** to their closed position (rigid-body motion through the revolute joints)
and the parts actually **flexing** under load. The time history separates them cleanly:

- Total deformation ramps up entirely during **step 1** (closing) and is **flat through
  step 2** (when the grip load is applied).
- If the 37 mm were elastic flex from the grip load, it would grow during step 2 as the
  force ramps on. It does not.

So the 37 mm is the jaw tips moving through space as the gripper closes — approximately
the pushrod stroke — **not** the structure deflecting. The earlier single-step runs that
reported ~23 mm were measuring the same closing travel (made non-physical by loading
before the linkage engaged).

The number that matters for grip accuracy is the **additional** deflection the grip load
causes: deformation(t = 2) − deformation(t = 1). The nearly flat step-2 curve shows this
is small, i.e. the closed mechanism is stiff under the 50 N grip. It should be quantified
by evaluating total deformation at t = 1 and t = 2 (or a deformation probe on a jaw tip
across step 2).

### Trustworthiness

Stress and deflection away from the pin/joint locations (jaw bodies, bracket, flange) are
reliable. Values at or near the revolute-joint locations (pin holes, pivot bores) carry
the MPC caveat of §4.3 and should not drive design decisions until the frictional-contact
model is in place.

---

## 6. Outstanding work

1. **Quantify grip-load deflection** — extract deformation(t = 2) − deformation(t = 1)
   at the jaw tips; confirm it is within the grip-accuracy target (order 0.5–1 mm).
2. **Frictional pin contacts** — replace MPC revolute joints with Augmented-Lagrange
   frictional cylindrical contacts to clear the over-constraint warnings and produce
   trustworthy pin-bore stresses.
3. **Fillet the singular edge + mesh convergence** — add a fillet at the peak-stress
   edge, refine locally, and confirm stress changes < 5% between refinements.
4. **Refine linkage geometry** — replace estimated dimensions in the force calculator
   with exact CAD measurements.
5. **Fatigue analysis** — S-N life estimation at pin holes, lightening holes, and fillets.
6. **Pushrod buckling check** — linear eigenvalue buckling with the peak axial pushrod
   force from the linkage calculator.
7. **Pin bearing stress** — once contacts replace joints, extract contact pressure at
   each pin-bore and compare to bearing strength.

---

## 7. Formulation reference

| Formulation | Method | Over-constraint risk | Status |
|-------------|--------|----------------------|--------|
| MPC | Rigid mathematical node coupling | High — caused recurring warnings | Currently used (Ansys revolute-joint default) |
| Lagrange Multiplier | Rigid constraint, different solver | Same as MPC | Evaluated, not recommended |
| Augmented Lagrange | Penalty-based stiffness | No conflict risk | Recommended replacement for pin contacts |
| Pure Penalty | Penalty, less accurate | Low | Not used |

---

## 8. Software and settings summary

| Parameter | Value |
|-----------|-------|
| Software | Ansys Mechanical Enterprise (Workbench) 2026 R1 |
| Analysis type | Static Structural (adopted over Transient) |
| Material | Structural steel, E 200 GPa, ν 0.3, ρ 7850 kg/m³ |
| Units | Metric (mm, kg, N, s) |
| Solver | Direct (sparse) |
| Large deflection | On |
| Weak springs | Off |
| Number of steps | 2 |
| Step 1 substeps | Initial 50, min 20, max 200 |
| Step 2 substeps | Initial 20, min 10, max 100 |
| Mesh | Patch conforming |
| Gravity | Standard earth gravity (on) |
| Connections | 1 translational joint, 10 revolute joints (MPC), bonded contacts on fixed interfaces |
| Convergence | Clean — both load steps; residual non-fatal MPC warnings |

---

*Peak stress 14.12 MPa and total deformation 37.12 mm are from the static two-step run at
t = 2 s. The 37.12 mm is rigid-body closing travel, not elastic deflection (see §5).*
