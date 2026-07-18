# Fuse Selection – Gen 2 Distillation Controller

## Requirement

Provide primary branch-circuit protection for the appliance mains input.

Design goals:

* Compact PCB footprint
* UL/CSA/VDE recognized components
* Suitable for worldwide 120 VAC / 230 VAC operation
* Internal fuse (not user serviceable)
* Low assembly cost
* Appropriate for production volumes of hundreds to thousands of units
* Located immediately after line input

---

## Candidate Fuse Types

### 1. 5 × 20 mm Cartridge Fuse

**Status:** Rejected unless field replacement is required.

Advantages:

* Very common
* Easy troubleshooting
* User replaceable

Disadvantages:

* Large footprint
* Requires fuse holder
* Higher BOM cost
* Consumes valuable HV board space

---

### 2. 3.6 × 10 mm Cartridge Fuse

**Status:** Acceptable

Advantages:

* Significantly smaller than 5 × 20 mm
* Available in 250 VAC ratings
* PCB clips readily available

Disadvantages:

* Still requires holder or clips
* Larger than radial alternatives

Examples:

* Littelfuse 396 Series
* Bel Fuse miniature cartridge series

---

### 3. TR5 Radial Lead Fuse

**Status:** Preferred Option

Advantages:

* Very compact footprint
* No separate holder required
* Lower assembly cost
* Common in appliance designs
* Good agency approval coverage
* Easy PCB integration

Typical footprint:

* Body approximately 8 mm × 8 mm
* Lead spacing approximately 5 mm

Examples:

* Littelfuse TR5 Series
* Eaton TR5 Series

Notes:

* Recommended as the baseline solution for Gen 2.
* Time-delay version likely appropriate due to heater and power supply startup currents.

---

### 4. TE5 Radial Lead Fuse

**Status:** Alternative

Advantages:

* IEC-oriented appliance format
* Excellent international approvals
* Available in higher interrupt ratings

Disadvantages:

* Slightly larger than TR5

Examples:

* Littelfuse TE5 Series

---

### 5. Nano2 Surface Mount Fuse

**Status:** Only if board area becomes critical

Advantages:

* Smallest footprint
* Fully automated assembly

Disadvantages:

* Harder field troubleshooting
* Less visually obvious during repair
* More difficult manual replacement

Examples:

* Littelfuse Nano2 Series

---

## Preliminary Recommendation

Use a:

* TR5 radial lead fuse
* 250 VAC rated
* Agency approved (UL/CSA/VDE)
* Time-delay type

Final current rating to be determined after:

* Heater current calculation
* Power supply input current calculation
* Certification review

Expected rating range:

* 8 A to 10 A

---

## PCB Placement Requirements

The fuse shall:

1. Be located immediately adjacent to the mains line input.
2. Be upstream of all appliance loads.
3. Protect:

   * Heater circuit
   * Power supply
   * Any future mains-powered accessories
4. Be accessible for inspection during manufacturing and service.

---

## Open Items

### TODO

* Determine final steady-state line current.
* Determine startup/inrush current.
* Verify required interrupt rating for worldwide operation.
* Confirm certification agency requirements.
* Evaluate board area impact versus cartridge styles.
* Review whether existing SSR inventory affects fuse sizing.

---

## Current Design Direction

**Preferred:** TR5 Time-Delay Fuse

Rationale:

* Small footprint
* Low cost
* Appliance-industry standard
* No fuse holder required
* Simplifies HV board layout
* Appropriate for non-user-serviceable protection
