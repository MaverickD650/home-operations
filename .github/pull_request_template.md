# Manual Pull Request

## 🚀 Summary

<!-- Describe what you've changed and why. Be concise but clear. -->

## 📦 Affected Areas

- [ ] HelmRelease
- [ ] Kustomization
- [ ] Secrets (SOPS)
- [ ] Cluster Bootstrap / Repos
- [ ] Ingress
- [ ] Monitoring
- [ ] VolSync / Backups
- [ ] CI
- [ ] Other: <!-- Specify -->

---

## ✅ Checklist

- [ ] Secrets encrypted with SOPS and committed as encrypted only
- [ ] No plaintext secrets committed
- [ ] Base domain, hostnames, and URLs are templated correctly
- [ ] Certifcate issuer set correctly
- [ ] Required Helm repositories are defined in `./repositories/helm`
- [ ] Integrations set correctly
- [ ] Chart versions are correct vs upstream
- [ ] Ingress annotations and rules are correct
- [ ] Volsync setup correctly
- [ ] Kustomizations reference correct chart and path
- [ ] CI workflows (if applicable) run successfully
- [ ] Namespace is correct and consistent
- [ ] Privileged setup where necessary

---

## 🧠 Notes for Myself

<!-- Leave any reminders, notes, or follow-ups for future you here. -->
