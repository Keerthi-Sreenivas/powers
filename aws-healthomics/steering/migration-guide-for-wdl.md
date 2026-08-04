# SOP: WDL Workflow Migration to AWS HealthOmics

## Purpose

This SOP defines how you, the agent, migrate on-prem or Cromwell-variant WDL workflows to run in AWS HealthOmics. This involves container migration, runtime configuration, storage migration, and output path standardization.

## Constraints

AWS HealthOmics requires:
- All containers MUST be in ECR repositories accessible to HealthOmics.
- All input files MUST be in S3.
- All tasks MUST have explicit CPU and memory runtime attributes.
- Output files are automatically collected from task outputs.
- WDL 1.0+ syntax is required (draft-2 is NOT supported).
- WDL 1.1 syntax is preferred

## Non-Goals

- DO NOT modify the scientific logic of the workflow.
- DO NOT change the workflow structure or task dependencies.
- DO NOT perform performance optimization beyond HealthOmics requirements.

## Procedure

### Phase 1: Container Inventory and Migration

**Objective**: Identify all containers and make them available to HealthOmics from private ECR.

**Steps**:
1. Extract all unique container URIs from runtime sections:
   - Scan all WDL files for `docker:` and `container:` runtime attributes.
   - Check imported WDL files and sub-workflows.
   - Identify containers in struct/object definitions.
2. Generate `container_inventory.csv` with columns: Task name, Original container URI, Container registry, Tool names and versions, Target ECR URI, Approach (`registry-map` or `uri-replacement`).
3. For each container, CHOOSE one approach — see **Registry Map or URI Replacement** under Technical Patterns:
   - **Registry map (PREFERRED)**: the WDL keeps its original public URIs and HealthOmics redirects them to your pull-through caches. Use this WHERE the image itself is unchanged — same repository, same tag, served from your private ECR instead of the public registry.
   - **URI replacement**: edit the WDL. Use this WHERE the migration changes which repository a task pulls from, because the command block's correctness then depends on which image is used.
4. Stage the containers using the MCP tools — PREFER these over hand-written `docker pull`/`docker push` scripts. Follow the [ECR Pull Through Cache SOP](./ecr-pull-through-cache.md):
   - Call `ListPullThroughCacheRules` first. IF a valid cache already exists for an upstream registry, reuse it — DO NOT create another.
   - Call `CreatePullThroughCacheForHealthOmics` for each remaining upstream registry (`docker-hub`, `quay`, `ecr-public`). This also sets the registry permissions policy and repository creation template that HealthOmics needs.
   - Call `CheckContainerAvailability` with `initiate_pull_through: true` to populate and confirm each image.
   - Call `CloneContainerToECR` for registries NOT supported by pull-through cache.
   - IF using a registry map: call `CreateContainerRegistryMap`, then pass it to `CreateAHOWorkflow` via `container_registry_map` (or `container_registry_map_uri`).
   - IF replacing URIs: update the `docker`/`container` values in the WDL task runtime sections, and PREFER parameterizing the registry base path.

   > **The registry map is a workflow attribute, not a run parameter.** It is supplied at workflow creation, and registration does not appear to validate the definition against it: a workflow whose tasks name public registries registers as `ACTIVE` and is unrunnable. The run then fails partway through with `has an invalid structure. Provide a valid ECR image URI`, naming the task it reached rather than the missing attribute, after earlier tasks have already consumed billed compute. ENSURE the map is attached at creation — a successful `CreateAHOWorkflow` does NOT confirm it.
5. Verify container CONTENTS, not just availability. For each task:
   - List the binaries its `command` block invokes. Take the first bare word of each pipeline stage and of each line, including inside `if`/`for` bodies, and discard shell builtins (`set`, `cd`, `echo`, `if`, `then`, `fi`, `for`, `do`, `done`, `mkdir`, `mv`, `cp`) and `~{}` interpolations.
   - Confirm each one is present in that task's image:
     ```bash
     docker run --rm <image> sh -lc 'command -v bwa samtools bcftools'
     ```
   - IF any binary is absent, the task WILL fail at run time — resolve it via step 6 before proceeding.

   > No static check substitutes for this. `CheckContainerAvailability`, `aws ecr describe-images`, `miniwdl check`, and `CreateWorkflow` all pass for an image that exists but lacks a tool the command block pipes to; the task then dies with `command not found`. Single-tool biocontainer images are a frequent source: an image named for one tool often does NOT carry the others in the same pipeline.
6. IF a task needs several tools in one command, use a multi-tool image (for example a `mulled-v2-*` biocontainer). This changes the repository, so treat it as URI replacement per step 3, and record every tool's version in `container_inventory.csv` — they MAY differ from the single-tool images they replace.
7. Create `healthomics.inputs.json` with an ECR registry base path parameter IF container references are parameterized.

