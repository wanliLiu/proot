include(CheckCSourceCompiles)
include(CheckLinkerFlag)
include(CheckSymbolExists)
find_package(Git QUIET)

option(PROOT_FETCH_DEPENDENCIES "Download required third-party libraries from the network" ON)

set(TALLOC_TARGET "")
set(LIBARCHIVE_TARGET "")
set(TALLOC_INCLUDE_DIRS "")
set(LIBARCHIVE_INCLUDE_DIRS "")
set(TALLOC_CFLAGS_OTHER "")
set(TALLOC_LDFLAGS_OTHER "")
set(LIBARCHIVE_CFLAGS_OTHER "")
set(LIBARCHIVE_LDFLAGS_OTHER "")
set(PROOT_DEP_TARGETS "")

if(PROOT_FETCH_DEPENDENCIES)
    include(ExternalProject)
    set(PROOT_DEPS_PREFIX ${CMAKE_BINARY_DIR}/third_party)
    set(PROOT_DEPS_INSTALL ${PROOT_DEPS_PREFIX}/install)
    file(MAKE_DIRECTORY ${PROOT_DEPS_INSTALL})
    file(MAKE_DIRECTORY ${PROOT_DEPS_INSTALL}/include)
    file(MAKE_DIRECTORY ${PROOT_DEPS_INSTALL}/lib)

    set(TALLOC_URL https://download.samba.org/pub/talloc/talloc-2.4.3.tar.gz)
    set(LIBARCHIVE_URL https://www.libarchive.org/downloads/libarchive-3.8.0.tar.gz)

    ExternalProject_Add(talloc_project
        URL ${TALLOC_URL}
        URL_HASH SHA256=dc46c40b9f46bb34dd97fe41f548b0e8b247b77a918576733c528e83abd854dd
        PREFIX ${PROOT_DEPS_PREFIX}/talloc
        CONFIGURE_COMMAND ./configure --prefix=${PROOT_DEPS_INSTALL}
        BUILD_COMMAND make
        INSTALL_COMMAND make install
        BUILD_IN_SOURCE ON
        DOWNLOAD_NO_PROGRESS ON
        BUILD_BYPRODUCTS ${PROOT_DEPS_INSTALL}/lib/libtalloc.so
    )

    set(TALLOC_INCLUDE_DIRS ${PROOT_DEPS_INSTALL}/include)
    set(TALLOC_LIBRARY ${PROOT_DEPS_INSTALL}/lib/libtalloc.so)
    add_library(talloc_external SHARED IMPORTED)
    set_target_properties(talloc_external PROPERTIES
        IMPORTED_LOCATION ${TALLOC_LIBRARY}
        INTERFACE_INCLUDE_DIRECTORIES ${TALLOC_INCLUDE_DIRS}
    )
    add_dependencies(talloc_external talloc_project)
    add_library(talloc::talloc ALIAS talloc_external)
    set(TALLOC_TARGET talloc::talloc)
    list(APPEND PROOT_DEP_TARGETS talloc_external)

    ExternalProject_Add(libarchive_project
        URL ${LIBARCHIVE_URL}
        URL_HASH SHA256=191b5b24811499d5c2e5efa3248975fa6daa5e6a227700cc7b8e54d6d7c06eef
        PREFIX ${PROOT_DEPS_PREFIX}/libarchive
        CONFIGURE_COMMAND ./configure --prefix=${PROOT_DEPS_INSTALL}
        BUILD_COMMAND make
        INSTALL_COMMAND make install
        BUILD_IN_SOURCE ON
        DOWNLOAD_NO_PROGRESS ON
        BUILD_BYPRODUCTS ${PROOT_DEPS_INSTALL}/lib/libarchive.so
    )

    set(LIBARCHIVE_INCLUDE_DIRS ${PROOT_DEPS_INSTALL}/include)
    set(LIBARCHIVE_LIBRARY ${PROOT_DEPS_INSTALL}/lib/libarchive.so)
    add_library(libarchive_external SHARED IMPORTED)
    set_target_properties(libarchive_external PROPERTIES
        IMPORTED_LOCATION ${LIBARCHIVE_LIBRARY}
        INTERFACE_INCLUDE_DIRECTORIES ${LIBARCHIVE_INCLUDE_DIRS}
    )
    add_dependencies(libarchive_external libarchive_project)
    add_library(libarchive::archive ALIAS libarchive_external)
    set(LIBARCHIVE_TARGET libarchive::archive)
    list(APPEND PROOT_DEP_TARGETS libarchive_external)
    set(PROOT_FETCHED_LIB_DIR ${PROOT_DEPS_INSTALL}/lib)
else()
    include(FindPkgConfig)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(TALLOC REQUIRED IMPORTED_TARGET talloc)
    pkg_check_modules(LIBARCHIVE REQUIRED IMPORTED_TARGET libarchive)
    set(TALLOC_TARGET PkgConfig::TALLOC)
    set(LIBARCHIVE_TARGET PkgConfig::LIBARCHIVE)
    set(TALLOC_INCLUDE_DIRS ${TALLOC_INCLUDE_DIRS})
    set(LIBARCHIVE_INCLUDE_DIRS ${LIBARCHIVE_INCLUDE_DIRS})
    set(TALLOC_CFLAGS_OTHER ${TALLOC_CFLAGS_OTHER})
    set(TALLOC_LDFLAGS_OTHER ${TALLOC_LDFLAGS_OTHER})
    set(LIBARCHIVE_CFLAGS_OTHER ${LIBARCHIVE_CFLAGS_OTHER})
    set(LIBARCHIVE_LDFLAGS_OTHER ${LIBARCHIVE_LDFLAGS_OTHER})
    set(PROOT_FETCHED_LIB_DIR "")
endif()

if(PROOT_FETCH_DEPENDENCIES)
    add_custom_target(proot_third_party
        COMMAND ${CMAKE_COMMAND} -E echo "Preparing external dependencies (talloc, libarchive)..."
        DEPENDS ${PROOT_DEP_TARGETS}
        USES_TERMINAL
        VERBATIM
    )
else()
    add_custom_target(proot_third_party)
endif()

# ------------------------------------------------------------
# Detect optional kernel features and project version metadata
# ------------------------------------------------------------

set(CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
check_symbol_exists(process_vm_readv "sys/uio.h" HAVE_PROCESS_VM)

check_c_source_compiles([[
    #include <sys/prctl.h>
    #include <linux/seccomp.h>
    #include <linux/filter.h>
    #include <linux/audit.h>
    #include <stddef.h>
    int main(void) {
        const size_t arch_offset = offsetof(struct seccomp_data, arch);
        const size_t syscall_offset = offsetof(struct seccomp_data, nr);
        struct sock_filter filter[] = {
            BPF_STMT(BPF_LD + BPF_W + BPF_ABS, arch_offset),
            BPF_JUMP(BPF_JMP + BPF_JEQ + BPF_K, AUDIT_ARCH_X86_64, 0, 1),
            BPF_STMT(BPF_LD + BPF_W + BPF_ABS, syscall_offset),
            BPF_JUMP(BPF_JMP + BPF_JEQ + BPF_K, 0, 0, 1),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_TRACE)
        };
        struct sock_fprog program = { .filter = filter, .len = sizeof(filter) / sizeof(filter[0]) };
        (void)prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
        (void)prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &program);
        return 0;
    }
]] HAVE_SECCOMP_FILTER)

set(VERSION_STRING "${PROJECT_VERSION}")
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-list --tags --max-count=1
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE GIT_TAG_RESULT
    )
    if(GIT_TAG_RESULT EQUAL 0 AND GIT_TAG)
        execute_process(
            COMMAND ${GIT_EXECUTABLE} describe --tags ${GIT_TAG}
            WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
            OUTPUT_VARIABLE GIT_VERSION
            OUTPUT_STRIP_TRAILING_WHITESPACE
            RESULT_VARIABLE GIT_VERSION_RESULT
        )
    endif()

    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-list --all --max-count=1
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_COMMIT
        OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE GIT_COMMIT_RESULT
    )
    if(GIT_COMMIT_RESULT EQUAL 0 AND GIT_COMMIT)
        string(SUBSTRING "${GIT_COMMIT}" 0 8 GIT_COMMIT_SHORT)
    endif()

    if(GIT_COMMIT_SHORT)
        if(GIT_VERSION_RESULT EQUAL 0 AND GIT_VERSION)
            set(VERSION_STRING "${GIT_VERSION}-${GIT_COMMIT_SHORT}")
        else()
            set(VERSION_STRING "-${GIT_COMMIT_SHORT}")
        endif()
    endif()
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/build.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/build.h
               @ONLY)

