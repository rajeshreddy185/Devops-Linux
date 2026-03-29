# Helm to ArgoCD Migration Process

This repository demonstrates a **zero-downtime migration process** for transitioning existing Helm-managed deployments to ArgoCD GitOps management without disrupting running applications.

## 🎯 Process Overview

The migration process follows a **systematic, safety-first approach** that ensures no service interruption while establishing GitOps control over existing Kubernetes resources.

## 🔄 Phase-by-Phase Migration Process

### **Phase 1: Discovery & Analysis**
```
Objective: Understand and document the exact current state
```
```bash
# Capture deployment state
kubectl get deployment <app-name> -o yaml > current-deployment.yaml

# Capture service state  
kubectl get service <app-name> -o yaml > current-service.yaml
```

**Critical Information to Extract:**
- **Deployment metadata**: name, namespace, labels, annotations
- **Selector configuration**: matchLabels (CRITICAL - immutable)
- **Pod template**: labels, containers, ports, resources
- **Service configuration**: type, ports, selector, external settings
- **Replica count**: Current scaling configuration

#### **Identify Immutable Fields**
```yaml
# Fields that cannot be changed on existing resources:
spec:
  selector:              # ❌ Cannot modify after creation
  template:              # ❌ Changes trigger rolling update
    metadata:
      labels:             # ❌ Changes trigger rolling update
```

### **Phase 2: Helm Chart Construction**
```
Objective: Create Helm chart that generates identical manifests
```

#### **Chart Structure Setup**
```bash
mkdir -p helm-charts/<app-name>/templates
cd helm-charts/<app-name>
```

#### **Values Configuration**
```yaml
# values.yaml - Extracted from existing deployment
replicaCount: <extracted-value>
image:
  repository: <extracted-repo>
  tag: "<extracted-tag>"
  pullPolicy: <extracted-policy>
service:
  type: <extracted-type>
  ports: <extracted-ports>
  selector: <extracted-selector>
```

#### **Template Creation**
```yaml
# templates/deployment.yaml - EXACT MATCH REQUIRED
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <exact-deployment-name>      # Must match exactly
  labels:
    app: <exact-label-name>          # Must match exactly
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: <exact-selector-label>   # Must match exactly
  template:
    metadata:
      labels:
        app: <exact-pod-label>      # Must match exactly
    spec:
      containers:
      - name: <container-name>
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        # ... exact container configuration
```

**Template Precision Rules:**
- **No helper functions** initially - Hardcode exact values
- **No extra labels** - Only use existing labels
- **No additional annotations** - Preserve original metadata
- **Exact name matching** - Prevents resource conflicts

### **Phase 3: ArgoCD Application Configuration**
```
Objective: Configure ArgoCD with safety mechanisms
```

#### **Basic Application Setup**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
spec:
  source:
    repoURL: <git-repository-url>
    targetRevision: main
    path: <path-to-helm-chart>
    helm:
      releaseName: <helm-release-name>
      valueFiles:
      - values.yaml
      skipCrds: true              # Don't modify cluster-wide resources
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>
```

#### **Safety Mechanisms Configuration**
```yaml
syncPolicy:
  automated:
    prune: false              # Don't delete existing resources
    selfHeal: false           # Don't auto-fix differences
  syncOptions:
  - CreateNamespace=true       # Ensure namespace exists
  - PrunePropagationPolicy=foreground  # Safe deletion order
  - PruneLast=true          # New resources first, then cleanup
  - RespectIgnoreDifferences=true   # Honor ignoreDifferences
  - ApplyOutOfSyncOnly=true  # Only change what's different
  - SkipDryRunOnMissingResource=true  # Safe operations
  - ServerSideApply=true    # Better field ownership tracking
```

#### **Comprehensive ignoreDifferences**
```yaml
ignoreDifferences:
- group: apps
  kind: Deployment
  jsonPointers:
  - /spec/replicas              # Ignore scaling changes
  - /spec/selector             # Ignore selector (immutable field)
  - /spec/template/metadata    # Ignore pod metadata changes
  - /spec/template/spec        # Ignore pod specification changes
  - /metadata/labels           # Ignore deployment label differences
  - /metadata/annotations      # Ignore deployment annotation differences
- group: ""
  kind: Service
  jsonPointers:
  - /metadata/labels           # Ignore service label differences
  - /metadata/annotations      # Ignore service annotation differences
  - /spec/ports               # Ignore service port differences
  - /spec/selector            # Ignore service selector differences
  - /spec/type                # Ignore service type differences
```

### **Phase 4: Migration Execution**
```
Objective: Apply ArgoCD configuration and verify zero-downtime
```

#### **Apply ArgoCD Application**
```bash
kubectl apply -f argocd-apps/<app-name>.yaml
```

#### **Critical Monitoring**
```bash
# Monitor for pod restarts (should be NONE)
watch "kubectl get pods -l app=<app-label>"

# Monitor ReplicaSet creation (should be NO new ones)
kubectl get replicasets -l app=<app-label> --watch

# Monitor ArgoCD sync status
argocd app get <app-name> --watch
```

**Expected Healthy Indicators:**
- ✅ **No pod restarts** during initial sync
- ✅ **No new ReplicaSets** created
- ✅ **ArgoCD status shows "Synced"**
- ✅ **Application remains healthy**

#### **Verification Process**
```bash
# Verify deployment unchanged
kubectl get deployment <app-name> -o yaml | grep -A5 -B5 "template:"

