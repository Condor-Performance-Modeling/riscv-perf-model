project (core)
add_library(core
  Core.cpp
  Fetch.cpp
  Decode.cpp
  Rename.cpp
  Dispatch.cpp
  Dispatcher.cpp
  Execute.cpp
  ExecutePipe.cpp
  Inst.cpp
  InstArchInfo.cpp
  InstGroup.cpp
  InstGenerator.cpp
  ROB.cpp
  LSU.cpp
  MMU.cpp
  DCache.cpp
  MavisUnit.cpp
  Preloader.cpp
  CPU.cpp
  CPUFactory.cpp
  CPUTopology.cpp
)
get_property(SPARTA_INCLUDE_PROP TARGET SPARTA::sparta PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
target_include_directories(core SYSTEM PRIVATE ${SPARTA_INCLUDE_PROP})
