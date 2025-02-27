include(CheckCXXCompilerFlag)
include(CheckLibraryExists)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/..)
include_directories(${LLDB_SOURCE_DIR}/source)
include_directories(MacOSX/DarwinLog)

include_directories(MacOSX)

function(check_certificate identity result_valid)
  execute_process(
    COMMAND security find-certificate -Z -p -c ${identity} /Library/Keychains/System.keychain
    RESULT_VARIABLE exit_code OUTPUT_QUIET ERROR_QUIET)
  if(exit_code)
    set(${result_valid} FALSE PARENT_SCOPE)
  else()
    set(${result_valid} TRUE PARENT_SCOPE)
  endif()
endfunction()

function(get_debugserver_codesign_identity result)
  string(CONCAT not_found_help
    "This will cause failures in the test suite."
    "Pass '-DLLDB_USE_SYSTEM_DEBUGSERVER=ON' to use the system one instead."
    "See 'Code Signing on macOS' in the documentation."
  )

  # Explicit override: warn if unavailable
  if(LLDB_CODESIGN_IDENTITY)
    set(${result} ${LLDB_CODESIGN_IDENTITY} PARENT_SCOPE)
    check_certificate(${LLDB_CODESIGN_IDENTITY} available)
    if(NOT available AND NOT LLDB_USE_SYSTEM_DEBUGSERVER)
      message(WARNING "LLDB_CODESIGN_IDENTITY not found: '${LLDB_CODESIGN_IDENTITY}' ${not_found_help}")
    endif()
    return()
  endif()

  # Development signing identity: use if available
  check_certificate(lldb_codesign available)
  if(available)
    set(${result} lldb_codesign PARENT_SCOPE)
    return()
  endif()

  if(NOT LLDB_USE_SYSTEM_DEBUGSERVER)
    message(WARNING "Development code sign identiy not found: 'lldb_codesign' ${not_found_help}")
  endif()

  # LLVM pendant: fallback if available
  if(LLVM_CODESIGNING_IDENTITY)
    check_certificate(${LLVM_CODESIGNING_IDENTITY} available)
    if(available)
      set(${result} ${LLVM_CODESIGNING_IDENTITY} PARENT_SCOPE)
      return()
    endif()
  endif()

  # Ad-hoc signing: last resort
  set(${result} "-" PARENT_SCOPE)
endfunction()

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libc++ -Wl,-sectcreate,__TEXT,__info_plist,${CMAKE_CURRENT_SOURCE_DIR}/../resources/lldb-debugserver-Info.plist")

check_cxx_compiler_flag("-Wno-gnu-zero-variadic-macro-arguments"
                        CXX_SUPPORTS_NO_GNU_ZERO_VARIADIC_MACRO_ARGUMENTS)