# Verify service unchanged
kubectl get service <app-name> -o yaml | grep -A5 -B5 "spec:"

# Test application functionality
curl <service-endpoint>/health
```

## 🛡️ Safety Mechanisms Deep Dive

### **Template-Level Safety**
```
Purpose: Prevent conflicts at the source
```

1. **Exact Name Matching**
   - **Deployment name** = Existing deployment name
   - **Service name** = Existing service name
   - **Container names** = Existing container names

2. **Label Consistency**
   - **Selector labels** = Existing selector labels exactly
   - **Pod template labels** = Existing pod labels exactly
   - **No additional k8s.io labels** = Prevents automatic additions

3. **Configuration Preservation**
   - **Image and tag** = Existing image exactly
   - **Resource limits** = Existing resources exactly
   - **Port configurations** = Existing ports exactly

### **ArgoCD-Level Safety**
```
Purpose: Handle expected differences gracefully
```

1. **Field-Specific Ignoring**
   - **Immutable fields** → /spec/selector
   - **Metadata fields** → /metadata/labels, /metadata/annotations
   - **Template fields** → /spec/template/metadata, /spec/template/spec
   - **Scaling fields** → /spec/replicas (if manual scaling desired)

2. **Conservative Sync Behavior**
   - **No automated pruning** → Prevents accidental deletion
   - **No self-healing** → Prevents unwanted "fixes"
   - **Manual sync only** → Human oversight required

3. **Server-Side Application**
   - **Field ownership tracking** → Kubernetes manages conflicts
   - **Strategic merge patches** → Better handling of complex objects
   - **Reduced client-side conflicts** → More reliable applies

```yaml
    - SkipDryRunOnMissingResource=true
    - ServerSideApply=true
    
spec:
  syncPolicy:
    automated:
      prune: false      
      selfHeal: false 
```


## 🔄 Process Flow Diagram

```
Phase 1: Discovery
┌─────────────────┐
│ Extract Current │
│ Configuration  │
└─────────┬─────┘
          │
          ▼
Phase 2: Chart Creation
┌─────────────────┐
│ Build Helm     │
│ Chart          │
└─────────┬─────┘
          │
          ▼
Phase 3: ArgoCD Setup
┌─────────────────┐
│ Configure      │
│ Safety         │
│ Mechanisms     │
└─────────┬─────┘
          │
          ▼
Phase 4: Migration
┌─────────────────┐
│ Apply &       │
│ Verify        │
└─────────┬─────┘
          │
          ▼
Result: Zero-Downtime GitOps
```

## 📊 Success Metrics & Indicators

### **Process Health Indicators**
```
Successful Migration:
✅ Zero pod restarts during entire process
✅ No new ReplicaSets created
✅ Service continuity maintained 100%
✅ ArgoCD shows "Synced" status
✅ All configurations preserved exactly
```

### **Failure Mode Detection**
```
Common Issues & Solutions:
🚨 Pod restarts → Check template metadata differences
🚨 Apply conflicts → Expand ignoreDifferences coverage
🚨 Selector errors → Verify exact label matching
🚨 Rolling updates → Disable automated sync
```

## 🎯 Process Principles

### **1. Non-Disruption First**
- **Never terminate** running pods unnecessarily
- **Never recreate** resources that can be preserved
- **Always maintain** service availability

### **2. Exact Matching Critical**
- **Names must match** existing resources exactly
- **Labels must match** existing selectors exactly
- **Configuration must match** existing state exactly

### **3. Safety Layers Redundant**
- **Template precision** prevents source conflicts
- **ignoreDifferences** handles expected variations
- **Conservative sync** prevents unwanted automation
- **Server-side apply** handles edge cases

### **4. Verification**
- **Monitor continuously** during migration
- **Verify each phase** before proceeding
- **Rollback ready** if issues detected


## 📋 Migration Completion Checklist

### **✅ Before Removing ignoreDifferences:**
- [ ] Zero-downtime migration confirmed
- [ ] ArgoCD shows "Synced" status
- [ ] No pod restarts occurred
- [ ] Application functioning normally

### **✅ After Removing ignoreDifferences:**
```bash
# Edit ArgoCD application
vim argocd-apps/<app-name>.yaml

# Remove ignoreDifferences section
# Remove migration-specific syncOptions
# Enable automated sync
# Apply changes
kubectl apply -f argocd-apps/<app-name>.yaml
```

### **✅ Post-Migration Verification:**
- [ ] GitOps control enabled
- [ ] Scaling via values.yaml works
- [ ] Configuration changes apply correctly
- [ ] No unexpected rolling updates
- [ ] ArgoCD automation functioning

## 🎯 **The Final State**

### **Clean GitOps Configuration:**
```yaml
# No migration artifacts
# No safety mechanisms needed
# Pure GitOps control
spec:
  syncPolicy:
    automated:
      prune: true      # Normal GitOps
      selfHeal: true   # Normal GitOps
  # No ignoreDifferences section
```

**Bottom line**: Once migration is complete, **remove all ignoreDifferences** to enable proper GitOps functionality. The safety mechanisms served their purpose during transition but now just block normal operations.
