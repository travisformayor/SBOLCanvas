# SBML Export Design & Research

This document consolidates the design decisions, research findings, and implementation details for SBML export from SBOLCanvas, enabling genetic circuit designs to be simulated in iBioSim.

## 1. Overview

**Goal**: Export SBOLCanvas genetic circuit designs to SBML format for quantitative simulation in iBioSim.

**Workflow**: Design (SBOLCanvas) -> Export (SBML) -> Simulate (iBioSim)

**Approach**: Map SBOLCanvas visual genetic circuit elements (backbones, glyphs, interactions) to iBioSim-compatible SBML structures (species, reactions, kinetic laws).

**Target Use Case**: Genetic toggle switch with repression, complex formation, and degradation.

---

## 2. Scope & Constraints

### 2.1 Design Assumptions

**User Knowledge Assumption**: This implementation assumes the user understands how to correctly create genetic circuit designs for SBML export. The code is optimized for clarity and feature implementation, NOT for user assistance or error handling.

**Philosophy**: Keep the initial implementation simple and feature-focused. This makes the first PR's git diff clearly show the SBML export features. Future PRs can add comprehensive validation, safety checks, and user assistance code separately, making that work easier to review in isolation.

**What this means**:
- User knows design rules (one promoter per backbone, no mixed regulation, single regulator per promoter, etc.)
- Code does not validate every constraint or handhold through error cases
- Invalid designs may fail with generic errors or produce incorrect SBML
- Validation and user assistance features are documented as out of scope for later implementation

### 2.2 In Scope

**Interaction Types**:
- Degradation (mass action)
- Complex formation (reversible binding)
- Genetic production with repression (Hill equation)
- Genetic production with activation (Hill equation, implemented but untested)
- Single repressor or single activator per promoter

**Features**:
- Molecular species with initial amounts and boundary conditions
- Promoter species (abstract TU (Transcription Unit) representation)
- Per-reactant cooperativity (nc) for complex formation
- Visual layout export (SBML Layout Extension)
- ID sanitization for SBML compliance