if (CXX_SUPPORTS_NO_GNU_ZERO_VARIADIC_MACRO_ARGUMENTS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-gnu-zero-variadic-macro-arguments")
endif ()

check_cxx_compiler_flag("-Wno-zero-length-array"
                        CXX_SUPPORTS_NO_ZERO_LENGTH_ARRAY)
if (CXX_SUPPORTS_NO_ZERO_LENGTH_ARRAY)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-zero-length-array")
endif ()

check_cxx_compiler_flag("-Wno-extended-offsetof"
                        CXX_SUPPORTS_NO_EXTENDED_OFFSETOF)
if (CXX_SUPPORTS_NO_EXTENDED_OFFSETOF)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-extended-offsetof")
endif ()

check_library_exists(compression compression_encode_buffer "" HAVE_LIBCOMPRESSION)

add_subdirectory(MacOSX)

set(LLDB_CODESIGN_IDENTITY "" CACHE STRING
    "Identity override for debugserver; see 'Code Signing on macOS' in the documentation (Darwin only)")

get_debugserver_codesign_identity(debugserver_codesign_identity)

# Override locally, so the identity is used for targets created in this scope.
set(LLVM_CODESIGNING_IDENTITY ${debugserver_codesign_identity})

# Use the same identity later in the test suite.
set_property(GLOBAL PROPERTY
  LLDB_DEBUGSERVER_CODESIGN_IDENTITY ${debugserver_codesign_identity})

if(APPLE)
  if(IOS)
    find_library(BACKBOARD_LIBRARY BackBoardServices
      PATHS ${CMAKE_OSX_SYSROOT}/System/Library/PrivateFrameworks)
    find_library(FRONTBOARD_LIBRARY FrontBoardServices
      PATHS ${CMAKE_OSX_SYSROOT}/System/Library/PrivateFrameworks)
    find_library(SPRINGBOARD_LIBRARY SpringBoardServices
      PATHS ${CMAKE_OSX_SYSROOT}/System/Library/PrivateFrameworks)
    find_library(MOBILESERVICES_LIBRARY MobileCoreServices
      PATHS ${CMAKE_OSX_SYSROOT}/System/Library/PrivateFrameworks)
    find_library(LOCKDOWN_LIBRARY lockdown)

    if(NOT BACKBOARD_LIBRARY)
      set(SKIP_TEST_DEBUGSERVER ON CACHE BOOL "" FORCE)
    endif()
  else()
    find_library(COCOA_LIBRARY Cocoa)
  endif()
endif()

if(HAVE_LIBCOMPRESSION)
  set(LIBCOMPRESSION compression)
endif()

if(LLDB_USE_ENTITLEMENTS)
  if(IOS)
    set(entitlements ${CMAKE_CURRENT_SOURCE_DIR}/debugserver-entitlements.plist)
  else()
    # Same entitlements file as used for lldb-server
    set(entitlements ${LLDB_SOURCE_DIR}/resources/debugserver-macosx-entitlements.plist)
  endif()
endif()

set(generated_mach_interfaces
  ${CMAKE_CURRENT_BINARY_DIR}/mach_exc.h
  ${CMAKE_CURRENT_BINARY_DIR}/mach_excServer.c
  ${CMAKE_CURRENT_BINARY_DIR}/mach_excUser.c
  )
add_custom_command(OUTPUT ${generated_mach_interfaces}
  COMMAND mig ${CMAKE_CURRENT_SOURCE_DIR}/MacOSX/dbgnub-mig.defs
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/MacOSX/dbgnub-mig.defs
  )

set(DEBUGSERVER_VERS_GENERATED_FILE ${CMAKE_CURRENT_BINARY_DIR}/debugserver_vers.c)
set_source_files_properties(${DEBUGSERVER_VERS_GENERATED_FILE} PROPERTIES GENERATED 1)

set(lldbDebugserverCommonSources
  DNBArch.cpp
  DNBBreakpoint.cpp
  DNB.cpp
  DNBDataRef.cpp
  DNBError.cpp
  DNBLog.cpp
  DNBRegisterInfo.cpp
  DNBThreadResumeActions.cpp
  JSON.cpp
  StdStringExtractor.cpp
  # JSON reader depends on the following LLDB-common files
  ${LLDB_SOURCE_DIR}/source/Host/common/StringConvert.cpp
  ${LLDB_SOURCE_DIR}/source/Host/common/SocketAddress.cpp
  # end JSON reader dependencies
  libdebugserver.cpp
  PseudoTerminal.cpp
  PThreadEvent.cpp
  PThreadMutex.cpp
  RNBContext.cpp
  RNBRemote.cpp
  RNBServices.cpp
  RNBSocket.cpp
  SysSignal.cpp
  TTYState.cpp

  MacOSX/CFBundle.cpp
  MacOSX/CFString.cpp
  MacOSX/Genealogy.cpp
  MacOSX/MachException.cpp
  MacOSX/MachProcess.mm
  MacOSX/MachTask.mm
  MacOSX/MachThread.cpp
  MacOSX/MachThreadList.cpp
  MacOSX/MachVMMemory.cpp
  MacOSX/MachVMRegion.cpp
  MacOSX/OsLogger.cpp
  ${generated_mach_interfaces}
  ${DEBUGSERVER_VERS_GENERATED_FILE})

add_library(lldbDebugserverCommon ${lldbDebugserverCommonSources})
set_target_properties(lldbDebugserverCommon PROPERTIES FOLDER "lldb libraries/debugserver")

target_link_libraries(lldbDebugserverCommon
                      INTERFACE ${COCOA_LIBRARY}
                      ${CORE_FOUNDATION_LIBRARY}
                      ${FOUNDATION_LIBRARY}
                      ${BACKBOARD_LIBRARY}
                      ${FRONTBOARD_LIBRARY}
                      ${SPRINGBOARD_LIBRARY}
                      ${MOBILESERVICES_LIBRARY}
                      ${LOCKDOWN_LIBRARY}
                      lldbDebugserverArchSupport
                      lldbDebugserverDarwin_DarwinLog
                      ${LIBCOMPRESSION})
if(HAVE_LIBCOMPRESSION)
  set_property(TARGET lldbDebugserverCommon APPEND PROPERTY
                COMPILE_DEFINITIONS HAVE_LIBCOMPRESSION)
endif()
set(LLVM_OPTIONAL_SOURCES ${lldbDebugserverCommonSources})
add_lldb_tool(debugserver ADD_TO_FRAMEWORK
  debugserver.cpp
  LINK_LIBS lldbDebugserverCommon
  ENTITLEMENTS ${entitlements}
)

set_target_properties(debugserver PROPERTIES FOLDER "lldb libraries/debugserver")

if(IOS)
  set_property(TARGET lldbDebugserverCommon APPEND PROPERTY COMPILE_DEFINITIONS
    WITH_LOCKDOWN
    WITH_FBS
    WITH_BKS
    )
  set_property(TARGET debugserver APPEND PROPERTY COMPILE_DEFINITIONS
    WITH_LOCKDOWN
    WITH_FBS
    WITH_BKS
    )
  set_property(TARGET lldbDebugserverCommon APPEND PROPERTY COMPILE_FLAGS
    -F${CMAKE_OSX_SYSROOT}/System/Library/PrivateFrameworks
    )

  add_library(lldbDebugserverCommon_NonUI ${lldbDebugserverCommonSources})
  target_link_libraries(lldbDebugserverCommon_NonUI
                      INTERFACE ${COCOA_LIBRARY}
                      ${CORE_FOUNDATION_LIBRARY}
                      ${FOUNDATION_LIBRARY}
                      lldbDebugserverArchSupport
                      lldbDebugserverDarwin_DarwinLog
                      ${LIBCOMPRESSION})
  if(HAVE_LIBCOMPRESSION)
    set_property(TARGET lldbDebugserverCommon_NonUI APPEND PROPERTY
                  COMPILE_DEFINITIONS HAVE_LIBCOMPRESSION)
  endif()

  add_lldb_tool(debugserver-nonui
    debugserver.cpp

    LINK_LIBS
      lldbDebugserverCommon_NonUI

    ENTITLEMENTS
      ${entitlements}
    )
endif()