# ------------------------------------------------------------
# Helpers to extract architecture-specific defines
# ------------------------------------------------------------

function(get_arch_define output_var define_name)
    execute_process(
        COMMAND ${CMAKE_C_COMPILER} ${ARGN} -E -dM -DNO_LIBC_HEADER ${PROJECT_SOURCE_DIR}/src/arch.h
        OUTPUT_VARIABLE compiler_defines
        OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE compiler_result
    )
    if(NOT compiler_result EQUAL 0)
        set(${output_var} "" PARENT_SCOPE)
        return()
    endif()
    string(REGEX MATCH "#define[ \t]+${define_name}[ \t]+([^ \n]+)" define_line "${compiler_defines}")
    if(define_line)
        string(REGEX REPLACE "#define[ \t]+${define_name}[ \t]+" "" define_value "${define_line}")
        string(STRIP "${define_value}" define_value)
        set(${output_var} "${define_value}" PARENT_SCOPE)
    else()
        set(${output_var} "" PARENT_SCOPE)
    endif()
endfunction()

get_arch_define(LOADER_ADDRESS_DEFINE LOADER_ADDRESS)
get_arch_define(HAS_LOADER_32BIT_DEFINE HAS_LOADER_32BIT)
get_arch_define(LOADER_ARCH_CFLAGS_DEFINE LOADER_ARCH_CFLAGS)
get_arch_define(LOADER_ADDRESS_M32_DEFINE LOADER_ADDRESS -m32)
get_arch_define(LOADER_ARCH_CFLAGS_M32_DEFINE LOADER_ARCH_CFLAGS -m32)