### 2.3 Future Work
**Unregulated Promoter Production**: Support for constitutive expression (no regulators). Currently not implemented. Future work should add a `buildUnregulatedFormula()` method. **Formula (needs verification**: `(P * ko * (ko_f/ko_r) * nr) / (1 + (ko_f/ko_r) * nr)`

**Activation Formula Validation**: Testing `buildActivationFormula()` export with real test case. Genetic toggle switch uses repression only, so activation formula is implemented but not validated.

**Mixed Regulation**: Handling promoters with both activation and repression. Currently, this throws an error. Future support should implement the `buildMixedRegulationPromoterLaw()` method using `BioModel.createProductionKineticLaw()` as a reference.

**Multiple Regulators**: Handling multiple repressors or activators on the same promoter. Currently, only the last regulator is used. Future work should extend kinetic law methods to iterate all regulators and sum their effects (Kr terms or Ka terms).

**Events System Improvements**:
- **Conditional Triggers**: Currently, triggers are hardcoded to `true`. Future work should support boolean expressions (e.g., `Species > 50`) and requires a math expression parser in the UI.
- **Multiple Assignments**: Currently, one event allows one assignment. Future work should support multiple assignments per event.
- **Event Priorities**: Handle simultaneous events with priority ordering.
- **UI Parity**: General UI improvements to match the flexibility of iBioSim's event editor.
- **Events Visual Layout**: Include event glyphs in the generated SBML Layout extension (currently only species and reactions are supported).

**SBML Import**: Converting SBML back to SBOLCanvas.

**SBOL Persistence**: Storing simulation parameters and events in SBOL files. Currently parameters are memory-only in SBOLCanvas

**Global Parameters**: Exporting defaults as global `<parameter>` elements instead of local parameters.

**Dedicated Params Tab**: Moving simulation parameters to a separate UI tab instead of the Info tab.

**Multiple TUs per Backbone**: Detecting multiple Transcriptional Units on a single backbone by scanning for promoter-terminator segments. Currently assumes one backbone equals one TU.

**Comprehensive Validation**: Pre-export check for invalid designs (e.g. loops, mixed regulation). Currently, basic errors are thrown for critical issues. Future work should add a holistic validation layer that checks all constraints upfront with user-friendly error messages.

### 2.4 Expected Design Rules

Users are expected to follow these rules when creating designs for SBML export:

**Critical (Enforced with Error Throws)**:
1. Each backbone must have exactly one promoter glyph -> Throws error if missing
2. Each promoter must have regulation (repressors or activators) -> Throws error if unregulated
3. Each promoter has EITHER repressors OR activators, not both -> Throws error if both present
4. Species referenced by all edges must exist -> Throws error if species not found
5. Association nodes must have at least one outgoing edge (product) -> Throws error if missing

**Expected (Not Enforced)**:
1. Each promoter has at most one regulator -> Uses last regulator if multiple present
2. Each CDS has at most one outgoing production arrow -> All arrows processed (creates duplicate products)

**Future**: Comprehensive pre-export validation with helpful error messages identifying specific problem elements. See "Design Validation & Error Checking" in Section 2.3.

---

## 3. Architecture Summary

### TU-Centric Three-Phase Export

**Phase 1: Create All Species**
- Promoter species: Scan backbones (circuitContainers), find first promoter, create abstract species (SBO:0000590)
- Molecular species: Scan view-level glyphs, create SBML species with SBO terms
- Store TUData and SpeciesData mappings

**Phase 2: Create All Reactions**
- Production: Collect edges by TU internally, create one reaction per TU with all products
- Degradation: Iterate degradation edges, create reactions
- Complex formation: Iterate association nodes, create reactions

**Phase 3: Create Visual Layout**
- Calculate canvas bounds
- Create SBML Layout with normalized coordinates
- Create species/reaction glyphs with curves

### Key Design Decisions / Assumptions

**One TU per Backbone**: Each backbone = one promoter species = one production reaction (multiple CDS create multiple products in ONE reaction).

**Two Kinetic Law Methods**: Repression and activation formulas are structurally different (separate methods, not universal formula). Unregulated out of scope for toggle switch.

**Keyed Parameter Storage**: Per-reactant nc as `nc_<sourceSpeciesURI>` enables independent values despite edge coupling.

**Fail-Fast Philosophy**: Invalid designs throw errors immediately.

### Example: Genetic Toggle Switch Export

**SBOLCanvas Design** (`examples/sbolcanvas_sbol-toggle.xml`):
- 2 backbones (one for pLac TU, one for pTet TU)
- Each backbone has: Promoter glyph, CDS glyph(s)
- Molecular species: LacI, TetR, GFP, IPTG, aTc, IPTG_LacI, aTc_TetR
- Inhibition arrows: LacI -> pTet promoter, TetR -> pLac promoter
- Production arrows: pLac CDS -> TetR, pLac CDS -> GFP, pTet CDS -> LacI
- Association nodes: IPTG + LacI -> IPTG_LacI, aTc + TetR -> aTc_TetR
- Degradation arrows: On LacI, TetR, GFP, complexes

**iBioSim SBML Output** (`examples/ibiosim_sbml-toggle-export.xml`):
- 2 promoter species: `pLac`, `pTet` (SBO:0000590, initialAmount=2)
- 7 molecular species: GFP, LacI, TetR, IPTG, aTc, IPTG_LacI, aTc_TetR
- 2 production reactions:
  - `Production_pLac`: Products=[GFP, TetR], Modifiers=[pLac, LacI as repressor]
  - `Production_pTet`: Products=[LacI], Modifiers=[pTet, TetR as repressor]
- 5 degradation reactions (one per degradable species)
- 2 complex formation reactions (IPTG+LacI, aTc+TetR)

**Key Insight**: SBOLCanvas's 2 backbones -> 2 promoter species -> 2 production reactions (not per-CDS).

---

## 4. SBML Mapping Reference

### Species Types

| SBOLCanvas Type    | SBML SBO Term | Description                   |
|--------------------|---------------|-------------------------------|
| Protein            | SBO:0000252   | Polypeptide chain             |
| Small Molecule     | SBO:0000247   | Simple chemical               |
| Complex            | SBO:0000253   | Non-covalent complex          |
| DNA                | SBO:0000251   | Deoxyribonucleic acid         |
| RNA                | SBO:0000250   | Ribonucleic acid              |
| Promoter (species) | SBO:0000590   | Logical element (abstract TU) |

### Interaction Types & SBML Reactions

| Interaction | Graph Structure | SBML Reaction | Formula | Parameters | Example |
|-------------|----------------|---------------|---------|------------|---------|
| **Degradation** | Arrow FROM species (no target) | Irreversible, SBO:0000179 | `kd * S` | kd | `<reaction id="Degradation_*">` lines 560-573 |
| **Complex Formation** | Node with incoming edges (reactants), outgoing edge (product) | Reversible, SBO:0000177 | `kc_f * A^nc_A * B^nc_B - kc_r * C` | kc_f, kc_r (node), nc per edge (keyed) | `<reaction id="Complex_*">` lines 574-614 |
| **Genetic Production (Repression)** | CDS->Species edges + Inhibition->Promoter | Irreversible, SBO:0000589 | See Section 6.1 | ko, ko_f, ko_r, nr, kr_f, kr_r, nc | `<reaction id="Production_*">` lines 435-496 |
| **Genetic Production (Activation)** | CDS->Species edges + Stimulation->Promoter | Irreversible, SBO:0000589 | See Section 6.2 | kb, ka, ko_f, ko_r, kao_f, kao_r, nr, ka_f, ka_r, nc | N/A (untested) |

**Note**: Example line numbers and tag paths reference `examples/ibiosim_sbml-toggle-export.xml`

**Graph Structure Details**:

**Degradation**: Arrow FROM species glyph (source) with no target. Represents breakdown of that species.

**Complex Formation**:
- Interaction NODE (not arrow) labeled "Association" or "Non-Covalent Binding" in UI
- Incoming edges: Point FROM individual species TO the interaction node (reactants)
- Outgoing edge: Points FROM the interaction node TO the complex species (product)
- Example: IPTG->Node, LacI->Node, Node->IPTG_LacI

**Genetic Production**:
- Arrow FROM CDS glyph (on backbone) TO molecular species glyph (product)
- Effect: Adds molecular species as PRODUCT to the TU's production reaction
- Constraint: Multiple CDS on same backbone create multiple products in ONE reaction
- All kinetic parameters come from promoter glyph, not from production arrow itself

**Note**: iBioSim has a "Degrades" checkbox on species. SBOLCanvas uses explicit degradation arrows for visual clarity.

### Promoter Species (Critical Concept)

In iBioSim SBML, each backbone (TU) becomes an **abstract species** (SBO:0000590):
- Named after first promoter glyph on backbone
- Initial amount = ng parameter (default: 2)
- Added as MODIFIER (SBO:0000598) to production reaction
- Multiplied in kinetic law formula (e.g., `pLac * ko * ...`)
- Visual layout: positioned at backbone midpoint with fixed 100x30 dimensions

**Example**: In `examples/ibiosim_sbml-toggle-export.xml`, `pLac` (line 407, `<species id="pLac">`) and `pTet` (line 408, `<species id="pTet">`) are promoter species with `sboTerm="SBO:0000590"`, `initialAmount="2"`, used in production reactions as modifiers (`<listOfModifiers>` lines 440-443, 502-505), and appear in kinetic law formulas (`<kineticLaw><math>` lines 444-495, 506-558).

### Conceptual Mapping

| SBOLCanvas Concept            | iBioSim SBML Concept            | Mapping                                                                |
|-------------------------------|---------------------------------|------------------------------------------------------------------------|
| Backbone (circuitContainer)   | Promoter Species (SBO:0000590)  | One backbone = one abstract promoter species                           |
| Promoter glyph on backbone    | Promoter parameters             | First promoter glyph defines TU name and parameters                    |
| CDS glyph on backbone         | Production product              | Each CDS->Molecular Species edge adds one product to TU's reaction     |
| Molecular Species glyph       | Species                         | Direct 1:1 mapping (proteins, small molecules, complexes)              |
| Genetic Production arrow      | Product in production reaction  | Multiple arrows from same backbone = multiple products in ONE reaction |
| Inhibition arrow to promoter  | Repressor modifier              | Adds modifier to production reaction, contributes Kr_term              |
| Stimulation arrow to promoter | Activator modifier              | Adds modifier to production reaction, contributes Ka_term              |
| Association interaction node  | Complex formation reaction      | Node with incoming/outgoing edges = reversible reaction                |
| Degradation arrow             | Degradation reaction            | Independent reaction per species                                       |

---

## 5. Implementation Details

### 5.1 Data Storage

**simulationData Field**: Hashtable<String, Object> added to GlyphInfo and InteractionInfo
- Stores SBML-specific parameters (ko, kb, ka, nc, kd, etc.)
- Separate from SBOL data (avoids breaking existing data model)
- Keyed storage for per-reactant nc: `nc_<sourceSpeciesURI>`

**SBOLData.simulationConfig**: HashMap of default parameter values
- Keys: Role/interaction type names ("Pro (Promoter)", "Degradation", "Inhibition")
- Values: Parameter maps (e.g., {"kd": 0.0075})
- Source: iBioSim BioModel.java lines 195-227
- Backend: `getParam()` checks user value first, then `getDefaultValue()` from SBOLData
- Frontend: `/simulationConfig` endpoint serves defaults to UI

**mxGraph Hierarchy**:
```
Cell "0" (root)
  Cell "1" (layer)
    ViewCell
      CircuitContainer (backbone)
        Promoter glyph (child of container)
        CDS glyph (child of container)
      MolecularSpecies glyph (child of view, NOT on backbone)
      InteractionEdge (child of view)
      InteractionNode (child of view)
```

Key: `glyph.getParent()` returns circuitContainer for backbone glyphs, used to map production edges back to TU.

### 5.2 Phase 1: Promoter Species Creation

**Method**: `createPromoterSpecies()`

**Algorithm**:
```
For each backbone (circuitContainer) in the design:
  1. Get all glyphs: getChildCells(backbone, true, false)
  2. Find first promoter: First glyph where partRole contains "Promoter"
  3. If no promoter found: Throw error (fail fast)
  4. Create promoter species:
     - ID: Use promoter glyph's name (or displayID)
     - SBO term: SBO:0000590 (Logical element)
     - Initial amount: ng parameter from promoter's simulationData
  5. Store mapping: backbone -> promoter glyph (for later use)
  6. Store TUData for Phase 2, SpeciesData for Phase 3
```

**Key Implementation Details**:

**Promoter Identification**:
- Check: `glyphInfo.getPartRole() != null && glyphInfo.getPartRole().contains("Promoter")`
- Role string is "Pro (Promoter)" from SBOLData.java
- Use the FIRST promoter found (fail fast if none)

**Parent Hierarchy**:
- `glyph.getParent()` returns the backbone (circuitContainer)
- Used in MxToSBOL.java lines 599, 616: `glyph.getParent().getIndex(glyph)`
- This allows mapping production edges back to their backbone/TU

**Helper Classes**:
```java
class TUData {
  mxCell promoterGlyph;        // For finding regulation edges
  Species promoterSpecies;     // JSBML Species object
  List<mxCell> productionEdges;
}

class SpeciesData {
  Species species;      // JSBML object
  mxGeometry geometry;  // For layout
}
```

### 5.3 Phase 1B: Molecular Species Creation

**Method**: `createMolecularSpecies()` calls `createSpecies()` for each glyph

**Algorithm**:
1. Filter molecular species glyphs at view level (molecularSpeciesFilter)
2. For each: create SBML species, set SBO term based on partType
3. Set initialAmount and boundaryCondition from simulationData
4. Sanitize ID, store SpeciesData

**SBO Term Mapping**:
- Protein -> 252, DNA -> 251, RNA -> 250, Small Molecule -> 247, Complex -> 253

**ID Sanitization** (`sanitizeId()`):
- Replace `[^a-zA-Z0-9_]` with `_`
- Prefix `_` if starts with digit
- Append `_N` for duplicates (tracked in usedIds map)

**Examples**:
- `pLac` -> `pLac` (first occurrence)
- `pLac` -> `pLac_2` (second occurrence with same name)
- `LacI-Repression` -> `LacI_Repression`
- `123start` -> `_123start`

### 5.4 Phase 2: Reaction Creation

**Phase 2A: Production Reactions** (`createProductionReactions()`)

Internally collects production edges by TU, then creates reactions:
```
// Collect production edges by TU (internal)
For each edge in view:
  If edge is "Genetic Production":
    source = edge.getSource() // This is a CDS glyph
    backbone = source.getParent() // CDS's parent is the backbone
    Store: backbone -> list of production edges

// Create reactions
For each backbone with production edges:
  If no production edges: Skip (TU has no products)

  Create ONE reaction named "Production_<promoterName>":
    - Add promoter species as modifier (SBO:0000598)
    - For each production edge: Add target species as product
    - Get repressors: getIncomingEdges(promoter) filtered by Inhibition
    - Get activators: getIncomingEdges(promoter) filtered by Stimulation
    - Add repressors/activators as modifiers
    - Determine regulation type:
      * If repressors only: buildRepressionFormula()
      * If activators only: buildActivationFormula()
      * If neither: throw error (unregulated out of scope)
      * If both: throw error (mixed regulation out of scope)
```

**Regulation Discovery**:
- Inhibition arrows target the promoter glyph (not CDS, not backbone container)
- Use: `mxGraphModel.getIncomingEdges(graphModel, promoterGlyph)`
- Filter by interaction type: SBO:0000169 (Inhibition) or SBO:0000170 (Stimulation)

**Multiple Products per TU**:
- iBioSim format: ONE production reaction with multiple `<speciesReference>` in `<listOfProducts>`
- Example: `Production_pLac` produces both GFP and TetR (`<listOfProducts>` lines 436-439 in `ibiosim_sbml-toggle-export.xml`)
- Implementation: Collect all production edges from a backbone, add all targets as products

**Phase 2B: Degradation Reactions** (`createDegradationReactions()`)
- Iterates all edges, creates `createDegradationReaction()` per degradation arrow

**Phase 2C: Complex Formation Reactions** (`createComplexReactions()`)
- Iterates all nodes, creates `createComplexFormationReaction()` per association node

### 5.5 Phase 3: Visual Layout

**Method**: `createVisualLayout()`

**Coordinate Systems**:
- **SBOLCanvas (mxGraph)**: Infinite canvas, top-left origin (0,0), Y-axis increases downward, coordinates via `cell.getGeometry().getX/Y/Width/Height()`, child cells relative to parent
- **SBML Layout Extension**: Fixed canvas dimensions, top-left origin (0,0), Y-axis increases downward (matches mxGraph), units in pixels, all coordinates absolute

**Coordinate Transformation**:

**Bounding Box Calculation**:
1. Scan all glyphs (molecular species and backbones) during export
2. Track: minX, minY, maxX, maxY of all positions
3. Calculate canvas dimensions: `width = maxX - minX + 2*buffer`, `height = maxY - minY + 2*buffer`
4. Use buffer of 75 pixels on all sides

**Normalization**:
- Normalized X = `originalX - minX + buffer`
- Normalized Y = `originalY - minY + buffer`
- This shifts all coordinates so leftmost/topmost elements start at (buffer, buffer)

**Parent-Relative Coordinates**:
- For glyphs on backbones: Add parent position to child position
- Absolute X = `parent.getGeometry().getX() + child.getGeometry().getX()`
- Absolute Y = `parent.getGeometry().getY() + child.getGeometry().getY()`

**Species Glyph Mapping**:

**Molecular Species Glyphs** (proteins, small molecules, complexes):
- Source: Molecular species glyphs filtered by molecularSpeciesFilter at view level
- Coordinates: Already absolute (view-relative), just normalize
- ID: `"Glyph__" + speciesId`
- Position: Top-left corner from `glyph.getGeometry()`
- Dimensions: Width/height from `glyph.getGeometry()`

**Promoter Species Glyphs** (abstract TU representations):
- Source: Backbone cells (circuitContainer)
- Coordinates: Backbone midpoint represents the TU
- Position X: `backbone.getGeometry().getX() + backbone.getGeometry().getWidth() / 2.0`
- Position Y: `backbone.getGeometry().getY() + backbone.getGeometry().getHeight() / 2.0`
- Dimensions: Fixed default size (100 x 30) - matches iBioSim standard
- Rationale: Promoter species is an abstract logical element, uses standard glyph size regardless of backbone width

**Reaction Glyph Mapping**:
- Position: Product species center
- Dimensions: 0x0 (point location, as used in iBioSim)
- ID Format: `"Glyph__" + reactionId`

**JSBML Layout API Usage**:

**Enable Layout Extension**:
```java
sbmlModel.enablePackage("layout");
LayoutModelPlugin layoutPlugin = (LayoutModelPlugin) sbmlModel.getPlugin("layout");
Layout layout = layoutPlugin.createLayout("iBioSim");
layout.createDimensions(canvasWidth, canvasHeight, 0);
```

**Create Species Glyph**:
```java
SpeciesGlyph sg = layout.createSpeciesGlyph("Glyph__" + speciesId, speciesId);
BoundingBox bbox = sg.createBoundingBox();
bbox.createPosition(normalizedX, normalizedY, 0);
bbox.createDimensions(width, height, 0);
```

**Create Reaction Glyph**:
```java
ReactionGlyph rg = layout.createReactionGlyph("Glyph__" + reactionId, reactionId);
BoundingBox bbox = rg.createBoundingBox();
bbox.createPosition(normalizedX, normalizedY, 0);
bbox.createDimensions(0, 0, 0);  // Point location
```

**Included Features**:
- Species glyph positions and dimensions (bounding boxes)
- Reaction glyph positions (point locations)
- SpeciesReferenceGlyphs with curves (line segments for edges)
- Compartment glyph for "Cell" with proper dimensions
- Canvas dimensions (auto-calculated from species bounds)

**References**:
- Example: `examples/ibiosim_sbml-toggle-export.xml` `<layout:listOfLayouts>` lines 7-398
- JSBML Docs: `org.sbml.jsbml.ext.layout` package
- JSBML User Guide: Section 4.6 page 47 (Layout extension diagram)

### 5.6 Phase 4: Events Creation

**Method**: `createEvents()`

**Algorithm**:
1. Iterate `eventDict` (loaded from graph cell 0).
2. For each `EventInfo`:
   - Create SBML Event.
   - **Trigger**: Set to `true` (always fires at delay time).
   - **Delay**: Set to `eventInfo.delay`.
   - **Assignment**: Create `EventAssignment` for `targetSpecies` with `assignmentValue`.

**Implementation details**:
- **Data Source**: `eventDict` stores `EventInfo` objects (delay, target, value).
- **Simplification**: Trigger is always true; events are purely time-based.
- **Limitation**: One assignment per event.

### 5.7 Special Cases

**Per-Reactant nc for Complex Formation**:
- Problem: Edges to association node share one InteractionInfo
- Solution: Store as `nc_<sourceSpeciesURI>` in simulationData
- Backend: `getKeyedParam()` tries keyed, then default
- Frontend: `getReactantParamKey()` constructs key, `getInteractionParamValue()` for consistent value access
- Output: `nc_<speciesId>` local parameters (matches iBioSim)

**Helper Data Structures**:
- `glyphToSpeciesData`: Maps glyph URI -> SpeciesData
- `tuMap`: Maps backbone -> TUData
- `usedIds`: Tracks IDs for uniquification
- `layoutBounds`: Tracks canvas bounds

---

## 6. Kinetic Law Formulas

### 6.1 Repression-Only Production

**Method**: `buildRepressionFormula()`

**Formula**:
$$\frac{P \cdot ko \cdot \frac{ko\_f}{ko\_r} \cdot nr}{1 + \frac{ko\_f}{ko\_r} \cdot nr + \left(\frac{kr\_f}{kr\_r} \cdot R\right)^{nc}}$$

**Parameters exported**: ko, ko_f, ko_r, nr, kr_f_<repId>, kr_r_<repId>, nc_<repId>

**Note on parameter naming**: `_R` in the formula above is a placeholder. In actual export, `<repId>` is replaced with the sanitized repressor species ID. For example, if LacI is the repressor, parameters become `kr_f_LacI`, `kr_r_LacI`, `nc_LacI`.

**Note**: Does NOT use kb (basal rate). Only ko in numerator.

**Source**: `examples/ibiosim_sbml-toggle-export.xml` `<reaction id="Production_pLac"><kineticLaw><math>` lines 444-495
**iBioSim Code**: `BioModel.createProductionKineticLaw()`

### 6.2 Activation-Only Production

**Method**: `buildActivationFormula()`

**Formula**:
$$\frac{P \cdot \left( kb \cdot \frac{ko\_f}{ko\_r} \cdot nr + ka \cdot \frac{kao\_f}{kao\_r} \cdot nr \cdot \left(\frac{ka\_f}{ka\_r} \cdot A\right)^{nc} \right)}{1 + \frac{ko\_f}{ko\_r} \cdot nr + \frac{kao\_f}{kao\_r} \cdot nr \cdot \left(\frac{ka\_f}{ka\_r} \cdot A\right)^{nc}}$$

**Parameters exported**: kb, ka, ko_f, ko_r, kao_f, kao_r, nr, ka_f_<actId>, ka_r_<actId>, nc_<actId>

**Note on parameter naming**: `_A` in the formula above is a placeholder. In actual export, `<actId>` is replaced with the sanitized activator species ID.

**Note**: Uses kb for basal expression when activator absent.

**iBioSim Code**: `BioModel.createProductionKineticLaw()`

### 6.3 Complex Formation

**Method**: `createComplexFormationReaction()`

**Formula**:
$$kc\_f \cdot [A]^{nc\_A} \cdot [B]^{nc\_B} - kc\_r \cdot [Complex]$$

**Parameters exported**: kc_f, kc_r (from node), nc_A, nc_B (keyed per reactant)

**Per-Reactant nc**: Stored as `nc_<speciesId>` local parameters (iBioSim format).

**Source**: `examples/ibiosim_sbml-toggle-export.xml` `<reaction id="Complex_IPTG_LacI"><kineticLaw><math>` lines 582-609
**iBioSim Code**: BioModel.java lines 1450-1500

### 6.4 Degradation

**Method**: `createDegradationReaction()`

**Formula**:
$$kd \cdot [Species]$$

**Parameters exported**: kd

**Source**: `examples/ibiosim_sbml-toggle-export.xml` `<reaction id="Degradation_IPTG_LacI"><kineticLaw><math>` lines 564-571
**iBioSim Code**: ReactionNode.java

### Parameter Definitions

- `Ko` = `ko_f / ko_r` (RNAP binding equilibrium)
- `Kao` = `kao_f / kao_r` (Activated RNAP binding equilibrium)
- `Ka` = `ka_f / ka_r` (Activation binding equilibrium)
- `Kr` = `kr_f / kr_r` (Repression binding equilibrium)
- `Kc` = `kc_f / kc_r` (Complex formation equilibrium)

---

## 7. Parameter Storage Details

### Promoter Glyph Parameters (simulationData)

**Location**: Promoter glyph on backbone (identified by `partRole` containing "Promoter")

**Purpose**: Defines the TU (Transcriptional Unit) for that backbone. Parameters used to create abstract promoter species and production reaction kinetics.

**Parameter names match iBioSim GlobalConstants.java exactly**:
- `ng` (Initial promoter count) - Default: 2 - Used as `initialAmount` for promoter species
- `np` (Stoichiometry of production) - Default: 10 - Set on product species references
- `ko` (Open complex production rate) - Default: 0.05 - Used in repression/unregulated
- `kb` (Basal production rate) - Default: 0.0001 - Used in activation only
- `ka` (Activated production rate) - Default: 0.25 - Used in activation only
- `Ko_f` (Forward RNAP binding rate) - Default: 0.033 - Ko = Ko_f/Ko_r
- `Ko_r` (Reverse RNAP binding rate) - Default: 1.0
- `Kao_f` (Forward activated RNAP binding rate) - Default: 1.0 - Kao = Kao_f/Kao_r
- `Kao_r` (Reverse activated RNAP binding rate) - Default: 1.0
- `nr` (Initial RNAP count) - Default: 30.0

### Molecular Species Parameters (simulationData)

**Location**: Species glyphs (Protein, Small Molecule, Complex, etc.) at view level (not on backbones)

**Purpose**: Define initial conditions for simulation

**Parameters**:
- `initialAmount` - Default: 0 - Set to non-zero for input species (e.g., IPTG, aTc)
- `boundaryCondition` (Boolean) - Default: false - Set to true for constant inputs

**Note**: Molecular species created by `molecularSpeciesFilter` which finds glyphs at view level.

### Interaction Parameters (simulationData)

**Inhibition arrows** (Repression):
- **Graph**: Arrow FROM repressor species TO promoter glyph on backbone
- **Effect**: Adds repressor as modifier to TU's production reaction, contributes Kr_term to Hill equation denominator
- **Parameters**: `Kr_f` (Forward repression binding) - Default: 0.5, `Kr_r` (Reverse) - Default: 1.0, `nc` (Cooperativity) - Default: 2
- **Usage**: Creates per-repressor local parameters in production reaction: kr_f_R, kr_r_R, nc_R

**Stimulation arrows** (Activation):
- **Graph**: Arrow FROM activator species TO promoter glyph on backbone
- **Effect**: Adds activator as modifier, contributes Ka_term to Hill equation numerator and denominator
- **Parameters**: `Ka_f` (Forward activation binding) - Default: 0.0033, `Ka_r` (Reverse) - Default: 1.0, `nc` (Cooperativity) - Default: 2
- **Usage**: Creates per-activator local parameters: ka_f_A, ka_r_A, nc_A

**Degradation arrows**:
- **Graph**: Arrow FROM species (no target)
- **Effect**: Creates independent degradation reaction for that species
- **Parameters**: `kd` (Degradation rate) - Default: 0.0075

**Association nodes**:
- **Graph**: Interaction NODE with incoming/outgoing edges
- **Parameters**: `Kc_f` (Forward complex formation) - Default: 0.05, `Kc_r` (Reverse) - Default: 1.0

**Association edges** (arrows TO node):
- **Parameters**: `nc` (Cooperativity per reactant) - Default: 2
- **Storage**: Keyed as `nc_<sourceSpeciesURI>` in node's shared simulationData
- **Effect**: Creates per-reactant local parameters in SBML: nc_A, nc_B

**Default Source**: iBioSim BioModel.java lines 195-227

---

## 8. Implementation Challenges

**Basal term kb incorrectly added**: Original `buildProductionKineticLaw()` unconditionally added `+ kb` to all formulas. iBioSim repression formula has NO kb term. **Fix**: Refactored into 2 distinct methods (`buildRepressionFormula()`, `buildActivationFormula()`) with correct formulas. Unregulated moved to out of scope (not needed for toggle switch).

**Double-sanitization ID mismatch**: Calling `sanitizeId()` twice for same species caused layout glyphs to reference non-existent IDs. **Fix**: Store Species objects in SpeciesData, use `species.getId()` instead of re-sanitizing.

**Edge-centric vs TU-centric reactions**: Initial approach created one reaction per production edge. iBioSim uses one reaction per TU with multiple products. **Fix**: Collect edges by backbone, create one production reaction per TU with all products.

**Promoter species positioning**: Unclear whether to use promoter glyph position or backbone midpoint for abstract promoter species in layout. **Fix**: Use backbone midpoint (represents entire TU), with fixed 100x30 dimensions.

**Missing visual layout**: Promoter species created but iBioSim couldn't display them without layout data. **Fix**: Implemented full SBML Layout Extension with species/reaction glyphs and curves.

**Missing required SBML elements**: ReactionGlyphs lacking listOfSpeciesReferenceGlyphs, SpeciesReferenceGlyphs lacking boundingBox elements. **Fix**: Added complete edge structures with curves and bounding boxes per SBML spec.

**Parameter storage location**: Avoid breaking existing SBOL data model. **Fix**: Added `simulationData` field to GlyphInfo and InteractionInfo for SBML-specific parameters.

**Code duplication between MxToSBOL and MxToSBML**: Shared methods (parseGraph, loadDictionary) were duplicated. **Fix**: Extracted to Converter base class.

**Handling default values**: Parameter defaults needed single source of truth. **Fix**: Centralized in SBOLData.simulationConfig.

**Per-reactant nc values**: Multiple edges to association nodes share one InteractionInfo, couldn't set independent nc values. **Fix**: Keyed parameter storage (`nc_<sourceSpeciesURI>`), backend `getKeyedParam()` method, frontend `getReactantParamKey()` and `getInteractionParamValue()` methods for consistent parameter access.

**Silent failure paths**: Used `System.out.println()` for parse errors, silent catch blocks for invalid parameter values, returning from methods instead of throwing. **Fix**: Converted all to throw exceptions (fail-fast).

**Dead exploratory code**: `processInteractionNode()` originally had extensive commented exploration and unused branches. **Fix**: Removed dead code, deleted method entirely. Logic inlined into `createComplexReactions()`.

**Complex nested iteration**: Re-scanning glyphs multiple times across phases. **Fix**: SpeciesData class stores Species + geometry upfront in single map, eliminates re-scanning.

**Mixed-concern phase methods**: Original `collectProductionEdgesAndProcessOtherInteractions()` combined data collection (production edges) with reaction creation (degradation, complex), violating single responsibility. Method name revealed the smell. **Fix**: Refactored into clean separation aligned with SBML output structure. Phase 1: Create all species. Phase 2: Create all reactions (production internally collects edges, degradation and complex iterate and create directly). Phase 3: Create layout. Clarity prioritized over iteration efficiency.

**One TU per backbone simplification**: Needed to map visual backbones to iBioSim's promoter species concept without complex TU detection logic. **Challenge avoided**: Scanning for multiple promoter-terminator segments per backbone requires positional sorting, handling missing terminators, nested structures, and ambiguous boundaries. **Fix**: Simplified assumption: each backbone = one TU = one promoter species = one production reaction. Uses first promoter found, ignores positional structure. Defers multi-TU support to future work (see Out of Scope: "Multiple TUs per Backbone").

**Fail-fast vs validation trade-off**: Original code had some fail-fast checks, but not comprehensive. **Fix**: Removed partial validation for simplicity (user knows rules). Parse failures still throw errors. Comprehensive validation deferred to future PR.

**SBO term-based type distinction**: Needed to distinguish promoter species from molecular species in layout generation. **Fix**: Use `species.getSBOTerm() == 590` to identify promoter vs molecular species.

**Stable identifier for lookups**: Needed consistent key across phases. **Fix**: Use `glyph.getValue()` (GlyphInfo URI) as stable identifier throughout export process.

**Reaction ID naming mismatch**: Original code used `info.getDisplayID()` (e.g., "Interaction_rk0ITP2F") for degradation and complex formation reactions. iBioSim uses descriptive IDs: "Degradation_<speciesName>", "Complex_<productName>". **Fix**: Construct descriptive IDs from species names: `Degradation_<speciesId>` and `Complex_<productId>`. **Why important**: Descriptive IDs make SBML files human-readable and enable tools like iBioSim to identify reaction types without parsing the full reaction structure. Matches iBioSim naming convention and SBML best practices.

**ID Sanitization requirement**: SBOLCanvas allows special characters in names (hyphens, spaces, etc.) that are invalid in SBML identifiers. **SBML SId rules** (from SBML Level 3 specification): Must start with letter or underscore, can only contain letters, digits, and underscores. Formal grammar: `SId ::= (letter | '_') (letter | digit | '_')*`. **Fix**: `sanitizeId()` method replaces invalid characters with underscore, prefixes with underscore if starts with digit, ensures uniqueness. **Sources**: [SBML Level 3 Core Specification, Section 3.1.7](https://pmc.ncbi.nlm.nih.gov/articles/PMC5451324/).

**Visual layout necessity**: Originally implemented to fix promoter species visualization issues in iBioSim. After implementation, teacher feedback explained that the visualization issue was actually just an alternate (still valid) rendering method, not an error. However, the SBML Layout Extension implementation was retained because: (1) it provides explicit positioning control matching SBOLCanvas design, (2) enables consistent visual presentation across tools, (3) fully complies with SBML Layout spec. **Challenge motivation**: Thought we had a rendering bug, implemented comprehensive solution, discovered it wasn't required but kept it anyway.

---

## 9. Implementation Status

### Completed Features

**Data Model Extensions**:
- Added `simulationData` field to GlyphInfo and InteractionInfo (frontend and backend)
- Created `/simulationConfig` endpoint to serve defaults from SBOLData to frontend
- Added UI fields in info-editor for all simulation parameters

**Parameter Centralization**:
- All default values in `SBOLData.simulationConfig` (single source of truth)
- No hardcoded fallbacks in code (fail-fast approach)
- Frontend fetches defaults from backend via HTTP

**ID Sanitization**:
- `sanitizeId()` method handles invalid characters, digit prefixing, and uniquification
- Species IDs accessed via `species.getId()` (no separate getSpeciesId method)

**Reactions**:
- Degradation reactions (mass action: `kd * [Species]`)
- Complex formation reactions (mass action: `kc_f * [A]^nc_A * [B]^nc_B - kc_r * [Complex]`)
- Production reactions - 2 distinct methods, each exports only required parameters:
  - `buildRepressionFormula()` - ko, ko_f, ko_r, nr, kr_f/kr_r/nc per repressor (tested with toggle switch)
  - `buildActivationFormula()` - kb, ka, ko_f, ko_r, kao_f, kao_r, nr, ka_f/ka_r/nc per activator (implemented, untested)

**Code Quality**:
- Moved shared methods (`parseGraph`, `loadDictionary`) to Converter base class
- Removed duplicate code between MxToSBOL and MxToSBML
- Added parameter reading helper methods (`getParam()`, `getKeyedParam()`)
- Fixed TypeScript type safety: `simulationData` properly initialized as object `{}` not array `[]`
- Consistent parameter access pattern: all interaction params use `getInteractionParamValue()`

**TU-Centric Architecture**:
- Phase 1: Species creation (promoter + molecular)
- Phase 2: Reaction creation (production with internal edge collection, degradation, complex formation)
- Phase 3: Visual layout with SBML Layout Extension
- Promoter species included in kinetic law formulas
- Fail-fast validation for mixed regulation (throws error)
- Single-regulator constraint (uses first, ignores additional)
- Clean separation between species and reactions aligned with SBML output structure

**Per-Reactant nc Values**:
- Keyed parameter storage (`nc_<sourceSpeciesURI>` in shared simulationData)
- Backend: `getKeyedParam()` for keyed lookup with default fallback
- Frontend: `getReactantParamKey()` for key construction, `getInteractionParamValue()` for consistent value access
- UI: Each edge to association node can have independent nc value
- SBML: Exports as `nc_<speciesId>` local parameters (matches iBioSim)

**Visual Layout** (SBML Layout Extension):
- Layout bounds tracking during species creation
- Canvas dimension calculation with normalization
- Species glyphs for molecular species (normalized coordinates)
- Species glyphs for promoter species (backbone midpoint)
- Reaction glyphs (product center, point locations)
- Coordinate normalization with buffer on all sides
- SpeciesReferenceGlyphs with curves for all reaction edges
- Compartment glyph with proper dimensions

---

## 10. References

### Example Files
- `examples/ibiosim_sbml-toggle-export.xml` - iBioSim SBML export (reference implementation)
- `examples/sbolcanvas_sbol-toggle.xml` - Same toggle in SBOLCanvas SBOL format

### iBioSim Codebase
- `iBioSim/dataModels/.../BioModel.java`
  - Lines 195-227: Default parameters
  - Lines 1200-1400: Production equations
  - Lines 1450-1500: Complex formation
- `iBioSim/analysis/.../ReactionNode.java` - Reaction evaluation

### JSBML
- API: https://sbml.org/jsbml/files/doc/api/1.6.1/overview-summary.html
- User Guide: Section 4.6 page 47 (Layout extension)
- Packages: `org.sbml.jsbml` (core), `org.sbml.jsbml.ext.layout`

**Dependency Setup**:
- `SBOLCanvasBackend` is a standard Java Web App (WAR)
- JSBML JAR location: `SBOLCanvasBackend/WebContent/WEB-INF/lib/jsbml-1.6.1-with-dependencies.jar`
- Dockerfile compiles with classpath: `WebContent/WEB-INF/lib/*`

**Core API Methods Used**:
1. `SBMLDocument doc = new SBMLDocument(3, 2);` - Level 3, Version 2
2. `Model model = doc.createModel("ModelID");`
3. `Species s = model.createSpecies("SpeciesID");`
4. `Reaction r = model.createReaction("ReactionID");`
5. `FormulaParser` - Parse infix math strings to AST for kinetic laws

### SBML Specification
- Level 3, Version 2
- Layout Extension
- SBO terms

## 12. Modified Files

| Status | File Path |
| :---: | :--- |
| A | `example_files/ibiosim_sbml-toggle-export.xml` |
| A | `example_files/sbolcanvas_sbml-export.xml` |
| A | `example_files/sbolcanvas_sbol-toggle.xml` |
| M | `SBOLCanvasBackend/src/data/GlyphInfo.java` |
| M | `SBOLCanvasBackend/src/data/InteractionInfo.java` |
| M | `SBOLCanvasBackend/src/servlets/Convert.java` |
| M | `SBOLCanvasBackend/src/servlets/Export.java` |
| A | `SBOLCanvasBackend/src/data/EventInfo.java` |
| A | `SBOLCanvasBackend/src/utils/Converter.java` |
| A | `SBOLCanvasBackend/src/utils/MxToSBML.java` |
| M | `SBOLCanvasBackend/src/utils/MxToSBOL.java` |
| M | `SBOLCanvasBackend/src/utils/SBOLData.java` |
| M | `SBOLCanvasFrontend/src/app/app.module.ts` |
| M | `SBOLCanvasFrontend/src/app/canvas/canvas.component.ts` |
| A | `SBOLCanvasFrontend/src/app/eventInfo.ts` |
| M | `SBOLCanvasFrontend/src/app/glyph-menu/glyph-menu.component.html` |
| M | `SBOLCanvasFrontend/src/app/glyph-menu/glyph-menu.component.ts` |
| M | `SBOLCanvasFrontend/src/app/glyphInfo.ts` |
| M | `SBOLCanvasFrontend/src/app/graph-base.ts` |
| M | `SBOLCanvasFrontend/src/app/graph-helpers.ts` |
| M | `SBOLCanvasFrontend/src/app/graph.service.ts` |
| M | `SBOLCanvasFrontend/src/app/info-editor/info-editor.component.css` |
| M | `SBOLCanvasFrontend/src/app/info-editor/info-editor.component.html` |
| M | `SBOLCanvasFrontend/src/app/info-editor/info-editor.component.ts` |
| M | `SBOLCanvasFrontend/src/app/interactionInfo.ts` |
| M | `SBOLCanvasFrontend/src/app/metadata.service.ts` |
| M | `SBOLCanvasFrontend/src/assets/glyph_stencils/utils/.bundle_order` |
| A | `SBOLCanvasFrontend/src/assets/glyph_stencils/utils/event.xml` |
