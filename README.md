This is a fork of [philuxes's guide](https://github.com/philuxe/pve/blob/main/5700g_PciPassthrough.md) to passthrough the AMD Ryzen 7 5700G iGPU to a Ubuntu VM on Proxmox VE.

I left the original guide at [5700g_PciPassthrough.md](5700g_PciPassthrough.md) in this repo for reference.

# Differences with the original guide

In my case, my hardware is different than in philuxes's guide: I have a **Beelink SER5 PRO Ryzen 7 5700U**; different CPU model, different PC manufacturer, etc.

Although the original guide is for Ryzen 7 5700G, it worked nicely for my 5700U (with the necessary adjustments for the different hardware).

So, I decided to fork it and reflect the minor changes I followed for my hardware configuration for future reference (it has also few additional notes).

## Sources

Original sources from philuxes's guide:

-  https://www.bilibili.com/video/BV11d4y1G7Nk/
-  [matt22207's notes](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a) on Proxmox with 5700G APU GPU PCI Passthrough
-  [isc30's repo](https://github.com/isc30/ryzen-7000-series-proxmox) with vbios rom files

Additional resources I found useful:

-  [dioden94's comment on Reddit](https://www.reddit.com/r/Proxmox/comments/st7zlv/comment/kxfio6x/) about machine type switching
-  [boxmox: ASRock 4x4 BOX 5800U + JBOD + Proxmox](https://www.reddit.com/r/homelab/comments/11l0s5j/boxmox_asrock_4x4_box_5800u_jbod_proxmox/)
-  [The Ultimate Beginner's Guide to GPU Passthrough](https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/)
-  [i12bretro's guide](https://i12bretro.github.io/tutorials/0650.html)