set(DEFAULT_BUILD_LOADER_M32 OFF)
if(HAS_LOADER_32BIT_DEFINE)
    set(DEFAULT_BUILD_LOADER_M32 ON)
endif()

option(BUILD_LOADER_M32 "Build 32-bit loader helper" ${DEFAULT_BUILD_LOADER_M32})

check_linker_flag(C "-Wl,--build-id=none" LINKER_SUPPORTS_NO_BUILD_ID)

set(LINKER_TOOL ${CMAKE_LINKER})
if(NOT LINKER_TOOL)
    find_program(LINKER_TOOL ld)
endif()
if(NOT LINKER_TOOL)
    message(FATAL_ERROR "Unable to locate linker executable for binary embedding")
endif()

# ------------------------------------------------------------
# Helper to embed textual assets as binary objects
# ------------------------------------------------------------

function(embed_file output_var output_basename input_path)
    set(output_object "${CMAKE_CURRENT_BINARY_DIR}/${output_basename}.o")
    add_custom_command(
        OUTPUT ${output_object}
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${input_path} ${CMAKE_CURRENT_BINARY_DIR}/${output_basename}
        COMMAND ${LINKER_TOOL} -r -b binary -o ${output_object} ${output_basename}
        BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/${output_basename}
        DEPENDS ${input_path}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Embedding resource ${output_basename}"
    )
    set_source_files_properties(${output_object} PROPERTIES GENERATED TRUE EXTERNAL_OBJECT TRUE)
    set(${output_var} ${output_object} PARENT_SCOPE)
endfunction()

embed_file(LICENSE_OBJECT licenses ${PROJECT_SOURCE_DIR}/COPYING)
embed_file(CARE_MANUAL_OBJECT manual ${PROJECT_SOURCE_DIR}/doc/care/manual.rst)

# ------------------------------------------------------------
# Build the loader helpers
# ------------------------------------------------------------

set(LOADER_COMPILE_FLAGS -fPIC -ffreestanding)
if(LOADER_ARCH_CFLAGS_DEFINE)
    separate_arguments(LOADER_ARCH_CFLAGS_LIST UNIX_COMMAND "${LOADER_ARCH_CFLAGS_DEFINE}")
    list(APPEND LOADER_COMPILE_FLAGS ${LOADER_ARCH_CFLAGS_LIST})
endif()

add_executable(proot_loader
    loader/loader.c
    loader/assembly.S
)
target_include_directories(proot_loader PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_definitions(proot_loader PRIVATE _GNU_SOURCE _FILE_OFFSET_BITS=64)
target_compile_options(proot_loader PRIVATE ${LOADER_COMPILE_FLAGS})

set(LOADER_LINK_FLAGS -static -nostdlib "-Wl,-z,noexecstack")
if(LOADER_ADDRESS_DEFINE)
    list(APPEND LOADER_LINK_FLAGS "-Wl,-Ttext=${LOADER_ADDRESS_DEFINE}")
endif()
target_link_options(proot_loader PRIVATE ${LOADER_LINK_FLAGS})
if(LINKER_SUPPORTS_NO_BUILD_ID)
    target_link_options(proot_loader PRIVATE -Wl,--build-id=none)
endif()
set_target_properties(proot_loader PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})

