cmake_minimum_required(VERSION 3.20)

# Build with Rulebook (RLC) and disable CMake's default C and C++ build rules.
project(KuhnPoker LANGUAGES NONE)

if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux" OR
   NOT CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    message(FATAL_ERROR
        "This project requires native Linux x86_64. "
        "Detected ${CMAKE_SYSTEM_NAME}/${CMAKE_SYSTEM_PROCESSOR}. "
        "Configure and build it on the Linux server with environment.yml.")
endif()

# Enable CTest.
enable_testing()

# ==========================================================
# 1. Find build and test tools in the active Conda environment.
# Use RLC and LLVM 22 from the rulebook environment.
# ==========================================================
# Conda sets CONDA_PREFIX to the path of the active environment.
if(NOT DEFINED ENV{CONDA_PREFIX})
    message(FATAL_ERROR "Activate the environment first with conda activate rulebook_linux")
endif()

set(CONDA_BIN "$ENV{CONDA_PREFIX}/bin")
set(CONDA_LIB "$ENV{CONDA_PREFIX}/lib")

# Require the RLC compiler.
find_program(RLC NAMES rlc
    PATHS "${CONDA_BIN}" NO_DEFAULT_PATH REQUIRED)

# Require the RLC unit test tool.
find_program(RLC_TEST NAMES rlc-test
    PATHS "${CONDA_BIN}" NO_DEFAULT_PATH REQUIRED)

# Locate the Rulebook trace replay tool in the active Conda environment.
find_program(RLC_ACTION NAMES rlc-action
    PATHS "${CONDA_BIN}" NO_DEFAULT_PATH REQUIRED)

# Python tools otherwise infer this from --rlc, which points to our build-local
# launcher rather than the installed compiler. Trace replay needs learn.rl.
get_filename_component(RLC_REAL "${RLC}" REALPATH)
get_filename_component(RLC_BIN_DIR "${RLC_REAL}" DIRECTORY)
get_filename_component(RLC_PREFIX "${RLC_BIN_DIR}" DIRECTORY)
find_path(RLC_STDLIB NAMES learn.rl
    PATHS "${RLC_PREFIX}/lib/rlc/stdlib" "${CONDA_LIB}/rlc/stdlib"
    NO_DEFAULT_PATH REQUIRED)

# Prefer Conda's target driver, which selects its Linux sysroot and linker.
find_program(CLANG NAMES x86_64-conda-linux-gnu-clang clang-22 clang
    PATHS "${CONDA_BIN}" NO_DEFAULT_PATH REQUIRED)

# opt processes LLVM IR.
find_program(LLVM_OPT NAMES opt-22 opt
    PATHS "${CONDA_BIN}" NO_DEFAULT_PATH REQUIRED)

# llvm-symbolizer turns ASan error addresses into readable symbols.
find_program(SYMBOLIZER NAMES llvm-symbolizer-22 llvm-symbolizer
    PATHS "${CONDA_BIN}" NO_DEFAULT_PATH REQUIRED)

# RLC's adapter is separate from LLVM's libFuzzer/ASan runtime.
find_file(FUZZ_ADAPTER NAMES libfuzzer.a
    PATHS "${CONDA_LIB}" NO_DEFAULT_PATH REQUIRED)

# Check the external LLVM tools, not RLC's separately bundled LLVM backend.
function(check_llvm_version tool)
    execute_process(COMMAND "${${tool}}" --version
        OUTPUT_VARIABLE version_output ERROR_VARIABLE version_error
        RESULT_VARIABLE version_result)
    if(NOT "${version_result}" STREQUAL "0")
        message(FATAL_ERROR "Cannot run ${tool}: ${${tool}}\n${version_error}")
    endif()
    string(REGEX MATCH "[Vv]ersion[ \t]+([0-9]+\\.[0-9]+\\.[0-9]+)"
        version_match "${version_output}")
    if(NOT CMAKE_MATCH_1 STREQUAL "22.1.8")
        message(FATAL_ERROR
            "${tool} must be LLVM 22.1.8, as pinned in environment.yml.\n${version_output}")
    endif()
    message(STATUS "${tool}: ${${tool}} (LLVM ${CMAKE_MATCH_1})")
