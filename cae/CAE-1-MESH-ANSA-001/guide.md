# Guide: ANSA Meshing of Turbocharger Compressor Housing

1. **Fix geometry before meshing** — never attempt to mesh over defects. Geometry cleanup is a prerequisite, not an optional step.

2. **Sliver face removal**: Use ANSA's Topo > Fix > Slivers tool. Verify topology integrity after each fix — removing a sliver can create new topology issues if adjacent faces are not stitched correctly.

3. **Gap closure strategy**: Project edges to close gaps rather than extending surfaces. Edge projection preserves the original surface curvature; surface extension can distort it.

4. **Volute tongue is the most geometry-sensitive region** — expect to do manual cleanup here. Automated tools often fail at the tongue because of the tight curvature and converging surfaces.

5. **Surface mesh first, then volume mesh**: Generate a triangular surface mesh, inspect it thoroughly for free edges and T-connections, and only then proceed to tetrahedral volume meshing.

6. **Growth rate between refinement zones and global mesh**: Keep the maximum growth rate at 1.3. Larger growth rates cause element quality degradation at the transition.

7. **Boundary layer meshing is not needed** for this structural analysis. Do not add prism layers — this is not a CFD model.

8. **Through-wall element count**: Use ANSA's "Layers" check tool to verify element count through the wall thickness. Do not rely on visual inspection — it is unreliable for thin sections.

9. **Quality statistics**: Always check the worst 1% of elements, not just the average quality. A good average can hide a cluster of bad elements in a critical region.

10. **Node equivalencing**: Use the default tolerance of 0.01mm. Increasing the tolerance risks merging nodes that should remain separate, especially at thin features.

11. **Check for inverted elements after volume meshing** — ANSA does not always flag inverted tets automatically. Run an explicit Jacobian check on the volume mesh.

12. **Feature edge preservation**: Set the feature angle threshold to 30 degrees. This preserves sharp edges at the flange and volute tongue while allowing smooth transitions elsewhere.

13. **Bolt hole meshing**: The 4-bolt mounting flange bolt holes require washer faces with finer mesh for accurate contact stress representation in downstream analysis.

14. **Element size at mounting flange**: Set refinement at bolt holes to approximately bolt hole diameter divided by 4. This provides adequate resolution for bolt load transfer.

15. **Duplicate node check after export**: Some mesh export formats (particularly Nastran) can introduce duplicate nodes at partition boundaries. Run a duplicate node check on the exported file.

16. **Export in both formats for solver compatibility verification**: Comparing element and node counts between Nastran and Abaqus exports catches format-specific export issues.
