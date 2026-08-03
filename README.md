# dotnet-vmr-trial

Scratch repo for one question: does a [dotnet/dotnet](https://github.com/dotnet/dotnet)
VMR build fit inside conda-forge CI limits?

Run `VMR build trial` from the Actions tab (manual trigger). It measures peak
disk, peak memory and wall-clock for either build mode, against conda-forge's
360-minute `timeout_minutes` ceiling.

Context: conda-forge/dotnet-feedstock#16. This lives in a scratch repo rather
than the feedstock fork because the workflow is fully standalone (it clones the
VMR itself), and because `workflow_dispatch` only registers on a default branch.