endfunction()
check_llvm_version(CLANG)
check_llvm_version(LLVM_OPT)
check_llvm_version(SYMBOLIZER)

execute_process(COMMAND "${CLANG}" -dumpmachine
    OUTPUT_VARIABLE CLANG_TARGET OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_VARIABLE target_error RESULT_VARIABLE target_result)
if(NOT "${target_result}" STREQUAL "0" OR
   NOT CLANG_TARGET MATCHES "^x86_64-.*linux")
    message(FATAL_ERROR "Expected a Linux x86_64 Clang target; got '${CLANG_TARGET}'.\n${target_error}")
endif()
message(STATUS "Clang target: ${CLANG_TARGET}")
message(STATUS "RLC: ${RLC}")
message(STATUS "RLC test runner: ${RLC_TEST}")
message(STATUS "RLC trace runner: ${RLC_ACTION}")
message(STATUS "RLC standard library: ${RLC_STDLIB}")
message(STATUS "RLC fuzzer adapter: ${FUZZ_ADAPTER}")

# ==========================================================
# 2. Keep all generated files inside the selected build directory.
# ==========================================================
# CMake provides the following directory variables.
# CMAKE_CURRENT_SOURCE_DIR points to the project source directory.
# CMAKE_CURRENT_BINARY_DIR points to the selected build directory.
set(SRC "${CMAKE_CURRENT_SOURCE_DIR}")
set(BUILD "${CMAKE_CURRENT_BINARY_DIR}")

set(RESULT_DIR "${BUILD}/result")
set(TEMP_DIR "${BUILD}/temp")
set(FUZZ_TEMP "${TEMP_DIR}/fuzzer")
set(TEST_TEMP "${TEMP_DIR}/test")
set(ASAN_TEMP "${TEMP_DIR}/Sanitizer")

# Create these directories during configuration.
# libFuzzer stores test inputs in corpus and failure cases in artifacts.
file(MAKE_DIRECTORY
    "${RESULT_DIR}"
    "${FUZZ_TEMP}/corpus"
    "${FUZZ_TEMP}/artifacts"
    "${TEST_TEMP}"
    "${ASAN_TEMP}/corpus"
    "${ASAN_TEMP}/artifacts"
)

# Project source files.
set(GAME_SOURCE "${SRC}/kuhn_poker.rl")
set(GAME_MAIN_SOURCE "${SRC}/kuhn_poker_main.rl")
set(TEST_SOURCE "${SRC}/kuhn_poker_test.rl")
set(FUZZ_SOURCE "${SRC}/kuhn_poker_fuzz.rl")
set(HELPER "${SRC}/cmake/kuhn_poker_game.cmake")

# Final executables belong to this build, never to a copied source tree.
set(GAME_BIN "${RESULT_DIR}/kuhn_poker")
set(FUZZ_BIN "${RESULT_DIR}/kuhn_poker_fuzzer")

# Python CLI tools also compile/link through RLC. A launcher selects the same
# Clang for them without the CLI's space-splitting --extra-rlc-args option.
string(REPLACE "'" "'\\''" RLC_SHELL "${RLC}")
string(REPLACE "'" "'\\''" CLANG_SHELL "${CLANG}")
set(RLC_DRIVER "${BUILD}/rlc-with-clang")
file(WRITE "${RLC_DRIVER}"
    "#!/bin/sh\nexec '${RLC_SHELL}' '--clang=${CLANG_SHELL}' \"$@\"\n")