**Done WHEN**:
- `container_inventory.csv` documents all containers and records the approach chosen for each.
- All containers are pullable by HealthOmics from private ECR.
- IF using URI replacement: all WDL task runtime sections use ECR URIs, and zero references to external registries remain.
- IF using a registry map: every external registry still referenced in the WDL has a mapping entry, and the map is passed to `CreateAHOWorkflow`.
- For every task, each command invoked in its `command` block is confirmed present in that task's image using the check in step 5.

### Phase 2: Runtime Attribute Audit

**Objective**: Ensure all tasks have CPU and memory runtime declarations.

**HealthOmics Limits**: Min 2 vCPUs / 4 GiB memory. Max 96 vCPUs / 768 GiB memory.

**Steps**:
1. Scan all WDL files for runtime sections.
2. Identify tasks missing `cpu`, `memory`, or `disks` attributes.
3. Check for dynamic resource calculations.
4. Add or update runtime attributes in all tasks:
   ```wdl
   runtime {
       docker: "..."
       cpu: 4
       memory: "8 GiB"
   }
   ```
5. Document resource requirements per task in `docs/healthomics_resources.md`.
6. Create validation script to confirm no task lacks runtime attributes.

**Done WHEN**:
- All tasks have `docker` (or `container` for WDL 1.1), `cpu`, and `memory` runtime attributes.
- All resources meet HealthOmics minimums (≥2 vCPU, ≥4 GB).

### Phase 3: WDL Version Compatibility

**Objective**: Ensure WDL 1.0+ compatibility.

**Steps**:
1. Scan all WDL files for version statements. Identify draft-2 syntax usage.
2. Upgrade syntax as needed:
   - Update version declaration to `version 1.0` or `version 1.1`.
   - Replace `${}` with `~{}` for command interpolation.
   - Update type declarations.
   - Replace deprecated functions.
   - Update struct definitions if using WDL 1.1.
   - Replace `command { ... }` with `command <<< ... >>>` for WDL 1.1+.
3. Validate imports:
   - Ensure all imported WDL files are the same version as the main workflow.
   - Update import statements to use proper aliasing.
   - Check for circular dependencies.
4. Lint:
   - Call `LintAHOWorkflowDefinition` or `LintAHOWorkflowBundle` to verify syntax.
   - For large workflows, use `miniwdl check` if available locally.
   - Resolve all issues.

**Done WHEN**:
- All WDL files declare version 1.0 or higher.
- No draft-2 syntax remains.
- Syntax validation passes for all WDL files.
- All imports resolve correctly.

### Phase 4: Reference and Input File Migration

**Objective**: Migrate all reference files and inputs to S3.

**Steps**:
1. Identify input files and reference data:
   - Extract all `File` and `File?` input parameters.
   - Scan for hardcoded file paths in command sections.
   - List reference files in workflow inputs.
   - Identify files in `Array[File]` inputs.
   - Generate reference inventory with sizes.
2. Design S3 bucket structure appropriate for the workflow. Example:
   ```
   s3://<bucket>/
   ├── references/
   │   ├── Homo_sapiens/
   │   │   ├── GATK/GRCh38/
   │   │   └── NCBI/GRCh38/
   │   └── Mus_musculus/
   ├── annotation/
   └── inputs/
       └── samples/
   ```
3. Create `scripts/migrate_references_to_s3.sh` to:
   - Copy from existing S3 locations if available.
   - Upload local files if needed.
   - Obtain and upload `http(s)://` and `ftp://` resources to S3.
   - Set appropriate S3 storage class (Intelligent-Tiering).
   - Validate checksums after upload.
4. Create `healthomics.inputs.json` with S3 URIs for all File inputs.
5. Update any hardcoded paths in command sections to use input variables.

**Done WHEN**:
- Reference inventory CSV lists all files and sizes.
- All reference files accessible from S3.
- `healthomics.inputs.json` uses S3 URIs exclusively.
- No hardcoded file paths in command sections.

### Phase 5: Output Collection Strategy

**Objective**: Ensure all workflow outputs are properly declared.

**Key Rule**: Intermediate files are automatically cleaned up unless declared as workflow outputs.

**Steps**:
1. Audit workflow outputs:
   - Identify all task outputs that should be retained.
   - Check workflow output section completeness.
   - Verify output types (`File`, `Array[File]`, etc.).
2. Update workflow output section:
   ```wdl
   output {
       File final_vcf = CallVariants.vcf
       File final_vcf_index = CallVariants.vcf_index
       Array[File] bam_files = AlignReads.bam
       File metrics_report = CollectMetrics.report
   }
   ```