set(LOADER_ELF "${CMAKE_CURRENT_BINARY_DIR}/loader.elf")
add_custom_command(
    OUTPUT ${LOADER_ELF}
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:proot_loader> ${LOADER_ELF}
    DEPENDS proot_loader
    COMMENT "Preparing 64-bit loader image"
)

set(LOADER_WRAPPED_OBJECT "${CMAKE_CURRENT_BINARY_DIR}/loader-wrapped.o")
add_custom_command(
    OUTPUT ${LOADER_WRAPPED_OBJECT}
    COMMAND ${LINKER_TOOL} -r -b binary -o ${LOADER_WRAPPED_OBJECT} loader.elf
    DEPENDS ${LOADER_ELF}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Embedding loader.elf"
)
set_source_files_properties(${LOADER_WRAPPED_OBJECT} PROPERTIES GENERATED TRUE EXTERNAL_OBJECT TRUE)

set(LOADER_M32_WRAPPED_OBJECT "")
if(BUILD_LOADER_M32 AND LOADER_ADDRESS_M32_DEFINE)
    set(LOADER_M32_COMPILE_FLAGS -fPIC -ffreestanding -m32)
    if(LOADER_ARCH_CFLAGS_M32_DEFINE)
        separate_arguments(LOADER_ARCH_CFLAGS_M32_LIST UNIX_COMMAND "${LOADER_ARCH_CFLAGS_M32_DEFINE}")
        list(APPEND LOADER_M32_COMPILE_FLAGS ${LOADER_ARCH_CFLAGS_M32_LIST})
    endif()

    add_executable(proot_loader_m32
        loader/loader.c
        loader/assembly.S
    )
    target_include_directories(proot_loader_m32 PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
    target_compile_definitions(proot_loader_m32 PRIVATE _GNU_SOURCE _FILE_OFFSET_BITS=64)
    target_compile_options(proot_loader_m32 PRIVATE ${LOADER_M32_COMPILE_FLAGS})

    set(LOADER_M32_LINK_FLAGS -static -nostdlib -m32 "-Wl,-z,noexecstack")
    list(APPEND LOADER_M32_LINK_FLAGS "-Wl,-Ttext=${LOADER_ADDRESS_M32_DEFINE}")
    target_link_options(proot_loader_m32 PRIVATE ${LOADER_M32_LINK_FLAGS})
    if(LINKER_SUPPORTS_NO_BUILD_ID)
        target_link_options(proot_loader_m32 PRIVATE -Wl,--build-id=none)
    endif()
    set_target_properties(proot_loader_m32 PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})

    set(LOADER_M32_ELF "${CMAKE_CURRENT_BINARY_DIR}/loader-m32.elf")
    add_custom_command(
        OUTPUT ${LOADER_M32_ELF}
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:proot_loader_m32> ${LOADER_M32_ELF}
        DEPENDS proot_loader_m32
        COMMENT "Preparing 32-bit loader image"
    )

    set(LOADER_M32_WRAPPED_OBJECT "${CMAKE_CURRENT_BINARY_DIR}/loader-m32-wrapped.o")
    add_custom_command(
        OUTPUT ${LOADER_M32_WRAPPED_OBJECT}
        COMMAND ${LINKER_TOOL} -r -b binary -o ${LOADER_M32_WRAPPED_OBJECT} loader-m32.elf
        DEPENDS ${LOADER_M32_ELF}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Embedding loader-m32.elf"
    )
    set_source_files_properties(${LOADER_M32_WRAPPED_OBJECT} PROPERTIES GENERATED TRUE EXTERNAL_OBJECT TRUE)
endif()

# ------------------------------------------------------------
# Build shared objects and executables
# ------------------------------------------------------------

set(COMMON_INCLUDE_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
    ${PROJECT_SOURCE_DIR}/lib/uthash/include
)

set(COMMON_COMPILE_OPTIONS -Wall -Wextra -O2 -g)
set(COMMON_LINK_OPTIONS -Wl,-z,noexecstack)

