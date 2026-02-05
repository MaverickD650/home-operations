# Manual Pull Request

## 🚀 Summary

<!-- Describe what you've changed and why. Be concise but clear. -->

## 📦 Affected Areas

- [ ] HelmRelease
- [ ] Kustomization
- [ ] Secrets (External Secrets, SOPS)
- [ ] Cluster Bootstrap
- [ ] Ingress
- [ ] Monitoring
- [ ] VolSync / Backups
- [ ] CI
- [ ] Other: <!-- Specify -->

---

## ✅ Checklist

- [ ] Secrets encrypted with SOPS or specified as External Secrets
- [ ] No plaintext secrets committed
- [ ] Base domain, hostnames, and URLs are templated correctly
- [ ] Certifcate issuer set correctly
- [ ] Required OCI repositories set with correct version and image
- [ ] Integrations set correctly
- [ ] Chart versions are correct vs upstream
- [ ] Gateway API routes and policies set correctly
- [ ] Volsync setup correctly
- [ ] Kustomizations reference correct chart and path
- [ ] CI workflows (if applicable) run successfully
- [ ] Namespace is correct and consistent
- [ ] Privileged setup where necessary

---

## 🧠 Notes for Myself

<!-- Leave any reminders, notes, or follow-ups for future you here. -->