file(CHMOD "${RLC_DRIVER}" PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

# ==========================================================
# 3. Build the game executable into the build directory's result/ folder.
# ==========================================================
# Use the selected Clang when RLC links the native Linux executable.
# Define how to generate a specific output file.
# OUTPUT is the executable in the build directory's result/ folder.
# COMMAND is the command to execute.
# DEPENDS lists the inputs so Ninja rebuilds the executable when source files change.
# Ninja can usually reuse an existing output when its inputs are unchanged.
# This is an incremental build.
# WORKING_DIRECTORY is the directory in which the command runs.
# COMMENT is the description displayed during the build.
# VERBATIM is CMake's recommended way to handle escaping and quoting in custom commands.
add_custom_command(
    OUTPUT "${GAME_BIN}"
    COMMAND "${RLC}" "${GAME_MAIN_SOURCE}" "--clang=${CLANG}" -g -o "${GAME_BIN}"
    DEPENDS "${GAME_MAIN_SOURCE}" "${GAME_SOURCE}" "${RLC}" "${CLANG}"
    WORKING_DIRECTORY "${SRC}"
    COMMENT "Building the Kuhn Poker game executable"
    VERBATIM
)

# Give the build task a target name so it can be run by name.
# 
add_custom_target(game_build ALL DEPENDS "${GAME_BIN}")

# ==========================================================
# 4. Build the fuzzer using RLC's built-in --fuzzer mode and Conda LLVM 22.
# ==========================================================
# Store the LLVM IR generated by RLC as temp/fuzzer/kuhn.raw.ll.
set(RAW_IR "${FUZZ_TEMP}/kuhn.raw.ll")
# Store the ASan-instrumented LLVM IR as temp/Sanitizer/kuhn.asan.ll.
set(ASAN_IR "${ASAN_TEMP}/kuhn.asan.ll")
# kuhn.asan.o is the object file containing compiled machine code.
set(ASAN_OBJ "${ASAN_TEMP}/kuhn.asan.o")

add_custom_command(
    OUTPUT "${FUZZ_BIN}"

    # Step 1. Generate LLVM IR using RLC's built-in --fuzzer flag.
    # The helper captures RLC's standard output with OUTPUT_FILE in place of shell redirection.
    COMMAND "${CMAKE_COMMAND}"
        "-DMODE=ir"
        "-DRLC=${RLC}"
        "-DSOURCE=${FUZZ_SOURCE}"
        "-DINCLUDE_DIR=${SRC}"
        "-DOUTPUT=${RAW_IR}"
        -P "${HELPER}"

    # Step 2. Add AddressSanitizer instrumentation.
    COMMAND "${LLVM_OPT}"
        -S -passes=forceattrs,asan
        -force-attribute=sanitize_address
        "${RAW_IR}" -o "${ASAN_IR}"

    # Step 3. Compile the LLVM IR into a .o object file.
    COMMAND "${CLANG}"
        -g -O0 -fno-omit-frame-pointer
        -c "${ASAN_IR}" -o "${ASAN_OBJ}"

    # Step 4. Link the LLVM libFuzzer runtime and RLC adapter library.
    COMMAND "${CLANG}"
        -g
        -fsanitize=fuzzer,address
        "${ASAN_OBJ}" "${FUZZ_ADAPTER}"
        -o "${FUZZ_BIN}"

    # DEPENDS triggers a fuzzer rebuild when the game, fuzzer entry point, or helper changes.
    # BYPRODUCTS lists the three intermediate files generated alongside kuhn_poker_fuzzer.
    DEPENDS "${GAME_SOURCE}" "${FUZZ_SOURCE}" "${HELPER}"
        "${RLC}" "${LLVM_OPT}" "${CLANG}" "${FUZZ_ADAPTER}"
    BYPRODUCTS "${RAW_IR}" "${ASAN_IR}" "${ASAN_OBJ}"
    WORKING_DIRECTORY "${SRC}"
    COMMENT "Building the RLC fuzzer with ASan instrumentation"
    VERBATIM
)
add_custom_target(fuzzer_build ALL DEPENDS "${FUZZ_BIN}")

# ==========================================================
# 5. Register the game, unit, short ASan, and fuzz tests with CTest.
# ==========================================================
# Test 1. The game's main() returns 0 on success.
add_test(NAME game_test COMMAND "${GAME_BIN}")

# Test 2. Run rlc-test and write the output to temp/test/unit_test.log.
add_test(NAME unit_test
    COMMAND "${CMAKE_COMMAND}"
        "-DMODE=unit"
        "-DRLC_TEST=${RLC_TEST}"
        "-DRLC=${RLC_DRIVER}"
        "-DRLC_STDLIB=${RLC_STDLIB}"
        "-DSOURCE=${TEST_SOURCE}"
        "-DSOURCE_DIR=${SRC}"
        "-DOUTPUT=${TEST_TEMP}/unit_test.log"
        -P "${HELPER}"
)

# Test 3. Run the ASan-instrumented fuzzer for 100 iterations.
add_test(NAME sanitizer_test
    COMMAND "${FUZZ_BIN}"
        -runs=100 -seed=418 -max_len=256 -timeout=5
        "-artifact_prefix=${ASAN_TEMP}/artifacts/"
        "${ASAN_TEMP}/corpus"
)

# Test 4. Run 10,000 coverage-guided fuzzing iterations with a separate corpus.
add_test(NAME fuzzer_test
    COMMAND "${FUZZ_BIN}"
        -runs=10000 -seed=418 -max_len=256 -timeout=5
        -print_final_stats=1
        "-artifact_prefix=${FUZZ_TEMP}/artifacts/"
        "${FUZZ_TEMP}/corpus"
)

set_tests_properties(sanitizer_test fuzzer_test PROPERTIES
    ENVIRONMENT "ASAN_SYMBOLIZER_PATH=${SYMBOLIZER}"
    TIMEOUT 180
    RUN_SERIAL TRUE
)

# Test 5: replay a fixed trace and check the expected final game state.
add_test(NAME trace_test
    COMMAND "${CMAKE_COMMAND}"
        "-DMODE=trace"
        "-DRLC_ACTION=${RLC_ACTION}"
        "-DRLC=${RLC_DRIVER}"
        "-DRLC_STDLIB=${RLC_STDLIB}"
        "-DSOURCE=${GAME_SOURCE}"
        "-DSOURCE_DIR=${SRC}"
        "-DTRACE_FILE=${SRC}/traces/game1.txt"
        "-DEXPECTED_FILE=${SRC}/traces/game1.expected.txt"
        "-DOUTPUT=${TEST_TEMP}/trace_test.log"
        -P "${HELPER}"
)

set_tests_properties(trace_test PROPERTIES TIMEOUT 90)

# ==========================================================
# 6. Build and run all tests with one command.
# ==========================================================
# CMAKE_CTEST_COMMAND points to the CTest executable.
# --output-on-failure shows the output of failed tests.
# --output-log saves CTest output to a log file.
# WORKING_DIRECTORY specifies the directory in which CTest runs.
# USES_TERMINAL gives the build command access to the terminal to display test progress.
add_custom_target(verify_all
    COMMAND "${CMAKE_CTEST_COMMAND}"
        --output-on-failure
        --output-log "${TEST_TEMP}/ctest.log"
    WORKING_DIRECTORY "${BUILD}"
    COMMENT "Running all Kuhn Poker tests"
    USES_TERMINAL
    VERBATIM
)
# verify_all runs the tests after game_build and fuzzer_build finish.
add_dependencies(verify_all game_build fuzzer_build)

message(STATUS "Kuhn Poker configuration complete")
message(STATUS "Final executables: ${RESULT_DIR}")
message(STATUS "Intermediate build and test files: ${TEMP_DIR}")
