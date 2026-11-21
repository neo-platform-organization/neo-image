# neo image

The live-image repository for neo. Its canonical source is neo-os/image.neo.
The current image contains language/platform acceptance exercises and a small
text tool. It is not yet the finished OS desktop.

This repository is a sibling of the independent surface VM and depth kernel:

```text
workdir/
  neo/        surface VM and root Makefile
  depth/      platform/depth plugins
  neo-image/  this repository
```

From neo/, build and test with make. The root Makefile uses this image directly
and never copies a loose image.neo into build/. Standalone builds embed it in
the boot bundle. Windows outputs remain build/x86_64/Windows/neo.exe.

The image owns program behavior, application state, drawing decisions, and pure
calculations. Platform adapters, resource mechanisms, preOS management, and the
launcher are C code in the other repositories.

The text-tool counts UTF-8 bytes. It can receive launch arguments and use
explicitly granted asynchronous byte channels. Current transfer and token limits
are small and are documented in the VM milestones; the languages are not yet
complete. documentation/whitepaper.md is a preserved design reference.

The archived neo-foundation-snapshot repository remains a separate historical
baseline. This repository is for ongoing image development.
