---
name: gatekeeper-policy-skill
description: 编写符合 Gatekeeper 规范的 ConstraintTemplate、Constraint 和 Rego 策略，支持 gator 测试验证
---

# Gatekeeper Policy Skill

Use this skill to create production-ready Gatekeeper policies with proper ConstraintTemplate definitions, Constraints, and Rego rules that pass gator validation.

## Output Format

Always produce:

1. **ConstraintTemplate** - CRD definition with embedded Rego
2. **Constraint** - Instance of the template with parameters
3. **Test Cases** - gator test files for validation
4. **Test Results** - Expected pass/fail scenarios

## Workflow

1. **Understand the policy requirement** - Identify what Kubernetes resources and rules to enforce
2. **Design the ConstraintTemplate** - Define parameters, CRD schema, and Rego logic
3. **Create the Constraint** - Instantiate template with specific values
4. **Write gator tests** - Create test cases covering pass and fail scenarios
5. **Validate with gator** - Ensure all tests pass before delivery

## Output directory layout and relative paths

Create a top-level directory named after the policy. All generated test artifacts must be placed under this policy directory.

```
<policy name>/
  policy/
    suite.yaml
    template.yaml
    constraint.yaml
  cases/
    case1.yaml
    case2.yaml
```

### How relative paths are resolved

`gator verify` resolves any relative path in `suite.yaml` **from the directory that contains `suite.yaml`** (not from the repo root).  
In this layout, that base directory is:

- `<policy name>/policy/`

### What to reference from `suite.yaml`

- `template.yaml` and `constraint.yaml` are in the same folder as `suite.yaml`, so reference them directly:
  - `template: template.yaml`
  - `constraint: constraint.yaml`

- Case manifests live in `<policy name>/cases/`, which is a sibling of `policy/`.  
  From `<policy name>/policy/` to `<policy name>/cases/`:
  - go up one level: `..`
  - then enter `cases/`

  So each case object path must be:
  - `object: ../cases/<case-file>.yaml`

### Example

```yaml
tests:
  - name: example
    template: template.yaml
    constraint: constraint.yaml
    cases:
      - name: case1
        object: ../cases/case1.yaml
      - name: case2
        object: ../cases/case2.yaml


## ConstraintTemplate Guidelines

### Structure

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: <template-name>
spec:
  crd:
    spec:
      names:
        kind: <ConstraintKind>
      validation:
        openAPIV3Schema:
          type: object
          properties:
            <parameters>
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package <package-name>
        
        violation[{"msg": msg}] {
          # violation logic
        }
```

### Rego Best Practices

- **Package naming**: Use `package <template_name>` matching the template name
- **Violation rule**: Always use `violation[{"msg": msg}]` format
- **Input access**: Use `input.review.object` for the resource being validated
- **Parameters**: Access via `input.parameters.<param_name>`
- **Error messages**: Be specific and actionable, include resource name and violation reason
- **Performance**: Use efficient iteration, avoid nested loops when possible

### Common Patterns

```rego
# Check required field exists and is not empty
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.image
  msg := sprintf("Container <%v> must have an image specified", [container.name])
}

# Check field against allowed values
violation[{"msg": msg}] {
  allowed := {p | p := input.parameters.allowedImages[_]}
  container := input.review.object.spec.containers[_]
  not startswith(container.image, allowed[_])
  msg := sprintf("Container <%v> has disallowed image <%v>", [container.name, container.image])
}

# Check required label exists
violation[{"msg": msg}] {
  required := input.parameters.requiredLabels[_]
  not input.review.object.metadata.labels[required]
  msg := sprintf("Resource must have label: %v", [required])
}
```

## Constraint Guidelines

### Structure

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: <ConstraintKind>
metadata:
  name: <constraint-name>
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["default"]
  parameters:
    <parameter-values>
```

### Match Patterns

```yaml
# Match all Pods in specific namespaces
match:
  kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  namespaces: ["default", "production"]

# Match Deployments and ReplicaSets
match:
  kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "ReplicaSet"]

# Exclude certain namespaces
excludedNamespaces: ["kube-system", "gatekeeper-system"]
```

## Gator Test Guidelines (Correct)

Gatekeeper Gator supports two workflows:

- **`gator test`**: ad-hoc evaluation (no assertions), useful for quick checks.
- **`gator verify`**: suite-based tests with assertions, suitable for CI and regression testing.

This guide focuses on the **correct `gator verify` Suite format**, and also includes a minimal `gator test` example.

---

### 1) Use `gator verify` with a Suite (Recommended for CI)

#### Directory Layout (Required by Suite format)

A Suite references other files by **relative path**:

<policy name>/
  policy/
    suite.yaml
    template.yaml
    constraint.yaml
  cases/
    case1.yaml
    case2.yaml


#### Suite File Format (Correct)

```yaml
# suite.yaml
kind: Suite
apiVersion: test.gatekeeper.sh/v1alpha1
metadata:
  name: <suite-name>
  annotations:
    description: "<optional description>"