3. Document output structure in `docs/healthomics_outputs.md`.
4. Verify all task output declarations and glob patterns.

**Done WHEN**:
- Workflow output section includes all desired outputs.
- All task outputs properly declared.
- Output types correctly specified.

### Phase 6: Configuration and Testing

**Objective**: Create HealthOmics-specific configuration and validate.

**Steps**:
1. Create comprehensive `healthomics.inputs.json` with all required inputs using S3 URIs.
2. Create `test_healthomics.inputs.json`:
   - Use small test dataset (e.g., chr22 only).
   - Minimal sample set (1-2 samples).
   - Use DYNAMIC storage for test runs.
3. Execute test plan:
   - Stage 1: Validate WDL syntax and lint.
   - Stage 2: Test on HealthOmics with minimal dataset.
   - Stage 3: Test with full-size dataset.
   - Stage 4: Resource optimization.
4. IF a test run fails, call `DiagnoseAHORunFailure` to identify issues and remediate.

**Done WHEN**:
- `healthomics.inputs.json` complete with all required inputs.
- WDL validation passes.
- Test workflow completes successfully on HealthOmics.

## Technical Patterns

### Registry Map or URI Replacement

Decide per container by asking whether the migration changes the image's TRANSPORT or its IDENTITY.

| The change is | Same repository and tag, served from private ECR | A different repository (different tool set or versions) |
|---|---|---|
| Approach | Registry map | Edit the WDL |
| WDL | Unmodified — keeps the public URI | `docker`/`container` value replaced |
| Why | The image is identical, so the redirect hides nothing | The command block's correctness now depends on which image is used |

DO NOT hide an identity change inside a registry map. A task whose runtime says `bwa` while the map redirects it to a different repository executes something other than what the WDL says, and the next reader has no way to see it from the workflow definition.

PREFER pinning by digest (`repository@sha256:...`) over a mutable tag on either path. A tag can be reassigned upstream, in which case the same WDL resolves to a different image on a later run.

### Container Runtime (Before/After)

Both paths add the required `cpu` and `memory` attributes. They differ only in whether the image reference changes.

```wdl
# Before
runtime {
    docker: "quay.io/biocontainers/bwa:0.7.17--h5bf99c6_8"
}

# After — registry map path: the image reference is UNCHANGED.
# The map is passed to CreateAHOWorkflow, which redirects quay.io to your pull-through cache.
runtime {
    docker: "quay.io/biocontainers/bwa:0.7.17--h5bf99c6_8"
    cpu: 4
    memory: "8 GB"
}

# After — URI replacement path, for when the repository itself changes.
runtime {
    docker: "<account-id>.dkr.ecr.<region>.amazonaws.com/workflow-name/bwa:0.7.17--h5bf99c6_8"
    cpu: 4
    memory: "8 GB"
}
```

### WDL Version Upgrade (Before/After)
```wdl
# Before (draft-2)
workflow MyWorkflow {
    call MyTask { input: file = input_file }
}

# After (1.0+)
version 1.0

workflow MyWorkflow {
    input {
        File input_file
    }
    call MyTask { input: file = input_file }
    output {
        File result = MyTask.output_file
    }
}
```

### S3 Input (Before/After)
```json
// Before
{ "WorkflowName.reference_fasta": "/path/to/reference.fasta" }

// After
{ "WorkflowName.reference_fasta": "s3://bucket/references/Homo_sapiens/GATK/GRCh38/Sequence/reference.fasta" }
```

## WDL-Specific Considerations

- **Scatter-Gather**: Ensure scattered tasks have appropriate resources. Verify `Array[File]` outputs are properly collected.
- **Sub-Workflows**: Ensure all imported WDL files are migrated. Verify sub-workflow outputs are properly passed.
- **Optional Inputs**: Handle `File?` inputs gracefully. Use `select_first()` or `defined()` appropriately.
- **Command Section**: Use `~{}` for variable interpolation (WDL 1.0+). Avoid hardcoded paths. Use `sep()` for array joining.

## Dependencies

- AWS CLI configured with appropriate permissions
- ECR repositories created
- S3 bucket(s) created with appropriate permissions
- HealthOmics service access
- HealthOmics MCP server
- Docker/Finch/Podman installed for container operations, including verifying image contents (Phase 1)

## References

- [AWS HealthOmics Documentation](https://docs.aws.amazon.com/omics/)
- [WDL 1.0 Specification](https://github.com/openwdl/wdl/blob/main/versions/1.0/SPEC.md)
- [WDL 1.1 Specification](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md)
- [WDL on AWS HealthOmics](https://docs.aws.amazon.com/omics/latest/dev/workflows.html)
- [ECR Documentation](https://docs.aws.amazon.com/ecr/)