set(PROOT_SHARED_SOURCES
    cli/cli.c
    cli/note.c
    cli/proot.c
    execve/aoxp.c
    execve/auxv.c
    execve/elf.c
    execve/enter.c
    execve/exit.c
    execve/ldso.c
    execve/shebang.c
    path/binding.c
    path/canon.c
    path/glue.c
    path/path.c
    path/proc.c
    path/temp.c
    ptrace/ptrace.c
    ptrace/user.c
    ptrace/wait.c
    syscall/chain.c
    syscall/enter.c
    syscall/exit.c
    syscall/heap.c
    syscall/rlimit.c
    syscall/seccomp.c
    syscall/socket.c
    syscall/syscall.c
    syscall/sysnum.c
    tracee/event.c
    tracee/mem.c
    tracee/reg.c
    tracee/tracee.c
    extension/extension.c
    extension/fake_id0/fake_id0.c
    extension/kompat/kompat.c
    extension/link2symlink/link2symlink.c
    extension/portmap/map.c
    extension/portmap/portmap.c
)

add_library(proot_core OBJECT ${PROOT_SHARED_SOURCES})
target_include_directories(proot_core PRIVATE ${COMMON_INCLUDE_DIRS} ${TALLOC_INCLUDE_DIRS})
target_compile_definitions(proot_core PRIVATE _GNU_SOURCE _FILE_OFFSET_BITS=64)
target_compile_options(proot_core PRIVATE ${COMMON_COMPILE_OPTIONS} ${TALLOC_CFLAGS_OTHER})
add_dependencies(proot_core proot_third_party)

add_executable(proot
    $<TARGET_OBJECTS:proot_core>
    ${LICENSE_OBJECT}
    ${LOADER_WRAPPED_OBJECT}
)
if(LOADER_M32_WRAPPED_OBJECT)
    target_sources(proot PRIVATE ${LOADER_M32_WRAPPED_OBJECT})
endif()
target_link_libraries(proot PRIVATE ${TALLOC_TARGET})
target_link_options(proot PRIVATE ${COMMON_LINK_OPTIONS} ${TALLOC_LDFLAGS_OTHER})
add_dependencies(proot proot_third_party)

set(CARE_SOURCES
    cli/care.c
    extension/care/archive.c
    extension/care/care.c
    extension/care/extract.c
    extension/care/final.c
)

add_executable(care
    $<TARGET_OBJECTS:proot_core>
    ${CARE_SOURCES}
    ${LICENSE_OBJECT}
    ${CARE_MANUAL_OBJECT}
    ${LOADER_WRAPPED_OBJECT}
)
if(LOADER_M32_WRAPPED_OBJECT)
    target_sources(care PRIVATE ${LOADER_M32_WRAPPED_OBJECT})
endif()
target_include_directories(care PRIVATE ${COMMON_INCLUDE_DIRS} ${TALLOC_INCLUDE_DIRS} ${LIBARCHIVE_INCLUDE_DIRS})
target_compile_definitions(care PRIVATE _GNU_SOURCE _FILE_OFFSET_BITS=64)
target_compile_options(care PRIVATE ${COMMON_COMPILE_OPTIONS} ${TALLOC_CFLAGS_OTHER} ${LIBARCHIVE_CFLAGS_OTHER})
target_link_libraries(care PRIVATE ${TALLOC_TARGET} ${LIBARCHIVE_TARGET})
target_link_options(care PRIVATE ${COMMON_LINK_OPTIONS} ${TALLOC_LDFLAGS_OTHER} ${LIBARCHIVE_LDFLAGS_OTHER})
add_dependencies(care proot_third_party)

# Ensure resources are available during IDE builds
set(EMBED_RESOURCE_DEPENDS ${LICENSE_OBJECT} ${CARE_MANUAL_OBJECT} ${LOADER_WRAPPED_OBJECT})
if(LOADER_M32_WRAPPED_OBJECT)
    list(APPEND EMBED_RESOURCE_DEPENDS ${LOADER_M32_WRAPPED_OBJECT})
endif()
add_custom_target(embed_resources ALL DEPENDS ${EMBED_RESOURCE_DEPENDS})

if(PROOT_FETCH_DEPENDENCIES)
    set_target_properties(proot PROPERTIES BUILD_RPATH ${PROOT_FETCHED_LIB_DIR})
    set_target_properties(care PROPERTIES BUILD_RPATH ${PROOT_FETCHED_LIB_DIR})
endif()