tests:
  - name: <test-name>
    template: template.yaml       # path to ConstraintTemplate YAML
    constraint: constraint.yaml   # path to Constraint YAML

    cases:
      - name: <case-name-1>
        object: cases/case1.yaml  # IMPORTANT: string path to YAML file
        assertions:
          - violations: no

      - name: <case-name-2>
        object: cases/case2.yaml  # IMPORTANT: string path to YAML file
        assertions:
          - violations: yes
            message: "<substring expected in violation message>"
```

✅ Key rules

- kind MUST be Suite (not TestSuite).

- template and constraint MUST be file paths, not resource names.

- cases[].object MUST be a string file path to a YAML file.

Do not inline the Kubernetes object under object:. That will fail with:
cannot unmarshal object ... object of type string.

```bash
gator verify -f suite.yaml
```
#### 2) Assertion Types (Suite / gator verify)
```yaml
# No violations expected
assertions:
  - violations: no

# At least one violation expected
assertions:
  - violations: yes

# Violation expected and message contains substring
assertions:
  - violations: yes
    message: "must have"

# Specific number of violations expected (exact count)
assertions:
  - violations: 2
```
Notes: message is typically treated as a substring match; keep it stable (avoid full exact messages if formatting may vary).

#### 3) Use gator test for Quick Evaluation (No Suite)

gator test evaluates a set of YAML files and prints violations. It does not use assertions.

Files

- template.yaml (ConstraintTemplate)

- constraint.yaml (Constraint)

- pod-bad.yaml (should violate)

- pod-good.yaml (should pass)

```bash
gator test -f template.yaml -f constraint.yaml -f pod-bad.yaml -f pod-good.yaml
```
Expected behavior:

- pod-bad.yaml shows one or more violations

- pod-good.yaml shows no violations

#### 4) Workload vs Pod: Make Rego Match Your Constraint

If your constraint matches Deployment/StatefulSet/..., remember:

- Pod volumes: spec.volumes

- Workload volumes: spec.template.spec.volumes

Either:

restrict match.kinds to Pod only, or write Rego that handles both Pod and Workload PodSpec paths (recommended), e.g. read from spec for Pod and spec.template.spec for workloads.

This prevents false “passes” where the constraint matches a Deployment but the Rego only inspects Pod fields.

#### Summary

For CI/regression: use gator verify + Suite.

Suite must reference template/constraint/case objects by file paths.

cases[].object must be a string path, not an inline manifest.

Use gator test for fast, assertion-less evaluation.

### Test Best Practices

- **Cover both pass and fail cases** - Test valid and invalid resources
- **Test edge cases** - Empty values, missing fields, boundary conditions
- **Verify error messages** - Ensure messages are clear and actionable
- **Test match/exclude logic** - Verify namespace and kind matching works

## Quality Bar

### ConstraintTemplate
- [ ] CRD schema properly defines all parameters
- [ ] Rego package name matches template name
- [ ] Violation messages are specific and actionable
- [ ] Handles missing fields gracefully
- [ ] Follows Rego best practices

### Constraint
- [ ] Match criteria clearly defined
- [ ] Parameters match template schema
- [ ] Namespace exclusions considered
- [ ] Enforcement action specified (deny/dryrun)

### Gator Tests
- [ ] At least one passing test case
- [ ] At least one failing test case
- [ ] Edge cases covered
- [ ] All tests pass with `gator test`

## Common Policy Patterns

### 1. Required Labels
Enforce specific labels on resources.

### 2. Image Restrictions
Restrict container images to allowed registries.

### 3. Resource Limits
Require CPU/memory limits on containers.

### 4. Security Context
Enforce security settings (runAsNonRoot, readOnlyRootFilesystem).

### 5. Naming Conventions
Enforce resource naming patterns.

### 6. Forbidden Fields
Block specific field values or configurations.

## Validation Commands

```bash
gator verify -f suite.yaml
```

## Reference

- [Gatekeeper Documentation](https://open-policy-agent.github.io/gatekeeper/)
- [Rego Documentation](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gator CLI](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/)
- [ConstraintTemplate Examples](https://github.com/open-policy-agent/gatekeeper/tree/master/charts/gatekeeper/templates)
