# ARM Architecture (CAX Servers) Considerations
**Part:** 6 of 11

## ARM Architecture (CAX Servers) Considerations

### ARM64 Compatibility

**Good News:** k3s has excellent ARM64 support out of the box.

**Verified Compatible:**
- k3s v1.30+ (tested by hetzner-k3s)
- Containerd runtime
- Flannel CNI
- Cilium CNI
- Hetzner Cloud Controller Manager
- Hetzner CSI Driver
- Cluster Autoscaler
- System Upgrade Controller

**Potential Issues:**

1. **Container Images:**
   - Most official images support `linux/arm64`
   - Third-party images may be `linux/amd64` only
   - Check with `docker manifest inspect <image>`

2. **Build Tools:**
   - Native ARM builds are faster
   - Cross-compilation requires QEMU (slower)

3. **Performance:**
   - CAX servers use Ampere Altra (ARM Neoverse N1)
   - Excellent single-threaded performance
   - Better price/performance than x86 for many workloads

**Recommendation:** Start with ARM (CAX), but be prepared to add x86 node pool if needed

**See:** Project has extensive testing on ARM64, no special configuration needed

---

**Previous:** [05-terraform-decisions.md](./05-terraform-decisions.md)
**Next:** [07-networking.md](./07-networking.md) *(To be created)*
