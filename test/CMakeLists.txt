function(add_python_test_target name test_script args comment)
  set(PYTHON_TEST_COMMAND
    ${PYTHON_EXECUTABLE}
    ${test_script}
    ${args}
    )

  add_custom_target(${name}
    COMMAND ${PYTHON_TEST_COMMAND} ${ARG_DEFAULT_ARGS}
    COMMENT "${comment}"
    DEPENDS ${LLDB_TEST_DEPS}
    USES_TERMINAL
    )
endfunction()

set(LLDB_TEST_DEPS lldb)

# darwin-debug is an hard dependency for the testsuite.
if (CMAKE_SYSTEM_NAME MATCHES "Darwin")
  list(APPEND LLDB_TEST_DEPS darwin-debug)
endif()

if(TARGET lldb-server)
  list(APPEND LLDB_TEST_DEPS lldb-server)
endif()
  
if(TARGET debugserver)
  if(NOT CMAKE_HOST_APPLE OR LLDB_CODESIGN_IDENTITY)
    list(APPEND LLDB_TEST_DEPS debugserver)
  endif()
endif()

if(TARGET lldb-mi)
  list(APPEND LLDB_TEST_DEPS lldb-mi)
endif()

# The default architecture with which to compile test executables is the default LLVM target
# architecture, which itself defaults to the host architecture.
string(TOLOWER "${LLVM_TARGET_ARCH}" LLDB_DEFAULT_TEST_ARCH)
if( LLDB_DEFAULT_TEST_ARCH STREQUAL "host" )
  string(REGEX MATCH "^[^-]*" LLDB_DEFAULT_TEST_ARCH ${LLVM_HOST_TRIPLE})
endif ()

# Allow the user to override the default by setting LLDB_TEST_ARCH
set(LLDB_TEST_ARCH
	${LLDB_DEFAULT_TEST_ARCH}
	CACHE STRING "Specify the architecture to run LLDB tests as (x86|x64).  Determines whether tests are compiled with -m32 or -m64")

# Users can override LLDB_TEST_USER_ARGS to specify arbitrary arguments to pass to the script
set(LLDB_TEST_USER_ARGS
  ""
  CACHE STRING "Specify additional arguments to pass to test runner. For example: '-C gcc -C clang -A i386 -A x86_64'")

set(LLDB_TEST_COMMON_ARGS
  --arch=${LLDB_TEST_ARCH}
  --executable $<TARGET_FILE:lldb>
  -s
  ${CMAKE_BINARY_DIR}/lldb-test-traces
  -S nm
  -u CXXFLAGS
  -u CFLAGS
  -C ${LLDB_TEST_C_COMPILER}
  )

if ( CMAKE_SYSTEM_NAME MATCHES "Windows" )
  # All tests are currently flaky on Windows, so rerun them all once when they fail.
  set(LLDB_TEST_COMMON_ARGS ${LLDB_TEST_COMMON_ARGS} --rerun-all-issues)
  
  set(LLDB_TEST_DEBUG_TEST_CRASHES
    0
    CACHE BOOL "(Windows only) Enables debugging of tests in the test suite by showing the crash dialog when lldb crashes")

  set(LLDB_TEST_HIDE_CONSOLE_WINDOWS
    1
    CACHE BOOL "(Windows only) Hides the console window for an inferior when it is launched through the test suite")

  if (LLDB_TEST_DEBUG_TEST_CRASHES)
    set(LLDB_TEST_COMMON_ARGS ${LLDB_TEST_COMMON_ARGS} --enable-crash-dialog)
  endif()

  if (NOT LLDB_TEST_HIDE_CONSOLE_WINDOWS)
    set(LLDB_TEST_COMMON_ARGS ${LLDB_TEST_COMMON_ARGS} --show-inferior-console)
  endif()
endif()

if(LLDB_CODESIGN_IDENTITY)
  list(APPEND LLDB_TEST_COMMON_ARGS --codesign-identity "${LLDB_CODESIGN_IDENTITY}")
endif()

if(LLDB_BUILD_FRAMEWORK)
  list(APPEND LLDB_TEST_COMMON_ARGS --framework $<TARGET_FILE_DIR:liblldb>)
endif()

if (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Windows|Darwin")
  list(APPEND LLDB_TEST_COMMON_ARGS
    --env ARCHIVER=${CMAKE_AR} --env OBJCOPY=${CMAKE_OBJCOPY})
endif()

if(CMAKE_HOST_APPLE)
  if(LLDB_CODESIGN_IDENTITY)
    set(DEBUGSERVER_PATH $<TARGET_FILE:debugserver>)
  else()
    execute_process(
      COMMAND xcode-select -p
      OUTPUT_VARIABLE XCODE_DEV_DIR)
    string(STRIP ${XCODE_DEV_DIR} XCODE_DEV_DIR)
    set(DEBUGSERVER_PATH
      "${XCODE_DEV_DIR}/../SharedFrameworks/LLDB.framework/Resources/debugserver")
  endif()
  message(STATUS "Path to the lldb debugserver: ${DEBUGSERVER_PATH}")
  list(APPEND LLDB_TEST_COMMON_ARGS --server ${DEBUGSERVER_PATH})
endif()

set(LLDB_DOTEST_ARGS ${LLDB_TEST_COMMON_ARGS};${LLDB_TEST_USER_ARGS})

add_python_test_target(check-lldb-single
  ${LLDB_SOURCE_DIR}/test/dotest.py
  "--no-multiprocess;${LLDB_DOTEST_ARGS}"
  "Testing LLDB with args: ${LLDB_DOTEST_ARGS}"
  )

# If tests crash cause LLDB to crash, or things are otherwise unstable, or if machine-parsable
# output is desired (i.e. in continuous integration contexts) check-lldb-single is a better target.
add_python_test_target(check-lldb
  ${LLDB_SOURCE_DIR}/test/dotest.py
  "-q;${LLDB_DOTEST_ARGS}"
  "Testing LLDB (parallel execution, with a separate subprocess per test)"
  )

# If we're building with an in-tree clang, then list clang as a dependency
# to run tests.
if (TARGET clang)
  add_dependencies(check-lldb clang)
endif()

# LLD is required to link test executables on Windows.
if (CMAKE_SYSTEM_NAME MATCHES "Windows")
  if (TARGET lld)
    add_dependencies(check-lldb lld)
  else ()
    message(WARNING "lld required to test LLDB on Windows")
  endif ()
endif ()

add_custom_target(lldb-test-depends DEPENDS ${LLDB_TEST_DEPENDS})
# This will add LLDB's test dependencies to the depenednecies for check-all and
# include them in the test-depends target.
set_property(GLOBAL APPEND PROPERTY LLVM_LIT_DEPENDS ${ARG_DEPENDS})
